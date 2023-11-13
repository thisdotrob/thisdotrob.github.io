---
layout: post
title:  "Getting ServUO running"
date:   2023-10-20 09:55:51 +0100
tags: ["ServUO", "UO server project"]
---

I've spent the last few days pouring over the [ServUO Ultima Online server emulator](https://github.com/ServUO/ServUO/) code.

First step was to get the server running on my MacBook with Mono. The readme instructions make it seem like this is a simple task - just install Mono and then run `make`. Unfortunately it wasn't this simple. The first hurdle was a compile error after running `make`:
```sh
Scripts: Compiling C# scripts...Error:
System.TypeInitializationException: The type initializer for
'Microsoft.CodeDom.Providers.DotNetCompilerPlatform.CompilationSettingsHelper' threw an exception.
---> System.EntryPointNotFoundException: IsDebuggerPresent assembly:<unknown assembly>
type:<unknown type> member:(null)
```

A bit of digging on the [ServUO forums](https://www.servuo.com/) and [ServUO Discord](https://discord.gg/0cQjvnFUN26nRt7y) turned up a few other people with the same error but no solution. I cloned the previous ServUO version (`57.2`) and `make` ran fine.

Next hurdle encountered was this prompt after `make` had finished compiling and started the server:
```sh
Enter the Ultima Online directory:
>
```

A bit more digging and I discovered that you also need to get copies of the `.mul` files from a UO client installation and give the server access to them (via the directory passed to the prompt above). The `.mul` file extension seems to be specific to UO, and the files contain various types of data including graphics, animation data, maps and patch contents.

This [ServUO forum post](https://www.servuo.com/threads/howto-compile-and-run-servuo-using-mono-in-linux.13357/) was helpful in figuring out what needed to be done.

To get the `.mul` files I needed to install the official UO client, so I needed to borrow my son's Windows PC as there is no client for Linux or MacOS. I coped any `.mul` files in the Windows UO directory into a `UO_DATA` subdirectory in the `ServUO` repo.

Providing a unix style path to this directory at the prompt then worked:
```sh
Enter the Ultima Online directory:
> ./UO_DATA
DataPath: ./UO_DATA
Regions: Loading...done
World: Loading...
...done (0 items, 0 mobiles, 0 customs) (0.03 seconds)
```

I found out you can set this path in `Config/DataPath.cfg` to avoid having to answer the prompt every time you start the server:
```cfg
# Config/DataPath.cfg
CustomPath=./UO_DATA
```

After providing the next prompt with a username and password for the server admin account, the server was running.

Next I wanted to get a client connected to the locally running server so I could inspect packets if I needed to. You can't use the official UO client to connect to a ServUO server, so I researched what other clients were compatible. The [UO Alive](https://uoalive.com/) free shard had a good [guide on client set up](https://uoalive.com/playnow.html) which I followed to install the [ClassicUO](https://www.classicuo.eu/) client on my son's Windows PC. ClassicUO offers a web client too, but there's gatekeeping over which servers it can connect to, so connecting to my local ServUO instance wouldn't be possible.

I could see from the server output that ServUO was listening on my local network on port 2593 so I created a new profile in the ClassicUO launcher set to connect to the MacBook's LAN ip on that port, hit connect and after creating a character I was in the (very empty) UO world.
