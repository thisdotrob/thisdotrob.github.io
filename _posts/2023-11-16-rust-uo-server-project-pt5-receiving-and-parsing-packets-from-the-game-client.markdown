---
layout: post
title:  "Rust UO server project pt5: Receiving and parsing packets from the game client"
date:   2023-11-16 18:00:00 +0000
tags: ["UO server project", "Rust"]
---

In my last journal post on my Rust Ultima Online server project I said I would be looking at accessing and using game state in between timer calls that need to mutate it.

I started looking into this and realised that the main reason for needing access to state inbetween timer calls is to communicate back to the client that the state had been updated. Knowing this I instead decided to dive into the networking side first, and deal with accessing the state after investigating that. It should be easier once I've looked at the networking and know more about what packets need to be sent and received.

### Listening for the first packets sent from the game client

The first thing I did was set up a TCP listening in the server, listening on the port that Ultima Online uses for communiation between the game client and game server (2593):

```rust
// src/tcp.rs
use std::net::TcpListener;

pub fn start() { // <-- called from the server's main() function
    let listener = TcpListener::bind("127.0.0.1:2593").unwrap();

    for stream in listener.incoming() {
        let mut stream = stream.unwrap();
        let mut buffer = [0; 256];
        stream.read(&mut buffer).unwrap();
        println!("{:#04X?}", buffer);
    }
}
```

The code above will simply print out any bytes received from a connected client on the UO port. `:#04X?` in the format string means it will print the hexadecimal representation of the data.

