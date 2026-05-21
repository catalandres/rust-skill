# Public API & crate design

Read this when designing a **public API, library, or crate** — the surface other
code (or other people) will depend on. Internal code can be looser; a published
API is a contract.

Distilled from the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/checklist.html)
and [Effective Rust](https://www.lurklurk.org/effective-rust/). The guideline
codes (`C-*`) and item numbers point back to the source.

## Implement the obvious traits eagerly

Derive (or implement) the common traits so your types compose with the ecosystem
(`C-COMMON-TRAITS`, `C-DEBUG`, Effective Rust Item 10):

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
pub struct Point { pub x: i32, pub y: i32 }
```

- **`Debug` on every public type** (`C-DEBUG`) — non-negotiable; people need to
  print your types. Make it non-empty.
- Add `Clone`, `PartialEq`/`Eq`, `Hash`, `Ord` when they make sense; `Copy` only
  for small, plain-data types.
- `Default` when there's an obvious zero value; `Display` for user-facing text
  (not the same as `Debug`).
- `Send`/`Sync` fall out automatically — preserve them; don't accidentally add a
  `Rc`/raw pointer that removes them (`C-SEND-SYNC`).
- Implement Serde `Serialize`/`Deserialize` behind a feature for data types
  (`C-SERDE`).

## Conversions & naming

- Use the standard conversion traits: `From`/`Into` for infallible, `TryFrom`/
  `TryInto` for fallible, `AsRef`/`AsMut` for cheap reference conversions
  (`C-CONV-TRAITS`, Item 5). Implement `From`, get `Into` for free.
- Method-name cost conventions (`C-CONV`): `as_*` = cheap borrow, `to_*` =
  expensive/owned clone, `into_*` = consuming conversion.
- **`AsRef` vs `Borrow` vs `Deref`** — pick deliberately: `AsRef<T>` is a cheap
  reference conversion (`fn f<S: AsRef<str>>(s: S)`); `Borrow<T>` is `AsRef` *plus*
  the guarantee that the borrowed form `Hash`/`Eq`/`Ord`-compares identically to the
  owned form, which is exactly why `HashMap<String, _>::get` accepts `&str`; `Deref`
  is for smart pointers only (see the Deref-polymorphism anti-pattern). Implement
  `Borrow` only when that equality guarantee holds.
- Iterator methods are named `iter` / `iter_mut` / `into_iter` (`C-ITER`), and the
  returned iterator type is named after them (`C-ITER-TY`).
- Getters are `fn field(&self)` (no `get_` prefix), setters `set_field` (`C-GETTER`).
- Casing follows RFC 430 (`C-CASE`): `CamelCase` types, `snake_case` values.
- Don't encode types in names (Hungarian notation like `str_name`, `i_count`) — the
  type system and editor tooling already show types, so the prefix is pure noise.
  Name by meaning; use shadowing across transformations rather than `_typed` suffixes.

## Constructors, builders, and type-safe arguments

- Constructors are inherent static methods, usually `new` (`C-CTOR`). For fallible
  construction use `parse`/`try_new` returning `Result`.
- Use a **builder** for types with many optional fields (`C-BUILDER`, Item 7)
  rather than a giant `new(...)` or many constructors.
- Arguments convey meaning through **types, not `bool`/`Option`** (`C-CUSTOM-TYPE`):
  prefer `Compression::Strong` over `true`. Newtypes give static distinctions —
  `UserId(u64)` can't be passed where an `OrderId(u64)` is expected (`C-NEWTYPE`,
  Item 6).
- For a set of flags, use the `bitflags` crate, not a hand-rolled enum
  (`C-BITFLAG`).

## Validation, errors, and docs

- **Validate arguments** at the boundary (`C-VALIDATE`); reject bad input with a
  `Result`, don't trust-and-crash.
- Error types are meaningful and well-behaved (`C-GOOD-ERR`): implement
  `Debug` + `Display` + `std::error::Error`. Prefer typed errors (`thiserror`) in
  libraries (Item 4) — see [error-handling.md](error-handling.md).
- Every public item has a rustdoc example (`C-EXAMPLE`); examples use `?`, not
  `unwrap`/`try!` (`C-QUESTION-MARK`).
- Document the failure surface (`C-FAILURE`): a `# Errors`, `# Panics`, and
  `# Safety` section wherever the function can fail, panic, or is `unsafe`.
- Crate-level docs are thorough with examples (`C-CRATE-DOC`); fill in `Cargo.toml`
  metadata (`C-METADATA`).

## Generics vs trait objects

Choose deliberately (Item 12, `C-GENERIC`/`C-OBJECT`):

- **Generics (`fn f<T: Trait>` / `impl Trait`)** — monomorphized, zero-cost,
  static dispatch. Default choice. Cost: code bloat, longer compiles.
- **Trait objects (`&dyn Trait` / `Box<dyn Trait>`)** — dynamic dispatch, one
  copy of the code, heterogeneous collections (`Vec<Box<dyn Trait>>`). Cost: a
  vtable indirection; the trait must be object-safe.
- Make a trait object-safe if it might ever be used as `dyn` (`C-OBJECT`).

## Trait design notes

Smaller, situational rules from the
[Tour of Rust's Standard Library Traits](https://github.com/pretzelhammer/rust-blog/blob/master/posts/tour-of-rusts-standard-library-traits.md):

- **Implement `IntoIterator` for `T`, `&T`, and `&mut T`** on a collection type, so
  `for x in c`, `for x in &c`, and `for x in &mut c` all work (consuming, borrowing,
  mutably borrowing).
- **Operator traits: implement owned *and* borrowed forms** (`impl Add for Point`
  *and* `impl Add for &Point`) or they silently diverge; delegate one to the other.
  Note `AddAssign`/`*Assign` take `&mut self`, not `self`.
- **`Cow<'a, T>`** lets an API defer the allocate-or-borrow choice to runtime —
  return `Cow::Borrowed` on the common no-change path, `Cow::Owned` only when you
  actually had to allocate.
- **Blanket impls aren't overridable.** `impl<T> MyTrait for T` conflicts with any
  later `impl MyTrait for Concrete` (coherence). Use a blanket impl only when you
  truly want one behavior for a whole family — relevant when writing extension
  traits (see [domain-modeling.md](domain-modeling.md)).
- **`?Sized`** on a generic bound (`fn f<T: ?Sized>(t: &T)`) lets it accept
  `&dyn Trait`, `&str`, `&[T]` — the implicit `Sized` bound otherwise excludes them.

## RAII guards

Tie resource cleanup to a value's lifetime via `Drop` (Item 11): a `MutexGuard`,
a file handle, a transaction that rolls back unless `commit()` is called. The
compiler then guarantees cleanup happens. Destructors must never fail
(`C-DTOR-FAIL`); offer an explicit `close()`/`commit()` for fallible teardown
(`C-DTOR-BLOCK`).

## Future-proofing

Keep room to evolve without breaking downstream code:

- **Private struct fields** (`C-STRUCT-PRIVATE`) — expose accessors, not fields,
  so layout can change.
- **`#[non_exhaustive]`** on public structs/enums you expect to grow — forces
  downstream `match`es to keep a `_` arm and blocks struct literal construction,
  so adding a variant/field isn't a breaking change.
- **Sealed traits** (`C-SEALED`) when a trait is for your use only and shouldn't be
  implemented downstream (a private supertrait it can't name).
- **Newtypes hide implementation** (`C-NEWTYPE-HIDE`) and can hide a complex inner
  type or iterator.

## Macros

Prefer functions and generics first; reach for macros only when there's no other
way (Item 28). When you do write them: input syntax should evoke the output
(`C-EVOCATIVE`), they should compose with attributes and visibility
(`C-MACRO-ATTR`, `C-MACRO-VIS`), and work anywhere the equivalent item works
(`C-ANYWHERE`). Document them with examples like any other API.

## Crate & dependency hygiene

- **Semantic versioning** is a promise (Item 21): breaking changes = major bump.
  Adding a public enum variant without `#[non_exhaustive]` is breaking.
- **Minimize visibility** (Item 22): default to private; expose the smallest API
  that works. `pub(crate)` over `pub` unless it's truly public.
- **No wildcard imports** (`use foo::*`) in library code (Item 23) — they make
  provenance unclear and break when upstream adds names.
- **Re-export dependency types that appear in your public API** (Item 24) so
  callers don't need to add and version-match the dependency themselves.
- **Manage the dependency graph** (Item 25) and **resist feature creep**
  (Item 26): every dependency and feature is surface area and supply-chain risk.
  Prefer small, focused crates (Rust Design Patterns: "Prefer small crates").
- Keep public dependencies of a stable crate stable (`C-STABLE`); use a
  permissive license compatible with the ecosystem (`C-PERMISSIVE`).

## Checklist

- [ ] Public types derive `Debug` (+ `Clone`/`Eq`/`Hash`/`Default` where sensible)
- [ ] Conversions use `From`/`TryFrom`/`AsRef`; methods follow `as_`/`to_`/`into_`
- [ ] Constructors inherent (`new`/`try_new`); builders for many-field types
- [ ] Arguments are meaningful types, not bare `bool`/`Option`; newtypes for IDs
- [ ] Errors implement `Error`+`Display`; docs have `# Errors`/`# Panics`/`# Safety`
- [ ] Every public item has a `?`-using rustdoc example
- [ ] Generics vs `dyn` chosen deliberately; object-safe traits where useful
- [ ] `Drop`/RAII for cleanup; destructors don't fail
- [ ] Private fields + `#[non_exhaustive]` + sealed traits where evolution expected
- [ ] Minimal visibility, no wildcard imports, public-API deps re-exported, semver respected
