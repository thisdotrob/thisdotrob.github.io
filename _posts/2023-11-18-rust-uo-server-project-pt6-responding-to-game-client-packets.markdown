---
layout: post
title:  "Rust UO server project pt6: Responding to game client packets"
date:   2023-11-18 18:00:00 +0000
tags: ["UO server project", "Rust"]
---

In my previous journal post I got to the point that my Ultima Online server implementation could receive the first two packets sent from the game client whilst logging in and parse them. The game client then hangs at this point because it is awaiting a response from the server.

This journal post covers how I updated the server code to send the correct responses, parse the subsequent replies from the client and carry on until the login flow is complete.

### The "Encrypted Login Seed" packet

This is the first packet sent by the client and does not require a response, hence why my server was receiving the next packet from the client without sending a response to it. What will need to happen is the seed included in this packet should be persisted in memory as it needs to be included in future packets sent between client and server. For now I skipped storing and using the seed as it is not needed in packets until after the login flow is complete.

### Responding to the "Account Login Request" packet

This packet does require a response and is why the game client was not progressing further. To figure out what should be sent back I ran the game client and ServUO and inspected packets in Wireshark. I found a packet sent back by ServUO with packet ID of `0xA8`. This is the ["Server List"](https://necrotoolz.sourceforge.net/kairpacketguide/packeta8.htm) packet detailed in this [packet guide](https://necrotoolz.sourceforge.net/kairpacketguide/).

Before delving in to creating the response packet, I needed to update the code to allow sending a response:

```rust
// src/tcp.rs

pub fn start() { // <-- called from the server's main() function
    // snip

    for stream in listener.incoming() {
        // snip
        stream.read(&mut buffer).unwrap();
        parse_packets(buffer, &mut stream);
    }
}

fn parse_packets(buffer: [u8; 1024], mut stream: &mut TcpStream) {
    //snip

    while buffer_slice.len() > 0 {
        // snip

        match packet_id {
            // snip
            0x80 => {
                handle_account_login_request_packet(&mut buffer_slice);
                send_server_list_packet(&mut stream);
            },
            // snip
        }
    }
}

fn send_server_list_packet(stream: &mut TcpStream) {
    let mut buffer: [u8;46] = [0;46];

    // Write the necessary bytes to the buffer...

    stream.write_all(&buffer).unwrap();
    stream.flush().unwrap();
}
```

The updated code above adds a second parameter to the `parse_packets()` function. This parameter is the `TcpStream` struct that needs to be used to send response packets on the connection. This struct is passed to a new `send_server_list_packet()` function which is called after the account login request packet has been handled. The `send_server_list_packet()` function constructs a new buffer and sends it to the game client using the `TcpStream` struct's `write_all()` method.

Next I needed to populate the buffer with the correct bytes per the packet guide. The packet is interesting because it involves looping and has a dynamic length. The 5th and 6th bytes in the packet need to be a `u16` for the number of servers in the list the packet contains. The number here determines how many bytes follow - for each server in the list a further 40 bytes need to be written. The number is given so that the game client knows how many bytes to read when the packet is received, and can apply the correct boundaries to work out which bytes are related to which server in the list.

For my initial purposes I only needed to provide a list of one server, so it was easiest to hardcode the bytes in the response instead of implementing any looping logic.

The first 2 bytes to write were simple:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    let mut buffer: [u8;46] = [0;46];

    buffer[0] = 0xA8; // packet ID

    buffer[2] = 0x2E; // packet length

    stream.write_all(&buffer).unwrap();
    stream.flush().unwrap();
}
```

The next byte was "Flags" according to the packet guide. Apparently this is "Server list flags." I could see that ServUO hardcodes this to an `0x5D` byte, but I wanted to understand what the flags were and how they were used so I looked at the source code for the game client I was testing with ([ClassicUO](https://github.com/ClassicUO/ClassicUO)).

I found the game client code that handles this packet and could see that it pulls out the "Flags" byte in [this line](https://github.com/ClassicUO/ClassicUO/blob/41922f4a33b4bc7768a61b168f2098bcfff6bd16/src/ClassicUO.Client/Game/Scenes/LoginScene.cs#L577C21-L577C21). It assigns the byte to a `flags` variable but then the variable is not used anywhere.

For now I set the value of this byte to be `0x00` given that it didn't seem to be used by the client:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    // snip

    buffer[3] = 0x00;

    // snip
}
```

