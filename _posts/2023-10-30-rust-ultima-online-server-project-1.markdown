---
layout: post
title:  "Rust Ultima Online server project - basic timer logic"
date:   2023-10-30 18:00:00 +0100
---

I've finished v1 of the Rust UO server timer logic. The logic covers:
  - registering new timers
  - calculating their next run time
  - checking registered timers for those due to be run
  - removing timers that are due to be run and "running" them
  - re-registering timers for next repetitions

At the moment there are no callbacks attached to the timers so when they are "run" nothing actually happens beyond dequeing the timer (and re-registering a repeat of it if there are repetitions remaining).

To test the logic I added a CLI module which allows starting timers from the command line and monitoring their repetitions being "run" via progress bars:

![rust_uo_server_timer_logic_v1_demo.gif](/assets/rust_uo_server_timer_logic_v1_demo.gif)

### Code walkthrough

#### Timer structs

First I created the `Timer` struct in a new *timer* module:
```rust
// src/timer.rs
pub struct Timer {
    name: String,
    repetitions: isize,
    interval: i64,
    next: i64,
}
```
`repetitions` are the number of repetitions of the timer left to run. `interval` is the time in ms between each repetition. `next` is the next server tick that the timer is due to run at.

`next` will be calculated by the server: when a timer is registered for the first time it will be *current ticks + timer interval*. When timer repetitions are registered it will be calculated as *previous next + interval*.

Because `next` will be calculated by the server as above, it won't be provided to the registration logic when code elsewhere requests a timer be registered for the first time. I therefore created a second `TimerArgs` struct to represent the subset of timer properties that *will* be provided to the registration thread by code elsewhere:

```rust
// src/timer.rs
pub struct TimerArgs {
    pub name: String,
    pub repetitions: isize,
    pub interval: i64,
}
```

In the future this will also allow properties to be optional when passed to the thread, with the thread then being responsible for setting defaults e.g. `TimerArgs` provided with `Option::None` for `repetitions` would have `repetitions` set to 1 when the `Timer` struct was created by the registration thread.

#### Server ticks

Knowing that calculating `next` for a timer required the current server ticks, I next set up a function to return them:

```rust
// src/timer.rs
use chrono::prelude::*;

fn current_ticks() -> i64 {
    let utc_now = Utc::now();
    utc_now.timestamp_millis()
}
```

I opted to go with a tick equalling a millisecond to begin with, but the resolution will need to be increased later.

#### Registration thread

Next I set up the registration thread:

```rust
// src/timer.rs
use std::sync::{Arc, Mutex, mpsc};

mod registration_thread;

// snip

fn start() -> mpsc::Sender<TimerArgs> {
    let (register_tx, register_rx) = mpsc::channel::<TimerArgs>();

    let new_timers: Vec<Timer> = vec![];
    let new_timers = Arc::new(Mutex::new(new_timers));

    registration_thread::spawn(register_rx, Arc::clone(&new_timers));

    return register_tx
}
```

In the *timer* module I added a `start()` function which will be called by `main()` in `src/main.rs`. This function returns an mpsc transmitter which the `main()` function will use to send `TimerArgs` to the registration thread. In the future it won't be `main()` that uses this transmitter, but other parts of the code responsible for requesting timer registration.

The `start()` function also sets up a `new_timers` vec. Ownership of this vec will be shared between the registration thread and the prioritisation thread. The registration thread will push new timers into it, the prioritisation thread will remove them from it and prioritise them. To allow shared ownership, the vec is wrapped in a `Mutex` to allow each thread to modify it after acquiring a lock. Because the mutex needs to be passed to two threads, it is wrapped in an `Arc`.

`start()` calls the `spawn()` function from the *registration_thread* submodule, passing it the mpsc receiver so it can receive `TimerArgs` and the wrapped `new_timers` vec, so it can add `Timers` to it.

The `spawn()` function was declared in `src/timer/registration_thread.rs`:

```rust
// src/timer/registration_thread.rs
use std::sync::{Arc, Mutex, mpsc};
use std::thread;
use super::{Timer, TimerArgs, current_ticks};

pub fn spawn(register_rx: mpsc::Receiver<TimerArgs>, new_timers: Arc<Mutex<Vec<Timer>>>) {
    thread::spawn(move || {
        for timer_args in register_rx {
            let TimerArgs { name, repetitions, interval } = timer_args;
            let next = current_ticks() + interval;
            let timer = Timer { name, repetitions, interval, next };
            let mut new_timers = new_timers.lock().unwrap();
            new_timers.push(timer);
        }
    });
}
```

