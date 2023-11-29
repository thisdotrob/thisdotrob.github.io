---
layout: post
title:  "Rust UO server project pt10: Non-blocking connection handling"
date:   2023-11-29 14:00:00 +0000
tags: ["UO server project", "Rust"]
---

I spent yesterday exploring different approaches to making the connection handling in my Ultima Online server non-blocking.

Before making any changes, the code in `main.rs` looked this like:

```rust
// src/main.rs

// snip
mod tcp;

fn main() {
    // snip

    tcp::start();
}
```

And in the `tcp` module:

```rust
// src/tcp.rs

use byteorder::{BigEndian, ByteOrder};
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;
use std::str;

pub fn start() {
    let listener = TcpListener::bind("127.0.0.1:2593").unwrap();
    for stream in listener.incoming() {
        let mut stream = stream.unwrap();
        let mut buffer = [0; 1024];
        let mut num_bytes_received = stream.read(&mut buffer).unwrap();
        while num_bytes_received > 0 {
            parse_packets(buffer, &mut stream);
            buffer = [0; 1024];
            num_bytes_received = stream.read(&mut buffer).unwrap();
        }
        // the connection has been closed by the client
        // if num_bytes_received == 0
    }
}

// snip
```

The issue with this is that only one client at a time can be connected. Because `stream.read()` is blocking, the thread can not get any more connections from the `listener.incoming()` iterator until the current connection closes. 

This is obviously no good as there's not much point in a MMORPG which can only be played by one player.

To allow the multiple long lived connections necessary, the server needed to do two things concurrently:

1. Listen for new connections and accept them

2. Repeatedly call `read()` on existing connections to receive packets and respond to them

### Non-blocking approach one: two threads and a Mutex

With this approach the listening and accepting of new connections happens in a separate thread. When a connection is received it is added to a `Mutex<Vec<TcpStream>>` which is shared with the other thread. The other thread iterates over this shared vec and reads/writes to each connection.

My first attempt at implementing this looked like this:

```rust
// src/tcp.rs

fn bind(connections: Arc<Mutex<Vec<TcpStream>>>) {
    thread::spawn(move || {
        let listener = TcpListener::bind("127.0.0.1:2593").unwrap();

        for stream in listener.incoming() {
            let mut connections = connections.lock().unwrap();
            let stream = stream.unwrap();
            let addr = stream.peer_addr().unwrap();
            println!("Connection received from: {}", addr);
            connections.push(stream);
        }
    });
}

pub fn start() {
    let connections: Vec<TcpStream> = vec![];

    let connections = Arc::new(Mutex::new(connections));

    bind(Arc::clone(&connections));

    loop {
        thread::sleep(Duration::from_millis(1));

        let mut connections = connections.lock().unwrap();

        let mut next_connections = vec![];

        while let Some(mut stream) = connections.pop() {
            let mut buffer = [0; 1024];

            let num_bytes_read = stream.read(&mut buffer).expect("Reading from stream should not error");

            if num_bytes_read == 0 {
                let addr = stream.peer_addr().unwrap();
                println!("Connection closed by: {}", addr);
            } else {
                parse_packets(buffer, &mut stream);
                next_connections.push(stream);
            }
        }

        *connections = next_connections;
    }
}
```

The `thread::sleep()` call means that for at least 1ms the lock over `connections` will be acquirable by the listener thread. This can be reduced but for the purposes of experimenting I didn't think it necessary. So for a duration of 1ms any waiting connections are accepted by the listener thread. After 1ms the other thread resumes, takes each existing connection (including any just added) and parses and responds to any packets they have received. As soon as each connection has been read once and responded to the loop restarts and the listener thread checks for new connections again.

However this still didn't allow multiple connections, because there is still a blocking call to `stream.read()`. The call blocks the server when the game client is waiting on the player to do something before sending the next packets. No bytes are being sent but the connection is stil open and so the default behaviour of `TcpStream` is in effect and the `read()` call blocks until the next bytes are sent. Other connections can not be accepted or read from until the client closes the connection.

To work around this I configured each `TcpStream` to have the smallest possible read timeout:

