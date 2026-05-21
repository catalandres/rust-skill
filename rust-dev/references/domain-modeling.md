# Domain modeling — delete code by changing types

Read this when a function takes several `bool`s or raw strings, when "config" or
"command" is a stringly-typed `HashMap`, when the same value gets validated in
many places, or when parallel collections track "the same thing" different ways.
The theme: **the right type makes whole functions and entire bug classes
disappear.**

## 1. Parse, don't validate

*Validation* checks a value and hands back the same loosely-typed value, trusting
every later reader to remember the check. *Parsing* converts it into a type that
*proves* the invariant, so later code can't get it wrong.

```rust
// Validate (fragile): everyone downstream still holds a u16 and must remember
// it's non-zero and in range.
fn check_port(p: u16) -> bool { p != 0 }

// Parse (robust): the success type carries the proof.
fn parse_port(s: &str) -> Result<NonZeroU16, InvalidPort> {
    let n: u16 = s.parse().map_err(|_| InvalidPort::Invalid)?;  // upper bound free
    NonZeroU16::new(n).ok_or(InvalidPort::Zero)                 // non-zero proven
}
```

Once a function returns `NonZeroU16`, no caller can forget the port is non-zero —
the type won't let them construct a zero. (Coined by Alexis King,
["Parse, don't validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).)

Do the parsing **once, at the boundary** (the edge of your program / module), and
let everything inside work with the typed value.

The same principle rejects the **init-then-populate** habit (`let c = Config::new();
c.load(path)?;`), which leaves a window where the object exists but isn't valid.
Return a fully-initialized, valid value straight from the constructor instead —
`Config::from_file(path)?` — so an invalid instance can never be observed.

## 2. Make illegal states unrepresentable

If a combination of values is nonsense, the type system should forbid
constructing it.

```rust
// Two bools = four states, but (is_authenticated=false, is_admin=true) is nonsense.
fn can_access(path: &str, is_authenticated: bool, is_admin: bool) -> bool { /*…*/ }

// Three real states; the impossible one can't be built.
enum UserStatus { Anonymous, Authenticated, Admin }
fn can_access(path: &str, status: UserStatus) -> bool { /*…*/ }
```

Symptoms that you need this:

- Multiple `bool` parameters/fields, especially ones that are only valid in
  certain combinations.
- Comments like "if `a` is set, `b` must be `None`".
- Runtime asserts guarding combinations that "shouldn't happen".

Fixes: enums for mutually exclusive states; `enum` variants that *carry* the data
relevant to that state (e.g. `Command::Append(usize)`); split a struct into two
types when fields are valid only in different phases.

## 3. Newtypes and `NonZero*` — encode invariants in the type

```rust
// Instead of passing raw u64s that could be confused:
struct UserId(u64);
struct OrderId(u64);   // can't accidentally pass an OrderId where a UserId is wanted

// Instead of "a count that's never zero", checked everywhere:
use std::num::NonZeroU32;
```

A newtype is a zero-cost wrapper. Add a validating constructor (`fn new(...) ->
Result<Self, _>`) and keep the field private, so the only way to get one is
through the check — "parse, don't validate" at the type level.

## 4. `TryFrom` / `FromStr` at the boundary

Model "string (or loose map) in, typed value or error out" with the standard
conversion traits, so callers use familiar syntax.

```rust
// FromStr gives you `.parse()` for free:
impl FromStr for SrtTimestamp {
    type Err = ParseError;
    fn from_str(s: &str) -> Result<Self, Self::Err> { /*…*/ }
}
let t: SrtTimestamp = "00:01:23,456".parse()?;

// TryFrom for "convert this whole map/struct into my typed config":
impl TryFrom<&HashMap<String, String>> for Config {
    type Error = ConfigError;
    fn try_from(map: &HashMap<String, String>) -> Result<Self, Self::Error> { /*…*/ }
}
```

This replaces a pile of per-field getters that each re-parse the source and
silently default on bad input. Parse once into `Config`; the rest of the program
reads `config.port`, never the raw map. (In production this is what `serde` +
`envy`/`figment`/`config` automate — but the shape is the same.)

## 5. Extension traits — add ergonomics to foreign types

You can't add inherent methods to a type you don't own (`str`, `Vec`, a
dependency's type), but you *can* implement your own trait for it (the orphan rule
allows `impl Trait for ForeignType` when the trait is local to your crate).

```rust
pub trait FunStr {
    fn shout(&self) -> String;
}
impl FunStr for str {
    fn shout(&self) -> String { self.to_uppercase() }
}
// turns sparkle(&shout(s.trim())) into s.trim().shout().sparkle() — reads in order
```

This is the pattern behind `itertools::Itertools`, `anyhow::Context`,
`tokio::AsyncReadExt`, `rayon`'s parallel iterators. Use it to give call sites
left-to-right method-chaining flow instead of inside-out nesting.

Prefer one concrete `impl Trait for str` (or per type) over a blanket
`impl<T> Trait for T`: a blanket impl can't be specialized later and conflicts
with any other impl (coherence), so it locks you in.

## 6. Collapse parallel collections into one typed model

Parallel maps/vecs that are "really about the same key" let inconsistent states
exist:

```rust
// Before — the SAME key can be a string AND a list AND a hash simultaneously.
struct Db {
    strings: HashMap<String, String>,
    lists:   HashMap<String, Vec<String>>,
    hashes:  HashMap<String, HashMap<String, String>>,
}

// After — a key maps to exactly one typed value; "wrong type" becomes a real,
// reportable error instead of silent corruption.
enum Value { String(String), List(Vec<String>), Hash(HashMap<String, String>) }
struct Db { store: HashMap<String, Value> }
```

The same idea applies to "an entity split across three vecs indexed by the same
position" — make it one vec of a struct.

## 7. Separate classification from policy

When one function answers two questions ("what kind of thing is this?" and "what
do we do about it?"), split them. The classifier becomes reusable and
independently testable; the policy shrinks to one arm per decision.

```rust
enum StatusClass { Informational, Success, Redirect, ClientError, ServerError, Unknown }
impl StatusClass { fn classify(status: u16) -> Self { /* the only place ranges live */ } }

// policy dispatches on the class, not raw numbers:
match StatusClass::classify(resp.status) { /* one arm per action */ }
```

## 8. State machines for context-dependent parsing

When parsing rules depend on what came before (inside quotes vs not, escaped vs
literal, expecting `=` vs reading a value), a fistful of `bool` flags will drift
out of sync. Model the states as an `enum` and transition on `match (state,
input)`:

```rust
enum State { LineStart, InKey, AfterKey, BeforeValue, InBareValue,
             InQuotedValue, InQuotedEscape, AfterQuotedValue, InComment }
```

Every input is handled in exactly one `(state, char)` arm, so a rule can't be
half-applied. The end-of-input state tells you whether to commit or report (e.g.)
an `UnterminatedQuote`. This is strictly more correct than `split` + flag-checking
for anything with quoting/escaping/nesting.

## 9. Defensive programming — make the compiler your safety net

Lean on types and lints so mistakes become compile errors, not runtime bugs
(from [Patterns for Defensive Programming](https://corrode.dev/blog/defensive-programming/)).

- **Slice patterns over indexing.** `vec[0]` panics; matching is total:

  ```rust
  match items.as_slice() {
      []                 => /* empty */,
      [only]             => /* exactly one */,
      [first, rest @ ..] => /* at least one */,
  }
  ```

  Lint: `clippy::indexing_slicing`.

- **Don't paper over struct growth with `..Default::default()`.** Naming every
  field in a literal means adding a field becomes a compile error you must
  consider. Inside trait impls, destructure so a new field surfaces a warning:

  ```rust
  impl PartialEq for Order {
      fn eq(&self, other: &Self) -> bool {
          let Self { size, toppings, crust: _ } = self;  // new field → must update
          // ...
      }
  }
  ```

- **Force construction through a validating constructor.** A private, zero-size
  field makes the struct unconstructable outside its module, so callers must go
  through `new`/`parse`/`try_new` (the type-level form of "parse, don't validate"):

  ```rust
  pub struct Email { addr: String, _private: () }
  impl Email { pub fn parse(s: &str) -> Result<Self, EmailError> { /* validate */ } }
  ```

- **`#[must_use]`** on a type or return value warns when it's silently dropped —
  use it for builders, RAII guards, and "you must apply this" results.

- **Exhaustive enum matching** over your own enums (no `_`) so a new variant is a
  compile error. Lint: `clippy::wildcard_enum_match_arm`.

- **Semantic params over booleans:** `process(data, Compression::Strong,
  Validation::Enabled)`, not `process(data, true, false)`. Lint:
  `clippy::fn_params_excessive_bools`.

- **`TryFrom` over `From` for fallible conversions** so call sites must handle the
  error. Lint: `clippy::fallible_impl_from`.

## Checklist

- [ ] No multi-`bool` parameter lists encoding impossible combinations (use an enum)
- [ ] Loose input parsed into a typed value once, at the boundary (`FromStr`/`TryFrom`)
- [ ] Invariants live in types (newtypes, `NonZero*`, private fields + validating ctor)
- [ ] No parallel collections that let "the same key" hold inconsistent state
- [ ] Classification separated from policy where a function does both
- [ ] Context-dependent parsing modeled as a state-machine enum, not flag soup
- [ ] Extension traits used to give foreign types ergonomic, chainable methods