`spawn()` moves the register receiver and the `Arc<Mutex<>>` new_timers into a newly spawned thread. It treats the register receiver as an iterator by using a for loop on it. This executes the for loop block each time `TimerArgs` are sent to the mpsc channel.

When `TimerArgs` are received, the thread calculates `next`, constructs a `Timer` struct using it and other properties from `TimerArgs` and then pushes it into `new_timers` after acquiring a lock.

At the moment I've left the `unwrap()` call when acquiring the lock. This will need to be replaced with better error handling in the future.

#### Prioritisation thread

Next I set up the thread that will periodically check the `new_timers` vec under lock and prioritise the `Timer` structs it finds there:

```rust
// src/timer.rs

mod prioritisation_thread;

pub fn start() -> mpsc::Sender<TimerArgs> {
    // snip

    let (execute_tx, execute_rx) = mpsc::channel::<Timer>();

    let new_timers: Vec<Timer> = vec![];
    let new_timers = Arc::new(Mutex::new(new_timers));

    // snip

    prioritisation_thread::spawn(execute_tx, new_timers);

    // snip
}
```

Similar to with the registration thread setup, `start()` now also calls the `spawn()` function from the *prioritisation_thread* submodule. It passes it an mpsc transmitter the prioritisation thread will use to send timers to be executed. It also passes the wrapped `new_timers` vec, so the prioritisation thread can pull `Timer` structs from it and prioritise them.

Nothing is done with the corresponding mpsc receiver yet - this will be given to the execution thread so it can receive the `Timer` structs to be executed.

The new `spawn()` function was declared in `src/timer/prioritisation_thread.rs`. Like the `spawn()` function in the registration thread module it spawns a new thread:

```rust
// src/timer/prioritisation_thread.rs
use super::Timer;
use std::sync::{Arc, Mutex, mpsc};
use std::thread;

pub fn spawn(execute_tx: mpsc::Sender<Timer>, new_timers: Arc<Mutex<Vec<Timer>>>) {
    thread::spawn(move || {});
}
```

In the new thread, I first implemented acquiring a lock on `new_timers` every millisecond and emptying it into a new vec that is owned solely by the thread:

```rust
// src/timer/prioritisation_thread.rs

// snip

pub fn spawn(execute_tx: mpsc::Sender<Timer>, new_timers: Arc<Mutex<Vec<Timer>>>) {
    thread::spawn(move || {
        let mut timers = vec![];

        loop {
            thread::sleep(Duration::from_millis(1));
            {
                let mut new_timers = new_timers.lock().unwrap();

                while let Some(timer) = new_timers.pop() {
                    timers.push(timer);
                }
            }
        }
    });
}
```

This means that the thread only locks `new_timers` once a millisecond and only for as long as it takes to remove all elements from it and push them to the new vec. This minimises the time that the registration thread is unable to acquire a lock and so can't process timer registrations.

Next I added logic to go through the `timers` vec and send any that are due to be run to the channel transmitter for the execution thread to pick up:

```rust
// src/timer/prioritisation_thread.rs
use super::{Timer, current_ticks};

// snip

pub fn spawn(execute_tx: mpsc::Sender<Timer>, new_timers: Arc<Mutex<Vec<Timer>>>) {
    thread::spawn(move || {
        let mut timers = vec![];

        loop {
            // snip

            let now = current_ticks();

            for timer in timers {
                if timer.next <= now {
                    execute_tx.send(timer).unwrap();
                }
            }
        }
    });
}
```

This won't compile because after the first iteration of the outer loop the `timers` vec has been moved by the for loop and the variable is invalid. `timers` needs to be reset at the end of the outer loop:

```rust
// src/timer/prioritisation_thread.rs

// snip

pub fn spawn(execute_tx: mpsc::Sender<Timer>, new_timers: Arc<Mutex<Vec<Timer>>>) {
    thread::spawn(move || {
        let mut timers = vec![];

        loop {
            // snip

            let mut not_due = vec![];

            let now = current_ticks();

            for timer in timers {
                if timer.next <= now {
                    execute_tx.send(timer).unwrap();
                } else {
                    not_due.push(timer);
                }
            }

            timers = not_due;
        }
    });
}
```

