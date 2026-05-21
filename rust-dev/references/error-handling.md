# Error handling

Read this when the code parses input, does I/O, returns `Result`, or `unwrap`s
something that can fail. The guiding principle: **errors are values; report them,
don't discard or panic on them.**

## `?` is the manual match, written once

`?` desugars to exactly the early-return you'd write by hand:

```rust
// these are equivalent
let file = match File::open(path) { Ok(f) => f, Err(e) => return Err(e.into()) };
let file = File::open(path)?;
```

Whenever you see `match ... Err(e) => return Err(e)`, replace it with `?`. The
`.into()` it inserts is why `From` impls (below) make `?` compose across error
types.

Before reaching for `?`, check whether std already does the whole job:
`fs::read_to_string`, `fs::write`, `str::parse`, `split_once` — many manual
open/read/close or split/count sequences collapse to one call.

## `unwrap` / `expect` — when they're acceptable

- **Tests and examples** — failure should abort the test loudly.
- **Truly impossible cases** — and then use `expect("why this can't fail")`, never
  a bare `unwrap()`. The message documents the invariant for the next reader.
- **Prototypes** — but mark them; don't let them reach a library API.

In anything a caller depends on, return `Result`/`Option` instead. A panic is not
an error-handling strategy for library code.

## Don't silently discard errors

Throwing away *why* something failed is its own bug class
([Bugs Rust Won't Catch](https://corrode.dev/blog/bugs-rust-wont-catch/)):

- `.ok()` drops the error, turning `Result` into `Option`.
- `.unwrap_or_default()` masks failure as a default value.
- `let _ = fallible();` discards it entirely.

Each is fine *only* when you genuinely don't care whether it failed. Otherwise
propagate with `?` or handle it. And remember: on untrusted input a panic
(`unwrap`/`expect`/indexing/overflow) is a denial-of-service — prefer `?`,
`.get`, `checked_*`. Enforce with `clippy::unwrap_used`, `clippy::expect_used`,
`clippy::indexing_slicing`.

## `Option` vs `Result` for fallible parsing

`Option` says *whether* it failed and throws away *why*. That's fine for
genuinely binary questions ("is there a first char?"). For parsing, the caller
usually wants the reason — prefer `Result` with a small error type:

```rust
// Loses information:
pub fn parse_port(s: &str) -> Option<u16> { /* abc? 99999? 0? all just None */ }

// Honest:
pub fn parse_port(s: &str) -> Result<NonZeroU16, InvalidPort> { /* ... */ }
```

Bridges between the two:

- `option.ok_or(err)?` / `option.ok_or_else(|| ...)?` — `Option` → `Result`.
- `result.ok()` — `Result` → `Option` (discarding the error).
- `result.map_err(|e| MyError::from(e))?` — convert the error type, then `?`.

## Custom error enums

For a library or any nontrivial module, define an enum that names each failure
mode. This lets callers `match` on what went wrong:

```rust
#[derive(Debug)]
pub enum ParseError {
    BadFormat,
    BadNumber(ParseIntError),
}

// Implement From so `?` auto-converts the inner error:
impl From<ParseIntError> for ParseError {
    fn from(e: ParseIntError) -> Self { ParseError::BadNumber(e) }
}

pub fn parse(s: &str) -> Result<Duration, ParseError> {
    let (h, rest) = s.split_once(':').ok_or(ParseError::BadFormat)?;
    let secs = h.parse::<u64>()?;   // ParseIntError → ParseError via From
    // ...
}
```

Keep the variant set tight. Two variants (`Invalid` / `Zero`) is often enough;
split into finer variants (`NotANumber` / `OutOfRange`) only when callers
actually need to distinguish them.

Carry context that helps the caller locate the problem — e.g. a line number for a
file parser:

```rust
pub struct ParseError { pub line: usize, pub kind: ErrorKind }
```

For a full standard `Error`, implement `Display` and `std::error::Error`:

```rust
impl std::fmt::Display for ParseError { /* human-readable message */ }
impl std::error::Error for ParseError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self { ParseError::BadNumber(e) => Some(e), _ => None }
    }
}
```

## The ecosystem crates (defaults for real projects)

Writing `Display`/`Error`/`From` by hand is fine for std-only code and good for
understanding. In a real project, reach for:

- **`thiserror`** — for *libraries*. Derives `Error`, `Display`, and `From` for
  your custom enum with attributes. You still expose a typed error API.

  ```rust
  #[derive(thiserror::Error, Debug)]
  pub enum ParseError {
      #[error("bad format")]
      BadFormat,
      #[error("bad number")]
      BadNumber(#[from] ParseIntError),  // generates From + source
  }
  ```

- **`anyhow`** — for *applications* / binaries, where you mostly want to
  propagate and report rather than match. `anyhow::Result<T>` plus `.context(...)`
  to add breadcrumbs:

  ```rust
  use anyhow::Context;
  let config = fs::read_to_string(path)
      .with_context(|| format!("reading config from {path}"))?;
  ```

Rule of thumb: **`thiserror` for libraries (typed errors callers match on),
`anyhow` for binaries (propagate + context).** Don't expose `anyhow::Error` in a
library's public API.

## Collecting fallible iterators

`collect` short-circuits on the first error when the item type is `Result`:

```rust
let nums: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse()).collect();
```

To instead keep the successes and ignore failures, use `filter_map(Result::ok)`.

To keep *both* sides — process the oks and report the errs — partition them.
`itertools::partition_map` does it in one pass with the values already unwrapped:

```rust
use itertools::{Itertools, Either};
let (oks, errs): (Vec<i32>, Vec<MyError>) = results.into_iter()
    .partition_map(|r| match r {
        Ok(v)  => Either::Left(v),
        Err(e) => Either::Right(e),
    });
```

(std's `Iterator::partition` works too, but leaves each side as `Result`s you must
still unwrap.)

## Panic safety

A panic *unwinds* — it can fire partway through an operation (often inside a
user-supplied closure) and abandon it mid-flight. Don't leave a data structure in a
broken state across a point that might panic ([Rustonomicon: exception safety](https://doc.rust-lang.org/nomicon/exception-safety.html)):

- Do the risky/foreign step (e.g. call the closure) *before* mutating shared state,
  or stage changes and commit them only after it succeeds.
- This is exactly why `std::sync::Mutex` *poisons* on a panic-while-locked: the data
  may be inconsistent. Handle `lock()`'s `Result` (or use a non-poisoning lock like
  `parking_lot`) deliberately rather than blindly `unwrap`-ing it.

## Checklist

- [ ] No `unwrap()`/`panic!`/`expect()` on fallible paths in library code
      (tests excepted; impossible cases use `expect("reason")`)
- [ ] `?` instead of manual `match ... return Err(e)`
- [ ] No errors silently discarded (`.ok()` / `.unwrap_or_default()` / `let _ =`)
      unless failure genuinely doesn't matter
- [ ] Fallible parsers return `Result<_, E>` with a named error type, not bare
      `Option`, when the caller would want the reason
- [ ] `From` impls (or `#[from]`) wired so `?` composes across error types
- [ ] Errors carry enough context to locate the failure
- [ ] `thiserror` for library error types, `anyhow` + `.context` for binaries
