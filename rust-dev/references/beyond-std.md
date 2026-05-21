# Beyond standard Rust — FFI & no_std

Read this when **interfacing with C (or exposing a C ABI)** or **targeting
constrained environments** (embedded, kernels, minimal WASM) without the standard
library. Both are specialized; skip unless you're actually doing one of them.

Sources: [Effective Rust ch. 6](https://www.lurklurk.org/effective-rust/) (Items
33–35) and the [Rust Design Patterns FFI chapters](https://rust-unofficial.github.io/patterns/).

## FFI — calling C / being called by C

### Layout and linkage
- `#[repr(C)]` on any struct/enum that crosses the boundary — Rust's default
  layout is unspecified and may differ between compilations.
- `extern "C" fn` for functions exposed to C; declare imported C functions in an
  `unsafe extern "C" { ... }` block.
- Everything across the boundary is `unsafe` — keep the unsafe surface tiny and
  wrap it in a safe Rust API (see [security.md](security.md) on containing unsafe).

### Strings
C strings are NUL-terminated byte arrays, not `String`/`&str`:
- **Passing a string to C:** build a `CString` (it adds the NUL and rejects
  interior NULs); pass `.as_ptr()`. Keep the `CString` alive for the whole call —
  don't pass a pointer into a temporary.
- **Accepting a string from C:** wrap the pointer in `CStr::from_ptr(ptr)` (unsafe;
  the caller guarantees validity + NUL termination), then `.to_str()?` to get a
  checked `&str`, or `.to_bytes()` for raw bytes.

### Never panic across the boundary
Unwinding into C is undefined behavior. At every `extern "C"` entry point, either
guarantee no panic, or wrap the body in `std::panic::catch_unwind` and convert a
caught panic into an error code. Translate Rust `Result`/errors into C-friendly
return codes / out-params at the boundary, not Rust error types.

### Designing the boundary (FFI patterns)
- **Object-based / opaque-handle APIs:** hand C an opaque pointer (`*mut Foo` to a
  `Box`-allocated value) plus `foo_new` / `foo_do` / `foo_free` functions. C never
  sees the struct layout; ownership rules are explicit in the function contract.
- **Consolidate types into wrappers:** present one coherent wrapper type per
  resource rather than leaking many raw handles.
- **Prefer `bindgen`** to generate bindings from C headers rather than
  hand-writing `extern` declarations (Item 35) — hand-mapping drifts from the
  header and is error-prone. Control exactly what crosses the boundary (Item 34).

## no_std — without the standard library

For embedded, bare-metal, kernel, or minimal-WASM targets (Item 33):

- `#![no_std]` opts out of `std`. You still get **`core`** (most of the language:
  `Option`, `Result`, iterators, `slice`/`str` methods, traits, `fmt`).
- Add **`alloc`** (via `extern crate alloc;`) if you have an allocator and want
  `Box`, `Vec`, `String`, `Rc` — these live in `alloc`, not `core`.
- No `std`-only features: no OS (`std::fs`, `std::net`, threads), no `HashMap`
  (it's in `std`; use `alloc::collections::BTreeMap` or a `no_std` map crate), no
  `println!` (use a target-specific logger / `defmt`).
- Make a library **`no_std`-compatible without forcing it** on everyone: gate std
  usage behind a default `std` feature.

  ```rust
  #![cfg_attr(not(feature = "std"), no_std)]
  #[cfg(feature = "alloc")] extern crate alloc;
  ```
- Keep most logic generic over `core` types so it works in both worlds; isolate
  the `std`-only conveniences behind the feature flag.

## Checklist

- [ ] All FFI types `#[repr(C)]`; the unsafe surface is small and wrapped in safe Rust
- [ ] Strings cross as `CString`/`CStr`, kept alive for the call; no interior NULs
- [ ] No panic can unwind into C (guaranteed, or `catch_unwind` at each entry point)
- [ ] Opaque handles + explicit new/free for object-style C APIs; `bindgen` over hand-mapping
- [ ] `no_std` crates use `core`/`alloc`; `std`-only features gated behind a feature flag