The last thing needed was to schedule repetitions of timers that had been sent for execution:

```rust
// src/timer/prioritisation_thread.rs

// snip

pub fn spawn(execute_tx: mpsc::Sender<Timer>, new_timers: Arc<Mutex<Vec<Timer>>>) {
    thread::spawn(move || {
        let mut timers = vec![];

        loop {
            // snip

            let mut not_due = vec![];

            let now = current_ticks();

            for timer in timers {
                if timer.next <= now {
                    if timer.repetitions > 1 {
                        let next_repetition = Timer {
                            name: String::from(&timer.name),
                            repetitions: timer.repetitions - 1,
                            interval: timer.interval,
                            next: timer.next + timer.interval,
                        };
                        not_due.push(next_repetition);
                    }

                    // snip
                } else {
                    // snip
                }
            }

            timers = not_due;
        }
    });
}
```

Now new `Timer` structs are constructed for repetitions, with the `repetitions` property decremented and `next` incremented by the `interval`. The repetition instances are also pushed to the `not_due` vec which is then used as the `timers` vec for the next iteration of the outer loop.

See the bottom of this post for the finished code.

#### Execution thread

The execution thread follows the same module pattern to the other two timer threads:

```rust
// src/timer.rs

mod execution_thread;

pub fn start() -> mpsc::Sender<TimerArgs> {
    // snip

    let (execute_tx, execute_rx) = mpsc::channel::<Timer>();

    execution_thread::spawn(execute_rx);

    // snip
}
```

```rust
// src/timer/execution_thread.rs

use std::collections::HashMap;
use std::sync::{Arc, Mutex, mpsc};
use std::thread;
use super::Timer;

pub fn spawn(execute_rx: mpsc::Receiver<Timer>) {
    thread::spawn(move || {
        for timer in execute_rx {
            println!("{}", timer.name);
        }
    });
}
```

The execution thread module spawns a new thread, moves the execute receiver into it, treats it as an iterator and prints the name of each `Timer` received. In the future it will call a callback on each `Timer`.

#### Starting the threads from `main.rs`

At the moment nothing happens when the program runs because the `main()` is empty. Next I updated it to call the `start()` function from the *timer* module:
```rust
// src/main.rs
mod timer;

fn main() {
    let timer_register_tx = timer::start();
}
```

So far so good, but nothing happens when the program runs because nothing is sending `TimerArgs` to the registration thread. I hardcoded this in the `main()` function to test it worked:

```rust
// src/main.rs
use crate::timer::TimerArgs;

// snip

fn main() {
    let timer_register_tx = timer::start();

    let timer_args = TimerArgs {
        name: String::from("A Timer!"), repetitions: 1000, interval: 50
    };

    timer_register_tx.send(timer_args).unwrap();
}
```

#### Creating timers from the command line

The code written so far isn't much good with only a single hard coded timer being registered. Next I updated the code to allow starting timers from the command line as the program is running:

```rust
// src/main.rs
mod timer;
mod cli;

fn main() {
    let timer_register_tx = timer::start();
    cli::start(timer_register_tx);
}
```

`main()` now calls a `start()` function from a new *cli* module:

```rust
// src/cli.rs
use crate::timer::TimerArgs;
use std::io;
use std::sync::mpsc;

pub fn start(timer_register_tx: mpsc::Sender<TimerArgs>) {
    loop {
        println!("Provide stdin with a string in the following format to register a new timer:");
        println!("name repetitions interval(ms)");
        println!("e.g.: \"timer0 100 50\"");

        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();

        let mut split_input = input.split_whitespace();

        let name = split_input.next().unwrap();
        let repetitions = split_input.next().unwrap();
        let repetitions: isize = repetitions.parse().expect("Failed to parse numeric string");
        let interval = split_input.next().unwrap();
        let interval: i64 = interval.parse().expect("Failed to parse numeric string");

        let timer_args = TimerArgs {
            name: String::from(name), repetitions, interval
        };

        timer_register_tx.send(timer_args).unwrap();
    }
}
```

The *cli* module's `start()` function starts a loop which waits for input from stdin. On the command line I could now enter arguments for a new timer in a space separated format of "name repetitions interval" (e.g. `timer0 1000 50`) and the module will parse the string, construct `TimerArgs` and send them to the registration thread.

