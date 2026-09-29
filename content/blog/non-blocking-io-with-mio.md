+++
title = "Non-Blocking I/O in Rust with Mio"
date = 2026-09-29
+++

## Introduction

`Mio` is a little crate I had to use recently and I tought "Why not write a blog post out of it?", so here we are.

Keep in mind that this blog post is mostly for presenting `Mio` in a "scrolling on the toilet" format.
For reading material to actually use while using the crate reffer to the [official documentation](https://docs.rs/mio/latest/mio/).

## What is `Mio`?

First of all what does `Mio` do? In `Mio`'s own words:

> Mio is a fast, low-level I/O library for Rust focusing on non-blocking APIs and event notification for
> building high performance I/O apps with as little overhead as possible over the OS abstractions.

This sounds cool, but I don't think it does the trick of making me understand what it does.
What we need to know is that, the operating system has event notification mechanisms, for
example Linux has `epoll`, Windows has `IOCP` and all other general purpose
SOs have their alternatives. This mechanisms make it possible for applications
to *register* themselves as interested in a specific *event*, lets say reading from a socket, and the OS will keep an eye
on it while the app does other stuff. Once the app is interested again the the state of the *event* it can *poll* if
the event happend, in our case the socket is readable, and act on it. Mio provides a rust-idiomatic and
easy way of using this mechanisms.

> NOTE:
>
> One important thing that I misunderstood when reaserching this
> is the meaning of 'polling'. I was thinking of 'busy polling'.
> If you don't know the difference I recommand reading about it.

## How to use Mio?

Mio has at its base 6 basic types:
- `Poll` (which we use to poll for events)
- `Token` (which we use to identify events)
- `Event` (the event)
- `Events` (a container in which to store events)
- `Interest` (types of events to poll for)
- `Source` (trait that defines pollable objects)

Now with an API surface so small you can imagine that it's not that complicated to use. And you are right!

First thing we need to do is identify what do we want to poll for, luckily mio provides some Non-Blocking I/O objects so we don't
have to implement one from the ground app (but you can do that with the `event::Source` trait). In this example we will be using
the `TcpListener` object exactly like in the [official documentation](https://docs.rs/mio/latest/mio/).

```rust
// Production server address
const address: &str = "127.0.0.1";

let mut listener = TcpListener::bind(address)?;
```

We decided that we want to poll on the listener. To do that we will need a `Poll` object.

```rust
let poll = Poll::new();

// And we can register an [`Interest`] in the [`Registry`] of our [`Poll`] object,
// but befor doing that we must define a [`Token`].

// A mio::Token is an identifier associated with an event source.
const SERVER: Token = Token(0);

poll
    .registry() // This gives us a Registry object
    .register(
        &mut listener,     // Source
        SERVER,            // Token
        Interest::READABLE // Interest
    )?;
```

> NOTE: We can register multiple event sources of course but for our purposes one will do.

> NOTE: A READABLE event doesn't mean that there was something read from the server,
>       it means that the searver is ready to be read from.


Now lets say that we want to poll on this, but this action will give as a number of events (in our case only 1 but endolge me).
Luckily again, `Mio` provides a purposes built container for storring events and it is called `Events` (naming is peak in this crate).

```rust
// The capacity is the ammount of memory initially allocated for the container.
// We will give it a capacity of 1, but keep in mind that the container is a [`Vec`]
// underneath and it can expand on its own if needed, so this donesn't constrain us
// to only storing one event.
let events = Events::with_capacity(1);
```

Now finally we have everything we need to *poll*:
```rust
// Start event loop
loop {
    // Poll the OS for events, waiting at most 100 milliseconds.
    poll.poll(&mut events, Some(Duration::from_millis(100)))?;

    // Process each event.
    for event in events.iter() {
        // We can use the previously provided token
        // to determine for which type the event is.
        match event.token() {
            SERVER => loop {
                // One or more connections are ready, so we'll attempt to
                // accept them (ina loop).
                match listener.accept() {
                    Ok((connection, address)) => {
                        println!("Got a connection from: {}", address);
                    },
                    // A "would block error" is returned if the operation is not ready,
                    // so we'll stop trying to accept connections.
                    Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => break,
                    Err(err) => return Err(err),
                }
            }
        }
    }
}
```

## Conclusion

In the end `Mio` is a very useful crate that can be used in almost all fairly sized Rust applications. I hope you enjoyed the short overview,
keep in mind all the examples were taken from the [official documentation](https://docs.rs/mio/latest/mio/) and have a nice day!
