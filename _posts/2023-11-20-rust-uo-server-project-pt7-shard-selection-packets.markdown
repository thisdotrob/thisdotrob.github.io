---
layout: post
title:  "Rust UO server project pt7: Receiving and responding to shard selection packets"
date:   2023-11-20 18:00:00 +0000
tags: ["UO server project", "Rust"]
---

I've been working on completing the login and shard selection flow in my Ultima Online server implementation. In the last journal post I got to the point of receiving a couple of login related packets from the game client and responding with a server list packet which the game client displayed.

Next I needed to add code to the server to handle the packets sent by the client when the player selected a server.

### Parsing the "Server Select" packet

This packet is sent from the game client to the server when the player clicks on a server in the server list.

It's a short packet, only 3 bytes long, as it just contains the packet ID (`0x0A`) and a 16 bit integer for the index of the server selected. Here's the code I added to handle it, following the same pattern of parsing and printing as for previous incoming packets:

```rust
// src/tcp.rs

// snip

fn parse_packets(buffer: [u8; 1024], mut stream: &mut TcpStream) {
    //snip

    while buffer_slice.len() > 0 {
        // snip

        match packet_id {
            // snip
            0xA0 => {
                handle_server_select_packet(&mut buffer_slice);
            },
            // snip
        }
    }
}

fn handle_server_select_packet(buffer_slice: &mut &[u8]) {
    let packet_length = 2;
    let (mut bytes, rest) = buffer_slice.split_at(packet_length);
    *buffer_slice = rest;
    let server_index = read_u16(&mut bytes);
    println!("server_index: {}", server_index);
}
```

For now I just want to hack something together that will get all the way through the login flow so I'm not doing anything with the server index from the packet. There's only one server to chose from anyway so it will always be `0`.

`read_u16()` needed to be implemented for the above:

```rust
// src/tcp.rs

// snip

fn read_u16(input: &mut &[u8]) -> u16 {
    let (int_bytes, rest) = input.split_at(2);
    *input = rest;
    u16::from_be_bytes(int_bytes.try_into().unwrap())
}
```

This follows a similar pattern to the `read_u8()` function covered in a previous journal post, but takes advantage of the `from_be_bytes()` function from the standard library.

The `try_into()` is necessary because `from_be_bytes()` expects a `[u8; 2]` but `int_bytes` is a `&[u8]`.

Attempting to use `try_into()` with `from_be_bytes()` on a slice that is longer than two bytes long *would* panic:

```rust
let (int_bytes, rest) = input.split_at(3); // <-- splitting at 3 instead of 2

// snip

u16::from_be_bytes(int_bytes.try_into().unwrap())
// thread 'main' panicked at 'called `Result::unwrap()` on
// an `Err` value: TryFromSliceError(())',
```

I know this will not panic because the `input.split_at(2)` will give a slice that is two bytes long. But I changed the `unwrap()` to an `expect()` so a message could be provided with the necessary context:

```rust
u16::from_be_bytes(
    int_bytes
        .try_into()
        .expect("int_bytes should always be two bytes long"),
)
```

### Responding to the "Server Select" packet

The response needed to be a "Server Redirect" packet (`0x8C`):

```rust
// src/tcp.rs

// snip

fn parse_packets(buffer: [u8; 1024], mut stream: &mut TcpStream) {
    //snip

    while buffer_slice.len() > 0 {
        // snip

        match packet_id {
            // snip
            0xA0 => {
                // snip

                send_server_redirect_packet(&mut stream);
            },
            // snip
        }
    }
}

fn send_server_redirect_packet(stream: &mut TcpStream) {
    let mut buffer: [u8; 11] = [0; 11];

    buffer[0] = 0x8C; // packet ID

    // server address
    buffer[1] = 0x7F; // 127;
    buffer[2] = 0x00; // 0;
    buffer[3] = 0x00; // 0;
    buffer[4] = 0x01; // 1;

    // server port
    buffer[5] = 0x0A; // 10;
    buffer[6] = 0x21; // 33;

    // encryption key
    buffer[7] = 0x43;
    buffer[8] = 0x2F;
    buffer[9] = 0x3F;
    buffer[10] = 0xF0;

    stream.write_all(&buffer).unwrap();
    stream.flush().unwrap();
}
```

Interestingly the server address needed to be sent as big endian here, unlike in the earlier "Server List" packet covered in a previous journal post, which had it sent as little endian.

