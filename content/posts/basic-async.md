---
title: "Basics of Asynchronous I/O"
date: 2026-09-19T00:22:00+02:00
author: Martin Kauppinen
---

I've been meaning to get into at least the basics of asynchronous I/O because I
keep seeing odd system calls when running async programs and want to understand
how they work. So I did a bunch of reading and experimentation, and this post is
the result!

I'll go over what I've learned about the various I/O multiplexing system calls
on *nix: `select()`, `poll()`, `epoll()` (Linux-specific), and even `kqueue()`
(BSD-specific)!

Sidenote: I recently stumbled upon the
[`<details>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/details)
HTML tag and have incorporated it into my blog theme. Occasionally I will have
some side discussions or tangents to talk about. They will be neatly folded
into a box like the one below, so that I don't spend too much time talking
about a different topic.
{{< details summary="This is a clickable sidenote." >}}
In here I might elaborate on a thought in a way I feel is too long for a
footnote, but too short or unrelated to warrant its own section of the article.
{{< /details >}}
Enough preamble, let's get into the topic at hand!

{{< toc >}}

## What's the plan and why?
The plan is to create a simple service: a TCP echo server. Anything you send it,
it will send back. The twist is that instead of spawning a new process or new
thread for each incoming connection, everything will be handled in the same
process and thread. This way, the thread is not wasting cycles checking if there
are new connections, or if there is new data on existing connections. As long as
there is work to do, the thread will do it. Asynchronous I/O might be a bit of a
misnomer in the title, and I'm mostly talking about non-blocking I/O. But I
might elaborate more in a future post and make this into a series.

A truly production-ready async service would be able to put things on
thread-pools, delegate tasks, not just I/O, and various other things you can see
in your favorite async frameworks. To keep the scope manageable, we'll just
stick to the description I gave above. Also most code in the article will skip
error-checking.

So the plan is to open a TCP socket in a non-blocking way, accept new
connections and make sure _those_ are non-blocking as well, and basically just
loop over our sockets when we receive data/a new connection. Non-blocking is
important. Otherwise we'd have to wait for data on each socket in turn, which is
fine if you spawn a thread/process for each socket. But for our use-case, we
want to move on to the next socket if the current one doesn't have any data yet.
Otherwise the whole system stalls.

{{< details summary="So just a `for`-loop, then?" >}}
Not quite. While it would _work_ to loop over all the open connections and the
listening socket to accept new connections, it's a waste of cycles. You're
basically just busy-waiting spinning in a loop asking _every_ socket if it has
data or not. The trick of these implementations is that we block and only wake
up _when we have something to do_. If there's nothing to do, we'll just wait
instead of running around doing pointless work.

Oh yeah, I'm also going to use these boxes for the socratic method at times.
{{< /details >}}

Also, this whole post will be in C, which is a detour from my usual Rust-based
antics. This is because I want to understand this stuff with as few abstractions
as possible. The only thing I will accept axiomatically is how the kernel
implements this stuff. It's somewhat straightforward to imagine the
implementation when seeing the usage anyway.

## One service, four programs
There are four different ways of achieving what I'm trying to do, and they all
revolve around their own system call. I'll go roughly in chronological order of
their availability (flipping `epoll` and `kqueue` simply because I'm less
familiar with BSD).

In all programs, I want the same echo service implementation, a simple way to
open a TCP socket, log info and error messages, and also handle some dynamic
array stuff. As such I created some helper files:

<!----------------------------------------------------------------------------->
{{< details name="helpers" summary="log.h" >}}
```c {linenos=inline}
#ifndef LOG_H
#define LOG_H

#include <stdio.h>
#include <time.h>

#define _LOG_IMPL(fmt, ...) do {       \
    struct timespec t;                 \
    clock_gettime(CLOCK_REALTIME, &t); \
    fprintf(                           \
        stderr,                        \
        "%lu.%09d " fmt "\n",          \
        t.tv_sec,                      \
        (int)t.tv_nsec                 \
         __VA_OPT__(,)                 \
        __VA_ARGS__ );                 \
} while (0)