#### Progress bars

The last change I made wasn't strictly necessary but the `println!` of each timer's name when it was executied was bugging me because it would get in the way of trying to enter another timer on the command line. I decided it would be fun to display progress bars instead so went ahead and opened up that can of worms.

I pulled in the [indicatif](https://github.com/console-rs/indicatif) library and used it in the timer *execution thread* module to display a progress bar for each timer created and increment them on each repetition:

```rust
/// src/timer/execution_thread.rs
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};
use std::collections::HashMap;
// snip

pub fn spawn(execute_rx: mpsc::Receiver<Timer>) {
    thread::spawn(move || {
        let progress_bars = MultiProgress::new();

        let sty = ProgressStyle::with_template(
            "[{elapsed_precise}] {bar:40.cyan/blue} {pos:>7}/{len:7} {msg}",
        ).unwrap().progress_chars("##-");

        let mut progress_bars_lookup: HashMap<String, ProgressBar> = HashMap::new();

        for timer in execute_rx {
            let progress_bar = progress_bars_lookup.get(&timer.name);
            match progress_bar {
                Some(pb) => {
                    pb.inc(1);
                }
                None => {
                    let total: u64 = timer.repetitions.try_into().unwrap();
                    let pb = progress_bars.add(ProgressBar::new(total));
                    pb.set_style(sty.clone());
                    pb.set_message(String::from(&timer.name));
                    pb.inc(1);
                    progress_bars_lookup.insert(String::from(&timer.name), pb);
                }
            }
        }
    });
}
```

Instead of printing the timer's name on execution, the thread now tries to find an existing progress bar for the timer's name from a hash map and increments it if one is found. If one doesn't exist, it is created and added to the hash map.

This worked okay, but whenever the progress bars were incremented it would still disrupt the input of a new timer on the command line. I found that indicatif provided a `suspend()` method that could be called to temporarily pause the display of the progress bars. The method needed to be called on `progress_bars`:

```rust
let progress_bars = MultiProgress::new();

// ...

progress_bars.suspend(|| {
    // do something
});

// progress bars reappear
```

The problem here is that the progress bars were created in the execution thread, but it would be the *cli* module (running in the main thread) that would know when the progress bars needed to be suspended. The `progress_bars` needed to have ownership shared between the main thread and the execution thread - the main thread so it could suspend them and the execution thread so it could create new bars and increment them.

I did the quickest thing I could think of and moved creation of `progress_bars` into the `main()` function and wrapped it in `Arc` and `Mutex` before giving it to both the execution thread and the cli module:

```rust
// src/main.rs
use indicatif::MultiProgress;
use std::sync::{Arc, Mutex};

mod timer;
mod cli;

fn main() {
    let progress_bars = Arc::new(Mutex::new(MultiProgress::new()));

    let timer_register_tx = timer::start(Arc::clone(&progress_bars));

    cli::start(progress_bars, timer_register_tx);
}
```

```rust
// src/timer.rs

// snip

pub fn start(progress_bars: Arc<Mutex<MultiProgress>>) -> mpsc::Sender<TimerArgs> {
    // snip

    execution_thread::spawn(execute_rx, progress_bars);

    // snip
}
```

```rust
// src/timer/execution_thread.rs

// snip

pub fn spawn(execute_rx: mpsc::Receiver<Timer>, progress_bars: Arc<Mutex<MultiProgress>>) {
    thread::spawn(move || {
        // snip

        for timer in execute_rx {
            // snip
            match progress_bar {
                Some(pb) => {
                    // snip
                }
                None => {
                    let progress_bars = progress_bars.lock().unwrap();
                    // snip
                }
            }
        }
    });
}
```

And in the *cli* module I added the call to `suspend()` when a return key was sent to stdin:

```rust
// src/cli.rs

// snip

pub fn start(progress_bars: Arc<Mutex<MultiProgress>>, timer_register_tx: mpsc::Sender<TimerArgs>) {
    loop {
        println!("Press return to start adding a new timer");
        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();

        let progress_bars = progress_bars.lock().unwrap();

        progress_bars.suspend(|| {
            println!("Provide stdin with a string in the following format to register a new timer:");
            println!("name repetitions interval(ms)");
            println!("e.g.: \"timer0 100 50\"");

            // snip
        })
    }
}
```