The encryption key, according to the [packet guide](https://necrotoolz.sourceforge.net/kairpacketguide/packet8c.htm), is *"The gameplay encryption key. This is usually the same as the account number."* For now I hardcoded it to the value in a sample of this packet I found in Wireshark when running the client against ServUO. I think this key will be sent back and forth with future packets to check their authenticity.

After sending this packet, the client responds with a ["Post Login"](https://necrotoolz.sourceforge.net/kairpacketguide/packet91.htm) packet.

### Parsing and responding to the "Post Login" packet

Following the same pattern as for previous packets I added the code to parse the packet:

```rust
// src/tcp.rs

// snip

fn parse_packets(buffer: [u8; 1024], mut stream: &mut TcpStream) {
    //snip

    while buffer_slice.len() > 0 {
        // snip

        match packet_id {
            // snip
            0x91 => {
                handle_post_login_packet(&mut buffer_slice);
            }

            // snip
        }
    }
}

fn handle_post_login_packet(buffer_slice: &mut &[u8]) {
    let packet_length = 64;
    let (mut bytes, rest) = buffer_slice.split_at(packet_length);
    *buffer_slice = rest;
    let encryption_key = read_u32(&mut bytes);
    print!("encryption_key: {}, ", encryption_key);
    let username = read_string(&mut bytes, 30);
    print!("username: {}, ", username);
    let password = read_string(&mut bytes, 30);
    println!("password: {}", password);
}
```

As anticipated, the encryption key is being sent back by the client. The username and password would need to be checked again here, but for now I'm hacking my way through and ignoring that logic.

`read_32()` needed adding, being almost identical to `read_16()`:

```rust
// src/tcp.rs

// snip

fn read_u32(input: &mut &[u8]) -> u32 {
    let (int_bytes, rest) = input.split_at(4);
    *input = rest;
    u32::from_be_bytes(
        int_bytes
            .try_into()
            .expect("int_bytes should always be four bytes long"),
    )
}

```

Checking what happened in wireshark I could see that two packets needed to be sent in response to this, a "Features" packet and a "Character List" packet:

```rust
// src/tcp.rs

// snip

fn parse_packets(buffer: [u8; 1024], mut stream: &mut TcpStream) {
    //snip

    while buffer_slice.len() > 0 {
        // snip

        match packet_id {
            // snip
            0x91 => {
                // snip
                send_features_packet(&mut stream);
                send_character_list_packet(&mut stream);
            }

            // snip
        }
    }
}

fn send_features_packet(stream: &mut TcpStream) {
    let mut buffer: [u8; 3] = [0; 3];

    buffer[0] = 0xB9; // packet ID

    // flags
    buffer[1] = 0x00;
    buffer[2] = 0x00;

    stream.write_all(&buffer).unwrap();
    stream.flush().unwrap();
}

fn send_character_list_packet(stream: &mut TcpStream) {
    let mut buffer: [u8; 6] = [0; 6];

    buffer[0] = 0xA9; // packet ID
                      //
    buffer[1] = 6; // packet size

    buffer[2] = 0x00; // number of characters

    buffer[3] = 0x00; // number of cities

    // flags
    buffer[4] = 0x00;
    buffer[5] = 0x00;

    stream.write_all(&buffer).unwrap();
    stream.flush().unwrap();
}
```

Looking at the [packet guide](https://necrotoolz.sourceforge.net/kairpacketguide/packetb9.htm) for the "Features" packet, I saw that the single `u16` it contains relates to certain optional game features a server can have enabled. For now I set it to `0x00` for `None` to keep things simple. Each bit in the `u16` relates to a single flag (1 = enabled, 0 = off), so multiple can be set at once.

I also hardcoded most of the "Character List" packet values to `0x00` which corresponds to an account with no characters created yet. In the future this will need to return details of existing characters.

Unfortunately, sending these two packets to the client did not allow it to progress, and it hung on the following screen:

![rust_uo_server_pt7_0.png](/assets/rust_uo_server_pt7_0.png)

After comparing the response packets my server was sending to the ones sent by ServUO I realised they were completely different. Digging in to the ServUO and ClassicUO code revealed that starting with these two packets, all packets sent from the server are compressed. Just when I thought I'd found a flow and could blitz through implementing more packets quickly I'd hit a big hurdle.

### Next steps

I'll be taking a look at the compression algorithm and writing my own implementation of it. Another issue that needs resolving is that the TCP logic is blocking at the moment, so multiple connections to the server are not possible. I'll be spending some time thinking about how to fix this too.