Next I started the game client ([ClassicUO](https://github.com/ClassicUO/ClassicUO)) and attempted to log in from this first screen:

![rust_uo_server_pt5_0.png](/assets/rust_uo_server_pt5_0.png)

Then checked what packets were received:

```
[
    0xEF, 0x01, 0x00, 0x00, 0x7F, 0x00, 0x00, 0x00, 0x07, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x63, 0x00, 0x00, 0x00,
    0x01, 0x80, 0x61, 0x64, 0x6D, 0x69, 0x6E, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x61, 0x64, 0x6D, 0x69, 0x6E, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0xFF, 0x00, 0x00, 0x00, 0x00, ...
]
```

The first byte received is `0xEF`. This should be a packet ID since all Ultima Online packets start with this.

Looking up `0xEF` in [this UO packet guide](https://necrotoolz.sourceforge.net/kairpacketguide/) did not reveal any matches, which had me confused.

Next I searched the [ServUO](https://github.com/ServUO/ServUO/) codebase and found this [line](https://github.com/ServUO/ServUO/blob/c11047d380248e014a63c54648171cc2890423cf/Server/Network/PacketHandlers.cs#L129):

```c#
Register(0xEF, 21, false, LoginServerSeed);
```

This line registers a function (`LoginServerSeed`) to be called whenever a packet with packet ID `0xEF` is recevied.

From this I deduced that `0xEF` is the ID for the ["Encrypted Login Seed" packet](https://necrotoolz.sourceforge.net/kairpacketguide/help.htm) from the packet guide. I'm not sure why no packet ID is given for the packet in the guide, all the other packets have them given.

The packet guide also seemed to have an incorrect length given for the packet. The 4 bytes length it states doesn't leave us with a next byte that matches any packet ID. Instead of relying on the packet guide to determine the packet length I instead looked back at the ServUO code. The `21` in the line of C# above is the `length` argument that `Register` receives, so I checked if I could use that and indeed it did leave me with a next byte that matched a packet ID:

```
[
     *21 bytes removed*

      ... 0x80, 0x61, 0x64, 0x6D, 0x69, 0x6E, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x61, 0x64, 0x6D, 0x69, 0x6E, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0xFF, 0x00, 0x00, 0x00, 0x00, ...
]
```

The next packet is `0x80`, which per the packet guide is the ID for an ["Account Login Request" packet](https://necrotoolz.sourceforge.net/kairpacketguide/packet80.htm). That makes sense since I just hit the login button on the client! Length per the packet guide is `0x3E` or 62 bytes long.

Moving on 62 bytes from the `0x80` takes us past the last non-zero byte received from the connection. So it seems like only two packets have been sent to the server at this point and the client is now awaiting a response. It looks that way since the client is paused on a "Verifying Account..." dialogue.

### Base logic for parsing packet data

Now I knew that I could take a stream of bytes and identify the start and end of each packet I worked on the scaffolding code which would loop over it extracting the packet ID and then sending the remaining bytes in the packet to handler functions to be processed:

```rust
// src/tcp.rs

// snip

pub fn start() { // <-- called from the server's main() function
    // snip

    for stream in listener.incoming() {
        // snip
        stream.read(&mut buffer).unwrap();
        parse_packets(buffer);
    }
}

fn parse_packets(buffer: [u8; 1024]) {
    let mut buffer_slice = &buffer[..];

    while buffer_slice.len() > 0 {
        let packet_id = read_u8(&mut buffer_slice);

        match packet_id {
            0xEF => {
              // call handler function for encryption login seed packet
            },
            0x80 => {
              // call handler function for account login request packet
            },
            _ => continue,
        }
    }
}

fn read_u8(input: &mut &[u8]) -> u8 {
    let (int_bytes, rest) = input.split_at(1);
    *input = rest;
    return *&int_bytes[0];
}
```

To loop over the stream of bytes efficiently I created an array slice over the buffer (`&buffer[..]`), and then passed a mutable reference to this slice to allow it to be changed to point to the `rest` of the bytes in the original array as each byte is consumed.

For example the `read_u8()` function splits the array slice into two array slices, one which points to the first byte in the original array (`int_bytes`) and the second which points to the remaining bytes (`rest`). The `input` variable is then reassigned to the `rest` slice so the next time a function uses it it will be positioned at the next byte along in the original array.

`read_u8()` extracts each byte in turn and then puts it through a match expression. If the byte matches one of the two packet IDs that I have seen so far, it executes a block which is empty for now but will call the relevant handler function once implemented. For any other byte it uses `continue` to proceed to the next byte.

Next I took a deeper look at the two packets and implemented the handler functions for each. They will need to extract the correct number of bytes for the particular packet and process them.

### The "Encrypted Login Seed" packet

Here is the full set of 21 bytes that correspond to this packet:

```
    0xEF, 0x01, 0x00, 0x00, 0x7F, 0x00, 0x00, 0x00, 0x07, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x63, 0x00, 0x00, 0x00,
    0x01
```

The packet guide says the packet should contain a `u32` "encryption seed" which should be used in future encrypted login packets. Looking at the ServUO code that handles this packet, I could also see that after that `u32` there were a further four `u32` values read which contain information on the game client version. I wrote a handler and `read_u32` helper function to extract these bytes, printing them to stdout for now:

```rust
// src/tcp.rs

// snip

fn parse_packets(buffer: [u8; 1024]) {
    let mut buffer_slice = &buffer[..];

    while buffer_slice.len() > 0 {
        let packet_id = read_u8(&mut buffer_slice);

        match packet_id {
            0xEF => {
              handle_encrypted_login_seed_packet(&mut buffer_slice);
            },
            // snip
        }
    }
}

fn handle_encrypted_login_seed_packet(buffer_slice: &mut &[u8]) {
    let packet_length = 20;
    let (mut bytes, rest) = buffer_slice.split_at(packet_length);
    *buffer_slice = rest;
    let seed = read_u32(&mut bytes);
    println!("seed: {}", seed);
    let major = read_u32(&mut bytes);
    let minor = read_u32(&mut bytes);
    let revision = read_u32(&mut bytes);
    let patch = read_u32(&mut bytes);
    println!("client version: {}.{}.{}.{}", major, minor, revision, patch);
}

fn read_u32(input: &mut &[u8]) -> u32 {
    let (int_bytes, rest) = input.split_at(4);
    *input = rest;
    u32::from_be_bytes(int_bytes.try_into().unwrap())
}

// snip
```

After reading the `0xEF` packet ID byte and the five `u32` values for the login seed packet, the buffer slice is positioned at the start of the next packet, the account login request packet (packet ID `0x80`).

### The "Account Login Request" packet

Here are the full 62 bytes received for this packet:

```
    0x80, 0x61, 0x64, 0x6D, 0x69, 0x6E, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x61, 0x64, 0x6D, 0x69, 0x6E, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0xFF
```

The packet guide said that after the packet ID, there are two 30 character strings where each character is a byte (i.e. ASCII).

Following the same pattern as for the previous packet I added a handler and a helper function to pull out the strings:

```rust
// src/tcp.rs

// snip

fn parse_packets(buffer: [u8; 1024]) {
    //snip

    while buffer_slice.len() > 0 {
        // snip

        match packet_id {
            // snip
            0x80 => {
                handle_account_login_request_packet(&mut buffer_slice);
            },
            // snip
        }
    }
}

fn handle_account_login_request_packet(buffer_slice: &mut &[u8]) {
    let packet_length = 61;
    let (mut bytes, rest) = buffer_slice.split_at(packet_length);
    *buffer_slice = rest;
    let username = read_string(&mut bytes, 30);
    println!("username: {}", username);
    let password = read_string(&mut bytes, 30);
    println!("password: {}", password);
}

fn read_string<'a>(input: &'a mut &[u8], length: u8) -> &'a str {
    let (string_bytes, rest) = input.split_at(length.into());
    *input = rest;
    return str::from_utf8(string_bytes).unwrap();
}
```

Using `str::from_utf8` is fine here because ASCII characters are valid UTF-8.

The lifetime parameter is needed on `read_string()` because the compiler does not know which of the two arguments' lifetimes the return value is related to. The lifetime declaration added tells it that it relates to the first argument's.

### Next steps

At this point the game client was stuck because it was expecting packets sent back:

![rust_uo_server_pt5_1.png](/assets/rust_uo_server_pt5_1.png)

Next I'll be looking at working out what these should be and adding the code to send them.
