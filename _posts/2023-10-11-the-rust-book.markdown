---
layout: post
title:  "The Rust Book"
date:   2023-10-11 17:03:00 +0100
---

I've finished the first step of my Rust learning plan, reading "The Rust Programming Language" a.k.a "The Rust Book". It's great that Rust has an "official" book that strikes the balance well between being accessible enough for newcomers to the language while also going into depth on and covering most of the features of the language. It made starting out a lot easier as there was no confusion about *where* to start, the decision was made quickly once I saw that the official book was recommended a bunch.

Normally when picking up a new language I prefer to learn by writing code very early on, rather than relying on video tutorials, reading books or guides. Since Rust had an official book and it was recommended a lot as a starting point I took a different approach this time and read it cover to cover, only writing code when concepts weren't 100% clear and I needed to explore them to fully grok.

I think the book itself does a good job at catering to readers that will have a variety of backgrounds. For me, the sections on the ownership model, references, trait objects, lifetimes and unsafe code were the most interesting, useful (and challenging) because they weren't as familiar to me from other languages I have worked with before. I was able to skim through a lot of the other sections and was able to get through the book quickly.

The book is excellent for an introduction to the language but it would be hard to use as a reference in the future. Because it's aimed at newcomers with different levels of programming experience and different language backgrounds I would need to read a lot of text to find the info I needed if I were to refer back to it in the future when trying to remember how to do something in Rust. To fix this, I made notes as I went, condensing the book down into bullet point form tailored to my needs by covering only new or unfamiliar concepts and differences in syntax that I could see myself needing to refer back to.

After reading the book I'm even more excited about learning Rust and to start writing some code in it. A lot of the language design choices I've read about seem sound:
- no null type and requiring error types to be handled.
- exceptions in the form of panics that actually are exceptions and so halt execution.
- immutability by default.
- enforcing rules over memory ownership in the compiler so the burden isn't left to the user to manage access and check for mistakes.
- requiring minimal type annotations - only in places that allow the compiler to infer types for the rest of your code so annotations aren't required everywhere.
- a compiler that provides super helpful feedback that can be relied on as a guide when writing code.
- automatic copying of memory only when it doesn't impact performance (i.e. no deep copying by default)
- keeping non-zero cost abstractions to a minimum.
- providing a solid but minimal standard library, relying on community crates to build on and extend the functionality offered.
- preferring message passing for safe concurrency.
- a compiler that is conservative, emphasising safety at runtime and reliability but allowing an escape hatch (unsafe code blocks) for situations where you have more information than the compiler can be aware of.
- limiting syntax to allow the compiler to be simpler and make more guarantees e.g. requiring all of a `match` expression arm's to evaluate to the same resultant type.
- surfacing important complexities and giving control over how they are tackled rather than abstracting over them as other languages do e.g. not allowing indexing into strings.
- easy and reliable dependency management with a very user friendly CLI.

Next I plan to convert the notes I've taken on the book into the start of a Rust knowledge graph in Logseq.