The next data that needed to be written to the packet was the server count. I only wanted one server, so the value needed to be `1`, but written as a 16 bit integer. The packet guide didn't mention whether it should be written as big or little endian but from inspecting the packets in wireshark I found it should be big endian. In the future I'll bring in the [byteorder crate](https://crates.io/crates/byteorder) to make writing multi-byte integers like this more ergonomic, but for now I manually assigned each byte:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    // snip

    // server count = big endian '1'
    buffer[4] = 0x00;
    buffer[5] = 0x01;

    // snip
}
```

Next I needed to write the bytes for the server details for the only server in the list. The details are the server index, server name, percent full (of players) and server address. They need to be written in that order.

Server index is another 16 bit integer. Assuming the list is zero indexed I wrote 0:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    // snip

    // server index = big endian '0'
    buffer[6] = 0x00;
    buffer[7] = 0x00;

    // snip
}
```

Next was the server name, which needed to be 32 (ASCII) characters. The buffer was initialised with all 0 bytes, so I found calling `copy_from_slice()` on the range of bytes equalling the number of bytes that needed to be non-zero was easiest:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    // snip

    buffer[8..16].copy_from_slice("My Shard".as_bytes()); // server name

    // snip
}
```

The range `8..16` updates the 8 necessary non-zero bytes for the `"My Shard"` string. The remaining 24 bytes of the server name value are left at their initialised 0 values.

Next was the server full percentage, which for now I just hardcoded to 0% by leaving the byte at the relevant index to the already initialised 0 value.

I took the same approach for the next byte that needed writing, for timezone. I figured leaving it at 0 might set it to UTC which is what I wanted anyway. The packet guide says it should only be a single byte but from looking at the packets in wireshark I could see it actually needed to be 4 bytes, as the next info (the server address) didn't start until after that.

The last information needed to be written to the packet was the server address. The packet guide states it should be a 32 bit integer or 4 bytes. I assumed that each byte would correspond to one of the 4 parts of an ipv4 address. Assuming this would also be big endian given that earlier multi-byte integers were, I set it to the loopback address the server is running on: `[127, 0, 0, 1]`. This didn't work because as it turned out, the server address needed to be written as little endian. Again the byteorder crate would make this code cleaner but for now I hardcoded the bytes:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    // snip

    // server address
    buffer[42] = 0x01;
    buffer[43] = 0x00;
    buffer[44] = 0x00;
    buffer[45] = 0x7F;

    // snip
}
```

With that, the server list packet was complete and could be sent back. Here's the full code:

```rust
// src/tcp.rs

// snip

fn send_server_list_packet(stream: &mut TcpStream) {
    let mut buffer: [u8;46] = [0;46];

    buffer[0] = 0xA8; // packet ID

    buffer[2] = 0x2E; // packet length

    buffer[3] = 0x00; // flags (unused)

    // server count = big endian '1'
    buffer[4] = 0x00;
    buffer[5] = 0x01;

    // server index = big endian '0'
    buffer[6] = 0x00;
    buffer[7] = 0x00;

    // server name
    buffer[8..16].copy_from_slice("My Shard".as_bytes());

    buffer[37] = 0x00; // server percent full

    // server timezone
    buffer[38] = 0x00;
    buffer[39] = 0x00;
    buffer[40] = 0x00;
    buffer[41] = 0x00;

    // server address
    buffer[42] = 0x01;
    buffer[43] = 0x00;
    buffer[44] = 0x00;
    buffer[45] = 0x7F;

    stream.write_all(&buffer).unwrap();
    stream.flush().unwrap();
}
```

I ended up hardcoding the 0 value items anyway, even though they are initialised that way, since I thought it would be clearer later which bytes needed changing if I wanted to set them to non-zero values.

After receiving the packet above, the game client progresses to the next screen and waits for the player to select the shard they want to play on:

![rust_uo_server_pt6_0.png](/assets/rust_uo_server_pt6_0.png)

Step one in the login flow complete 🎉
