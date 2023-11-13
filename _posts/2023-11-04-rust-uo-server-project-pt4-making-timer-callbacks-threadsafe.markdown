---
layout: post
title:  "Rust UO server project pt4: Making Timer Callbacks Threadsafe"
date:   2023-11-04 18:00:00 +0000
tags: ["UO server project", "Rust"]
---

As mentioned at the end of my [last journal post]({% post_url 2023-10-31-rust-uo-server-project-pt3-adding-timer-callbacks %}), attempting to send a `Timer` struct between threads after a `callback` property had been added it to it did not compile. This post explores the compiler errors seen when passing `Timer` structs between threads after adding callbacks to them, and how to fix them. In my current implementation, `Timer` structs need to be sent between threads using both an mpsc channel and a shared vec wrapped in a `Mutex`. Channels are used when sending timers to the registration thread, and from the prioritisation thread to the execution thread. The shared vec `Mutex` is used when timers are being transferred from the registration thread to the prioritisation thread.

### Using function pointers for the callback

As covered in my previous journal post, it is not possible to capture game state from the environment when using a function pointer. So this approach needed the state to be stored on the `Timer` and then passed as arguments to the callback. To allow `Timer` structs to operate with different types of state, I used a trait bound:

```rust
trait State {}

struct Timer<T: State> {
    state: T,
    update: fn(&mut T),
}
```

And then implemented the trait on the state types (this is a simplified version of the code in the previous post):

```rust
struct Character { hitpoints: u32 }
impl State for Character {}
impl Character {
    fn update(&mut self) {
        self.hitpoints += 10;
    }
}

struct Monster { anger: u32 }
impl State for Monster {}
impl Monster {
    fn update(&mut self) {
        self.anger += 10;
    }
}
```

But passing a `Timer` via a channel now doesn't work if you allow the compiler to try and infer the type for the channel sender:

```rust
    let character = Character { hitpoints: 100 };
    let monster = Monster { anger: 0 };
    
    let character_timer = Timer {
        state: character,
        update: Character::update,
    };

    let monster_timer = Timer {
        state: monster,
        update: Monster::update,
    };

    let (tx, rx) = mpsc::channel();

    tx.send(monster_timer).unwrap(); // <-- The type of 'tx' is inferred here
                                     //     to be 'mpsc::Sender<Timer<Monster>>'.

    tx.send(character_timer).unwrap();  // error[E0308]: mismatched types
    // ---- ^^^^^^^^^^^^^^^ expected `Timer<Monster>`, found `Timer<Character>`
```

With the types and the trait bound as they are there is no way around this. With trait bounds Rust performs monomorphisation of the code and then does static dispatch.

During monomorphisation, Rust replaces the generic definitions with the concrete ones needed. So the `Timer<T: State>` definition is replaced with the following for the example above:

```rust
struct Timer_Character {
    state: Character,
    update: fn(&mut Character),
}

struct Timer_Monster {
    state: Monster,
    update: fn(&mut Monster),
}
```

On the first call to the `send()` function the type of the mpsc `Sender` is then inferred as `mpsc::Sender<Timer_Monster>`.