```rust
// src/tcp.rs

fn bind(connections: Arc<Mutex<Vec<TcpStream>>>) {
    thread::spawn(move || {
        // snip

        for stream in listener.incoming() {
            //snip

            let stream = stream.unwrap();
            stream
                .set_read_timeout(Some(Duration::from_nanos(1)))
                .expect("set_read_timeout call failed");

            // snip
        }
    });
}
```

Now `read()` will only block for 1 nanosecond before either returning a `Result::Ok` containing `num_bytes_read` or a `Result::Err` of the `WouldBlock` kind. The rest of the code needed updating as below to handle this different behaviour:

```rust
pub fn start() {
    // snip

    loop {
        // snip

        while let Some(mut stream) = connections.pop() {
            // snip

            let read_result = stream.read(&mut buffer);

            match read_result {
                Ok(num_bytes_read) => {
                    if num_bytes_read == 0 {
                        let addr = stream.peer_addr().unwrap();
                        println!("Connection closed by: {}", addr);
                    } else {
                        parse_packets(buffer, &mut stream);
                        next_connections.push(stream);
                    }
                }
                Err(e) => match e.kind() {
                    WouldBlock => next_connections.push(stream),
                    _ => println!("Unexpected error from stream.read(): {:?}", e),
                },
            }
        }

        // snip
    }
}
```

If a `Result::Err` of the `WouldBlock` kind is returned from `stream.read()`, it indicates that the connection is still alive and well but no bytes have been sent by the client since the last read. The connection is pushed to `next_connections` in this case so that it can be checked again on the next iteration of the loop.

If any other type of `Result::Err` is returned, the error is printed and the connection is *not* added to `next_connections` to be checked on the next loop iteration.

