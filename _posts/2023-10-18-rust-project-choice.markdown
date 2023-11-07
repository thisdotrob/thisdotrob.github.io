---
layout: post
title:  "Rust project choice"
date:   2023-10-18 13:14:32 +0100
---

Today I've come up with a plan for a Rust project that I'm very excited about. I'm going to build an Ultima Online server.

I played Ultima Online as a kid back in '99 - '01 before moving on to Everquest. The internet was relatively new to me then and being able to play a game with 1000s of other people online in the same world was mind blowing. I have so many fond memories of in-game adventures with school friends and I can still remember my favourite spots in the world - the dungeons we'd hunt monsters in, the mining areas I'd nervously hack away at while looking out for PKs and the places I'd build houses.

I hadn't thought about Ultima Online for years, but during a conversation with my fiance about gaming I mentioned it. She surprised me on Valentines day this year with an Ultima Online gaming session. We researched how you could play in 2023, picked a free shard (Outlands), created our characters and spent the evening bashing monsters in a newbie dungeon.

Playing the game again bought more memories back. I remembered that there were helper programs that you could use to automate the mundane tasks such as making your miner character target areas for mining and then move to the next spot when the ore there was exhausted. These programs had their own scripting language which you wrote the instructions in. I realised that this must have been the first time I wrote code, I just didn't realise it at the time and had forgotten all about it.

Realising that the scripts I wrote for Ultima Online were probably the start of my journey to eventually writing code for a job as an adult, I started thinking about whether a similar approach could be used today to teach kids to code and get them interested in it. Both my sons have had a little exposure to coding in school and they've gone to coding camps in the holidays but it hasn't really sparked their interest. They are both into gaming, as are most of their peers, so perhaps combining the two would be a winning combination.

I came up with an idea for an education coding tool:
  - An online fantasy world like Ultima online
  - Each player has trade skilled characters (e.g. miners, blacksmiths, lumberjacks) which are permanently active in the world, even when the player is not logged in and playing
  - These trade characters are only controllable via scripts the players must write
  - The trade characters interact with each other (also via scripts) to form a production line for in game items
  - These in game items can be used by the non-trade skilled characters the players can create (e.g. warriors, mages)
  - The non-trade skilled characters can't be scripted, the players control them

The scripting of trade characters would introduce the players to coding, with the reward being the in game items they can then equip their non-trade characters with and use in adventures with other players.

My kids already get set homework they have to complete on gamified online platforms for subjects such as maths. I've noticed that they like looking at the leaderboards to see how they rank against their classmates, and spend time customising their characters with the items they are rewarded with for completing the homework challenges set. I can see this Ultima Online like coding game being appealing in a similar way. There could be a global server, or a server set up for an individual school or even class within a school.

I've taken a look at the current Ultima Online ecosystem to see what open source projects there are that I could either use as inspiration or fork and use to build what I want to. Most of the server and client emulators are built in C#, I couldn't find any Rust implementations.

As I mentioned at the start, my plan is to build an Ultima Online server in Rust which might end up being used to facilitate the educational version of the game I described. The first step I'll be taking is reviewing the code of the most popular server emulator out there today ([ServUO](https://github.com/ServUO/ServUO)) before hopefully re-implementing it in Rust.
