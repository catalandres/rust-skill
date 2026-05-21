# Async & concurrency

Read this when code uses `async`/`await`, spawns threads, or shares mutable state
across threads/tasks. Rust prevents *data races* at compile time (via `Send`/
`Sync`), but deadlocks, blocking the runtime, and cancellation bugs are still on
you.

Sources: curated entries from [mre/idiomatic-rust](https://github.com/mre/idiomatic-rust),
[Rust Atomics and Locks](https://marabos.nl/atomics/) (Mara Bos), and the
[Rayon](https://docs.rs/rayon) docs.

## Threads vs async — pick by workload

- **Threads** (`std::thread`, `rayon`): CPU-bound work, blocking syscalls, a
  bounded number of long-lived workers. Simple mental model.
- **async/await** (`tokio` is the default runtime): IO-bound work with high
  concurrency (thousands of in-flight connections). Tasks are cheap; the runtime
  multiplexes them onto a few threads.
- Don't reach for async just because it exists. A handful of CPU-bound jobs is
  simpler and faster with threads or Rayon.

## Don't block the async runtime

An async task runs cooperatively on a shared thread pool — blocking one task
starves the others.

- No `std::thread::sleep`, blocking file/network IO, or heavy CPU loops inside an
  `async fn`. Use async equivalents (`tokio::time::sleep`, `tokio::fs`) or offload
  blocking/CPU work with `tokio::task::spawn_blocking` (or a Rayon pool).
- A long *synchronous* computation between `.await`s blocks the thread too — break
  it up or move it off the runtime.

## Cancellation safety

A future can be dropped at any `.await` point (a timeout fires, a `select!` branch
loses, the caller stops polling). Everything after that `.await` simply never runs.

- Don't leave invariants half-updated across an `.await`. Mutate, *then* await — or
  make the update atomic.
- Watch "take from a buffer → await → put it back" shapes: if cancelled at the
  await, the data is gone. (See "Cancelling async Rust".)

## Don't hold a lock across `.await`

Holding a `std::sync::Mutex` guard across an `.await` can deadlock and makes the
future `!Send`. Either:

- Drop the guard before awaiting — `{ let v = m.lock().unwrap(); use(v); }` then
  `.await` — or
- Use an async-aware mutex (`tokio::sync::Mutex`) when the critical section
  genuinely must span awaits. Prefer shrinking the critical section first.

## Sharing state: message passing vs shared memory

- **Prefer message passing** (channels: `std::sync::mpsc`, `tokio::sync::mpsc`,
  `crossbeam`) where it fits. "Share by communicating" sidesteps lock contention
  and most deadlocks.
- **Shared memory** when you need it: `Arc<Mutex<T>>` / `Arc<RwLock<T>>` for shared
  *mutable* state, `Arc<T>` for shared *immutable*. Use `RwLock` only when reads
  vastly outnumber writes.
- **Atomics** (`AtomicUsize`, `AtomicBool`, …) for simple counters/flags — no lock
  needed. See [Rust Atomics and Locks](https://marabos.nl/atomics/) for the
  memory-ordering details.

## Send / Sync / 'static

- `tokio::spawn` / `thread::spawn` require the closure and everything it captures to
  be `Send + 'static` — the task may move between threads and outlive the spawner.
  So: `Arc` (which is `Send`) over `Rc` (which isn't), and owned data or `Arc` over
  borrows.
- If `Send + 'static` bounds fight you, consider structured concurrency: scoped
  threads (`std::thread::scope`) for borrowing local data, or `tokio::task::LocalSet`
  / a single-threaded runtime for `!Send` futures. (See "Async Rust without
  Send + Sync + 'static".)

## CPU-bound parallelism: Rayon

For data-parallel CPU work, `rayon` turns a sequential iterator parallel with a
one-word change:

```rust
use rayon::prelude::*;
let total: u64 = items.par_iter().map(expensive).sum();
```

It work-steals across cores. Reach for it instead of hand-rolling threads for "do
this to every element."

## Checklist

- [ ] Workload matched to the model (threads/Rayon = CPU-bound, async = IO-bound)
- [ ] No blocking calls or heavy CPU inside `async fn` (use `spawn_blocking`/Rayon)
- [ ] No lock guard held across `.await`; critical sections kept small
- [ ] Invariants not left half-updated across an `.await` (cancellation-safe)
- [ ] Shared state via channels where possible; `Arc<Mutex/RwLock>` or atomics otherwise
- [ ] Spawned tasks/threads satisfy `Send + 'static` via `Arc`/owned data, not borrows
