---
name: rust-dev
description: >
  Write, refactor, and review idiomatic, maintainable Rust. Apply Rust idioms —
  Option/Result combinators, iterator pipelines instead of manual loops, the `?`
  operator, FromStr/TryFrom parsers, custom error enums, newtypes, and "make
  illegal states unrepresentable" — and run fmt/clippy/test before finishing.
  Use whenever writing new Rust, cleaning up awkward or verbose Rust, fixing
  ownership/borrowing or error-handling smells, or reviewing a Rust diff — even
  if the user never says "idiomatic" or names a specific pattern.
license: MIT
metadata:
  source: Distilled from corrode/refactoring-rust (Matthias Endler) + Agent Skills best practices
  rust-edition: "2021/2024"
---

# Rust development

Write Rust the way an experienced Rustacean would: lean on the type system,
prefer combinators and iterators over manual control flow, push invariants into
types, and report errors honestly. This skill is the distilled form of a
refactoring workshop — apply it both when writing new code and when improving
existing code.

## Keep it simple (the default)

Idiomatic does not mean clever. Reach for the simplest thing that works and reads
well; add abstraction only when a concrete second use case forces it.

- Don't add generics, traits, or lifetimes for hypothetical future needs. A plain
  `&str` parameter beats `impl AsRef<str>` + lifetimes until you actually need it.
- "Performance crimes" (an extra `clone`, iterating twice) are fine in cold paths.
  Profile before optimizing — elegant code is easier to make fast than the reverse.
- Keep the common-case API obvious; push advanced knobs out of the default path.
- If a type's doc comment needs to say "X **and** Y", it probably does too much.
- Prefer functions and generics over macros. Reach for a `macro_rules!` or
  proc-macro only when there's genuinely no other way — macros are harder to read,
  document, and debug.
- Rust is multi-paradigm: functional pipelines for data transforms, plain
  imperative code for hot or hardware-level paths, structs/enums + traits
  (composition, not inheritance) for structure. Pick per situation.

