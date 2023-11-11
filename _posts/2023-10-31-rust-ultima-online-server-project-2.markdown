---
layout: post
title:  "Rust Ultima Online server project - adding callbacks to a struct in Rust"
date:   2023-10-31 18:00:00 +0100
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

`FnOnce` is implemented by closures which can only be called once. `Timer` structs need to be able to have the callback called multiple times, once for each repetition. So using a `FnOnce` closure would therefore not work:

```rust
struct Timer {
    callback: Box<dyn FnOnce() -> ()>,
}

fn main() {
    let t = Timer {
        callback: Box::new(|| println!("Hello from callback")),
    };

    (t.callback)();

    let t_repetition = Timer {
        callback: t.callback, // error[E0382]: use of moved value: `t.callback`
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
    let t = Timer {
        callback: Box::new(|| println!("Hello from callback")),
    };

    (t.callback)();

    let t_repetition = Timer {
        callback: t.callback,
    };

    (t_repetition.callback)();
}
```

But to be able to operate on some state, the closure would need to either capture values from it's environment or be supplied the state as arguments when executed. `Fn` implementing closures either don't move captured values out or mutate them OR capture nothing from their environment. This therefore won't work if the closure captures state and needs to mutate it e.g.:

```rust
struct Timer {
    callback: Box<dyn Fn() -> ()>,
}

fn main() {
    let mut c = Character {
        hitpoints: 10,
    };

    let t = Timer {
        callback: Box::new(move || c.hitpoints += 10),
        // error[E0594]: cannot assign to `c.hitpoints`, as it is
        // a captured variable in a `Fn` closure
    };
}
```

Passing state as arguments to the `callback` closure is possible when using a `Fn` trait object:

```rust
struct Timer {
    state: Character,
    callback: Box<dyn Fn(&mut Character) -> ()>,
}

fn main() {
    let c = Character {
        hitpoints: 10,
        stamina: 100,
    };

    let t = Timer {
        state: c,
        callback: Box::new(|c| c.hitpoints += 10),
    };

    let mut state = t.state;
    (t.callback)(&mut state);

    let t_repetition = Timer {
        state,
        callback: t.callback,
    };

    let mut state = t_repetition.state;

    (t_repetition.callback)(&mut state);
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
    let c = Character {
        hitpoints: 10,
        stamina: 100,
    };

    let t = Timer {
        state: Box::new(c),
        callback: Box::new(|character| {
            let state_delta = vec![
                StateDelta {
                    property: String::from("+hitpoints"),
                    delta: 10,
                },
            ];
            character.update(state_delta);
        }),
    };

    let mut state = t.state;
    (t.callback)(&mut state);

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

The state (a `Character` struct in this case) can then be captured by the closure and mutated. Care needs to be taken here though - the code below looks like it might work but only the `hitpoints` property is captured by the closure (and copied since it is a simple scalar type for which copying is relatively cheap):

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let mut t = Timer {
        callback: Box::new(move || {
            character.hitpoints += 10;
        }),
    };

    (t.callback)();

    let mut t_repetition = Timer {
        callback: t.callback,
    };

    (t_repetition.callback)();
}
```

This doesn't work because it leaves the `character` variable valid to be used after the `hitpoints` property has been copied into the closure. This means that you could try and retrieve `hitpoints` from it and it will not reflect the value that has been updated by the closure when it is called, because the closure is updating the copied `hitpoints` value, not the `hitpoints` value that is a property of the character:

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let mut t = Timer {
        callback: Box::new(move || {
            println!("in closure before update: {}", character.hitpoints);
            character.hitpoints += 10;
            println!("in closure after update: {}", character.hitpoints);
        }),
    };

    println!("in main before update: {}", character.hitpoints);
    (t.callback)();
    println!("in main after update: {}", character.hitpoints);
}

// This will print out:
//   in main before update: 10
//   in closure before update: 10
//   in closure after update: 20
//   in main after update: 10
```

This [users.rust-lang.org forum post](https://users.rust-lang.org/t/force-struct-to-be-moved-into-a-closure/79713) suggests using shadowing to force the entire struct to be moved into the closure. For the code above this means modifying it to look like this:

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let mut t = Timer {
        callback: Box::new(move || {
            let character = &mut character;
            println!("in closure before update: {}", character.hitpoints);
            character.hitpoints += 10;
            println!("in closure after update: {}", character.hitpoints);
        }),
    };

    (t.callback)();
    println!("in main after update: {}", character.hitpoints); // error[E0382]: borrow of moved value: `character`
}
```

Now attempting to use `character` after it has been moved into the callback results in a compiler error, as I wanted it to.

I also explored moving a mutable reference to the hitpoints property in, so that the value isn't copied: 

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let mut t = Timer {
        callback: Box::new(move || {
            let hitpoints = &mut character.hitpoints;
            println!("in closure before update: {}", hitpoints);
            *hitpoints += 10;
            println!("in closure after update: {}", hitpoints);
        }),
    };

    (t.callback)();
    println!("in main after update: {}", character.hitpoints);
}

// This will print out:
//   in closure before update: 10
//   in closure after update: 20
//   in main after update: 10
```

The above doesn't work, the `hitpoints` value is still copied and the mutable reference points to the copied value, not the original. To prevent the value being copied, the mutable reference needs to be created outside the closure and then moved in:

```rust
fn main() {
    let mut character = Character {
        hitpoints: 10,
    };

    let hitpoints = &mut character.hitpoints;

    let mut t = Timer {
        callback: Box::new(move || {
            *hitpoints += 10;
        }),
    };

    (t.callback)();
    println!("in main after update: {}", character.hitpoints);
    // error[E0502]: cannot borrow `character` as immutable because it is also borrowed as mutable
}
```

Now we get the same result as when we moved the entire `character` struct into the closure - we can't use it after the closure has been defined.

### Wrap up

I'm not 100% sure which of the above approaches is best at this point.

In my next journal post I'll explore passing the `Timer` struct between threads, since adding the callback property as in the code above will not compile at the moment when multiple threads are involved.

I'll then explore how the state that is mutated by the callback can be used from elsewhere in the code in between calls to the callback, which isn't possible with the code above since ownership of the state is either taken by the closure and not given back, or needs to be transferred back and forth between the closure and the execution thread on each repetition.

Allowing the state to be used by other code in between timer executions (and after the final execution) will be necessary. As an example: given a timer that increments `hitpoints` every few seconds, the updated hitpoints will need to be sent back to the game client after each execution so the client can display the correct hitpoints. Some mechanism is needed to allow ownership to be transferred elsewhere in between timer executions.

Exploring different approaches to solving these two issues next should guide me towards the best approach.

TODO: comment on StateDelta needing to sometimes not change by a delta, but just set a new value e.g. the name of an item

TODO: comment on using function pointers instead of closures