#define INFO(fmt, ...) _LOG_IMPL("[INFO] " fmt __VA_OPT__(,) __VA_ARGS__)
#define WARN(fmt, ...) _LOG_IMPL("[WARN] " fmt __VA_OPT__(,) __VA_ARGS__)
#define ERRO(fmt, ...) _LOG_IMPL("[ERR!] " fmt __VA_OPT__(,) __VA_ARGS__)

#endif // LOG_H
```
This gives macros that print logs to stdout on the form
```
1789588582.670519197 [INFO] An informative message
```
Also using the C23 feature `__VA_OPT__`. Further, `clock_gettime` requires POSIX
standard to be at least 200809.
{{< /details >}}
<!----------------------------------------------------------------------------->
{{< details summary="dynarr.h" >}}
A dynamic array utility. A bunch of macros to operate on struct of the form
(pseudo-C with generics):
```c
struct DynamicArray<type> {
    size_t n;
    size_t capacity;
    type *elems;
};
```

Rather than paste the entire implementation, I'll just summarize the usage of
this header, since it will come up in the course of implementing each server:

```c
// Declare new array type called `type_name`, holding `type`s
arr_decl(type_name, type);
// Initalize an array of `type`s
arr_init(type);
// Free an array
arr_free(arr);
// Index the array
arr_at(arr, index);
// Get the raw data pointer, for passing to APIs
arr_raw(arr);
// Get the length
arr_len(arr);
// Push an element to the end
arr_push(arr, x);
// Pop an element from the end into `x`
arr_pop(arr, x);
// Concatenate two arrays
arr_concat(dst, src);
// Remove an element from the array when `expr` is true
arr_remove(arr, it, expr);
// Clear elements from array
arr_clear(arr);
// Helper macro to iterate over an array
arr_foreach(arr, it) { /* Do something with *it */ }
```
The implementation of this was pretty fun, and makes use of C23's `typeof`
feature.
{{< /details >}}
<!----------------------------------------------------------------------------->
{{< details summary="socket-util.h" >}}
One function opens a listening socket, the other accepts new connections on it.
```c {linenos=inline}
#ifndef SOCKET_UTIL_H
#define SOCKET_UTIL_H

#include "dynarr.h"

// Dynamic array of file descriptors (ints)
arr_decl(FdArr, int);

int init_socket(short port);
void accept_new_connections(int socket, FdArr *connections);

#endif // SOCKET_UTIL_H

{{< /details >}}
<!----------------------------------------------------------------------------->
{{< details summary="echo.h" >}}
Literally just a single function, but that's all an echo service needs to be! It
takes an open file descriptor, receives data on it, and sends it back. If
something went wrong, it'll close the file descriptor.
```c {linenos=inline}
#ifndef ECHO_H
#define ECHO_H

int echo(int fd);

#endif // ECHO_H
```
{{< /details >}}
<!----------------------------------------------------------------------------->
{{< details summary="test.sh" >}}
We need a way to test our servers. This script will open however many
connections we specify and for each connection it will write some data, receive
the answer, save to file, wait one second, then close. At the end the script
prints how many lines were saved to the file, i.e. how many connections
successfully managed to invoke the echo service.
```bash {linenos=inline}
#!/usr/bin/env/bash

set -euo pipefail

remote=${IP:-127.0.0.1}
port=${PORT:3000}

rm -f output.txt

for i in $(seq $1); do
    printf "Hello from %d\n" $i \
        | nc -w1 "${remote}" "${port}" \
        >>output.txt &
hone

wait

wc -l <output.txt

```
{{< /details >}}

