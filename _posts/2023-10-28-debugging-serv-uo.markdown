---
layout: post
title:  "Debugging ServUO"
date:   2023-10-28 14:26:41 +0100
---

After getting [ServUO](https://github.com/ServUO/ServUO/) running locally I wanted to start looking over the code for inspiration for my Rust implementation. I started reviewing the code using Neovim but there were so many files and layers of indirection that it was really hard going. I needed to set up an IDE that would allow me to jump around the code efficiently and attach a debugger to the running server to observe the data flowing through it.

I assumed the quickest route to achieving this would be installing Visual Studio. There is a MacOS version of the IDE available that after a lot of reading I was under the impression should be able to provide code navigation and debugging using Mono. I spent way too much time trying to get this working, but gave up as I could not for the life of me set the Mono executable for the IDE to use. Whenever I started the debugger it would attempt to use an old version of Mono that either it installed itself or I had previously installed and forgotten about. This old version was obviously not compatible with ServUO as the compile step errored. Weirdly, attempting to run `make` from the terminal would then also try and use the old version and fail. I could reset this behaviour by running `make clean` but at this point I didn't want to waste any more time investigating. I got the strong impression that the path of least resistance offers a hell of a lot less resistance than the running-Visual-Studio-on-MacOS path I was currently following so I switched to using my son's Windows PC and installed Visual Studio on there.

Getting debugging working on the Windows machine ended up being difficult too. I eventually stumbled on a [ServUO forum post](https://www.servuo.com/threads/debugging-servuo-with-visual-studio-image-heavy.810/) which got it working.

I'm sure some of my difficulties getting an IDE and debugger working are due to my lack of experience with C# but I found the experience very, very frustrating. The variety of .NET frameworks and languages, the unintuitive UI, seemingly poor dev community and lack of clear tutorials and documentation made the process really difficult, even compared to setting up Neovim for a new language with it's well known steep learning curve.

Now I'm set up I plan to start looking through the ServUO code to get a high level understanding of the server's main components, and then pick one to implement in Rust.
