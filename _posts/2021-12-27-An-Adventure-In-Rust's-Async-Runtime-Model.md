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
