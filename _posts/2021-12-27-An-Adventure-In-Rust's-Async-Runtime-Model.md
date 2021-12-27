---
layout: post
---

# Abstraction

In Rust, we need to invent our self a async runtime because that Rust's team decides not to include
it in `std`, as it will nearly double the size of its code. We'll need to invent our runtime & provide
a library in order to use this feature. At this point, there are [`tokio-rs`](https://tokio.rs/) and
[`async-std`](https://async.rs/) for you to use directly. Both have reach production level quality and
pretty easy to use, especially, `tokio-rs` has been used widely in production.

While we have some awesome runtime libraries, it makes me wonder that how do those runtime actually work
and how does the runtime model look like. You know, sometimes we just want to DIY some cool stuff, so here
I am.

The code of this project can be found at [`smb374/thread-poll-server`](https://github.com/smb374/thread-poll-server).

## 1. The Model

The execution model is rather simple to explain (but we all know simple stuff can be hard to do well):

![async model overview](https://imgur.com/04f1nBO.png)

1. Async functions can be `spawn`ed using a global `Spawner`.
2. `Spawner` will wrap up the function to a `Task`, which will contain the boxed function.
3. `Spawner` will then notify the `Executor` to run it. Each task spawned will be queued by notification's order.
4. `Executor` will continue run tasks in queue until all the task have been reached its end state, `Pending` or `Ready(T)`.
   - `Pending` tasks will **sleep** until someone or the `Reactor` wakes it up.
   - `Ready(T)` means the task produced a return value of type `T`, the task will end and return.
5. `Executor` will notify `Reactor` to wait readiness events. Once it get some events, `Reactor` will requeue the corresponding tasks to `Executor`'s queue.

Sounds simple, right? But there are quite a lot of pitfalls that can make you shoot yourself in your foot:

- Global variables are simply a headache when writing rust, because the type of Global variables need to satisfy `Send + Sync`, which is decided by the compiler.
- Even if you're in single thread, you still need to take locking & synchronization into consideration because Rust can't tell whether you're multi-threading or not, it assumes all the program you do is multi-threaded.
- Rust only provides `Future` trait, meaning that you'll have to implement all this stuff.
- Not even this Model is a standard, there's no standardized model or procedure on how to run a async task, plus `Future` is lazy.
- etc.

There are different ways to drive the lazy `Future` to run, but this concept is used by most of the runtimes (I guess).

In the following sections I'll talk about my design approach on `Reactor`, single-threaded runtime,and multi-threaded runtime.

## 2. Reactor

`Reactor` takes the job of getting rediness events and wakeup corresponding tasks.
Mostly, these kind of events are tighted with IO events or timer events.
I've only implement the IO reactor with the help of `mio` library, which is a wrapper of the OS's IO multiplexing solution.

The only thing it need to do is to wait events and wake up corresponding tasks, that's all. A global `Registry` will accept
registration and deregistration to the reactor, and a global map will accept adding/removing wakers corresponding to the event.

Note that the `poll` method can only be accessed with mutable reference, which Rust guarentees that the will only be at most one mutable
reference access at the sametime, we'll have to separate a polling thread for `Reactor` in multi-threaded runtime.

## 3. Single-threaded runtime

The single-threaded runtime can be described briefly by the following graph:

![single thread model](https://imgur.com/F6R3Ykz.png)
