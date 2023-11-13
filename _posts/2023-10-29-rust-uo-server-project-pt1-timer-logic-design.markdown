---
layout: post
title:  "Rust UO server project pt1: Timer Logic Design"
date:   2023-10-29 18:00:00 +0100
tags: ["UO server project", "Rust"]
---

Today I've been thinking about a design for the timer logic in my Rust UO server implementation. I took a more detailed look at the timer logic in ServUO and made the following diagrams on my whiteboard to fully grok it:

![ServUO_timer_logic_diagram.png](/assets/image_1698346145359_0.png)

![ServUO_timer_logic_diagram_2.png](/assets/image_1698348649812_0.png)

After exploring diagrammatically I realised that the timer logic in ServUO is not as complicated as the code makes it seems. I think the OO implementation involves so many layers of indirection and state being passed around that a *lot* of jumping around code is needed to piece the logic together.

Once I'd plotted out the data structures and logic used at a level that ignores the language specific implementation details the simplicity of the design came through:
  - any time a timer is started for the first time or modified, a `TimerChangedEvent` is created and pushed to a list (`m_Changed`)
  - each `TimerChangedEvent` has a reference to the `Timer` it relates to
  - creating / modifying timers and pushing corresponding `TimerChangedEvent`s to the `m_Changed` list happens in the main thread, using a lock on `m_Changed`
  - in another thread (the "timer" thread), `m_Changed` is periodically cleared out under lock, with the `Timer` referenced by each `TimerChangedEvent` removed being pushed or removed from another list, `m_Timers`
  - `m_Timers` is a list of lists - each sub list corresponding to a priority category of timers. Timers are pushed and removed from the correct sub list according to the priority number set on the `Timer` instance
  - Once the timer thread has processed all of the `TimerChangedEvent`s have been processed and `m_Timers` updated, it proceeds to check the timers in `m_Timers`, under lock, to see if any are due to run
  - Any that are due to be run are removed from `m_Timers` and pushed to another list, `m_Queued`, under lock
  - Back in the main thread a lock is acquired over `m_Queued` and each timer is dequeued and has it's associated callback called to perform the logic that should be run on the current tick

I then thought about how similar logic could be implemented in Rust. I started by listing out the properties a `Timer` should have, assuming it would be implemented as a struct:

### `Timer` struct properties

- `callback` - the logic to execute when the timer elapses (can make this a string to print to begin with)
- `repetitions` - how many times to repeat the timer
- `interval` - how many milliseconds to wait between repetitions
- `delay` - how long to wait until the first call
- `next` - the next tick that the timer is due to run on
- `count` - how many times the timer has elapsed so far
- `priority` - how often (via which timer bucket) to check whether it should be queued (could skip priority buckets to begin with)
- `queued` - whether the timer has been queued to have its `callback` executed, so that only one of each timer can be queued at a time

To simplify as much as possible for my first go at hacking this logic together I'm going to skip the following properties:
  - `callback` - setting closures or functions as a property value in a Struct seems to be fairly complicated and there are a number of different approaches I need to read up on. All I want to do first is have some timer logic that is capable of prioritising timers and has the ability to dequeue them at the correct time. As long as I can prove this is being done correctly it doesn't matter what logic is actually performed when a timer runs, for now. Instead of `callback` I'll add a `name` property to the `Timer` struct and will print the name out and the current tick count to prove the timers are running when they should.
  - `delay` - for now I can just use `interval` as the initial delay.
  - `priority` - for now I'll skip creating separate buckets for each priority category of timer and just have all timers prioritised equally
  - `queued` - for now I'll skip this property and just allow the same timer to be queued multiple times if that happens

Next I looked at the `TimerChangedEvent` in the ServUO implementation to try and understand if it was necessary. The logic would be simpler if this data type and the associated data structure (`m_Changed`) could be removed. New and changed timers would need to be pushed straight into the main `timers` list instead. The trade-off here is in making sure that the 3 main pieces of logic do not make each other pause for longer than is necessary.

The 3 main pieces of logic are:
  - Registering timers (creating and updating them)
  - Prioritising timers (setting their priority bucket and next run time)
  - Selecting and executing timers (according to their priority bucket and next run time)

I explored 3 different approaches for splitting this logic across threads with shared ownership of data structures:

### Three threads approach with a `Mutex<T>` wrapped vector of `Timer` structs
This approach would have a separate thread for each of the main pieces of logic and a single, shared ownership, vector of `Timer` structs.

A "timer registration thread" would receive args for new or changed timers via an MPSC channel. It would create `Timer` structs using these args. It would then push newly created `Timer` structs to a `timers` vector under lock.

