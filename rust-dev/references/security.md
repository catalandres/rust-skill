# Security & untrusted-input safety

Read this when code **touches the filesystem with attacker-influenced paths, runs
with elevated privileges, or processes untrusted input/bytes.** Rust's guarantees
are about *memory* safety — they say nothing about logic bugs, OS-level races, or
privilege mistakes. Every bug below compiles cleanly and passes ordinary tests.

Primary source: [Bugs Rust Won't Catch](https://corrode.dev/blog/bugs-rust-wont-catch/).
Most mitigations here are Unix-specific; check platform assumptions.

The type-level antidote running through all of this: **parse untrusted input into a
validated type once, at the boundary** (see
[domain-modeling.md](domain-modeling.md)), so the rest of the code can't re-handle
it wrongly.

## TOCTOU — time-of-check / time-of-use file races

A path resolved twice can point at two different things if an attacker swaps a
component (e.g. plants a symlink) between syscalls.

```rust
// Vulnerable: between remove and create, dest can become a symlink to /etc/shadow
fs::remove_file(&dest)?;
let f = File::create(&dest)?;          // may now write through the symlink

// Safer: create_new refuses if anything — including a dangling symlink — exists
use std::fs::OpenOptions;
let f = OpenOptions::new().write(true).create_new(true).open(&dest)?;
```

For a *sequence* of operations on the same location, don't re-resolve the path
string each call. Open the parent directory once and work relative to that file
descriptor (`openat`-style) — the `cap-std` or `openat` crates provide this.

## Set permissions at creation, not after

Creating a file/dir and then `chmod`-ing it leaves a window where other users can
open it.

```rust
// Vulnerable: gap between create and set_permissions
fs::create_dir(&path)?;
fs::set_permissions(&path, Permissions::from_mode(0o700))?;

// Atomic: set the mode as part of creation
use std::os::unix::fs::{DirBuilderExt, OpenOptionsExt};
std::fs::DirBuilder::new().mode(0o700).create(&path)?;
OpenOptions::new().write(true).create_new(true).mode(0o600).open(&file)?;
```

## Path identity is not string equality

`..`, `.`, and symlinks make two different strings name the same file (and one
string name different files over time).

```rust
// Fooled by /../, /./, symlinks
if path == Path::new("/") { /* bypassed by "/../" */ }

// Resolve symlinks and relative components first
let real = fs::canonicalize(&path)?;
if real == fs::canonicalize("/")? { /* ... */ }
```

Caveat: `canonicalize` is itself a check that can race a later use. For strict
cases, canonicalize *and* keep the resolved handle open rather than re-opening by
path.

## Load everything before crossing a privilege boundary

Functions that lazily load dynamic libraries (NSS modules, locales, plugins) will
load them from wherever the process root currently points. Do that resolution
*before* `chroot`/`setuid`/`setgid`/`exec`, never after.

```rust
// Vulnerable: get_user_by_name() loads NSS modules from the NEW root, as uid 0
chroot(new_root)?;
let user = get_user_by_name(name);     // executes attacker-controlled code

// Safe: resolve users/groups and load needed libs first, then drop in
let user = get_user_by_name(name);
chroot(new_root)?;
setuid(user.uid)?;
```

## See also (general, already in the skill)

These untrusted-input traps live in SKILL.md gotchas and
[error-handling.md](error-handling.md):

- A panic (`unwrap`/`expect`/indexing/overflow) on untrusted input is a
  denial-of-service. Prefer `?`, `.get`, `checked_*`, `try_from`.
- Don't discard errors (`.ok()`, `.unwrap_or_default()`, `let _ =`) on
  security-relevant operations — a missed failure is a missed access check.
- Don't force UTF-8 on paths/byte streams; keep `&[u8]`/`OsStr` fidelity.

## Contain `unsafe`

`unsafe` turns off the compiler's safety checks — a single wrong `unsafe` block can
reintroduce the memory-safety bugs Rust exists to prevent. (Effective Rust Item 16;
Rust Design Patterns "Contain unsafety in small modules".)

- **Avoid it** unless you have a concrete reason (FFI, a proven performance need, a
  primitive the safe API can't express). Most code never needs it.
- **Isolate it** in the smallest possible module/function, and wrap it in a safe
  API that upholds the invariants so callers can't misuse it.
- **Document every block** with a `// SAFETY:` comment stating why it's sound, and
  put a `# Safety` section on every `unsafe fn` describing the caller's obligations.
- **Test it hard:** run it under [Miri](https://github.com/rust-lang/miri) to catch
  undefined behavior, and consider fuzzing.

## Checklist

- [ ] No check-then-use on attacker-influenced paths; use `create_new`, or
      operate relative to an open directory fd
- [ ] Permissions/modes set atomically at creation, not after
- [ ] Paths canonicalized before identity comparisons
- [ ] All inputs resolved and libraries loaded before `chroot`/`setuid`/`exec`
- [ ] No panics or discarded errors on untrusted-input or access-control paths
- [ ] Untrusted input parsed into a validated type at the boundary, not re-checked
      everywhere
- [ ] `unsafe` avoided where possible; otherwise isolated, `// SAFETY:`-documented,
      and tested under Miri
