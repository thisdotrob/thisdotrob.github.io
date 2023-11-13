---
layout: post
title:  "Rust UO server project pt3: Adding Timer Callbacks"
date:   2023-10-31 18:00:00 +0100
tags: ["UO server project", "Rust"]
---

Today I've carried on working on the timer logic in my Ultima Online server implementation project. Previously the logic only displayed progress bars that incremented as each repetition of a timer was run. Next I needed to implement functionality to allow some arbitrary code to be run when a timer is executed. This is the whole point of having timers - execute some logic at a certain time or at certain intervals.

This journal post documents my exploration of storing a callback on the `Timer` struct to be called by the timer execution thread. The logic that runs on each execution will need to modify some game state e.g. a character's hitpoints, so this post also explores how state can be passed to a callback property on a struct.

My first thought for how to implement the callback on the `Timer` struct was to set it as a `callback` property with the value being a closure. I wasn't sure what type to give the property, so explored that first:

```rust
pub struct Timer {
    // snip
    callback: ????, // some sort of closure type?
}
```

Closures do not have concrete types that can be used in a struct definition, but there are three traits that closures can implement: `Fn`, `FnOnce` and `FnMut`.

To use one of these traits to specify the type of `callback`, a trait object would be necessary e.g.:
```rust
struct Timer {
    callback: Box<dyn Fn() -> ()>,
}
```

### Using a `FnOnce` trait object for the callback

`FnOnce` is implemented by closures which can only be called once. `Timer` structs need to be able to have the callback called multiple times, once for each repetition. So using a `FnOnce` closure would therefore not work and we'll get the error demonstrated in the example below:

```rust
struct Timer {
    callback: Box<dyn FnOnce() -> ()>,
}

fn main() {
    let timer = Timer {
        callback: Box::new(|| println!("Hello from callback")),
    };

    (timer.callback)();

    let timer_repetition = Timer {
        callback: timer.callback, // error[E0382]: use of moved value: `t.callback`
    };
}
```

### Using a `Fn` trait object for the callback

`Fn` is implemented by closures which can be called multiple times, so the following will compile:

```rust
struct Timer {
    callback: Box<dyn Fn() -> ()>,
}

fn main() {
    let timer = Timer {
        callback: Box::new(|| println!("Hello from callback")),
    };

    (timer.callback)();

    let timer_repetition = Timer {
        callback: timer.callback,
    };

    (timer_repetition.callback)();
}
```

But to be able to operate on some state, the closure would need to either capture values from it's environment or be supplied the state as arguments when executed.

`Fn` implementing closures can either do one of two things:
  - don't move captured values out AND don't mutate them, or
  - capture nothing from their environment.

We therefore can't use state captured from the environment with a `Fn` closure, we'll get an error like that in this example:

```rust
struct Timer {
    callback: Box<dyn Fn() -> ()>,
}

fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let timer = Timer {
        callback: Box::new(move || character.hitpoints += 10),
        // error[E0594]: cannot assign to `character.hitpoints`, as it is
        // a captured variable in a `Fn` closure
    };
}
```

This leaves passing state as arguments to the `callback` closure, which is possible when using a `Fn` trait object:

```rust
struct Timer {
    state: Character,
    callback: Box<dyn Fn(&mut Character) -> ()>,
}

fn main() {
    let character = Character {
        hitpoints: 10,
        stamina: 100,
    };

    let timer = Timer {
        state: character,
        callback: Box::new(|character| character.hitpoints += 10),
    };

    let mut state = timer.state;
    (timer.callback)(&mut state);

    let timer_repetition = Timer {
        state,
        callback: timer.callback,
    };

    let mut state = timer_repetition.state;

    (timer_repetition.callback)(&mut state);
}
```

Each execution of the timer pulls out the `state` property from it and passes it to the `callback`.

The problem here is that `Timer` structs can now only operate on `Character` types. I could have multiple `Timer` struct definitions, one for each type of state they will operate on e.g. `CharacterTimer` but then specifying the types in the logic that creates the timers and executes them would then be very complicated, if not impossible.

I explored an alternative of setting the `state` type to be a trait object too:

```rust
struct Timer {
    state: Box<dyn State>,
    callback: Box<dyn Fn(&mut Box<dyn State>) -> ()>,
}
```

The `State` trait would categorise a struct that can have some state updated by a callback, for example through an `update` method:

```rust
trait State {
    fn update(&mut self) -> ();
}
```

