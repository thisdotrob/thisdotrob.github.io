---
layout: post
title:  "Rust UO server project pt9: Packet compression implementation"
date:   2023-11-26 18:00:00 +0000
tags: ["UO server project", "Rust"]
---

I've spent the last day implementing the Huffman coding compression logic I needed for my Ultima Online server project. This logic will be used to compress outgoing packets from the server to the client, after the player has selected the shard they want to play on.

This is the first part of my server implementation I have built for which I'm confident that the design will stick and no *big* changes will be needed later on. For that reason I spent a decent amount of time writing unit tests, and an integration test that verifies it will work for the purposes of an Ultima Online server. I opted not to write tests for the other parts of the server I've written so far, since I've been hacking away and experimenting. Manual testing is much faster for that kind of workflow, but I'll be going back and adding tests when a more final design has evolved and all the shortcuts in terms of skipped functionality need addressing.

I chose to implement the logic as a separate crate that can be used to compress bytes for any project, not just an Ultima Online server. The full code is on GitHub: [thisdotrob/rust_huffman_compression](https://github.com/thisdotrob/rust_huffman_compression). I may end up making it Ultima Online specific but still keep it as a crate. Perhaps this way others wanting to build Ultima Online servers or clients will be able to use it too. This has got me thinking what other components of the server could be written as crates for others to use modularly.

## Using the crate

This is mostly lifted from the crate's readme...

### Getting started

First construct a `HuffmanTable` which represents the encoding rules:

```rust
let values: [u32; 256] = [
    0b1111, 0b0111, 0b1011, 0b110,
    // snip
];

let bit_counts: [u8; 256] = [
    4, 4, 4, 4,
    // snip
];

let table = HuffmanTable { values, bit_counts };
```

`values` are the compressed bits that should be written for corresponding uncompressed bytes. `bit_counts` are the number of bits that should be written for each encoded value. The correct value and bit_count for each uncompressed byte are looked up by using that byte as an index to look up the elements in the two arrays.

For example, given an uncompressed byte of 0x03:
  - The value, `0b110`, lies at index 3 (i.e. `0x03`) of `values`
  - The bit count, `4`,  lies at index 3 (again, `0x03`) of `bit_counts`
  - The final compressed bits are the value after is has been left padded with 0s until the bit_count is reached. In this case that means `0110` is written.

The example above shows only the first 4 elements for each array but in reality you will need to populate all 256.

Next create a `Huffman`, passing it the table:

```rust
let mut huffman = Huffman::new(table, None); // <-- the None is for the termination code, see further down
```

Now, send bytes to be compressed and an output vec for the compressed bits to be pushed to:

```rust
let uncompressed_bytes = vec![0x00, 0x01, 0x02, 0x03];

let mut output = Vec::new();

huffman.compress(uncompressed_bytes, &mut output);
```

Now we can see that `output` has been populated with the compressed bits, separated into bytes: 

```rust
// as binary with underscores separating the original compressed bit groupings:
assert_eq!(output, vec![0b1111_0111, 0b1011_0110]);

// or as hex:
assert_eq!(output, vec![0xF7, 0xB6]);
```

### Byte boundaries and termination codes

If the compressed bits do not align with a byte boundary like they do in the example above, the crate will pad with zeroes:

```rust
// using the same table as in the previous example

let uncompressed_bytes = [0x00, 0x01, 0x02];

huffman.compress(uncompressed_bytes, &mut output);

// the compressed bits will now be 0b1111_0111_1011. This is only one and a half bytes, so
// four zeroes are added to the end to make up to the next byte boundary:
assert_eq!(output, vec![0b1111_0111, 0b1011_0000]);
```

This is fine so long as the zeroes added for padding do not clash with one of the compressed values for a byte. For example if a 0x04 byte had a compressed value of `0b00` and bit count of 2, the last four bits in the example above would decompress to two 0x04 bytes which is not what we want.

To avoid this, it is possible to append a termination code to the compressed bits which will signal to the consuming code that any bits following do not represent compressed bytes and should be ignored:

```rust
// this time, set up the Huffman with a TerminalCode:

let terminal_code = TerminalCode { value: 0b001, bit_count: 3 };
let huffman = Huffman::new(table, Some(terminal_code));

// compress as normal:

let uncompressed_bytes = [0x00, 0x01, 0x02];
huffman.compress(uncompressed_bytes, &mut output);

// now the termination code is appended to the output before padding with zeroes:

assert_eq!(output, vec![0b1111_0111, 0b1011_001_0]);
//                                           ^
//                                  termination code
```

## Code walkthrough

The entrypoint to the crate is the `Huffman` struct:

```rust
// src/lib.rs

pub struct Huffman {
    pub table: HuffmanTable,
    pub terminal_code: Option<TerminalCode>,
}

impl Huffman {
    pub fn new(table: HuffmanTable, terminal_code: Option<TerminalCode>) -> Huffman {
        return Huffman {
            terminal_code,
            table,
        };
    }

    // snip
}
```

The initialiser takes a `HuffmanTable` containing the compressed bit values and their corresponding bit counts, and an optional `TerminalCode`, which will be appended once the last compressed bit is written.

A `Huffman` has a single `compress()` method:

```rust
// src/lib.rs

impl Huffman {

    // snip

    pub fn compress(&mut self, src: Vec<u8>, output: &mut Vec<u8>) {
        let mut compressor = Compressor::new(&self.table);

        for byte in src {
            compressor.compress_byte(byte);

            for compressed_byte in &mut compressor {
                output.push(compressed_byte);
            }
        }

        if let Some(terminal_code) = &self.terminal_code {
            compressor.append_terminal_code(terminal_code);
        }

        compressor.end();

        for compressed_byte in &mut compressor {
            output.push(compressed_byte);
        }
    }
}
```

`compress()` takes a `src` of uncompressed bytes and a `Vec<u8>` to put the compressed `output` in. It iterates over the uncompressed bytes and uses the `compress_byte()` method on a `Compressor` struct instance to compress it. The `TerminalCode` is appended if provided, once all bytes have been compressed. To align on the byte boundary, `end()` is called. Any extra compressed bits written during aligning are then added to the `output`.

The `Compressor` struct takes a reference to the `HuffmanTable` and initialises a new `CompressorBuffer` which is used to build up the compressed bits:

```rust
// src/compressor.rs

pub struct Compressor<'a> {
    table: &'a HuffmanTable,
    buffer: CompressorBuffer,
}

impl<'a> Compressor<'a> {
    pub fn new(table: &'a HuffmanTable) -> Self {
        Compressor {
            table,
            buffer: CompressorBuffer::new(),
        }
    }

    // snip
}
```

The lifetime parameter `'a` signals that the `HuffmanTable` reference needs to live at least as long as the struct itself.

The `compress_byte()` method on `Compressor` takes an uncompressed byte, looks up the compressed bit count and value and passes them to the buffer to write.

```rust
// src/compressor.rs

impl<'a> Compressor<'a> {
    // snip

    pub fn compress_byte(&mut self, byte: u8) {
        let value = self.table.get_compressed_value(byte);
        let bit_count = self.table.get_compressed_value_bit_count(byte);
        self.buffer.write_bits(value, bit_count);
    }

    // snip
}
```

`Compressor` has two other methods that write to the buffer:

```rust
// src/compressor.rs

impl<'a> Compressor<'a> {
    // snip

    pub fn append_terminal_code(&mut self, terminal_code: &TerminalCode) {
        self.buffer
            .write_bits(terminal_code.value, terminal_code.bit_count);
    }

    pub fn end(&mut self) {
        let byte_boundary_offset = self.buffer.byte_boundary_offset();

        if byte_boundary_offset != 0 {
            let padding_value = 0b0;
            let padding_bit_count = 8 - byte_boundary_offset;
            self.buffer.write_bits(padding_value, padding_bit_count);
        }
    }

    // snip
}
```

`append_terminal_code()` takes a `TerminalCode` and passes it's compressed value and bit count to the buffer for writing. `end()` asks the buffer for the current offset from the byte boundary and asks it to write zeroes up to the next boundary so that a full last byte can be read. This logic should be the responsibility of the buffer itself, something I'll refactor later.

After bits have been written to the buffer, full compressed bytes can be read with the `get_compressed_byte()` method:

```rust
// src/compressor.rs

impl<'a> Compressor<'a> {
    // snip

    fn get_compressed_byte(&mut self) -> Option<u8> {
        self.buffer.read_byte()
    }
}
```

This will return `Option::None` if there are no full bytes of compressed bits left to be read.

The `Compressor` struct doesn't seem to do much more than call out to methods on `CompressorBuffer`, but it is a useful layer to abstract over the buffer API and is also where the `Iterator` trait is implemented so that compressed bytes can be read in a `for` loop:

```rust
// src/compressor.rs

// snip

impl<'a> Iterator for Compressor<'a> {
    type Item = u8;

    fn next(&mut self) -> Option<u8> {
        self.get_compressed_byte()
    }
}
```

Lastly, the buffer itself, `CompressorBuffer`:

```rust
// src/compressor/buffer.rs

pub struct CompressorBuffer {
    compressed_bits: u32,
    compressed_bit_count: u8,
}

impl CompressorBuffer {
    pub fn new() -> Self {
        Self {
            compressed_bits: 0,
            compressed_bit_count: 0,
        }
    }

    // snip
}
```

The initialiser sets two private properties to 0: `compressed_bits` stores the sequence of actual compressed values that haven't been read yet and `compressed_bit_count` keeps track of how many compressed bits have been written but not read yet.

`write_bits()` takes a compressed value and a bit count for that value (both previously looked up in the `HuffmanTable`), left shifts `compressed_bits` and appends the value and then increments `compressed_bit_count` by the value's bit count.

```rust
// src/compressor/buffer.rs

impl CompressorBuffer {
    // snip

    pub fn write_bits(&mut self, value: u32, bit_count: u8) {
        self.compressed_bits = self.compressed_bits << bit_count;
        self.compressed_bits = self.compressed_bits | value;
        self.compressed_bit_count = self.compressed_bit_count + bit_count;
    }

    // snip
}
```

`read_byte()` returns `Option::None` if there are less than a byte's worth of compressed values. Otherwise it removes and returns a byte by right shifting and applying a mask to leave only the first 8 bits in the compressed sequence, leaving the remainder there:

```rust
// src/compressor/buffer.rs

impl CompressorBuffer {
    // snip

    pub fn read_byte(&mut self) -> Option<u8> {
        if self.compressed_bit_count < 8 {
            return None;
        }

        self.compressed_bit_count = self.compressed_bit_count - 8;

        let byte = self.compressed_bits >> self.compressed_bit_count;

        let mask = if self.compressed_bit_count > 0 {
            u32::MAX >> (32 - self.compressed_bit_count)
        } else {
            0
        };

        self.compressed_bits = self.compressed_bits & mask;

        Some(byte as u8)
    }
}
```

The internal buffer (`compressed_bits`) is implemented as a `u32`. This means that if the compressed values written to it exceed 32 bits the program will panic. This is okay since I have control over the calling code (in `Compressor`) which always attempts to read a byte in between each write of a compressed value. A single compressed value cannot exceed 32 bits since they are stored in `HuffmanTable`'s `values` array which is declared to take only `u32` elements:

```rust
// src/huffman_table.rs

pub struct HuffmanTable {
    pub values: [u32; 256],

    // snip
}
```

If the `CompressorBuffer` needed to be used elsewhere at some point, in a place where it couldn't be guaranteed bytes would be read often enough to prevent the overflow described above, then I'd need to consider approaches to allowing the internal buffer to grow dynamically.

## Performance

I'm looking forward to benchmarking this code and seeing how it stacks up with the ServUO implementation. I'll return to this in the future as I'm keen to use it in my server ASAP, see if I can progress past the login flow and get a character appearing in game.

## Next steps

Use this compression lib in the server! This should allow it to progress past the shard selection screen and allow implementing the next packets which are for character creation and selection.

To see the full code for the crate, see the github repo: [thisdotrob/rust_huffman_compression](https://github.com/thisdotrob/rust_huffman_compression/)