The full code for this approach can be seen in [this commit](https://github.com/thisdotrob/rust-uo-server/tree/98c3641177aa20a9802b4dd63c103b8aa88e4336).

I also refactored the code above slightly to use `retain_mut()` on the `connections` vec to avoid needing a separate `next_connections` vec - see [this commit](https://github.com/thisdotrob/rust-uo-server/tree/1af0e99931279eee34f5f785ec8a07f79ea883ac).

One potential risk I can see with this approach is that anyone maliciously sending a significant amount of new connections to the server could make it hard for the second thread to acquire a lock on the connections vec, slowing down communication with existing clients. This could be fixed by adding throttling to the listener thread, which would introduce a delay that must happen before locking the vec and accepting a connection.

### Non-blocking approach two: one thread

With this approach I made the loop check once for new connections on each iteration and then proceed. In the previous approach the `TcpListener` was treated as an iterator when checking for new connections:

```rust
let listener = TcpListener::bind("127.0.0.1:2593").unwrap();

for stream in listener.incoming() {
    let stream = stream.unwrap();
    // do something with the new connection
}
```

This is blocking, preventing anything else happening after the for loop.

An alternative way of accepting new connections is to use the `accept()` method instead of treating the listener as an iterator:

```rust
while let Ok((stream, addr)) = listener.accept() {
    // do something with the new connection
}
```

This is still blocking however, as `accept()` blocks until a new connection is received.

To make it non-blocking, the `TcpListener` can be set to non-blocking mode:

```rust
let listener = TcpListener::bind("127.0.0.1:2593").unwrap();
listener
    .set_nonblocking(true)
    .expect("Cannot set non-blocking");
```

`accept()` will now not not block, and either return the new connection *or* a `Result::Err` of the `WouldBlock` kind.

After setting the listener to non blocking mode as above I was able to use a simple `if let` statement to check for new connections once per loop iteration:

```rust
// src/tcp.rs

pub fn start() {
    let listener = TcpListener::bind("127.0.0.1:2593").unwrap();
    listener
        .set_nonblocking(true)
        .expect("Cannot set non-blocking");

    let mut connections: Vec<TcpStream> = vec![];

    loop {
        thread::sleep(Duration::from_millis(1));

        if let Ok((stream, addr)) = listener.accept() {
            println!("Connection received from: {}", addr);
            stream
                .set_read_timeout(Some(Duration::from_nanos(1)))
                .expect("set_read_timeout call failed");
            connections.push(stream);
        }

        // iterate over connections and read from / write to them
    }
}
```

See [this commit](https://github.com/thisdotrob/rust-uo-server/tree/03042d8f53cf05f9f65de433189a95c946ac05ef) for the full code for this approach.

This approach doesn't risk being overloaded with connection requests since a maximum of one connection is accepted every 1 millisecond. But it might be slower because the check for new connections with `accept()` occurs on every iteration of the loop. The previous approach ran the checking in a separate thread and so only interrupted the other thread when a new connection had actually been received.

### Non-blocking approach three: using an async runtime

My third approach to non-blocking TCP connections was to use an async runtime to allow concurrent packet handling and connection accepting.

The runtime I chose was [async-std](https://github.com/async-rs/async-std), purely because that's what I have most familiarity with.

My async approach has two main async functions:

```rust
async fn connection_loop(mut stream: TcpStream) -> Result<()> {
    let mut buffer = [0; 1024];

    while let Ok(received) = stream.read(&mut buffer).await {
        if received == 0 {
            let addr = stream.peer_addr().unwrap();
            println!("Connection closed by: {}", addr);
            break;
        } else {
            parse_packets(buffer, &mut stream).await?;
            buffer = [0; 1024];
        }
    }
    Ok(())
}

async fn accept_loop(addr: impl ToSocketAddrs) -> Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let mut incoming = listener.incoming();
    while let Some(stream) = incoming.next().await {
        let stream = stream?;
        let addr = stream.peer_addr()?;
        println!("Connection received from: {}", addr);
        task::spawn(connection_loop(stream));
    }
    Ok(())
}

pub fn start() -> Result<()> {
    let fut = accept_loop("127.0.0.1:2593");
    task::block_on(fut)
}
```

`accept_loop()` uses the `async-std` version of `TcpListener` to asynchronously wait for new connections. Because there is no support for async for loops in Rust yet, a `while let Some()` is needed here. For each new connection received, an asychnronous task is spawned to start the `connection_loop()` function.

`connection_loop()` uses the `async-std` version of `TcpStream` to read bytes and pass them to `parse_packets()`. As before, the `TcpStream` is passed to `parse_packets()` so that it can be used to send responses to the client. Now that the async version of `TcpStream` is being passed, `parse_packets()` and the functions it calls need to be converted to async too:

```rust
async fn parse_packets(buffer: [u8; 1024], mut stream: &mut TcpStream) -> Result<()> {
    let mut buffer_slice = &buffer[..];

    while buffer_slice.len() > 0 {
        let packet_id = read_u8(&mut buffer_slice);

        match packet_id {
            // snip
            0x80 => {
                handle_account_login_request_packet(&mut buffer_slice);
                send_server_list_packet(&mut stream).await?; // <-- async function
            }
            // snip
            _ => continue,
        }
    }

    Ok(())
}

async fn send_server_list_packet(stream: &mut TcpStream) -> Result<()> {
    let mut buffer: [u8; 46] = [0; 46];

    buffer[0] = 0xA8; // packet ID

    buffer[2] = 0x2E; // packet length

    // snip

    stream.write_all(&buffer).await?; // <-- using the async-std version
    stream.flush().await?; // <------------|

    Ok(())
}
```

The `Result<()>` used for the return types in the code above is an alias:

```rust
type Result<T> = std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
```

The `Send` and `Sync` traits are needed in order to use `async-std`'s `task::spawn` because the async executor uses a thread pool behind the scenes. This tells the compiler that the `E` in the `Result<T, E>` is safe to be moved between threads, as `task::spawn` tells it it needs to be.

The full code for this approach is in [this commit](https://github.com/thisdotrob/rust-uo-server/tree/7eacf2f5e6499c35a1b1f1bfe84ca1d09874db45).

### Wrap up

I'm going to go with the third approach and use async. This is because it's best suited to a situation where there are lots of tasks and not so many threads available to be used, especially when time is typically spent waiting in each task. This is the situation I anticipate for the server given that there will be lots of connections and a limited number of threads available to the networking portion of it, and often the tasks will be waiting for bytes from the client. Instead of only having two threads as in the first approach or spinning up so many threads that the CPU spends a lot of time switching between them, async will allow keeping the number of threads to the optimal amount - within the number of virtual cpu cores available.
