---
layout: post
title:  "Rust UO server first steps"
date:   2023-10-30 11:44:00 +0100
---

Now I've finished reviewing the ServUO code, I'm ready to get started on my Rust UO server implementation.

Rather than trying to implement the server end to end I've picked one component to work on first. I'll get this component working, providing a CLI interface to interact with it in place of where other components would do so in the finished server.

The component I've chosen to work on first is the timer logic. This logic is responsible for prioritising and queuing code to be run at specific times and intervals.

I've picked this component to start with because I can implement it using only Rust's standard library. Other components such as the networking would likely be best implemented using external crates. If I picked one of these components I could start using just the standard library but would then want to re-write parts of it to use the external crates. Or I could spend time researching the external crates up front before writing. Either way, there's extra time required and I'd rather get stuck in and start writing rust ASAP. Picking the timer logic means I can put into practice what I've learnt from The Rust Book straight away.
