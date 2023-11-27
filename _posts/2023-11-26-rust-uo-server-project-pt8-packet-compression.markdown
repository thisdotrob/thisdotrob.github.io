---
layout: post
title:  "Rust UO server project pt8: Packet compression"
date:   2023-11-26 18:00:00 +0000
tags: ["UO server project", "Rust"]
---

As mentioned in my [last journal post]({% post_url 2023-11-20-rust-uo-server-project-pt7-shard-selection-packets %}), I had communication between the client and my Rust Ultima Online server working for the login flow up until the point that a shard was selected by the player. The packets sent from the server to the client after this point need to be compressed, and I hadn't implemented that yet.

Nothing is mentioned in the packet guide about compression, and after doing a decent amount of trawling through forum posts I couldn't find any information on who originally implemented it, if it wasn't the original game developers or how it was reverse engineered if it was.

All I had to go on was the compression and decompression logic in the ServUO server code and ClassicUO client, so I spent some time reading through it. I found that these open source Ultima Online server and client implementations use Huffman coding to compress packets. Compression is only performed on packets sent from the server to the client, not the other way around. The server has the compression logic only and the client only has decompression logic.

Why compression is only performed in one direction I am not sure, and couldn't find an answer to.

### Huffman coding - how it works

Huffman coding is a method for lossless data compression. Ultima Online uses it to compress each byte in a packet, so the following uses bytes as an example. Huffman coding can be used on other types of data e.g. whole words too but the focus here is on how it is used in the game.

For each of the 256 possible bytes a packet can include, a unique value of bits is assigned. These values are looked up for each byte in the original packet and written in their place to construct the compressed packet.

Shorter unique bit values are assigned to the more frequently seen input bytes and longer ones to less frequent bytes. The overall length of the encoded data should therefore be shorter, vs the uncompressed packets where every byte takes up 8 bits.

The client code is able to decompress the packet because the bit values are unique and so walking left to right through the compressed data it can determine what the original bytes were. For example, here's how the 4 most common bytes in Ultima Online packets are encoded:

```
| original byte | original bits | compressed value |
|---------------|---------------|------------------|
|      0x00     |    00000000   |       00         |
|      0x01     |    00000001   |       11111      |
|      0x40     |    01000000   |       01010      |
|      0x03     |    00000011   |       100010     |
```

An uncompressed packet containing `[0x01, 0x03, 0x00, 0x40, 0x03]` would therefore be compressed to the following stream of bits (underscores to distinguish each compressed value):

```
11111_100010_00_01010_100010
```

Total length of the packet has been reduced from 5 x 8 = 40 bits to 24 bits.

Rearranging into bytes gives the compressed packet:

```
[11111100, 01000010, 10100010]
```

or in hex:

```
[0xFC, 0x42, 0xA2]
```

To decompress, the receiving code needs to traverse a pre constructed tree built according to the same encodings as the data was compressed with:

```
          start
       _____|_____
      /           \
     0             1
    / \           / \
   /   \         0   1
0x00    1       /     \
       /       0       1
      0       /         \
       \     0           1
        1     \           \
       /       1          0x01
      /       /
   0x40    0x03
```

Starting from the top, the code should take each of the compressed bits one by one and go left (0) or right (1) depending on its value.

In the compressed packet from the example above the first bit is 1, so the code should go *right*. The next 4 bits are also 1s, so the code would continue branching right until it reached the leaf node of `0x01`. This is the original uncompressed byte that corresponds to these first five 1 bits.

It would then write this decompressed byte and carry on to the next bit, starting from the top of the tree again.

### Ultima Online's Huffman codes

The specific bit codes used by modern Ultima Online servers and clients can be seen in the `_huffmanTable` array variable in [this ServUO file](https://github.com/ServUO/ServUO/blob/c11047d380248e014a63c54648171cc2890423cf/Server/Network/Compression.cs#L13).

The array contains the bit codes and the number of bits required to write them, side by side. The bits required to write them (the bit counts) must be provided because codes that start with zeroes would otherwise have these zeroes stripped. The compression algorithm left shifts the compressed bits by the required bit count before writing each value, making sure that leading zeroes are preserved.

To find the correct bit code and bit count for an uncompressed byte, the algorithm uses the uncompressed byte as an index into the array (adding 1 to the index to get the bit code that sits next to the bit count).

The compression code uses unsafe code and `fixed` statements to keep the array fixed in memory and avoid garbage collection slowing things down.

I couldn't find any information on how the Huffman table was arrived at. Presumably it was based on analysis of real packet data sent between game clients to work out which packets were most common, with an optimised tree generated from this. It seems like this same Huffman table has been used by various open source Ultima Online projects since the early 00's, so it would be interesting to find out if a more efficient table could be generated by analysing a sample of the real packet data for the game as it is played today. Or even switching out the compression method completely.

Plotting out the distribution of bit counts by original uncompressed byte showed the following:

![rust_uo_server_pt8_0.png](/assets/rust_uo_server_pt8_0.png)

The distribution is heavily skewed towards only a small number of bytes actually being compressed i.e. with bit counts less than 8.

Here's the distribution showing only the 59 out of 256 bytes with 8 or less bit counts:

![rust_uo_server_pt8_1.png](/assets/rust_uo_server_pt8_1.png)

For this to be the optimum distribution there must be a pretty high frequency of those 3 bytes with < 6 bit count. The trade off here is that by allowing bytes to have very low bit counts (like 0x00 having a bit count of 2), you are losing a large section of the binary tree immediately below it, that could otherwise provide relatively low bit counts for other bytes. Those 3 bytes are 0x00, 0x40 (64) and 0x01. It makes sense that there would be lots of 0x00 bytes in UO packets - values are often padded with them e.g. where a character name is included in a packet there might be 30 bytes in the packet allocated but very few character names would use the full allocation. Likewise with 0x01 packets, there are probably lots of these bytes in packets because they signal an "on" or "enabled" state. The inverse is also true for 0x00, they will be used to signal "off" or "disabled" and so there's another reason they would be more frequent. Why 0x40 would be a high frequency byte I'm not so sure.

### Next

Now I've understood how packet compression works in Ultima Online, I'm going to write my own implementation of it from scratch in Rust for use in the UO server I am developing. I'm sure there are existing crates out there that can be used to perform Huffman compression, but writing it from scratch should be a fun challenge and I can tailor it exactly to the use case in an Ultima Online server.