`Character` would then needed to implement the trait:

```rust
impl State for Character {
    fn update(&mut self) {
      // update self
    }
}
```

You can't update state without some extra arguments to `update` though. These arguments can't be specific to the type that implements `State`, because then all types implementing `State` would need an `update` function that takes those arguments:

```rust
trait State {
    fn update(&mut self, hitpoints, stamina) -> ();
}

impl State for Character {
    fn update(&mut self, hitpoints, stamina) {
      self.hitpoints += hitpoints;
      self.stamina += stamina;
    }
}

impl State for TreasureChest { // A TreasureChest doesn't have "hitpoints" or "stamina"
    fn update(&mut self, gold) { // <-- type signature is incompatible with that of State
      self.gold += gold;
    }
}
```

Instead, `update` could take a more generic argument which provides information on the state property that needs updating and the delta to change it by:

```rust
struct StateDelta {
    property: String,
    delta: u8,
}
```

`update` could then take a vec of these structs:

```rust
trait State {
    fn update(&mut self, state_deltas: Vec<StateDelta>) -> ();
}
```

Implementing `State` for a `Character` would then look like this:

```rust
impl State for Character {
    fn update(&mut self, state_deltas: Vec<StateDelta>) {
        for state_delta in state_deltas {
            match state_delta.property.as_str() {
                "+hitpoints" => self.hitpoints += state_delta.delta,
                "-hitpoints" => self.hitpoints -= state_delta.delta,
                "+stamina" => self.stamina += state_delta.delta,
                "-stamina" => self.stamina -= state_delta.delta,
                _ => (),
            }
        }
    }
}
```

And then setting up the `Timer` that modifies the state of a `Character`:

```rust
fn main() {
    let character = Character {
        hitpoints: 10,
        stamina: 100,
    };

    let timer = Timer {
        state: Box::new(character),
        callback: Box::new(|character| {
            let state_deltas = vec![
                StateDelta {
                    property: String::from("+hitpoints"),
                    delta: 10,
                },
            ];
            character.update(state_deltas);
        }),
    };

    let mut state = timer.state;
    (timer.callback)(&mut state);

    // snip
}
```

This works, but storing the state separately on the `Timer` just to be able to pass it as an argument to `callback` and so satisfy the `Fn` trait seems unnecessary, when we could instead have the closure capture the state from its environment.

### Using a `FnMut` trait object for the callback

To allow capturing state from the environment and mutating it, a `FnMut` trait object is needed instead:

```rust
struct Timer {
    callback: Box<dyn FnMut() -> ()>,
}
```

The state (a `Character` struct in this case) can then be captured by the closure and mutated. But just attempting to mutate the state in the callback after making it a `FnMut` trait object won't work. An error relating to lifetimes will result as in this example:

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let mut timer = Timer {
        callback: Box::new(|| {
            character.hitpoints += 10;
            // error[E0597]: `character.hitpoints` does not live long enough

        }),
    };
} // - `character.hitpoints` dropped here while still borrowed
```

The problem is that the compiler is giving the trait object (`dyn FnMut() -> ()`) on the `Timer` struct a lifetime of `'static`, meaning it treats the lifetime of the callback closure as potentially as long as the program. But the closure captures the `hitpoints` property from `Timer` using a mutable reference, and that is only valid for the lifetime of the `Timer` struct it belongs to. The `Timer` struct is dropped at the end of the `main()` function and so has a shorter lifetime than `'static`. The solution is to provide the compiler more information by specifying a lifetime parameter on `Timer` - to tell it that the closure should only have a lifetime equal to that of the `Timer` struct it belongs to:

```rust
struct Timer<'a> {
    callback: Box<dyn FnMut() -> () + 'a>,
}
```

Now the code compiles and we can run repetitions of the timer too:

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let mut timer = Timer {
        callback: Box::new(|| {
            character.hitpoints += 10;
        }),
    };

    (timer.callback)();

    let mut timer_repetition = Timer {
        callback: timer.callback,
    };

    (timer_repetition.callback)();
}
```

### Using a function pointer for the callback

I explored the alternative of using a function pointer for the callback. A function pointer to a function on a struct can be set as a property on another struct e.g.:

```rust
struct Timer {
    callback: fn(),
}

struct Character {}

impl Character {
    fn update() {}
}

fn main() {
    let timer = Timer {
        callback: Character::update,
    };
}
```

