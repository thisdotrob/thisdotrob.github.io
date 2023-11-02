---
layout: post
title:  "Using Logseq for programming language notes"
date:   2023-10-16 14:33:01 +0100
categories: rust
published: true
---

I've been using Logseq for nearly a couple of years now as my sole note taking application. Prior to that I've used a number of different apps, most recently Obsidian. I switched from Obsidian begrudgingly at first because Shopify removed Obsidian from the approved list of programs we could run on our work MacBooks. Because the focus was on moving all my notes from Obsidian to Logseq as quickly as possible I didn't have much time to read up on the correct way to use Logseq at the time, so my workflow has evolved naturally as I've made notes, tried to retrieve and use them and come across difficulties which I then fix.

From the reading I was able to do, the recommended approach with Logseq seemed to be to forget about trying to make a heirarchical tree of notes as you would with a note taking app that uses a file and directory layout and instead write notes straight into the journal pages. As you write, tag the notes by enclosing key words and concepts in double square brackets to make a hypertext that will allow you to traverse a graph like structure later to retrieve information.

I was still following this approach when writing my notes on ["the Rust book"](https://doc.rust-lang.org/stable/book/). On any day that I read and made notes on the book I'd create a block with the chapter number and name like the one below, with the notes themselves nested in sub-blocks:
```
- Read and made notes on The Rust Programming Language - Chapter 4 'Understanding Ownership':
  - allows memory safety guarantees without a garbage collector
  - improves runtime performance as no checks are needed
  - etc...
```

I made attempts to tag these blocks keeping in mind the following retrieval scenarios that I envisaged as being likely:
  - There might be a broad topic that I want to review and I remember a particular chapter in the book covered it
  - There might be a broad topic that I want to review and I remember that the book covered it, but can't recall which chapter
  - There might be a specific topic that I want to review and I remember that the book covered it
  - There might be a broad or specific topic I want to review but I don't remember that it was covered in the book (but it was!)

My first attempt left the notes exactly as I had written them and simply added double square brackets around key words and phrases e.g.:
```
- Read and made notes on [[The Rust Programming Language - Chapter 4 'Understanding Ownership']]:
  - allows [[memory safety guarantees]] without a [[garbage collector]]
  - improves [[runtime performance]] as no checks are needed
  - etc...
```
This works okay for the scenario when I remember the chapter that contains the info I want to retrieve - I just search Logseq for "The Rust Programming Language Chapter 4" or "The Rust Programming Language Understanding Ownership" and the page tag will appear at the top of the results. Click that result and I'm taken to the page and can see the linked reference containing the sub-blocks with the info I need. But if the chapter contains a lot of information, not just on the topic I'm interested in, then the linked references will be too noisy. If I want to retrieve information about the garbage collector which I remember was in Chapter 4 then I can't simply search Logseq for "The Rust Programming Language Chapter 4 garbage collector" - Logseq search isn't good enough to surface the block I need. Even adding quotes around the separate search terms ("The Rust Programming Language..." and "garbage collector" doesn't help). To find specific information like this I found that a Logseq query was necessary:
```
{{query (AND [[The Rust Programming Language - Chapter 4 'Understanding Ownership']] [[garbage collector]]) }}
```
Whilst Logseq helps by offering auto-complete for the tags after you've opened the double square brackets and start typing, this is still a lot more time consuming that using Logseq's standard search.

I also tried getting Logseq queries to perform fuzzy search. For example what if sometimes the topic was tagged as `[[garbage collection]]`, sometimes as `[[garbage collector]]` and sometimes as `[[garbage collecting]]`? I tried the following but it didn't return any results:
```
{{query "garbage collect*"}}

{{query [[garbage collect*]]}}
```
I took a quick look at Logseq docs but couldn't easily surface an an answer for how to perform fuzzy searching so at this point decided to ditch using Logseq queries and restructure the notes instead. My aim being to allow searching (using normal Logseq search) for single tags for the main keywords only and then easily fold and unfold to reduce noise. This has left a structure which is a lot more heirarchical than I understood was desirable with Logseq, so I'm sure there's a better way but right now I need to keep on track with my learning plan for Rust and don't want to get side tracked.

This is the tagging structure I've ended up with:
```
- Read and made [[Rust]] notes on [[The Rust Programming Language book/Chapter 4]]:
  - [[Ownership]]:
      - allows [[memory safety]] guarantees without [[garbage collection]]
      - improves [[runtime performance]] as no checks are needed
      - etc...
```

Getting the tagging into this state required quite a lot of rewording. The notes now reflect the structure of the book less and the heirarchy of concepts as I would prefer to navigate through them more. The rewording took time, but it was a good refresher and there were some areas where I spent time playing around in code after realising I'd forgotten aspects or didn't fully understand.

I've spent a few days using the notes while writing code and I'm still not happy with the tagging structure. Having all the book notes written in blocks and sub-blocks in journal pages means that the only way to surface them when searching for tags is through linked references. Linked references are okay, but they appear most recently written first which isn't great for book notes as normally concepts start out basic and are built on - the oldest linked references are likely to be the ones that provide an overview of a tagged topic or an intro and are the ones you'd want to see first. I've also noticed that Logseq's graph view is barely populated when everything is written in journal pages with tags to other pages that are essentially empty, serving only as a link between blocks and sub-blocks.

I think this is all pointing to me going a bit overboard on only using journal pages and trying to let the graph evolve naturally instead of putting a bit of thought in upfront about tagging and organisation. I plan to move the Rust Book notes out of the journal pages and into pages of their own according to topic. The journal pages will then just have actual journal entries e.g.:
```
- Read and made notes on [[Rust/books/The Rust Programming Language/Chapter 4 Understanding Ownership]].
```

As above, I'd like to make more use of heirarchical tags. In the `[[.../Chapter 4 Understanding Ownership]]` page, I'd have something similar to the following:
```
- Covers [[Rust/Ownership]].
```

Then in the `[[Rust/Ownership]]` page I'd have the actual notes I made while reading the book:
```
- allows [[memory safety]] guarantees without [[garbage collection]]
- improves [[runtime performance]] as no checks are needed
- etc...
```

I'll see how this tagging strategy works. I'm not sure mixing heirarchical and non-heirarchical tagging is a good idea, but we'll see. I also think it will struggle to surface information where the link between two search terms is not made explicit e.g. searching for "Rust" and "memory safety" in an attempt to get all blocks that talk about memory safety in the context of Rust probably won't return the block talking about memory safety in the `[[Rust/Ownership]]` page example above