The implementation can be found in the [companion
repo](https://github.com/martinkauppinen/async-echoes). When I was creating the
`init_socket()` function, I read in the man-pages that you can add the
`SOCK_NONBLOCK` flag to the call to `accept()` and make the socket non-blocking
immediately! This is part of POSIX.1-2024, the latest standard. If I add the
compiler definition `-D_POSIX_C_SOURCE=202405L`, I should get access to this
functionality and avoid some extra setup. _And_ since it's in POSIX, this
**should** _just work_ on BSD later on[^foreshadowing]! I needed at least
200809 anyway for the timestamps in my logging.

[^foreshadowing]: ***Foreshadowing*** is a literary device in which a writer
    gives an advance hint of what is to come later in the story.

I think that's all the setup needed. If you've seen the declarations in the
helper files above, this should be pretty easy to follow along with.

## Structure of each server
All servers will essentially be an infinite loop in which the first step is to
wait for something to happen on one of the sockets we're monitoring. In
pseudo-code, it looks something like this:

```c
init_socket();
open_connections = init_array();

while (1) {
    wait_for_events(&open_connections);

    if (event on listening socket) {
        accept_new_connections();
        add_new_connections(&open_connections);
    }

    for (connection in open_connection) {
        if (connection received data)
            echo(data);

        if (error on connection) {
            remove_connection(&open_connections);
            close(connection);
        }
    }
}
```
They will differ slightly in the setup and event handling, but in essence this
is all we will do. Send data back if we got some, close the connection if
_anything_ went wrong.


## In the beginning, there was `select()`
Our first stop is at the
[`select(2)`](https://man7.org/linux/man-pages/man2/select.2.html) system call.
Reading the man-page, it takes three sets of _descriptors_ (essentially just
which file descriptors/sockets we want to be notified of events on), a timeout
value which we'll set to wait forever, and an int describing the number of file
descriptors.

```c
int select(
    int nfds,
    fd_set *readfds,
    fd_set *writefds,
    fd_set *exceptfds,
    struct timeval *timeout
);
```

We don't really care about `writefds`, so that will be `NULL`. We'll just assume
that the socket is ready to accept writes if we get a read. We'll keep a dynamic
array of active open connections around, and use that to populate `readfds` and
`exceptfds`. These arguments correspond to which file descriptors we want to
monitor for the possibility to read data from them, and for exceptional events
(errors mostly). In our case we want to monitor all open connections for both.

That `nfds` argument is _not_ the number of file descriptors we're watching.
It's actually supposed to be set to the _highest-numbered_ file descriptor in
any of the three sets, plus one. Kind of a weird setup. According to the
man-page, the maximum this can be is defined by `FD_SETSIZE`, which in glibc is
defined to be 1024. So we will not be able to monitor more than 1024
connections. I'll just take a shortcut here and set it to the max and call it a
day.

Essentially, `fd_set` is just a bit string (with fixed size 1024 in glibc). We
set the bit corresponding to the number of a specific file descriptor we want to
monitor for an event. So if we have a connection open with file descriptor 5, we
should set bit 5 in our `readfds` argument. We do this for each active
connection we have at the start of each loop.

When `select()` eventually returns, bits will be set in the `readfds` and
`exceptfds` according to which event happens. The header provides the `FD_ISSET`
macro to check if a specific bit is set or not.

So our loop body will look something like this:
```c
// Reset the fd_sets
FD_ZERO(&readfds);
FD_ZERO(&exceptfds);

// Monitor the listening socket
FD_SET(socket_fd, &readfds);
FD_SET(socket_fd, &exceptfds);

// Monitor all active connections
arr_foreach(&active_connections, connection) {
    FD_SET(*connection, &readfds);
    FD_SET(*connection, &exceptfds);
}

// Wait for events
int result = select(
    FD_SETSIZE, &readfds, NULL, &exceptfds, NULL);

// Accept new connections if the listening socket
// received a read event
FdArr new_connections = arr_init(int);
if (FD_ISSET(socket_fd, &readfds))
    accept_new_connections(socket_fd, &new_connections);

// Check for data on each open connection
arr_foreach(&active_connections, connection) {
    if (!FD_ISSET(*connection, &readfds))
        continue;

    if (FD_ISSET(*connection, &exceptfds)) {
        // Remove connection later if failed
        arr_push(&to_remove, *connection);
        continue;
    }

    int ret = echo(*connection);
    if (ret < 0)
        arr_push(&to_remove, *connection);
}

// Add new connections for next loop
arr_concat(&active_connections, &new_connections);

// Remove failed connections
arr_foreach(&to_remove, connection) {
    arr_remove(&active_connections, x, *x == *connection);
    close(*connection);
}

```

This is... fine. It's a bit finicky to have to have to reset and set which file
descriptors we want to monitor in every loop iteration. And we have to loop
through _every_ open connection, even ones that didn't have any events for us to
handle. It feels very manual. But it works! running `test.sh 1000`, it handles
1000 incoming connections completely without issue. Running `test.sh 2000`,
however:

```
$ ./test.sh 2000
1020
```

Yeah there we can see that only 1020 connections made it through.

{{< details summary="Why not 1024?" >}}
Because every process has file descriptors 0 through 2 corresponding to stdin,
stdout, and stderr open. Furthermore, we have file descriptor 3 open in the
server as the listening socket. So that only leaves 1020 possible connections
before hitting the ceiling.
{{< /details >}}

And if we try to scale back and run 1000 connections again?
```
$ ./test.sh 1000
0
```
The server has completely locked up. Not great. But at least it works for a
small number of descriptors.

## Enter `poll()`

So `select()` works, but is pretty limited. We'd like to be able to monitor way
more connections than 1020. Maybe even 10'000 or so. The man-page for `select()`
on my Linux machine even says this explicitly:

> "All modern applications should instead use `poll(2)` or `epoll(7)`, which do
> not suffer this limitation."
>
> Linux man-pages 6.17, `select(2)`

So let's try that first suggestion: `poll(2)`! It's also specified in POSIX, so
we'll be able to use it in BSD-land. First order of business would be to look at
the function signature to get a feel for how to use it:

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

That looks a whole lot simpler than the signature for `select`! It's just a
pointer to a list of `struct pollfd`s, how big that array is, and a timeout
value in milliseconds which we will set to wait forever (-1).

The `struct pollfd` is a really simple structure:
```c
struct pollfd {
    int fd;         /* file descriptor */
    short events;   /* requested events */
    short revents;  /* returned events  */
};
```
For each `pollfd`, you populate the `.events` with a bitmask of events you want
to be notified of (provided by some nice helper macros like `POLLIN` for input),
and when `poll()` returns, the `.revents` field will have bits set according to
which events happened.

I like it! Much better than having three separate structures you have to juggle
with that get overwritten each loop so you have to reset them. With this
solution, you can tell the kernel _exactly_ which events you're interested in,
and only the `.revents` field is modified. Which events you've requested stays
the same each iteration unless _you_ change it.

{{< details summary="What happens if I don't set `.events`?" >}}
There are three bits in `.events` which are always set: `POLLERR`,
`POLLHUP`, and `POLLNVAL`. So if you have some logic for modifying `.events`
that happens to set it to zero, you'll never end up having a ghost file
descriptor that you can't access in your loop. Sure, you're not monitoring it
for any other meaningful events, but if it does error out you do get the chance to
close it and move on.
{{< /details >}}

With that, it's pretty simple to modify the loop body for `select()` to instead
use `poll()`. Structurally they're pretty much identical.

```c {linenos=inline}
// Wait
int poll_result =
    poll(arr_raw(&poll_fds), arr_len(&poll_fds), -1);

// Accept new connections if any arrived
if (arr_get(&poll_fds, 0).revents & POLLIN) {
    FdArr new_connections = arr_init(int);
    accept_new_connections(sock_fd, &new_connections);
    arr_foreach(&new_connections, fd) {
        // Set up new polling struct for new connection
        struct pollfd new_pollfd = {
            .fd = *fd,
            .events = POLLIN,
            .revents = 0
        };
        arr_push(&poll_fds, new_pollfd);
    }
}

// Echo on all connections
arr_foreach(&poll_fds, poll_fd) {
    // Skip uninteresting events
    bool is_listening_socket = (poll_fd->fd == sock_fd);
    bool is_read_event = (poll_fd->revents & POLLIN);

    if (is_listening_socket || !is_read_event)
        continue;

    int ret = echo(poll_fd->fd);
    if (ret < 0)
        arr_push(&to_remove, *connection);
}

// Check for failures
arr_foreach(&poll_fds, poll_fd) {
    if (poll_fd->fd == sock_fd)
        continue; // Ignore failures on listening socket

    // Any event other than POLLIN is an error
    bool failed = (poll_fd->revents & ~POLLIN) != 0;

    if (failed) {
        INFO(
            "Connection %d closing:%s%s%s",
            poll_fd.fd,
            (poll_fd.revents & POLLHUP) ? " POLLHUP" : "",
            (poll_fd.revents & POLLNVAL) ? " POLLNVAL" : "",
            (poll_fd.revents & POLLERR) ? " POLLERR" : ""
        );
        arr_push(&to_remove, poll_fd);
    }
}

// Remove connections that failed this loop
arr_foreach(&to_remove, poll_fd) {
    close(poll_fd->fd);
    arr_remove(&poll_fds, x, x->fd == poll_fd->fd);
}
arr_clear(&to_remove);
```

As you can see, we're doing basically the same thing. But instead of having a
`readfds` and an `exceptfds`, we just tell `poll()` that we want to monitor for
`POLLIN` (read) events, and we get `POLLERR` & co. (error) events for free. Then
we can just read `.revents` to see what happened and handle it accordingly. As
you can see on lines 27 and 42, it's just some bit masking against the events
we're interested in.

Alright, so now we have a more powerful server, without the problem of limited
sets of file descriptors we can monitor! Surely this can handle 10'000
connections at once, right?

```
$ ./test.sh 10000
1020
```

What? I thought the whole point of `poll()` over `select()` was that I could
have more file descriptors!?

After a bunch of wild goose chasing[^geese], thinking there was a bug in my
code, it turns out my code was fine. It's just that by default, Linux limits
the number of file descriptors you can have open to... 1024. So all I had to do
was increase that limit to something higher than 10'000 and...
[^geese]: Seriously, who let these all these geese out?

```
$ ./test.sh 10000
10000
```

Nice!

{{< details summary="How did you do that?" >}}
`setrlimit(2)` from `<sys/resource.h>`. It can modify resource limits for your
program. In my case, I needed to modify the resource limit `RLIMIT_NOFILE` to
let me have more than 1024 file descriptors.
```c
const struct rlimit new_limit = {
    .rlim_cur = 4096,   /* soft cap */
    .rlim_max = 16384   /* hard cap */
};
setrlimit(RLIMIT_NOFILE, &new_limit);
```
{{< /details >}}

So now we have essentially the same server, but not limited to 1024 file
descriptors. But we also have the same server in that we're manually keeping
track of `struct pollfd`s in a dynamic array, having to juggle memory, and loop
through _all_ open descriptors. Even if no events have happened on them. If only
someone had invented a better interface for this.

## In Linux-land: `epoll()`

Oh hey look, a better interface for this! `epoll(7)` is a Linux-specific
API for I/O events. Like `poll(2)`, you ask it to keep track of certain events
on certain file descriptors by bitmasking a field in a structure. But _the way_
it does it is a lot better. Let's, as is tradition by now, check the function
signature to get a feel for how it works. However, this is not one function, but
three. But let's start with the main one, most similar to `select()` and
`poll()` and we'll get to the other two when relevant.

```c
int epoll_wait(
    int epfd,
    struct epoll_event *events,
    int n,
    int timeout
);
```

`timeout` we'll set to -1 to wait forever as before. `events` and `n` are
obviously related in that `n` is the size of the `events` array. But this is not
an array we're setting up and keeping track of ourselves. It's actually a memory
space we provide to the function, which it will write events into _for us_. It
will contain _only_ file descriptors which have had events. So we're not
looping over absolutely everything, only what's necessary. The return value is
the number of events, which we can use for our looping purposes.

The first argument, `epfd`, is a special file descriptor created by
`epoll_create1()` which is one of the other two functions. The file
descriptors we want to monitor are registered to `epfd`, making it a sort of
handle for the group of monitored descriptors. We register them using the last
of our set of functions:

```c
int epoll_ctl(
    int epfd,
    int op,
    int fd,
    struct epoll_event *event
);
```

This function can add, remove, or modify file descriptors registered to `epfd`.
That's what the `op` argument is. `fd` is the descriptor we want to monitor. The
`event` argument contains the file descriptor again for some reason, and which
events we want to be notified of, analagous to `poll()`'s `struct
pollfd.events`.

And that's it! We don't have to keep track of the file descriptors ourselves. We
don't even have to remove them from a buffer or call `epoll_ctl` to remove them
when we're done. If we `close()` a file descriptor registered to `epfd` it will
_automatically_ be removed from `epfd` too! This means our loop body can contain
a single for-loop:

```c
// Wait
int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

// Loop over all events
for (int n = 0; n < nfds; n++) {
    struct epoll_event event = events[n];

    // Accept new connections if any arrived
    if (event.data.fd == sock_fd) {
        FdArr new_connections = arr_init(int);
        accept_new_connections(sock_fd, &new_connections);
        arr_foreach(&new_connections, fd) {
            ev.events = EPOLLIN | EPOLLRDHUP | EPOLLHUP;
            ev.data.fd = *fd;
            epoll_ctl(epfd, EPOLL_CTL_ADD, *fd, &ev);
        }
        arr_free(&new_connections);
        continue;
    }

    INFO("fd: %d, events:%s%s%s%s%s%s", event.data.fd,
            (event.events & EPOLLIN) ? " EPOLLIN" : "",
            (event.events & EPOLLOUT) ? " EPOLLOUT" : "",
            (event.events & EPOLLRDHUP) ? " EPOLLRDHUP" : "",
            (event.events & EPOLLPRI) ? " EPOLLPRI" : "",
            (event.events & EPOLLERR) ? " EPOLLERR" : "",
            (event.events & EPOLLHUP) ? " EPOLLHUP" : ""
    );

    // Handle non-listening event
    if (event.events & EPOLLIN) {
        // Result can be ignored, because echo()
        // already closes the fd and logs why
        // if there is an error
        (void)echo(event.data.fd);
    }

    // Any event other than POLLIN is an error
    bool failed = (event.events & ~EPOLLIN) != 0;
    // Remove connections that failed this loop
    if (failed)
        close(event.data.fd);
}
```

Now this was nice. One simple inner loop, events are put nicely into the
`events` buffer to be looped over, and when we're done with a file descriptor we
just `close()` it. No fuss, no buffer juggling, no looping over file descriptors
that haven't had anything happen. Very nice!

Another thing `epoll` is capable of is _edge-triggered_ events. Unlike `select`
and `poll`, which only support _level-triggered_ events. I won't go into detail
about that in this article (it's long enough as it is), but it's a pretty cool
concept. Basically we can subscribe for events to trigger every time they
happen, even if their event condition is currently fulfilled, such as a file
having data available to read. So if we haven't read the data already, we'll get
a second event when data is ready again. `select` and `poll` only tell us that
data is ready, not how many times it has been. Pretty cool, but out of scope for
this article.

## A journey into BSD: `kqueue()`

In my initial research of the previous three syscalls, I had heard whispers of a
fourth one, implemented in BSD-land: `kqueue()`. Being curious, I spun up a VM
with FreeBSD 15.1 so I could implement the echo service with this fourth syscall
as well.

### Detour: Getting *nix'ed

Since I knew that I wanted to write a `kqueue()` implementation of the echo
server as well, I figured I should write all my code to POSIX standard. That way
I figured I wouldn't have any problems with the BSD implementation, because the
supporting was supposedly portable. I did it by simply slapping
`_POSIX_C_SOURCE=202405L` into my Makefile's `CFLAGS` like so:

```make
CFLAGS=-std=c23 -D_POSIX_C_SOURCE=202405L

socket-util.o: socket-util.c socket-util.h log.h dynarr.h
echo.o: echo.c echo.h log.h
%-server.o: %-server.c log.h dynarr.h

%-server: %-server.o socket-util.o
	$(CC) $(CFLAGS) $^ -o $@
```

I needed that feature test macro for `clock_gettime` and `SOCK_NONBLOCK` anyway.
This was my entire Makefile when developing the first three echo services, on
Linux. Then I figured "Hey, I before I develop the `kqueue` version, let's just
compile the `select` and `poll` versions on FreeBSD to see them work there too!"

{{< details summary="`make select-server poll-server`" >}}

```
cc -std=c23 -D_POSIX_C_SOURCE=202405L select-server.c  -o select-server
ld: error: undefined symbol: init_socket
>>> referenced by select-server.c
>>>               /tmp/select-server-b477ff.o:(main)

ld: error: undefined symbol: accept_new_connections
>>> referenced by select-server.c
>>>               /tmp/select-server-b477ff.o:(main)

ld: error: undefined symbol: echo
>>> referenced by select-server.c
>>>               /tmp/select-server-b477ff.o:(main)
cc: error: linker command failed with exit code 1 (use -v to see invocation)
```
{{< /details >}}

No such luck. Loads of linker errors when running `make`. This was really
confusing initially, because I was _sure_ this Makefile worked. It worked when I
was on Linux, after all!

Oh.

BSD's `make` is different from the one on Linux, which is GNU `make`. Of course.
These Makefiles are written in different dialects and have different features.
Indeed, inspecting what the compiler command ended up being (and using `make`'s
debug printing functionality with the `-d` flag) the problem was clear. BSD
`make` doesn't have the [pattern
rules](https://ftp.gnu.org/old-gnu/Manuals/make-3.80/html_node/make_105.html)
that exist in GNU `make`. I usually use these to write shorter makefiles when I know
I'll have common prefixes/suffixes to multiple targets. So since I wasn't trying
to built a literal `%-server`, BSD couldn't find that rule and instead used the
standard fallback rule, trying to build `select-server` from _only_
`select-server.c`. Of course we're getting linker errors!

No matter, we can define each of our implementations in an array variable and
use BSD `make`'s `.for` loop! But then we won't be able to use the same Makefile
on both Linux and BSD. Hm. In the [companion repo](https://github.com/martinkauppinen/async-echoes) I'll include both a GNU and
BSD-flavored Makefile. But for the sake of complete portability I will _also_
include a POSIX-flavored one! Either way, let's try compiling with the updated
targets.

{{< details summary="`make select-server poll-server`" >}}

```
cc  -std=c23 -D_POSIX_C_SOURCE=202405L -c socket-util.c -o socket-util.o
socket-util.c:14:51: error: use of undeclared identifier 'SOCK_NONBLOCK'
   14 |     int socket_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
      |                                                   ^
1 error generated.
```
{{< /details >}}

So that also fails. Turns out `SOCK_NONBLOCK` is _not_ POSIX? At least not
according to FreeBSD. Searching through the system headers, that macro is
guarded behind an [`#ifdef
__BSD_VISIBLE`](https://cgit.freebsd.org/src/tree/sys/sys/socket.h#n108)
directive. Defining that in the Makefile _works_, but whenever you're dealing
with double underscores you're probably doing something weird. So instead of
doing it like this, I'll just make the socket non-blocking using `fcntl()` in a
separate step instead, with `O_NONBLOCK`.[^openbsd]

[^openbsd]: OpenBSD [_does_ expose
    `SOCK_NONBLOCK`](https://github.com/openbsd/src/blob/master/sys/sys/socket.h#L75)
    if POSIX 2024 is specified. So I guess FreeBSD is just a bit behind here.

{{< details summary="`make select-server poll-server`" >}}

```
cc  -std=c23 -D_POSIX_C_SOURCE=202405L -c select-server.c -o select-server.o
cc  -std=c23 -D_POSIX_C_SOURCE=202405L -c socket-util.c -o socket-util.o
cc  -std=c23 -D_POSIX_C_SOURCE=202405L -c echo.c -o echo.o
cc select-server.o socket-util.o echo.o -o select-server
cc  -std=c23 -D_POSIX_C_SOURCE=202405L -c poll-server.c -o poll-server.o
cc poll-server.o socket-util.o echo.o -o poll-server
```

{{< /details >}}

Success! Now we can get on with the final implementation. Curiously, glibc
doesn't guard this at all. Not with a specific POSIX version or anything. Even
though it _is_ POSIX as of 2024, I expected enabling POSIX mode lower than 2024
to cause glibc to complain as well.

### Back on track: Implementing with `kqueue()`

Like `epoll`, `kqueue` operates on a special descriptor, returned by the
`kqueue()` function, puts events into a memory buffer provided, and returns how
many events were triggered.

As per our time-honored tradition: let's check the function signature. The
special descriptor is created by `kqueue()` and events are waited for with the
`kevent()` function:

```c
int kevent(
    int kq,
    const struct kevent *changelist,
    int nchanges,
    struct kevent *eventlist,
    int nevents,
    const struct timespec *timeout
);
```
Essentially the same as `epoll_wait`, but with _two_ lists instead of one.
`eventlist` is basically the same as for `epoll_wait`. It's the list the events
will be put into that we loop over. The first one, `changelist`, essentially
bakes the functionality of `epoll_ctl` into the `epoll_wait` function. Any
changes are applied before filling the `eventlist`.

Now let's compare `struct kevent` to `struct epoll_event`. The `epoll` variant
had two fields: `.fd` and `.events`. Simple. The descriptor and events we're
interested in. How much difference can there be?


```c
struct kevent {
    uintptr_t ident;
    short     filter;
    u_short   flags;
    u_int     fflags;
    int64_t   data;
    void      *udata;
    uint64_t  ext[4];
};
```

_Woah!_

The explanation for all of these events is long and best explained by the [`kqueue(2)`
man-page](https://man.freebsd.org/cgi/man.cgi?query=kqueue&apropos=0&sektion=0&format=html).

The gist of it is that `kqueue` can register multiple kinds of event filters to
each file descriptor being monitored. Each filter can put an event in the
`eventlist` when triggered. But each filter can also have a whole lot of extra
configuration, making them extremely powerful. Seriously, open the linked
man-page and read the configuration you can do to the filters beginning with
`EVFILT_`.

For our purposes, we only care about the filter `EVFILT_READ`, which will
trigger an event when there is data to read on the file descriptor, analogous to
`POLLIN` and `EPOLLIN`. On each accepted connection, we'll add a new event to
the `changelist` with this filter set, and the `EV_ADD` flag to add it to the
kqueue. Essentially what we did by calling `epoll_ctl`. We don't really care
about any extra data returned in the event either, so it will be extremely
similar to the `epoll` implementation.

```c
// Wait
int n_events = kevent(
    kq,
    arr_raw(&changelist),
    arr_len(&changelist),
    arr_raw(&eventlist),
    arr_len(&eventlist),
    NULL
);
arr_clear(&changelist);

// Loop over all events
for (int i = 0; i < n_events; i++) {
    struct kevent event = eventlist[i];

    // Accept new connections if any arrived
    if ((int)event.ident == sock_fd) {
        FdArr new_connections = arr_init(int);
        accept_new_connections(sock_fd, &new_connections);
        arr_foreach(&new_connections, fd) {
            struct kevent new_event = {
                .ident = *fd,
                .filter = EVFILT_READ,
                .flags = EV_ADD,
                .fflags = 0,
                .data = 0,
                .udata = NULL,
                .ext = {0, 0, 0, 0},
            };
            arr_append(&changelist, new_event);
        }
        continue;
    }

    // Closed file descriptors are removed
    // from the kqueue automatically
    (void)echo(event.ident);

    // Any non-read event is an error
    if (event.flags != 0) {
        INFO("fd: %d, events:%s%s", event.ident,
            (event.flags & EV_ERROR) ? " EV_ERROR" : "",
            (event.flags & EV_EOF) ? " EV_EOF" : ""
        );
        close(event.ident);
    }
}
```

And that's all, folks! Very similar to `epoll` but oh, so much more powerful. I
have to say that this is probably my favorite version of them all. Everything
good about `epoll` is in `kqueue` (including edge-triggered events) and more.
Baking the `changelist` into the waiting call instead of having a separate
control call feels pretty nice and the sheer amount of filters and options on
the filters makes it hard to beat. Reading some of the options in the man-page
really makes you wonder how Linux can fare without it all.

## Other options?

There are other options than these. There is POSIX's `aio` and Linux's
`io_uring` to learn about as well. However to my understanding these are a bit
different to the syscalls covered here. So I'll leave them for another day.

On Windows you have IOCP. But I don't do stuff for Windows very much at all, so
I left that out of scope for this article.

## Summary

All in all, asynchronous I/O in this manner is pretty simple. The kernel does
all the dirty work of looping over the file descriptors and just tells us which
ones has events we're interested in. They work in pretty similar ways, but some
are clearly better than others. Which one should you use?

* `select()` -- Just don't use it. The limit on the number of file descriptors
makes it unusable. On BSD you _can_ raise it, but why bother when `poll()` and
`kqueue()` exist?
* `poll()` -- Use it if you want asynchronous I/O but have strict POSIX
requirements.
* `epoll()` -- Absolutely use it if you only need Linux support.
* `kqueue()` -- Absolutely use it if you only need BSD support.

And which one is my favorite? Well, I said it only a few paragraphs ago, but
`kqueue()` is the clear winner for me. 

The complete code for all these echo service implementations is in the
[companion repo](https://github.com/martinkauppinen/async-echoes).
