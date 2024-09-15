---
title: "Going Super Sonic with Asio"
description: Gotta go fast! Lessons learned for squeezing the most out of Asio for your application's networking.
slug: going-super-sonic-with-asio
date: 2024-09-15 00:00:00+0000
author: Chaosvex
image: cover.jpg
categories:
    - Programming
tags:
    - programming
    - performance
    - networking
draft: false
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## Skippable preamble
MMO server emulation is a pretty niche topic and there likely isn't that much demand for a development blog on the topic. Yet, part of the fun of developing a sprawling backend system is that it's pretty easy to find an excuse to delve into almost any domain you get an itch to explore. At the very heart of any server emulator, though, is the networking library. In Ember, that heart is Boost.Asio.

As a disclaimer, my usage of Asio is limited to open source projects so I won't profess to be an expert in it. But, as most that users will know, its documentation can, at points, be *somewhat* cryptic and unopinionated on matters of performance.

Asio is certainly utilised in domains that demand performance but those paid to squeeze the most out of it are often tight-lipped on their techniques. I'm here to blow the lid on those techniques to ensure you can get your high-frequency trades in before they do. Alright, perhaps we'll settle for something a little more modest... like a more responsive chat app or MMO server. Still, a good starting point to tide you over until you need to play in the big leagues of [DPDK](https://en.wikipedia.org/wiki/Data_Plane_Development_Kit) and [SeaStar](https://github.com/scylladb/seastar).

## Getting down to it
The techniques discussed here are those that I've learned from my own experiences with Asio. There are rarely absolutes in development (apart from tabs being four spaces), so don't take them as gospel. Evaluate each technique and decide whether your own usecase could benefit. To put it bluntly, don't blame me if you submit a PR that makes your colleagues balk.

### Mo' cores, mo' problems
<img src="moarcores.jpg" width="250px" style="float:right; margin: 20px; border-radius: 5%;">I'm going to open with a real blinder. The machine you're running your Asio code on *probably* has a CPU with multiple cores and if you're looking to get the most from Asio, you want to be utilising all of them. That's it, now go forth and speed up your program.

Okay, it's a little more nuanced than that. The default approach to scaling Asio that I've run() across is to simply add additional worker threads to your `io_context`. Perhaps a little something like the below.

```cpp
auto concurrency = std::thread::hardware_concurrency();
boost::asio::io_context service(concurrency);

std::vector<std::jthread> workers;

// Behold! Instant scaling!
for (auto i = 0u; i < concurrency; ++i) {
    workers.emplace_back(
        static_cast<std::size_t(boost::asio::io_context::*)()>
        (&boost::asio::io_context::run), &service
    );
}

// ... other stuff...
```

> Note: `std::thread::hardware_concurrency()` can return `0` if it's unable to determine the number of cores on the machine. I've never seen it happen but you might want to handle it.

It looks appealing, right? You can continue to use your single `io_context` (or `io_service` if you're working in the dark ages) and make use of all your cores with just a few extra lines. While it is is a pretty good start, it has drawbacks that, in my humble opinion, mean it should not be the default approach you go for. To understand the *why not*, we need to first explore the *why*.

An `io_context` is effectively a work queue. You post work (e.g. read from a socket) to it and at some point, it'll give you a result, or an error. By adding multiple threads to the `io_context`, it's able to selectively dispatch work to a thread whose own queue is looking a little lighter than the rest. In other words, it helps Asio to balance work evenly across threads, ensuring no worker thread is sitting idle ('starvation') while another is overburdened.

This sounds pretty great, but there's a price to pay. The more threads you add, the more lock contention you'll get on those queues. On top of that, your IO objects now need their own synchronisation to ensure they cannot be accessed from multiple worker threads at the same time. This isn't a problem if you're using a half-duplex protocol as you'll see in many examples, but it becomes one once you go full-duplex. To illustrate what I mean by half-duplex, take this example.

```cpp
// pseudocode
void start() {
    read();
}

void read() {
    asio::read(_socket, _buffer, [&](auto size, auto error) {
        process_message(size);
    });
}

void process_message(auto size) {
    auto response = generate_response(_buffer, size);
    
    asio::write(_socket, asio::buffer(*response),
     [&, response](auto size, auto error) {
        read();
    });
}
```

In this fairly typical Asio starter example, traffic can only flow in a single direction at a time, following a basic request -> response pattern. This doesn't need to be synchronised because we only have a single completion handler associated with our IO object (`_socket`) at any given point.

To illustrate the handler flow, let's get retro with some ASCII art:

```
accept connection
       ↓
     start → read → process → write
               ↑                ↓
               ↑                ↓
                ←←←←←←←←←←←←←←←←
 ```

 This approach works well for many basic protocols, including HTTP/1.1 (which may or may not be half-duplex depending on who you ask), but soon falls apart when you need to communicate in both directions at the same time and where there isn't a 1:1 mapping between requests and responses... such as any MMO server.

For a full-duplex protocol in Asio, we need to be able to initiate write operations independently of the read handler. For example:

```cpp
// more pseudocode
asio::strand _strand(io_context); // done somewhere else
buf_type _buffer;

void start() {
    read();
}

void read() {
    asio::read(_socket, _buffer,
        _strand.wrap([&](auto size, auto error) {
            process_message(size);
            read();
        })
    );
}

void write(std::shared_ptr<buf_type> response) {
    asio::write(_socket, asio::buffer(*response),
        _strand.wrap([&, response](auto size, auto error) {
            // check for errors or whatever
        })
    );
}

// alternatively, instead of wrapping handlers in
// strands, we can do this when we create the socket
_socket = asio::ip::tcp::socket(asio::make_strand(io_context));
```

We've made sure that all handlers that use our socket are serialised. This is going to add a overhead but it's certainly preferable to introducing race conditions. 

So, we've now introduced lock contention for the work queues and overhead for handler serialisation in exchange for `io_context` concurrency, which is a decent trade-off... but it's also the *why not*. We haven't evaluated whether work balancing is a feature we *actually* need!

If your application is likely to have a high volume of computationally expensive work on the `io_context` threads, you may well benefit from work balancing. However, if you're just using Asio to shuttle packets around at warp speed without doing anything overly taxing on the threads, you might benefit from the `io_context` per thread approach. This reduces lock contention within Asio and removes the need to use strands to ensure serialised IO object (i.e. sockets, timers, files) access.

Here's an example of setting up the `io_context` per core approach:

```cpp
std::vector<std::jthread> threads;
std::vector<std::unique_ptr<asio::io_context>> services;

const auto core_count = std::thread::hardware_concurrency();

// create io_contexts
for (auto i = 0u; i < core_count; ++i) {
    services.emplace_back(
        std::make_unique<asio::io_context>(1)
    );
}

// create worker threads
for (auto i = 0u; i < core_count; ++i) {
    threads.emplace_back(static_cast<std::size_t(asio::io_context::*)()>
        (&asio::io_context::run), services[i].get());
    thread::set_affinity(threads[i], i);
}
```

All we do here is determine the number of cores in the system and create an `io_context` with a single worker thread for each one. Whether a 1:1 core to thread ratio is the right choice for your application will come down to profiling but it's a sensible starting point.

Ember's own implementation of this pattern can be found [here](https://github.com/EmberEmu/Ember/blob/development/src/libs/shared/shared/threading/ServicePool.cpp).

#### To affinity and beyond

In the previous example, we've also set the *affinity* of every worker thread. This is often referred to as *pinning* and effectively locks a thread to a single core, meaning it cannot be scheduled to run on any core other than the one specified. Pinning a thread can boost the chances of cached code and data still being available the next time it's scheduled to run on that same core (*'warm cache'*), whereas if it's moved to another core, it'll likely need to pull its data back in from a higher level cache or RAM (*'cold cache'*).

Setting affinity is platform specific, so you'll need to implement the functionality for each platform you want to target. Ember has an example implementation [here](https://github.com/EmberEmu/Ember/blob/development/src/libs/shared/shared/threading/Utility.cpp#L31).

The drawback is that if you have idle cores and pinned threads waiting to be scheduled on busy cores, the OS' scheduler is not allowed to reschedule the threads on the idle cores, leading to potential underutilisation of the CPU. If you have a heavy load application that's allowed to use all of your cores, it's likely an acceptable trade but once again, profile and decide whether it's the right choice.

As an addendum, this technique can play quite nicely with RSS (<cite>receive side scaling[^14]</cite>) if you've got capable hardware to play with.

<div style="clear:both">


### Luke, use the thread pool

Earlier, I mentioned that you might want to retain Asio's work balancing if you were doing computationally expensive work. Another scenario that might be holding Asio up is having to deal with IO bound operations such as file reads/writes or blocking APIs. Basically, if it's going to take a while to execute but doesn't need CPU cycles that could be better spent elsewhere, don't do it on your networking worker threads.

In Ember's case, we use MySQL Connector C++, a blocking API. Issuing database queries from within Asio's worker threads is going to cause a bottleneck as we sit around waiting for the database, potentially on a completely different machine, to get back to us. What we should instead do is palm this work off to a separate group of workers, allowing the networked clients to continue merrily on until the operation has completed.

Luckily, Asio has a `thread_pool` implementation that does the job as documented [here](https://think-async.com/Asio/asio-1.18.1/doc/asio/reference/thread_pool.html) for the standalone Asio and [here](https://www.boost.org/doc/libs/master/doc/html/boost_asio/reference/thread_pool.html) for Boost.Asio. Ember also has a basic one [here](https://github.com/EmberEmu/Ember/blob/development/src/libs/shared/shared/threading/ThreadPool.h), since it predates Asio introducing its own.

The idea is simple enough. Whenever we need to perform a blocking task, we simply wrap it up in a lambda and post it into the thread pool. If need be, we can then post the result back into `io_context` it originated from.

Cribbed from <cite>Asio's documentation[^5]</cite>:

```cpp
void my_task() {
    // do something
}

// Launch the pool with four threads.
asio::thread_pool pool(4);

// Submit a function to the pool.
asio::post(pool, my_task);

// Submit a lambda object to the pool.
asio::post(pool, []() {
    // do something
});
```

<img src="unacceptable.jpg" width="250px" style="float:left; margin: 20px; border-radius: 5%;">The ideal size of the pool is going to be specific to your application's workload and usage patterns. You may even consider multiple pools with differing sizes. For example, initiating disk IO from a large number of threads may be detrimental to performance, whereas issuing a large number of queries to a stonking database server might be perfectly acceptable.

> Note: Asio file IO was previously Windows-only but has introduced support for Linux, if your kernel has io_uring.

### Concurrency hints

When creating an `io_context`, Asio allows you set a concurrency hint to tell it how many threads you expect to be running within the context. I'm going to crib from Asio's documentation again. Let's take a look at a constructor overload for `io_context`.

> io_context::io_context (2 of 2 overloads)
>
> Constructor.
>
> io_context(int concurrency_hint);
>
> Construct with a hint about the required level of concurrency.
>
> Parameters:
>
> concurrency_hint
>
> A suggestion to the implementation on how many threads it
> should allow to run simultaneously.<br/>
> — <cite>Asio documentation[^1]</cite>

<img src="neature.jpg" width="250px" style="float:right; margin: 20px; border-radius: 5%;">That's pretty neat. Unfortunately, if you've chosen the multithreaded `io_context` path, it might not be able to do much for you. On Windows, it passes the value to IOCP (IO completion ports), which may or may not have any effect, but it has no direct impact on Asio.

If we've restrained ourselves to a single-threaded `io_context`, we can pass a concurrency hint of `1`, which the documentation tells us will eliminate a lock.

> Using thread-local operation queues in single-threaded use cases (i.e. when concurrency_hint is 1) to eliminate a lock/unlock pair.<br/>
> — <cite>Asio release notes[^2]</cite>

That was a pretty easy win, but there are [two other values of interest](https://think-async.com/Asio/asio-1.21.0/doc/asio/overview/core/concurrency_hint.html), although much greater care will need to be taken if you choose to use them.

####  Unsafe

`ASIO_CONCURRENCY_HINT_UNSAFE` is the ultimate hint, disabling locking in Asio's scheduler and reactor IO. The downside, though, is that it makes almost every operation thread unsafe, meaning even `post` or `dispatch` must only be issued from a single thread at a time. Additionally, you cannot use asynchronous resolve operations.

####  Unsafe IO

A step down but still good, `ASIO_CONCURRENCY_HINT_UNSAFE_IO` retains scheduler locking but disables reactor IO locking. All functionality will continue to be available and you can still safely `post` and `dispatch` to your `io_context` from multiple threads but you cannot access IO objects (i.e. sockets and timers) from multiple threads. Additionally, you cannot call `run` functions (`run_*`, `poll`, `poll_*`, `reset`, `restart`, `stop`) from multiple threads.

> Warning: Concurrency hints are generally not well understood or well documented, so take care in using them.<br/>
> — <cite>Asio GitHub[^3]</cite>

#### The nuclear option
There's one more option related to threading, and it isn't actually a hint but rather defining `BOOST_ASIO_DISABLE_THREADS` or `ASIO_DISABLE_THREADS` (standalone). This completely removes Asio's internal locking, replaces atomics with non-atomics equivalents and triggers various other changes. However, it also disables multiple features, including timers. The documentation for it is sparse and it [may behave differently on different platforms](https://www.boost.org/doc/libs/1_86_0/doc/html/boost_asio/overview/implementation.html), so I won't dedicate more words to it other than to say *best of luck*.

### Stay hungry
> This section only applies to TCP streams. If you're only interested in UDP, you can skip it.

<img src="mrgreedy.jpg" width="250px" style="float:left; margin: 20px; border-radius: 5%;">One common protocol design pattern is called **TLV**, or <cite>*type (or tag), length, value*[^4]</cite>. The idea is that each element starts with a message *type*, followed by the *length* of the message, finally followed by the *value* (or body) of the message. The type and length might be swapped but that doesn't change the underlying concept. The type and the length are fixed-size and known in advance, with the value being variable.

In the case of message definitions that follow the same pattern, I think it's reasonable to just treat the value as an aggregate type, with the type and the length representing the message header.

```
+---------------+-------------+ 
|     Type      |   Length    |
+---------------+-------------+  
|            Value            |
+-----------------------------+
```
One common way of dealing with this type of protocol is to initiate three socket reads; one to determine type, one to determine length, and a final read for the value/data. A slightly better implementation will group the T and L together and then read the value separately, giving us two socket reads per message.

> Terminology note: A message is the application level data that you send, receive and process. A packet is the outer layer that encapsulates zero or more messages. Packet is often used interchangeably with message but in this case, we're following this definition.

Ideally, we want to minimise the number of socket operations and that means being *greedy*. We should to aim to read as much data as possible per socket read. Because TCP is stream-oriented, it's allowed to coalesce multiple messages into a single packet. In high-throughput scenarios, it's quite possible that we could receive multiple messages with only a single socket read, rather than requiring 2*number of messages. You already know the drill, there's going to be a downside. In the worst case, it could increase the number of socket reads if we're dealing with large numbers of slow clients that are only sending a handful of bytes at a time, but this should rarely be the case. Unless your service is under attack.

Here's an example of the naïve implementation:

```cpp
// pseudocode
void read_loop() {
    while(true) {
        // read header (type, length)
        auto header = _socket.read(header_size);

        // extract length
        auto value_len = get_length(header);

        // read body (value)
        auto body = _socket.read(value_len);
        process_message(header, body);
    }
}
```

We start by reading the header (type and length), extracting the length, and then issuing a second read to get the rest of the message. It's simple, it's functional, but not the most efficient.

In the following example, we'll issue a single read on the socket and then process the data in a loop, allowing zero or more messages stored within the buffer to be processed. The buffer type is intentionally omitted as the bookkeeping strategy for tracking the read/write position within the buffer will depend on the concrete type. For example, if you were to use an `std::vector`, you'd need to keep track of the offsets separately.

```cpp
// pseudocode!
our_buffer_type<char> _buffer;

enum class read_state {
    header, body, done
} _state;

void read_loop() {
    while(true) {
        auto read_len = _socket.read_all(_buffer); // read *all* available data
        process_buffered_data();
    }
}

void parse_header() {
    // logic to read & check header completion
    // ...

    if(header_complete) {
        _state = read_state::body;
    }
}

void completion_check() {
    // logic to read & check body completion
    // ...

    if(body_complete) {
        _state = read_state::done;
    }
}

void process_message() {
    // logic to handle a completed message
    // ...

    // go back to square one
    _state = read_state::header;
}

void process_buffered_data() {
    while(!_buffer.empty()) {
        if(_state == read_state::header) {
            parse_header();
        }

        if(_state == read_state::body) {
            completion_check();
        }

        if(_state == read_state::done) {			
            process_message();
            continue; // check for next message
        }

        // if we get here, we have a partial message buffered
        break;
    }
}
```
It's a little tricker to get right than the first version, but definitely worth it if we want to wring the most out of our networking layer. For a concrete example of this pattern, you can take a look at how [Ember reads messages](https://github.com/EmberEmu/Ember/blob/8e20698c5cafdcfd5e873d8012191f23486043f2/src/gateway/ClientConnection.cpp#L124).

The same principle applies to writes, too. Asio supports scatter/gather, allowing you to send multiple buffers in a single call. This could be useful if you have a message queue that you're occasionally pumping into Asio. Here's an example of sending two buffers at once, from some adapted legacy code in Ember:

```cpp
std::array<asio::const_buffer, 2> buffers {
    asio::const_buffer { buffer_1.data(), buffer_1.size() },
    asio::const_buffer { buffer_2.data(), buffer_2.size() },
};

_socket.async_send(buffers, [&](...) {
    // the rest of your completion handler
});
```

In this basic example, the number of buffers is fixed at compile-time but if you need the number of buffers to be flexible, you can construct a *buffer sequence*, as <cite>per the documentation[^6]</cite>. Ember has a functional implementation <cite>here[^7]</cite>, since the documentation is a little sparse. The idea is to provide Asio with iterators over your sequence of buffers, with the lifetime of the underlying buffers left in your capable hands.

Buffer sequences could likely fill an article of their own but at least you know they exist and when to reach for them. Just make sure your buffer sequences are <cite>cheap to copy[^8]</cite>.

### A problem shared, is a performance halved

If you have any experience with Asio, you'll be familiar with how `std::shared_ptr` and `std::enable_shared_from_this` are encouraged for lifetime management of your connections. Internally, `shared_ptr` uses atomics for reference counting, and those counts need to be updated every time you call `shared_from_this()` and capture it within an Asio completion handler that has being copyable as a requirement. This is *pretty* slow and if we want to squeeze the most out of our application, we might want to consider eliminating it.

<img src="dragonslair.jpg" width="250px" style="float:right; margin: 20px; border-radius: 5%;">I'll be up front with you, this one is difficult to get right and if you're using a multithreaded `io_context`, you're on your own. If, however, you're able to guarantee that your connection objects are only ever accessed from a single thread, thusly having easier to reason about lifetime, you should be able to eliminate the need for passing a `shared_ptr` by value everywhere. That said, there are absolutely dragons here and if you're able to get it right on ~~first~~ ~~second~~ third attempt, we are not worthy of your presence.

Ember achieves this by storing a set of `unique_ptr<connection>`s. Initiating a connection shutdown will remove the connection from the set and pass the smart pointer to the connection object. The connection object will close all of its IO objects (sockets, timers), triggering an `operation_aborted` error on any pending completion handlers. Immediately after closing the IO resources, we `post` a final completion handler which will allow the smart pointer to drop off, freeing the connection object. The final `post` is why this strategy wouldn't work with a multithreaded `io_context`, since we wouldn't be able to guarantee that it'd only execute after Asio had cancelled all completion handlers associated with the object.

This all sounds a little barmy, and that's because it is, but I'm yet to come across a *clean* way to manage Asio connections without `shared_ptr`s. Claims have been made that it can be done but I've never seen any examples and I have asked. Other examples of avoiding `shared_ptr`s in their Asio code have equally unwieldly tricks to make it work. Herb Sutter has discussed better lifetime models that don't require such tricks, so perhaps in the future, there will be a better way.

If you're keen to see what the above looks like in practice, take a gander at Ember's version [here](https://github.com/EmberEmu/Ember/blob/8e20698c5cafdcfd5e873d8012191f23486043f2/src/gateway/SessionManager.cpp#L37).

### Timing out

One feature you'll often find yourself needing is the ability to time a network client out after a period of inactivity. The common way to achieve this in Asio is to use a timer and reset it each time there's activity on the client's socket. For example...

```cpp
using namespace std::chrono_literals;

boost::asio::steady_timer timer;

void start_session() {
    start_timer();
    read();
}

void read() {
    _socket.async_receive(..., [&](...) {
        start_timer();
        // handle read
        read();
    });
}

void write(auto buffer) {
    _socket.async_send(buffer, [&](...) {
        start_timer();
        // handle write
    });
}

void start_timer() {
    timer_.expires_from_now(60s);
    timer_.async_wait([&](const boost::system::error_code& ec) {
        if(ec == boost::asio::error::operation_aborted) {
            return;
        }

        timeout();
    });
}

void timeout() {
    // close the connection
}

```

All we're doing here is starting a 60 second timer. For each read or write to the socket, we're resetting the timer and giving the client another 60 seconds to live. If there's no activity on the socket within that period, we'll disconnect the client. The problem with this approach is, as it turns out, setting timers up in Asio is very expensive if you're doing it hundreds, or even thousands, of times per second.

A much better approach is to instead set the timer up and then allow the timer to expire before resetting it. Here's an improved example.

```cpp
using namespace std::chrono_literals;

boost::asio::steady_timer timer;
bool is_active = false; // make atomic if multithreaded IO objects

void start_session() {
    start_timer();
    read();
}

void read() {
    _socket.async_receive(..., [&](...) {
        is_active = true;
        // handle read
        read();
    });
}

void write(auto buffer) {
    _socket.async_send(buffer, [&](...) {
        is_active = true;
        // handle write
    });
}

void start_timer() {
    is_active = false;

    timer_.expires_from_now(60s);
    timer_.async_wait([&](const boost::system::error_code& ec) {
        if(ec == boost::asio::error::operation_aborted) {
            return;
        }

        if(is_active) {
            start_timer();
        } else {
            timeout();
        }
    });
}

void timeout() {
    // close the connection
}

```

With this approach, we're starting the timer when the socket starts and then allowing it to expire every 60 seconds. When there's socket activity, we set the `is_active` flag to `true`. This flag is then checked within the timer's completion handler. If it's set to true, it knows that there's been socket activity since the last check and it simply restarts the timer for another 60 seconds, otherwise, it disconnects the client as before. 

If we're dealing with high-traffic clients, the second approach *vastly* reduces the overhead of managing timers. It could mean the difference between doing it once per minute rather than tens of thousands of times per minute (assuming a 60 second timeout).

Now, there is a downside but it's probably inconsequential for most applications. The timeout in the second example isn't going to be as accurate as the first. Whereas the first example will trigger the disconnect at pretty much exactly the time specified, the second example could take up to n*2 seconds (where n is your timeout) to time the client out. It will, however, be just as accurate for a client that connects but never sends any data before timing out.

 For bonus points, use a single timer to manage multiple clients, as long as you can do so without locking.

### Polymorphic executors

By default, modern versions of Asio use a polymorphic executor, `any_io_executor`, which internally uses reference counting and may cause additional allocations. <cite>While improvements have been made to mitigate the performance overhead for the typical cases[^15]</cite>, you might stand to benefit if you don't need the flexibility offered by the polymorphic executor.

```cpp
#include <asio/ip/tcp.hpp>
#include <asio/io_context.hpp>

void foo(asio::ip::tcp::socket socket) {
    // do something with the socket
}

int main() {
    asio::io_context ctx;
    asio::ip::tcp::socket socket(ctx);
    asio::ip::tcp::socket strand_socket(asio::make_strand(ctx));

    // connect the sockets or what have you
    // ...

    foo(std::move(socket));        // OK
    foo(std::move(strand_socket)); // OK
}
```

Internally, these sockets are both using the default polymorphic executor. If we want to remove the polymorphism, we'll need to template our socket types on the concrete executor that we want to use.

```cpp
#include <asio/ip/tcp.hpp>
#include <asio/io_context.hpp>
#include <asio/strand.hpp>

using executor = asio::io_context::executor_type;
using tcp_socket = asio::basic_stream_socket<asio::ip::tcp, executor>;

void foo(tcp_socket socket) {
    // do something with the socket
}

int main() {
    asio::io_context ctx;
    tcp_socket socket(ctx);
    tcp_socket strand_socket(asio::make_strand(ctx)); // won't work

    // connect the sockets or what have you
    // ...

    foo(std::move(socket));        // OK
    foo(std::move(strand_socket)); // also wouldn't work
```

<img src="polymorph.jpg" width="250px" style="float:right; margin: 20px; border-radius: 5%;">We've removed the polymorphic executor by templating socket on the `io_context`'s executor type, but we've lost the flexibility of being able to pass around sockets that might be using different executors internally, such as our strand wrapped socket, `strand_socket`. Although we could wrap our completion handlers in a strand instead, we can solve this by defining an additional type.

```cpp
#include <asio/ip/tcp.hpp>
#include <asio/io_context.hpp>
#include <asio/strand.hpp>

using executor = asio::io_context::executor_type;

using tcp_socket = asio::basic_stream_socket<asio::ip::tcp, executor>;

using strand_tcp_socket = asio::basic_stream_socket<
	asio::ip::tcp, asio::strand<executor>
>;

void foo(auto socket) {
    // do something with the socket
}

int main() {
    asio::io_context ctx;
    tcp_socket socket(ctx);
    strand_tcp_socket strand_socket(asio::make_strand(ctx));

    // connect the sockets or what have you
    // ...

    foo(std::move(socket));        // OK
    foo(std::move(strand_socket)); // OK once again
```

`foo` is now templated on the socket type and all is well, except possibly our compilation times. We've lost some flexibility but we've now hopefully gained in performance. 

If you want to read more about this but find documentation lacking, you can find a good overview on this <cite>StackOverflow post[^16]</cite>.

As always, evaluate the pros and cons and decide whether you need to do this. Although, if you're looking to write fast code just because you find it fun, *hell yeah, brother!*

> Note: You could allow `foo()` to accept an `asio::ip::tcp::socket`, but you'd be back to using the polymorphic executor if you passed a `tcp_socket` to it


### Lighter than air buffers
A typical initial approach to managing buffered data is to use vectors. When a datagram is received or a <cite>TLV[^4]</cite> message is ready to read, the vector is resized to match and the data is read in. When an outgoing message is being built, the data is wrangled into a vector, which is then sent directly after wrapping in a shared pointer or moving it into a message queue.

If we don't need the networking to scale, this approach is *fine*. If we're writing an MMO server with a target of handling a five-figure number of players, allocating tens of thousands of times per second is less than ideal.

A better approach is to allocate the buffers *once* per connected client and then reuse them for the lifetime of that connection. Ember's approach is to use custom buffer types tailored to its usage patterns. The two main buffer types used are the *<cite>static buffer[^12]</cite>* and *<cite>dynamic buffer[^13]</cite>*.

The <cite>static buffer[^12]</cite> is a fixed-size buffer used *only* for inbound messages. Because we know the maximum message size from the client, and it isn't that large, the storage for this buffer is part of the client connection object. In essence, it's nothing more than an `std::array` wrapped with some helper functions to aid with bookkeeping, such as tracking the read/write position within the storage. It's not used for output because the size of outgoing data to each client is an unknown that's dependent on what's going on in the game world. If they're meandering through a dark grove on their lonesome, it won't be much. If they're in an intense world PvP battle at Azuregos, it'll likely be quite a lot. Additionally, because we're processing packets the moment they arrive, we're emptying the inbound buffer as quickly as it's filled, and so the socket will never find itself in a situation where the buffer is too full (unless the maximum allowed message size is exceeded).

<img src="raid.jpg" width="250px" style="float:right; margin: 20px; border-radius: 5%;">The <cite>dynamic buffer[^13]</cite> allows for an unbounded amount of data to be written to it. It achieves this by internally allocating and deallocating fixed-size chunks of memory and storing them in an intrusively linked list. To reduce the number of allocations, it will attempt to reuse the initially allocated chunk as much as possible. This means that as long as the amount of data stored in the buffer at any given point stays under the size of a single chunk, it will never allocate additional chunks. To mitigate the cost of allocations in high-traffic scenarios, custom allocators can be used. In Ember's case, a much larger thread-local storage shared pool is used to allocate from, only resorting to the system allocator if that's also exhausted. The dynamic buffer can be used for inbound traffic but the internal machinery imposes overhead that isn't necessary if you don't need to exceed the size of a single memory chunk.

The fixed-size nature of the dynamic buffer makes it great for throwing in arbirtrary data and pumping it out of Asio. A variation that you might see used is to allow the linked list of memory chunks to be of arbitrary size, meaning a chunk holds exactly one message. This might be preferable if you want to be able to add and remove chunks to/from the buffer to move them around your application or transfer them between buffers without copying the data.

Ultimately, the buffer choices made for one application design might not work for another. You'll need to take stock of your application's design and the protocols you're working with and then choose the right buffering strategy from there. Just do what you can to avoid allocations, copying and locking.

### Double buffering

If you're familiar with graphics, you'll likely have already come across the concept of double (or triple) buffering. While the *front* buffer is being used to draw to the display, the *back* buffer is used allow to application to keep rendering without overwriting the contents of the other buffer while it's being read from. Once ready, the two buffers are switched, with the back becoming the front and vice versa.

You can use the same concept with network message buffering. In Ember's case, we need to deal with multiple messages within a single invocation of a `read()` completion handler (see: stay hungry), meaning we may need to write multiple responses. Rather than using a message queue, we instead employ double buffering. The first response that's generated is written to the *back buffer*, which is then swapped to become the *front buffer*, and Asio is instructed to send the contents. Any additional messages will be written to the new back buffer, but will *not* invoke another Asio write operation, at least not directly. Instead, when Asio finishes sending the front buffer, it will swap the buffers, check whether the new front buffer has any data, then continue sending if so.

This simple technique removes the need for an explicit message queue, which also makes buffer lifetime and memory management easier. <cite>Ember has an example implementation[^21]</cite>.

### Deferred coroutines

If you're using Asio's coroutines, you'll notice that the examples generally make use of `use_awaitable`. An example from <cite>Asio's documentation[^9]</cite>:

```cpp
asio::co_spawn(executor, echo(std::move(socket)), asio::detached);

// ...

asio::awaitable<void> echo(tcp::socket socket)
{
  try
  {
    char data[1024];
    for (;;)
    {
      std::size_t n = co_await socket.async_read_some(asio::buffer(data), asio::use_awaitable);
      co_await async_write(socket, asio::buffer(data, n), asio::use_awaitable);
    }
  }
  catch (std::exception& e)
  {
    std::printf("echo Exception: %s\n", e.what());
  }
}
```

Unless you have a specific need for `asio::use_awaitable` (e.g. you need to store the coroutine in a container), use `asio::deferred` instead. `deferred` allows Asio to potentially skip allocating a stack frame for the coroutine. In fact, the latest Asio documentation is now being updated to favour `deferred`.

### Nagle's algorithm
> This section only applies to TCP streams. If you're only interested in UDP, you can skip it.

One [complaint](https://github.com/chriskohlhoff/asio/issues/1426) that [occasionally](https://stackoverflow.com/questions/2039224/poor-boost-asio-performance) crops up from beginners is how sluggish Asio seems to be when building their introductory TCP networking programs. These programs tend to deal with low traffic volumes in a request -> response pattern, causing them to run head-on into into Nagle's algorithm.

<img src="nagles.jpg" width="250px" style="float:left; margin: 20px; border-radius: 5%;">Nagle's algorithm, named after its creator, John Nagle, was designed back in the 1980s when bandwidth was at a premium and sending packets containing single keystrokes to a remote machine incurred significant overhead. To improve efficiency, the algorithm delays outgoing traffic in order to coalesce multiple messages into a single packet, thus reducing the overhead of having to send many TCP frames where only one is needed. The downside is that the delay introduced by the algorithm can sometimes have a noticeable impact on applications that are latency-sensitive, particularly those that are relatively low bandwidth or have pauses between outgoing traffic. It isn't just programmers that get bitten by Nagle's; it's not uncommon for gamers to disable Nagle's algorithm at a system level to help improve multiplayer responsiveness.

Although Nagle's algorithm can generally be controlled at the system level, it's a better idea to control it at the application level, allowing applications that might stand to benefit to continue to do so. I'd argue that those applications are now few and far between but that's a tangent. An example scenario where you *might* want to keep the algorithm enabled could be an application that heavily values throughput over latency.

Disabling the algorithm in Asio is simple enough, simply set the TCP socket option as such, *before* the socket is connected:

```cpp
socket.set_option(asio::tcp::no_delay(true));
```

This can also be done on the listening acceptor socket, causing any connections it accepts to also have the no delay flag set.

As always, profile your application to decide whether this is the right choice for you. If you want to dive deeper into mystical algorithms that operate within the shadows of the kernel, look into TCP corking.

### Include what you use

This one's going to take a different tack and give a brief nod to compile-time performance.

Asio is a header-only library, chock full of templated types. This can be anathema to compile times, especially if you're including the entire library everywhere. Both new and experienced users will occassionally be tempted to use Asio via `#include <asio/asio.hpp>`. It's to quick to type and you don't need to worry about which Asio types you'll end up needing within your translation unit. It's also dragging the kitchen sink along with it.

Asio is well-structured, so get into the habit of only including what you actually use. Only need a TCP socket? `<asio/ip/tcp.hpp>`. Strands? `<asio/strand.hpp>`. Most parts of Asio will have their own header. You get the idea.

### Custom allocator

This is going to be an honourable mention more than anything else, simply because you could fill a book on allocators. Just know that Asio supports custom allocators to be used in conjunction with your completion handlers. Here's an example:

```cpp
_socket.async_send(buffer, create_alloc_handler(_allocator,
    [this](boost::system::error_code ec, std::size_t size) {
        // do your thing
    })
);
```

For a glance at what Asio expects the allocator interface to look like, <cite>here's an example from Ember[^10]</cite> and <cite>Asio's own example[^11]</cite>. Just don't use Ember's example in production - it's only intended as a placeholder for future improvement.

<div style="clear:both">

### Profile io_uring 

Another honourable mention but an area that you might want to keep your eye on as the implementation matures. 

If you're building your application on a Linux distribution with kernel version 5.1+, you might stand to benefit from profiling Asio's `io_uring` backend. By default, Asio will use the `epoll` interface for most IO objects, including sockets, so to enable it, you'll need to <cite>define both `BOOST_ASIO_HAS_IO_URING` and `BOOST_ASIO_DISABLE_EPOLL`[^17]</cite>. Some users have reported a performance lift, while others have reported a hit, so you *really* do need to profile your application to decide whether it's the right move for your application.

### Platform specific techniques

There are other techniques that might come in handy in very specific scenarios but they're likely to be platform dependent. I'll briefly mention two of them here just so you're aware that they exist. There may be equivalents on other platforms but I don't have experience with them.

#### Socket splicing

<cite>splice()[^18]</cite> is a Linux system call that allows for moving data between file descriptors (that includes sockets) without having to copy the data between kernel and user space. This can be incredibly beneficial if, for example, you're working on an reverse proxy type application where you simply want to forward data from one socket to another (i.e. from client to backend service).

#### Page locking

When the system wants to read from or write to your networking buffers, page faults caused by the kernel deciding to page the data out to disk or moving its location within physical memory can be a real hit to latency.

Page locking is a technique whereby certain memory ranges are 'locked' into physical RAM, preventing the kernel from paging/swapping them out. This is done to try to reduce the number of page faults generated by accessing data that's no longer resident in memory. One function for doing this on Linux is <cite>mlock()[^19]</cite>, although it does not prevent the kernel from moving the pages around within memory, which can pose problems for DMA (direct memory access). This really is an entire, complex topic unto itself. If you want to to do a deep dive, look into <cite>get_user_pages()[^20]</cite> and friends.

## Closing ceremony

To recap:

* Give Asio all of your cores to work with
* Don't pay the price for work balancing if you don't need it
* Don't make IO intensive or blocking calls on your network threads
* Use concurrency hints to your advantage
* Send and receive as much data as possible with each call
* Use cheap to copy, non-allocating buffer types
* Disable Nagle's algorithm if low latency is important
* Avoid using `shared_ptr`s where you can
* Prefer to defer coroutines
* Use a custom allocator if you can do better than the default
* Tailor your buffering strategy to your protocol & workload
* Don't start/stop timers excessively
* Don't use polymorphic executors if you don't need them
* Only include what you use
* Investigate receive side scaling
* Investigate io_uring on Linux
* **Evaluate whether you need to use each technique before applying it!**

Hopefully some of these tips will be of value to other Asio users or even those dipping their feet into network programming with other libraries. If any of the provided techniques are wrong, woefully misguided, or you have insights for further performance gains, drop a comment below or somewhere else where I'll see it. 

Thanks for reading, happy programming, and may your packets be delivered swiftly and in abundance!

[^1]: Boost.Asio io_context [documentation](https://www.boost.org/doc/libs/1_86_0/doc/html/boost_asio/reference/io_context/io_context.html)
[^2]: Boost.Asio release [notes](https://www.boost.org/doc/libs/1_62_0/doc/html/boost_asio/history.html#boost_asio.history.asio_1_6_1___boost_1_48)
[^3]: Adds concurrency hints, [refuses to elaborate](https://github.com/chriskohlhoff/asio/issues/1466)
[^4]: [Wikipedia entry on TLV](https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value)
[^5]: Asio's thread pool [documentation](https://www.boost.org/doc/libs/1_86_0/doc/html/boost_asio/reference/thread_pool.html)
[^6]: Cryptic Asio [documentation](https://www.boost.org/doc/libs/1_86_0/doc/html/boost_asio/reference/ConstBufferSequence.html)
[^7]: Ember's buffer sequence [implementation](https://github.com/EmberEmu/Ember/blob/development/src/libs/spark/include/spark/buffers/BufferSequence.h)
[^8]: Asio [slap fight](https://github.com/chriskohlhoff/asio/issues/203) over the correct way to send large numbers of buffers
[^9]: Asio coroutine [documentation](https://think-async.com/Asio/asio-1.20.0/doc/asio/overview/core/cpp20_coroutines.html)
[^10]: Ember's custom Asio [allocator](https://github.com/EmberEmu/Ember/blob/development/src/libs/shared/shared/memory/AsioAllocator.h)
[^11]: Asio custom allocator [example](https://www.boost.org/doc/libs/1_86_0/doc/html/boost_asio/example/cpp11/allocation/server.cpp)
[^12]: Ember's [static buffer](https://github.com/EmberEmu/Ember/blob/development/src/libs/spark/include/spark/buffers/StaticBuffer.h)
[^13]: Ember's [dynamic buffer](https://github.com/EmberEmu/Ember/blob/development/src/libs/spark/include/spark/buffers/DynamicBuffer.h)
[^14]: MSDN RSS [documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/network/introduction-to-receive-side-scaling)
[^15]: [What's New in Asio performance](https://clearpool.io/pulse/posts/2020/Jul/13/whats-new-in-asio-performance/)
[^16]: [Sehe explains Asio better than the docs: Can this allocation be avoided with some custom allocator?](https://stackoverflow.com/questions/69042552/can-this-allocation-be-avoided-with-some-custom-allocator)
[^17]: [Asio release history, Asio 1.22.0 / Boost 1.78](https://www.boost.org/doc/libs/1_80_0/doc/html/boost_asio/history.html)
[^18]: splice [documentation](https://man7.org/linux/man-pages/man2/splice.2.html)
[^19]: mlock [documentation](https://man7.org/linux/man-pages/man2/mlockall.2.html)
[^20]: get_user_pages() [documentation](https://www.kernel.org/doc/html/latest/core-api/mm-api.html)
[^21]: Ember's [ClientConnection.cpp](https://github.com/EmberEmu/Ember/blob/cac2cc368dd545d572445310e52051cf0319655d/src/gateway/ClientConnection.cpp#L109C1-L109C8)