Sources: [Be Simple](https://corrode.dev/blog/simple/),
[Navigating Paradigms](https://corrode.dev/blog/paradigms/).

## The validation loop (do this every time)

Rust's tooling catches most non-idiomatic code for you. After any change to Rust
code, run — in this order — and fix what they report before moving on:

```bash
cargo fmt                       # canonical formatting, non-negotiable
cargo clippy --all-targets -- -D warnings   # treat lints as errors
cargo test                      # behavior is preserved
```

Clippy is the single most effective idiom teacher in the ecosystem — most rules
in this skill are things clippy will flag (browse the full
[lint list](https://rust-lang.github.io/rust-clippy/master/)). **Never silence a lint with `#[allow(...)]`
to make it pass; fix the underlying code.** Only allow a lint when you can state
why it's a deliberate false positive.

For stricter, defensive codebases, opt into extra lints (`warn` or `deny`):
`clippy::unwrap_used`, `clippy::expect_used`, `clippy::indexing_slicing`,
`clippy::wildcard_enum_match_arm`, `clippy::fn_params_excessive_bools`,
`clippy::fallible_impl_from`. See
[references/domain-modeling.md](references/domain-modeling.md) for what each buys
you ([Defensive Programming](https://corrode.dev/blog/defensive-programming/)).

Enforce `-D warnings` in **CI** (e.g. `RUSTFLAGS="-D warnings"`), not by baking
`#![deny(warnings)]` into crate source — a source-level deny breaks the build
whenever a new compiler version adds a lint or deprecates an API
([anti-pattern](https://rust-unofficial.github.io/patterns/anti_patterns/deny-warnings.html)).

When refactoring, the test suite is your safety net: refactors must not change
behavior. If tests don't exist for the code you're changing, add characterization
tests *first*, then refactor.

`cargo test` also runs doc-tests: the `///` examples on your public API are
compiled and executed, so they double as documentation that can't go stale (write
more than unit tests — Effective Rust Item 30).

## Universal rules — reach for these constantly

These are the highest-frequency idioms. They apply to almost every function you
touch.

**Signatures & types**
- Take `&str` not `String`, `&[T]` not `&Vec<T>`, `&T`/`Option<&T>` not `&Option<T>`
  for read-only parameters. Borrow at the API boundary; let callers keep ownership.
- Return `Option`/`Result` instead of sentinel values or panicking, in any code a
  caller depends on. Reserve `unwrap`/`expect` for tests, prototypes, and
  truly-impossible cases (and then `expect` with a reason, never bare `unwrap`).
- Parse into the *narrowest* type: parse a port straight into `u16`, not `i32`
  then range-check. Let the type erase whole classes of validation.

**Control flow**
- Replace `if cond { true } else { false }` with `cond`; drop `== true`.
- Collapse pyramids of nested `match`/`if let` on `Option`/`Result` into
  combinators (below) or a `let ... else { return ... }` early-out.
- Use `match` (exhaustive) over `if/else if` ladders on an enum or over ranges
  (`100..=199 => ...`). Prefer listing every variant of *your own* enums (no `_`
  catch-all) so adding a variant is a compile error you must handle; reserve `_`
  for open/numeric matches and foreign types.
- Nest or-patterns inside constructors: `Some(2 | 3 | 5 | 7)`, not
  `Some(2) | Some(3) | ...`.
- Rust is expression-oriented: let `if`/`match`/`loop` *produce* the value
  (`let x = if c { a } else { b };`, `let n = loop { ... break v; };`) and assign
  block results (`let data = { let mut v = ...; v.sort(); v };`) instead of
  pre-declaring a `mut`. Drop the trailing `return` on a block's last expression.

**Option / Result**
- `n.map(|x| x + 1)` over `match n { Some(x) => Some(x+1), None => None }`.
- `opt.is_some_and(|x| pred)` / `opt.map_or(default, f)` to collapse a
  match-with-two-arms into one line.
- `.filter(pred)` to turn a `Some` you don't want into `None` declaratively.
- `?` instead of `match ... Err(e) => return Err(e)`. That match *is* what `?`
  expands to.
- `opt.ok_or(err)?` / `res.map_err(convert)?` to bridge between `Option` and
  `Result`.

**Iterators (loops are usually a smell)**
- A `for` loop that accumulates is almost always `.map().sum()` / `.fold()` /
  `.max_by_key()` / `.filter().collect()`. No `mut` accumulator needed.
- `Iterator<Item = Result<T, E>>` collects into `Result<Vec<T>, E>` and
  short-circuits on the first error: `items.iter().map(parse).collect()`.
- `Option<T>` is iterable: `iter.flatten()` drops the `None`s; `filter_map` is
  the explicit form.
- `split_once('=')` over `split('=').collect::<Vec<_>>()` + length check (no
  allocation, gives you both halves).
- `split_whitespace()` over `split(' ')` — handles tabs and runs of spaces.
- `strip_prefix` / `strip_suffix` return `Option<&str>` = "matched, here's the
  rest". Use instead of manual slicing.
- Balance, though: combinators win when they read top-to-bottom. A deeply nested
  `opt.map_or(d, |x| ...).then_some(...)` chain is worse than a plain `match` —
  optimize for the next reader ([Thinking in Expressions](https://corrode.dev/blog/expressions/)).

**Data structures & cost**
- `Vec::contains` in a loop is O(n²). A `HashSet` gives O(1) membership; build
  the lookup table *once*, outside the loop, and pass it in.
- Prefer `to_ascii_lowercase`/`to_ascii_uppercase` over the Unicode `to_lowercase`
  when the data is ASCII (cheaper, no surprise allocations mid-comparison).
- Use `entry(k).or_insert_with(...)` for the counting/grouping idiom.

## Idiom catalog — load when refactoring or reviewing

For the full annotated before→after catalog (27 patterns from warm-ups through
capstones, with the reasoning behind each), read
[references/idioms.md](references/idioms.md). Load it when you're cleaning up a
nontrivial chunk of Rust or doing a focused review, rather than holding all of it
in context for a one-line tweak.

For deeper, situational guidance, load the matching file on demand:

- **Errors, `?`, `FromStr`, custom error types, `thiserror`/`anyhow`** →
  [references/error-handling.md](references/error-handling.md). Read it when the
  code parses input, does I/O, returns `Result`, or `unwrap`s things that can
  fail.
- **Parse-don't-validate, newtypes, illegal-states-unrepresentable, enums over
  bool soup, `TryFrom` at boundaries, extension traits, state machines** →
  [references/domain-modeling.md](references/domain-modeling.md). Read it when a
  function takes several `bool`s/strings, when "config" or "command" is modeled
  as a stringly-typed map, or when you're tempted to validate the same value
  repeatedly.
- **Filesystem races, privilege boundaries, untrusted-input safety, `unsafe`** →
  [references/security.md](references/security.md). Read it when code touches the
  filesystem with attacker-influenced paths, runs with elevated privileges,
  parses untrusted input/bytes, or uses `unsafe` — Rust's memory safety does *not*
  cover these.
- **Public API / library / crate design** →
  [references/api-design.md](references/api-design.md). Read it when designing a
  public API or publishing a crate: which traits to derive, naming and conversion
  conventions, builders, generics vs trait objects, future-proofing, and
  dependency hygiene.
- **Async, threads, and shared state** →
  [references/async-concurrency.md](references/async-concurrency.md). Read it when
  code uses `async`/`await`, spawns threads, or shares mutable state — blocking the
  runtime, deadlocks, and cancellation bugs aren't caught by the compiler.
- **FFI (C interop) and `no_std`/embedded** →
  [references/beyond-std.md](references/beyond-std.md). Read it only when calling C
  / exposing a C ABI, or targeting environments without the standard library.

## `unsafe` requires explicit human sign-off

`unsafe` switches off the guarantees this entire skill is built on. Treat it as a
last resort, never a convenience — and never a borrow-checker escape hatch.

- **Never introduce `unsafe` on your own initiative.** If you believe it's truly
  necessary, stop and ask the human. Ask **up to three times**, presenting the safe
  alternative each time, before writing a single line of `unsafe`. Only proceed with
  explicit human approval.
- **In every code review, flag every line of `unsafe` as a warning** — call out each
  `unsafe` block/fn explicitly, even when it looks correct, so a human always
  re-examines it.

Safe-isolation rules (and panic safety, `transmute`) live in
[references/security.md](references/security.md); the
[Rustonomicon](https://doc.rust-lang.org/nomicon/) is the reference *if* you must
write it.

## Gotchas — Rust traps that defy reasonable assumptions

- **`String::truncate` / slicing panics on a non-char-boundary.** `truncate("café", 4)`
  panics — byte 4 is inside `é`. Back up with `is_char_boundary`, or use char
  semantics (`s.chars().take(n).collect()`). Decide which the spec wants.
- **`&Option<T>` is almost always wrong.** Use `Option<&T>` (caller flexibility),
  or just `&T`, or — frequently — let an *empty collection* mean "none configured"
  and drop the `Option` entirely.
- **`Iterator::max`/`min` return `Option`.** A hand-rolled loop returning `0` on
  empty input becomes `.max().unwrap_or(0)` — don't silently drop the empty case.
- **`unwrap()` on `parse()` panics on bad input.** In anything library-like,
  return `Result` and `?`. `Option`-returning parsers throw away *why* it failed;
  a small error enum is usually more honest.
- **Multiple `bool` parameters encode impossible states.** `(is_authenticated:
  false, is_admin: true)` is nonsense the type system permits. Replace with an
  enum so illegal states can't be constructed.
- **Silent default-on-parse-failure hides operator errors.** `cfg.get("MAX").parse().unwrap_or(100)`
  turns `"banana"` into `100`. Parse once at the edge into a typed struct and
  surface a real error.
- **Cloning to satisfy the borrow checker is a smell, not a fix.** Indexing a
  `Vec` you own (`for i in 0..v.len() { v[i].clone() }`) when you could
  `into_iter()` and consume it; cloning output strings you could return as
  borrows with a lifetime. Reach for ownership/borrowing first, `.clone()` last.
- **`Rc<RefCell<T>>` to dodge the borrow checker is a design smell.** Use it only
  when you genuinely model shared ownership of mutable state. For dependency
  wiring, prefer passing parameters/callbacks or returning a value the caller
  applies — redesign to avoid long-lived shared mutable references.
  ([Bad Habits](https://adventures.michaelfbryan.com/posts/rust-best-practices/bad-habits))
- **Hand-written wire/format bytes drift.** Scattering `"+OK\r\n"` at every return
  site means one missing `\r\n` hangs a client. Give the reply a type and put the
  formatting in one `Display` impl.
- **Discarded errors hide failures.** `.ok()`, `.unwrap_or_default()`, and
  `let _ = fallible()` throw away *why* something failed. Propagate with `?` or
  handle it explicitly; don't swallow it unless you genuinely don't care.
- **`unwrap`/`expect`/indexing on untrusted input is a DoS.** A panic crashes the
  whole process. On any input you don't control, use `?`, `.get(...)`, `checked_*`,
  `try_from`. ([Bugs Rust Won't Catch](https://corrode.dev/blog/bugs-rust-wont-catch/))
- **Forcing UTF-8 on byte/OS data corrupts it.** Paths and external byte streams
  aren't guaranteed UTF-8; keep them as `&[u8]`/`Vec<u8>`/`OsStr` rather than
  `String::from_utf8_lossy` when fidelity matters.
- **Integer overflow panics in debug but *silently wraps* in release.** When
  operands can be large, use `checked_*` (→ `Option`), `saturating_*`, or
  `wrapping_*` explicitly to say what you mean. ([Pitfalls of Safe Rust](https://corrode.dev/blog/pitfalls-of-safe-rust/))
- **`as` casts truncate/overflow silently.** `300u32 as u8 == 44`. Use `TryFrom`/
  `try_into` for narrowing conversions (it returns `Result`); reserve `as` for
  known-safe widening.
- **`Hash` and `Eq` must agree, and floats have neither.** Derive `Hash`, `Eq`,
  `PartialEq` together — never one by hand. Types containing `f32`/`f64` can't be
  `Eq`/`Hash` (NaN ≠ NaN), so they can't be `HashMap`/`HashSet` keys.

## Before you finish — checklist

- [ ] `cargo fmt` applied
- [ ] `cargo clippy --all-targets -- -D warnings` clean (no new `#[allow]`)
- [ ] `cargo test` passes (added characterization tests first if refactoring)
- [ ] Public functions take borrowed params and return `Result`/`Option` (no
      stray `unwrap`/`panic` on fallible paths)
- [ ] Manual loops/nested matches replaced with iterators/combinators where it
      reads more clearly
- [ ] Invariants pushed into types where practical (newtypes, enums, parse-don't-validate)
- [ ] No comments that merely restate the code; doc comments (`///`) on public items
- [ ] No `unsafe` introduced without explicit human sign-off; any `unsafe` is
      isolated, `// SAFETY:`-documented, and flagged as a warning in review

## Sources & further reading

- [corrode/refactoring-rust](https://github.com/corrode/refactoring-rust) — the
  27-exercise workshop these idioms are distilled from (Matthias Endler).
- corrode.dev blog: [Be Simple](https://corrode.dev/blog/simple/),
  [Thinking in Expressions](https://corrode.dev/blog/expressions/),
  [Defensive Programming](https://corrode.dev/blog/defensive-programming/),
  [Bugs Rust Won't Catch](https://corrode.dev/blog/bugs-rust-wont-catch/),
  [Pitfalls of Safe Rust](https://corrode.dev/blog/pitfalls-of-safe-rust/),
  [Navigating Paradigms](https://corrode.dev/blog/paradigms/).
- [Tour of Rust's Standard Library Traits](https://github.com/pretzelhammer/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md)
  (pretzelhammer) — deep, practical reference on designing with std traits.
- [Bad Habits](https://adventures.michaelfbryan.com/posts/rust-best-practices/bad-habits)
  (Michael-F-Bryan) — common newbie habits and their idiomatic fixes.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — the reference for writing
  correct `unsafe` Rust (consult only when `unsafe` is unavoidable).
- [mre/idiomatic-rust](https://github.com/mre/idiomatic-rust) — a peer-reviewed,
  continuously curated index of idiomatic-Rust articles, talks, and repos; the
  best single place to mine further patterns.
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — community
  catalogue of idioms, design patterns, and anti-patterns.
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html)
  — the official library-design checklist.
- [Effective Rust](https://www.lurklurk.org/effective-rust/) — David Drysdale's 35
  items for writing better Rust (free online).
- Books: *Programming Rust* (Blandy, Orendorff, Tindall);
  [*Rust for Rustaceans*](https://rust-for-rustaceans.com/) (Gjengset);
  [*Rust Atomics and Locks*](https://marabos.nl/atomics/) (Bos, free online);
  [*Rust Cookbook*](https://rust-lang-nursery.github.io/rust-cookbook/);
  [*Elements of Rust*](https://github.com/ferrous-systems/elements-of-rust).