A "timer prioritisation thread" would iterate over the `Timer` structs in the `timers` vector, also under lock. It would inspect each timer and send any that are due to be run via MPSC channel to the third thread. Any that are not due to be run would be left in the vector to be checked later. If there are repetitons left to perform on any timers sent to the third thread, a new timer registration for the next repetition is triggered by sending updated args back to the first thread via channel.

The last thread would be a "timer executor thread". It would receive `Timer` structs due to be run via MPSC channel from the timer prioritisation thread. For each received it would run that `Timer`'s callback.

Ownership of the `timers` vec would be shared between the timer registration thread and the timer prioritisation thread. Each would acquire a lock on the vec to do its work. This would mean that while timers were being registered they would not be able to be prioritised. Timer prioritisation needs to happen once on each tick, so a blocking delay equal to the length of one tick could be added to the timer prioritisation thread. This would create a period between each tick when the `timers` vec was not locked by the timer prioritisation thread and the timer registration thread could acquire a lock to register new timers.

The potential problem here is if the logic in the timer prioritisation thread is not fast enough and timer registrations back up because the lock over `timers` cannot be acquired by the registration thread for long enough. I think this is why the ServUO implementation uses another data structure (`m_Changed`) - to minimise the amount of time spent waiting for a lock by the registration logic.

### Two threads approach with no shared ownership of the vector of `Timer` structs
This approach would use a "timer manager thread" and a "timer executor thread". They would use channels to communicate and a single `timers` vec, owned by the manager thread.

The timer manager thread would alternate between receiving timers to register via a channel and registering them, and checking registered timers to send for execution (again via channel).

The timer executor thread would receive timers ready for execution and run their callbacks.

To alternate between registering and queuing for execution, the timer manager thread could use the `recv_deadline` channel method to spend an amount of time receiving args from the channel. Args received would be used to create `Timer` structs which would then be pushed onto a `timers` vec. After the deadline elapses the thread would stop receiving args and start checking through the `Timer` structs in the vec for any that are due to be run. Any that are due to be run would be sent to the executor thread. If there are still repetitions left for a timer that was just sent, the thread would need to construct a new `Timer` struct and add it to the `timers` vec.

The potential problem with this approach is similar to that for the previous approach. No registration of new timers is possible for the entire time that timers are being checked and sent for execution. Even worse, re-registration of timer repetitions also needs to be done before any new timers can be registered. Again, introducing a second data structure to record new timers to be added would pause registrations only for the amount of time needed to iterate over that second data structure and move the elements into the `timers` vec.

### Three threads approach with a `Mutex<T>` wrapped `new_timers` vector of `Timer` structs
This approach introduces a second vector of `Timer` structs, containing timers that are new and need to be prioritised. Three threads would be needed:

A "timer registration thread" would receive timer args via channel, construct a `Timer` struct with them and add them to a `new_timers` vec under lock.

A "timer prioritisation thread" would consume the `new_timers` vec under lock. Each `Timer` struct consumed would be moved into a `timers` vec, into the correct priority sub list. After consuming the `new_timers` vec, the thread would iterate over the `timers` vec and remove any timers that are due to run. Removed timers would be sent via channel to the third thread for execution. If removed timers have repetitions still to run, args would be constructed for the next run and sent to the first thread via channel for re-registration.

A "timer execution thread" would receive timers to execute and run their callbacks.

Timer registration will only be paused for as long as the timer prioritisation thread takes to consume the `new_timers` vec under lock, which should be very quick.

### Design decision
I'm going to go with the third approach above since it should offer the best performance. It is similar to the approach taken in the ServUO implementation, with the following differences:
  - A single `Timer` type, rather than having a `Timer` type and `TimerChangedEvent` type. The `TimerChangedEvent` type seems like a wrapper around a `Timer` which serves little purpose other than indicating that the `Timer` is new and needs to be registered. This information is duplicated in the name of the data structure (`m_Changed`) that holds the `TimerChangedEvent` instances in the ServUO implementation. In my Rust implementation I will use only a `Timer` struct, and rely on the name of the vec that holds new instances (`new_timers`) to differentiate between timers that need to be registered and timers that have been registered and priortised.
  - Immutable timer instances. ServUO uses a single `Timer` instance to execute multiple repetitions of a timer. This instance is passed back and forth between threads and needs to keep its own reference to the priority list it is currently assigned to to make this work. Creating a `Timer` struct instance for each repetition instead keeps the code simpler and shouldn't affect performance much, assuming creating a struct is cheap. I might need to revisit this in the future but as a starting point I think it makes sense to more easily keep the borrow checker happy.