The `update()` function on Character in the example above does not take any arguments. In reality it would need to take a `self` argument so that it could update the `Character` instance. Adding a `self` parameter to `update()` means that the type for `callback` on `Timer` then needs to reflect that. The state also needs to be added as a property on `Timer` so it can be passed to the callback:

```rust
struct Timer {
    callback: fn(&Character),
    state: Character,
}

struct Character {}

impl Character {
    fn update(&self) {}
}

fn main() {

    let character = Character {};

    let timer = Timer {
        callback: Character::update,
        state: character,
    };

    (timer.callback)(&timer.state);
}
```

This compiles but now we have the same issue we saw before when passing state as arguments to a callback closure that implemented the `Fn` trait - that `Timer` would only work with `Character` structs and no other types of state.

To allow `Timer` to take callbacks which can operate on different types of state, a trait bound can be specified on `Timer`:

```rust
trait State {}

struct Character {}
struct Monster {}

impl State for Character {}
impl State for Monster {}

struct Timer<T: State> {
    state: Box<T>,
    callback: fn(Box<T>),
}

impl Character {
    fn update(self: Box<Self>) {}
}

impl Monster {
    fn update(self: Box<Self>) {}
}

fn main() {
    let character = Character {};

    let character_timer = Timer {
        state: Box::new(character),
        callback: Character::update,
    };

    (character_timer.callback)(character_timer.state);

    let monster_timer = Timer {
        state: Box::new(monster),
        callback: Monster::update,
    };

    (monster_timer.callback)(monster_timer.state);
}
```

Using `T: State` specifies that both the `state` and `callback` properties on `Timer` need to take a type that implements the `State` trait. Importantly it also means that for any particular instance of a `Timer`, the *concrete type* these two properties take must be the same i.e. you can't set up a `Timer` with a `Monster` for state but `Character::update` as the callback. It ensures that whatever concrete type is set for `state` must also be the concrete type that `callback` accepts as an argument:

```rust
    let character = Character { hitpoints: 100 };

    let timer = Timer {
        state: Box::new(character),
        callback: Monster::update,
        // error[E0308]: mismatched types
        // = note: expected fn pointer `fn(Box<Character>)`
        //   found fn item `fn(Box<Monster>) {Monster::update}`
    };
```

Now we have the ability to pass the state instance to the callback and the callback can modify it. But we need to pass additional arguments so the callback knows *how* to modify the state, e.g. the integer amount to increase a character's hitpoints by. The same approach as we took before with the `Fn` trait closure can be taken here - introducing a `StateDelta` type:

```rust
struct StateDelta {
    property: String,
    delta: u8,
}

struct Timer<T: State> {
    // snip
    callback: fn(Box<T>, Vec<StateDelta>),
}

impl Character {
    fn update(self: Box<Self>, state_deltas: Vec<StateDelta>) {}
}

// snip

fn main() {
    let character = Character {};

    let timer = Timer {
        state: Box::new(character),
        callback: Character::update,
    };

    let state_deltas = vec![
        StateDelta {
            property: String::from("+hitpoints"),
            delta: 10,
        },
    ];

    (timer.callback)(timer.state, state_deltas);
}
```

One thing to note here - `StateDelta` only allows a `u8` type for the `delta` property. This only allows passing numeric deltas, but what if the state needs a `String` property updating? A string could be represented as an integer which would then need to be converted back to a string by the `update()` method on the state type, but I'm unsure if that will be true for all the different types that need to be updated on state types, because I'm not sure what they all are yet. I'll revisit this later, if I decide to take this approach.

### Wrap up

I'm not 100% sure which of the above approaches is best at this point.

In my next journal post I'll explore passing the `Timer` struct between threads, since adding the callback property as in the code above will not compile at the moment when multiple threads are involved.

I'll then explore how the state that is mutated by the callback can be used from elsewhere in the code in between calls to the callback, which isn't possible with the code above since ownership of the state is either taken by the closure and not given back, or needs to be transferred back and forth between the closure and the execution thread on each repetition.

Allowing the state to be used by other code in between timer executions (and after the final execution) will be necessary. As an example: given a timer that increments `hitpoints` every few seconds, the updated hitpoints will need to be sent back to the game client after each execution so the client can display the correct hitpoints. Some mechanism is needed to allow ownership to be transferred elsewhere in between timer executions.

Exploring different approaches to solving these two issues next should guide me towards the best approach.
