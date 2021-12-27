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

`Reactor`'s wait code:

```rust
// src/lib/reactor.rs
pub fn wait(&mut self, timeout: Option<Duration>) -> io::Result<()> {
    // poll for IO events, block until one event appears.
    self.poll.poll(&mut self.events, timeout)?;
    let mut guard = WAKER_MAP.lock(); // lock global waker map.
    let wakers_ref = guard.deref_mut();
    for e in self.events.iter() { // iterate the events
        // find token in waker map
        if let Some(waker_set) = wakers_ref.get_mut(&e.token()) {
            // wake up read waiting tasks
            if e.is_readable() && !waker_set.read.is_empty() {
                // drain all the wakers to clean the vec at the same time.
                waker_set.read.drain(..).for_each(|w| w.wake_by_ref());
            }
            // wake up write waiting tasks
            if e.is_writable() && !waker_set.write.is_empty() {
                waker_set.write.drain(..).for_each(|w| w.wake_by_ref());
            }
        }
    }
    Ok(())
}
```

## 3. Single-threaded runtime

The single-threaded runtime can be described briefly by the following graph:

![single thread model](https://imgur.com/wSG4WTr.png)

1. `Executor` will be started by `block_on` an async function, which spawn the task & start the main loop.
2. Main loop will receive the `block_on` task and other task spawned.
3. For every task received, it will run itself until the under lying future returns either `Poll::Pending` or `Poll::Ready(())`
4. After all the tasks are either sleeped or exited, `Executor` will block and wait `Reactor` finish waiting events
5. `Reactor` will wake up corresponding tasks according to the event it received
6. Tasks that is woke up will send itself to `Executor`

The actual main loop & `block_on` code:

```rust
// src/lib/single_thread.rs
fn run(&self) {
    // setup reactor
    let mut reactor = reactor::Reactor::new();
    reactor.setup_registry();
    loop {
        // try to receive any task it got in queue, non-blocking
        // Will get `mpsc::TryRecvError::Empty` when no task is in queue,
        // meaning that all the tasks are either finished or slept.
        match self.rx.try_recv() {
            Ok(msg) => match msg {
                // run task
                Message::Run(task) => task.run(),
                // received disconnect message, cleanup and exit.
                Message::Close => break,
            },
            Err(mpsc::TryRecvError::Empty) => {
                // mio wait for io harvest
                reactor.wait(None).unwrap();
            }
            // no one is connected, bye.
            Err(mpsc::TryRecvError::Disconnected) => break,
        }
    }
}
pub fn block_on<F>(&self, future: F)
where
    F: Future<Output = ()> + 'static + Send,
{
    spawn(future);
    self.run()
}
```

For the rest of the part (e.g.: struct definition, imports, misc functions), please refer to `src/lib/single_thread.rs` & `src/lib/reactor.rs`.
The code of these two files is 300- SLOCs, pretty short and simple to implement and understand.
The problematic part is when you need to extend it to multi-threaded runtime.

## 4. Multi-threaded runtime

I think we all agree that things will get more complex than you thought when it comes to multi-threading.
At least it's totally true when writing an async runtime.

The first problem we'll encounter is task scheduling. How would you distribute tasks to all the worker threads has always been a problem.
Plus, different strategy suits for different scenarioes, it's hard to decide what to use.

The second problem is you have take atomic actions, locking problem, and starvation into consideration.
While a simple round-robin or hash distribution seems fair enough, but that's fair only on schedule time.
In reality, threads may starve for tasks, this will cause problem when we're designing a server software that may
encounter C10K problem, where you definitely don't want a thread just chill and do nothing. Also, atomic actions and locking
are pretty expensive, avoiding constantly doing them is also a hard problem.

I've designed two schedulers: the simple Round Robin scheduler and the more complicated Work Stealing scheduler.
I'm not very good at multi-threading, so if you guys think there's a better way, don't hasitate and tell me your thoughts,
simply launch an issue is a huge help.

### 4-1. The execution model

The brief execution model can be described by this graph:

![multi thread model](https://imgur.com/DQgJcZG.png)

1. `Spawner` spawns multiple tasks and send to the `Scheduler`
2. `Scheduler` will then schedule the tasks use chosen strategy to distribute the task to each worker threads
3. Each worker thread will run all the tasks until there's no tasks to run, then each of them will wait wakeups or task schedule
4. At the mean time, the poll thread will make `Reactor` to continously poll for readiness events and wake up tasks
5. It's up to the woke up tasks to wake up at the worker threads or to reschedule itself

### 4-2. The Scheduler trait and the Executor

To make us easy to swith or implement scheduling strategies, we defined a public `Scheduler` trait:

```rust
// src/lib/schedulers/mod.rs
pub trait Scheduler {
    fn init(size: usize) -> (Spawner, Self);
    fn schedule(&mut self, future: BoxedFuture);
    fn reschedule(&mut self, task: Arc<Task>);
    fn shutdown(self);
    fn receiver(&self) -> &Receiver<ScheduleMessage>;
}
```

This trait defined the needed behaviours that a `Scheduler` must satisfy to run with the `Executor`.
Take a look at the `Executor`'s code:

```rust
// src/lib/multi_thread.rs
impl<S: Scheduler> Executor<S> {
    pub fn new() -> Self {
        // get number of current cpu cores
        let cpus = num_cpus::get();
        let size = if cpus == 0 { 1 } else { cpus - 1 };
        // setup scheduler & spawner
        let (spawner, scheduler) = S::init(size);
        let (tx, rx) = channel::unbounded();
        // setup global SPAWNER
        SPAWNER.lock().deref_mut().replace(spawner);
        // create the poll thread
        let poll_thread_handle = thread::spawn(move || Self::poll_thread(rx));
        Self {
            scheduler,
            poll_thread_notifier: tx,
            poll_thread_handle,
        }
    }
    fn poll_thread(rx: Receiver<()>) {
        // poll thread will continously block until any rediness event
        // it will exit automatically when error occurs or shutdown notify.
        let mut reactor = reactor::Reactor::new();
        reactor.setup_registry();
        loop {
            match rx.try_recv() {
                // exit signal
                Ok(()) | Err(TryRecvError::Disconnected) => break,
                Err(TryRecvError::Empty) => {}
            };
            // blocking wait for readiness events
            if let Err(e) = reactor.wait(None) {
                eprintln!("reactor wait error: {}, exit poll thread", e);
                break;
            }
        }
    }
    fn run(mut self) {
        // The main loop will continously schedule tasks
        // until shutdown or error.
        loop {
            // blocking receive schedule messages
            match self.scheduler.receiver().recv() {
                // continously schedule tasks
                Ok(msg) => match msg {
                    ScheduleMessage::Schedule(future) => self.scheduler.schedule(future),
                    ScheduleMessage::Reschedule(task) => self.scheduler.reschedule(task),
                    ScheduleMessage::Shutdown => break,
                },
                Err(_) => {
                    eprintln!("exit...");
                    break;
                }
            }
        }
        // shutdown worker threads
        self.scheduler.shutdown();
        // shutdown poll thread
        self.poll_thread_notifier
            .send(())
            .expect("Failed to send notify");
        let _ = self.poll_thread_handle.join();
    }
    // The `block_on` function
    pub fn block_on<F>(self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        spawn(future);
        self.run();
    }
}
```

The logic of the `Executor` is similliar to the single-threaded runtime, except it needs to spawn a polling thread and the
tasks is scheduled to the worker threads.
However, the scheduler's implementation is up to you to spawn worker threads and schedule the tasks.

I've made two scheduler implementations, `RoundRobinScheduler` and `WorkStealingScheduler`.

Note that when implementing both schedulers, I use `crossbeam`'s channel rather than the `std::mpsc` one, for multiplexing channels and broadcasting.

### 4-3. The RoundRobinScheduler

This scheduler is a pretty simple one: it just round through all the worker threads and spread it evenly when scheduling,
but there's no guarantee that the workload of each worker thread is fair.

The code is relatively easy to understand, so I won't explain further, just look at the [code](https://github.com/smb374/thread-poll-server/blob/main/src/lib/schedulers/round_robin.rs).
