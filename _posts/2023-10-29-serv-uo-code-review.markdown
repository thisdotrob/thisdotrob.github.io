---
layout: post
title:  "ServUO code review"
date:   2023-10-29 12:18:07 +0100
---

I've spent the last few days looking through the [ServUO]() code to understand how the server works. It's been tough going as I'm not familiar with C#, there's zero documentation and it is a pretty complicated project. It's also a HUGE project. I'm in awe that the authors have been able to build a server emulator for such a complicated and feature filled game. I'm not sure if ServUO was built from scratch or based on a previous UO server emulator, but at some point there was a first server emulator that was built purely based on knowledge of how the game worked from playing it and presumably by inspecting packets set from client to server. Building it must have taken an insane amount of hours and I think that is testament to how passionate people are about the game and keeping it alive.

Here's my rundown of how the server works at a high level:

### Main thread and `main()` function
On server startup a `main()` function is called which does the following setup:
  - Sets up the game tick logic
  - Sets up a networking thread and starts the server listening
  - Sets up a timer thread which is responsible for prioritising and queuing code to be run at specific times and intervals
  - Runs a bunch of configuration and initialisation scripts which load and populate the world and start timers
  - Starts an infinite loop which pauses at the end of each loop until a signal is sent from elsewhere in the code

The loop started by the main thread does the following:
  - Processes network packets received from clients
  - Processes delta queues - queues of changes to items and mobiles (objects in the game world that are mobile e.g players, monsters and NPCs) that need to be enacted and communicated back to clients
  - Executes timer callbacks
  - Sends queued network packets to clients
  - Closes connections for disconnected clients

The signal to restart the loop in the main function is sent from various places whenever the loop needs to process the changes they have queued e.g. mobile and item deltas, timer executions.

### Timer thread
The timer thread does the following:
  - Initialises timer instances using config passed from elsewhere in the code to a constructor
  - Registers "timer change entries" for new or modified timers, in a list
  - Periodically goes through and clears out the timer change entries list, adding their associated timer instances into a main timers list (or removing them from it if the change entry indicates they should be)
  - Periodically checks the main timers list, finding any timers that are due to be run by comparing there scheduled run time to the current tick count.
  - Timers due to be run are pushed to a queue which the main thread consumes.
  - Timers that are pushed to the queue also have their properties reset for their next run time, or are removed from the main timers list if they will not run again. This is done by adding new entries to the timer change entries list.

The main timers list is actually a list of lists. Each sub-list corresponds to a timer *priority* of which there are 6. When adding a timer to the list the thread inserts it into the correct sub list according to the priority given to the timer's constructor. When checking the main timers list, sublists are only checked if the configured amount of time for the priority has elapsed since the last check.

The checking of the main timers list happens at least once a millisecond. The highest priority timer sub list has a priority delay of zero, meaning timers in this list will be checked on every check of the main list. The lowest priority timer sub list has a priority delay of 1 minute, meaning the timers in it will be checked once every 60,000 checks of the main list.

When the main thread pulls from the queue of timers to execute, it runs the callback stored on the dequeued timer. A timer can also have state objects stored on it which are provided as arguments to the callback if present.

Timers are created either during server start up or as the result of processing network packets.

### Registering and processing deltas

A "delta" represents a difference in the current and desired state of a mobile or item. They are registered and added to queues for processing in response to network packets received and timer callbacks executing.

The easiest way to explain how a delta is registered and processed is to run through an example. I've taken the example of a player clicking the "CONTINUE" button on the dialogue they are shown when their character is dead and near a healer who is offering to resurrect them. Clicking this button resurrects the ghost of the player's character and restores their hit points to 10.

A UI dialogue is referred to in UO as a "gump" (Graphical User Menu Pop-up). When the player clicks the "CONTINUE" button on this particular gump a packet is sent to the server from the client. All UO packets have a "packet ID" byte. The packet ID is used to lookup the correct packet handler for the packet type. For this particular packet, the packet ID is 177. This packed ID has the `DisplayGumpResponse` handler registered.

The `DisplayGumpResponse` handler is called with the packet. It pulls the next 3 bytes from the packet which are the "serial", "type ID" and "button ID". All three are generated by the server when the gump is first created, and are sent back to the server in any packets that are sent in response to player interactions with that gump. Serial is an ID for the specific gump and is incremented each time a new gump is created, starting at 1. The type ID corresponds to the category of the gump and is generated by taking the hash code of the gump's class name as a string. Button ID is the ID of the button within the gump. For the gump in question there are two buttons "CONTINUE" (ID = 1) and "CANCEL" (ID = 0). The button ID in this packet is therefore 1 as the player clicked "CONTINUE".

The handler finds the corresponding gump instance using the serial and type ID and calls an `OnResponse()` method on it, passing information from the packet and the connection's state. The gump found for this packet is of type `ResurrectGump`. Its `OnResponse()` method performs a lot of logic but for this example I'll focus on how it resets the character's hit points to 10. To do this, the method gets the `Mobile` instance associated with the connection's state (this will be the player's character) and calls a `resurrect()` method on it.

The `resurrect()` method sets the instance's hit points attribute to 10 and then registers a delta to be processed. The delta has a flag of "4", which means it is a hit points delta. The delta flag is registered by using a bitwise OR operation on the flag and the instance's `DeltaFlags` attribute. The attribute starts off as a 32 bit zero value. Applying bitwise OR to it with the hit points delta flag of "4" (`100`) leaves the attribute with a `1` bit in the third from right position. Other flags have values that correspond to unique digits in the `DeltaFlags` attribute e.g. the mana points delta flag is "8" (`1000`) and the stamina points flag is "16" (`10000`). This means the `DeltaFlags` attribute could be set to `11100` indicating there are three deltas to process: stamina, mana and hit points.

After the flag has been registered, the character's `Mobile` instance is added to a list of instances for which deltas need to be processed (one of the delta queues mentioned earlier). A signal is sent to the `main()` function which triggers the restart of its loop. This loop removes each `Mobile` instance in the delta queue and calls the `ProcessDelta()` method on it. This method inspects the `DeltaFlags` attribute to decide what packets should be queued for sending back to the client. In this example, `DeltaFlags` indicates a `MobileHits` packet should be queued for sending. The constructed packet has packet ID 161, which is written first. Next comes the serial of the player's character, then the max hit points and lastly the new hit points (10).

The client receives this `MobileHits` packet and updates the hit points displayed in game.