I asked [a question](https://users.rust-lang.org/t/passing-trait-bounded-structs-to-mpsc-channels/102455/2) on the Rust discourse and the reply lead me to change the code to use a callback method on `Timer` instead of storing the callback on `Timer` as a function pointer:

### Using a method on `Timer` for the callback

First, I changed `callback` to a method on `Timer` and defined that behaviour with another trait:

```rust
trait Callback { fn callback(&mut self); }

struct Timer<T: State> {
    state: T,
}

impl<T> Callback for Timer<T> where T: State {
    fn callback(&mut self) {
        self.state.update();
    }
}
```

The above gives a 'error[E0599]: no method named `update` found for type parameter `T` in the current scope' error. To fix this, the `update()` behaviour needs to be moved into the `State` trait, and the implementations moved into the `impl` blocks that implement that trait for the `Monster` and `Character` types:

```rust
trait State { fn update(&mut self); }

struct Character { hitpoints: u32 }

impl State for Character {
    fn update(&mut self) {
        self.hitpoints += 10;
    }
}

struct Monster { anger: u32 }

impl State for Monster {
    fn update(&mut self) {
        self.anger += 10;
    }
}
```

We no longer need to set the `callback` property on the `Timer`:

```rust
    let character_timer = Timer { state: character };

    let monster_timer = Timer { state: monster };
```

Lastly, the timers need to be wrapped in a pointer and a type cast added on the first call to `send()`:

```rust
    tx.send(Box::new(monster_timer) as Box<dyn Callback>).unwrap();
    tx.send(Box::new(character_timer)).unwrap();
```

Now that a timer could be sent to a channel, I looked at receiving it and calling its callback:

```rust
fn main() {
    // snip

    for mut timer in rx.recv() {
        timer.callback();
    }
}
```

The last thing to try was receiving from the channel in another thread:

```rust
use std::thread;

// snip

fn main() {
    // snip

    thread::spawn(move || {
        for mut timer in rx.recv() {
            timer.callback();
        }
    }
    // error[E0277]: `dyn Callback` cannot be sent between threads safely
}
```

To fix this error, the type on the channel needed to include `Send`:

```rust
fn main() {
    // snip

    tx.send(Box::new(monster_timer) as Box<dyn Callback + Send>).unwrap();

    // snip
}
```

Here's the full code:

```rust
use std::thread;
use std::sync::mpsc;

trait State { fn update(&mut self); }

struct Character { hitpoints: u32 }

impl State for Character {
    fn update(&mut self) {
        self.hitpoints += 10;
    }
}

struct Monster { anger: u32 }

impl State for Monster {
    fn update(&mut self) {
        self.anger += 10;
    }
}

trait Callback { fn callback(&mut self); }

struct Timer<T: State> {
    state: T,
}

impl<T> Callback for Timer<T> where T: State {
    fn callback(&mut self) {
        self.state.update();
    }
}

fn main() {
    let character = Character { hitpoints: 100 };
    let monster = Monster { anger: 0 };

    let character_timer = Timer { state: character };
    let monster_timer = Timer { state: monster };

    let (tx, rx) = mpsc::channel();

    tx.send(Box::new(monster_timer) as Box<dyn Callback + Send>).unwrap();
    tx.send(Box::new(character_timer)).unwrap();

    thread::spawn(move || {
        for mut timer in rx.recv() {
            timer.callback();
        }
    });
}
```

The code above still lacks parameterised state deltas (the changes to the state are hardcoded in the `update()` methods on `Monster` and `Character`, to increase a property by 10) and `Timer` structs can only run once - there is no repetition logic. The full code including these features can be found on [this commit](https://github.com/thisdotrob/rust-uo-server/tree/25774cd117a58d67ab5113b994598f2f67c8ee8c) in the server repo.

### Using a `FnMut` trait object for the callback

Here's the starting `FnMut` code from my last journal post:

```rust
struct Character { hitpoints: u32 }

struct Monster { anger: u32 }

struct Timer<'a> {
    callback: Box<dyn FnMut() -> () + 'a>,
}

fn main() {
    let mut character = Character { hitpoints: 100 };
    let mut monster = Monster { anger: 0 };

    let mut character_timer = Timer {
        callback: Box::new(|| {
            character.hitpoints -= 1;
        }),
    };

    let mut monster_timer = Timer {
        callback: Box::new(|| {
            monster.anger += 10;
        }),
    };

    (character_timer.callback)();
    (monster_timer.callback)();
}
```

Now passing the timer through a channel to another thread before calling its callback:

```rust
// snip

fn main() {
    // snip

    let (tx, rx) = mpsc::channel();

    tx.send(character_timer).unwrap();
    tx.send(monster_timer).unwrap();

    thread::spawn(move || {
        for mut timer in rx.recv() {
            (timer.callback)();
        }
    });
    // error[E0277]: `dyn FnMut()` cannot be sent between threads safely
    //     = help: the trait `Send` is not implemented for `dyn FnMut()`
}
```

The error is suggesting that we need to specify the `Send` trait again:

```rust
// snip

struct Timer<'a> {
    callback: Box<dyn FnMut() -> () + Send + 'a>,
}

// snip
```

Now another error appears:

```rust
fn main() {
    let character_timer = Timer {
        callback: Box::new(|| {
            character.hitpoints -= 1;
            // error[E0597]: `character.hitpoints` does not live long enough
        }),
    };
} // `character.hitpoints` dropped here while still borrowed
```

Now that the callback has `Send`, the compiler knows that it might be sent to another thread, and so it cannot guarantee that the mutable reference to `character.hitpoints` captured by the closure will live longer than `character` itself.

The solution I took was to use `move` with the closure to force it to take ownership of `character`:

```rust
fn main() {
    let character_timer = Timer {
        callback: Box::new(move || {
            character.hitpoints -= 1;
        }),
    };
}
```

Now that `move` is specified, we can drop the lifetime parameter from `Timer`:

```rust
struct Timer {
    callback: Box<dyn FnMut() -> () + Send>,
}
```

Here's the full example code:

```rust
use std::thread;
use std::sync::mpsc;

struct Character { hitpoints: u32 }

struct Monster { anger: u32 }

struct Timer {
    callback: Box<dyn FnMut() -> () + Send>,
}

fn main() {
    let mut character = Character { hitpoints: 100 };
    let mut monster = Monster { anger: 0 };

    let character_timer = Timer {
        callback: Box::new(move || {
            character.hitpoints -= 1;
        }),
    };

    let monster_timer = Timer {
        callback: Box::new(move || {
            monster.anger += 10;
        }),
    };

    let (tx, rx) = mpsc::channel();

    tx.send(character_timer).unwrap();
    tx.send(monster_timer).unwrap();

    thread::spawn(move || {
        for mut timer in rx.recv() {
            (timer.callback)();
        }
    });
}
```

The complete code that demonstrates running timer repetitions and parameterising the state deltas with this approach can be found in [this commit](https://github.com/thisdotrob/rust-uo-server/tree/55532411640dce317cdbc89d56f0210879d50ca6).

### Wrap up

I'm still not sure which of the two workable approaches above will be best. Using a callback method on `Timer` feels more robust but involves quite a lot more code and would take longer to understand for someone new to the codebase.

Next I'm going to explore how state can be used from elsewhere in the code in between calls to the callback that mutates it.
