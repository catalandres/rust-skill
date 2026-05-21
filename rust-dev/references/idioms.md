# Idiom catalog — the full before→after reference

A distilled catalog of idiomatic-Rust refactorings, grouped by theme. Each entry
shows the awkward starting point, an idiomatic rewrite, and *why*. There is
rarely a single right answer — these are defaults, and the reasoning matters more
than the exact spelling.

Use this as a lookup: find the smell you're looking at, apply the transformation.

---

## Expression-oriented Rust (applies throughout)

Almost everything in Rust produces a value — lean on that instead of mutable
scratch state ([Thinking in Expressions](https://corrode.dev/blog/expressions/)).

```rust
// if / match as expressions
let label = if n > 0 { "pos" } else { "non-pos" };
let color = match duck { Duck::Huey => "red", Duck::Dewey => "blue", _ => "?" };

// loop yields a value via `break`
let first_even = loop {
    match it.next() { Some(n) if n % 2 == 0 => break n, Some(_) => continue, None => break 0 }
};

// assign the result of a block instead of pre-declaring a `mut`
let sorted = { let mut v = get_vec(); v.sort(); v };  // v is immutable afterwards

// drop the trailing `return` on the last expression
fn make() -> Config { Config::default() }   // not `return Config::default();`
```

Balance: expressions and combinators are a win when they read top-to-bottom. A
deeply nested `opt.map_or(d, |x| ...).then_some(...)` chain is harder to read than
a plain `match` — optimize for the next person, not for brevity.

---

## Phase 1 — Types, signatures, simple matches

### 1. Own only what you must; let the type system answer questions

```rust
// Before
pub fn starts_with_uppercase(s: String) -> bool {
    if s.len() == 0 { return false; }
    let first = s.chars().nth(0).unwrap();
    if first.is_uppercase() == true { return true; } else { return false; }
}

// After
pub fn starts_with_uppercase(s: &str) -> bool {
    s.starts_with(char::is_uppercase)
}
```

- `&str` is the natural type for a read-only check; don't demand `String`.
- `len() == 0` → `is_empty()`; `chars().nth(0)` → `chars().next()`.
- `== true` is noise; `if x { true } else { false }` is just `x`.
- Empty string falls out naturally (no first char → `false`). The edge case
  disappears instead of needing a guard.

### 2. Or-patterns nest inside constructors

```rust
// Before
match value {
    Some(2) | Some(3) | Some(5) | Some(7) => "prime",
    Some(0) | Some(1) | Some(4) | Some(9) => "square",
    None => "nothing",
    _ => "something else",
}
// After
match value {
    Some(2 | 3 | 5 | 7) => "prime",
    Some(0 | 1 | 4 | 9) => "square",
    None => "nothing",
    _ => "something else",
}
```

Or-patterns live inside `Some(_)` (since Rust 1.53). Same behavior, half the
noise; the arms read like a table.

### 3. Collapse nested `match` on `Option` into a combinator

```rust
// Before
match path.extension() {
    Some(ext) => match ext.to_str() {
        Some(s) => s == "rs",
        None => false,
    },
    None => false,
}
// After
path.extension().is_some_and(|ext| ext == "rs")
```

- `is_some_and` collapses both `None` arms into a single `false`.
- `OsStr: PartialEq<str>`, so no `to_str` dance is needed.
- Equally valid: `.map_or(false, |e| e == "rs")`.

### 4. Respect UTF-8 boundaries (a correctness bug, not just style)

```rust
// Before — panics on truncate("café", 4): byte 4 is inside 'é'
let mut owned = s.to_string();
owned.truncate(max_bytes);
owned

// After — byte cap that respects char boundaries
pub fn truncate(s: &str, max_bytes: usize) -> String {
    if max_bytes >= s.len() { return s.to_string(); }
    let mut end = max_bytes;
    while !s.is_char_boundary(end) { end -= 1; }
    s[..end].to_string()
}
```

The real lesson is spec ambiguity: did you want "first N bytes" or "first N
chars"? Char semantics would be `s.chars().take(n).collect()`. Pick deliberately.

---

## Phase 2 — `Option`, `Result`, and iterators (everyday idioms)

`Option` and `Result` behave like iterators of 0-or-1 element. Almost any
manual unwrapping has a combinator.

### 5. `map` over match; `let ... else` for unwrap-or-bail

```rust
// Before
let x = match n { Some(x) => x, None => return None };
Some(x + 1)
// After
n.map(|x| x + 1)
```

When the rest of the body is long, `let Some(x) = n else { return None };` is the
language-level form of "unwrap or bail".

### 6. Chain transformations; let all failure paths share one `None`

```rust
// Before — two nested ifs, three returns
if let Some(s) = s {
    let trimmed = s.trim();
    if !trimmed.is_empty() { return Some(trimmed.to_uppercase()); }
    else { return None; }
} else { return None; }
// After
s.map(str::trim)
 .filter(|t| !t.is_empty())
 .map(str::to_uppercase)
```

`filter` turns the unwanted `Some("")` into `None` declaratively. The chain reads
top-to-bottom in the same order the logic happens.

### 7. `Option` is iterable — `flatten` drops the `None`s

```rust
// Before
let mut sum = 0;
for option in options { if let Some(v) = option { sum += v; } }
sum
// After
options.into_iter().flatten().sum()
```

`filter_map(|x| x)` is the explicit equivalent; `flatten` reads better since it
needs no closure.

### 8. `collect` turns an iterator of `Result` into a `Result` of collection

```rust
// Before
let mut result = Vec::new();
for value in values {
    match value.parse::<i32>() {
        Ok(num) => result.push(num),
        Err(e) => return Err(e),
    }
}
Ok(result)
// After
pub fn parse_values(values: &[String]) -> Result<Vec<i32>, ParseIntError> {
    values.iter().map(|v| v.parse()).collect()
}
```

`collect::<Result<Vec<_>, _>>()` short-circuits on the first `Err` — exactly the
early-return you wrote by hand. The return type drives inference for `parse`.

### 9. Pick the data structure that fits the algorithm

```rust
// Before — O(n²): Vec::contains scans every time
let mut seen: Vec<char> = Vec::new();
for c in s.chars() { if !seen.contains(&c) { seen.push(c); } }
seen.len()
// After — O(n)
s.chars().collect::<HashSet<_>>().len()
```

`HashSet` deduplicates and gives O(1) membership. Shorter *and* faster.

---

## Phase 3 — Aggregations (loops become iterator patterns)

Every accumulating `for` loop maps to a named adapter.

### 10. The four canonical accumulators

```rust
// sum:     rooms.iter().map(|r| r.adults).sum()
// count:   rooms.iter().map(|r| r.children.len() as i32).sum()
// max:     rooms.iter().map(|r| r.adults).max().unwrap_or(0)   // max returns Option!
// flatten: rooms.iter().flat_map(|r| r.children.iter().copied()).collect()
```

Preserve original empty-input behavior: a loop returning `0` becomes
`.max().unwrap_or(0)`, not a bare `.max()`.

### 11. `fold` threads a running value; seed it from the first element

```rust
// Before — manual min/max bookkeeping, parses parts[0] three times
// After
let mut nums = input.split_whitespace().map(|s| s.parse::<i64>().unwrap());
let first = nums.next().expect("at least one number");
nums.fold((first, first), |(max, min), n| (max.max(n), min.min(n)))
```

`split_whitespace` handles tabs and double spaces. (The honest version returns
`Result` so empty/non-numeric input is a real case, not a panic.)

### 12. `max_by_key` instead of hand-rolled "track the best"

```rust
// counting half is already idiomatic:
let mut occurrences: HashMap<i32, i32> = HashMap::new();
for &x in numbers { *occurrences.entry(x).or_insert(0) += 1; }
// finding half:
occurrences.into_iter().max_by_key(|&(_, count)| count).map(|(v, _)| v)
```

No temporaries, no double-seeding. Ideally return the `Option` and let the caller
decide what "empty" means.

### 13. Implement `Iterator` for lazy, composable sequences

```rust
struct Fibonacci { current: u64, next: u64 }
impl Iterator for Fibonacci {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        let current = self.current;
        (self.current, self.next) = (self.next, current + self.next);
        Some(current)
    }
}
// let v: Vec<u64> = Fibonacci::new().take(10).collect();
```

Naive recursive `fib` is exponential; this is O(n) and lazy. Implementing
`Iterator` unlocks `take`, `map`, `zip`, `collect`, … for free. For overflow
safety swap `+` for `checked_add` and return `None`.

---

## Phase 4 — Parsing & error handling (`?`, `FromStr`, custom errors)

See also [error-handling.md](error-handling.md) for the deeper version.

### 14. `?` is the manual match; std often has a one-shot helper

```rust
// Before — two matches that each just propagate the error
// After
use std::fs;
pub fn read_file_contents(path: &str) -> io::Result<String> {
    fs::read_to_string(path)
}
```

`match ... Err(e) => return Err(e)` *is* `?`. And `fs::read_to_string` opens,
reads, and closes for you — the function nearly stops needing to exist.

### 15. Parse, don't validate — push invariants into the type

```rust
// Before — Option throws away the reason; parses i32 then range-checks by hand
// After
pub fn parse_port(s: &str) -> Result<NonZeroU16, InvalidPort> {
    let n: u16 = s.parse().map_err(|_| InvalidPort::Invalid)?;
    NonZeroU16::new(n).ok_or(InvalidPort::Zero)
}
```

- Parsing straight to `u16` deletes the upper-bound check (the type enforces it).
- `NonZeroU16` makes "port 0" unrepresentable in the success type.
- A small error enum tells the caller *why* it failed.

### 16. Standard string methods over hand-rolled char loops

```rust
// Before — hand-rolled trim, tag-strip, and whitespace-collapse loops
// After
pub fn trim_log_line(line: &str) -> String {
    let trimmed = line.trim();
    let without_tag = ["[INFO]", "[WARN]", "[ERROR]"]
        .iter()
        .find_map(|tag| trimmed.strip_prefix(tag))
        .unwrap_or(trimmed);
    without_tag.split_whitespace().collect::<Vec<_>>().join(" ")
}
```

`trim`, `strip_prefix` (= "matched, here's the rest"), and
`split_whitespace().join(" ")` (collapse runs) replace ~25 lines of manual
state.

### 17. `split_once` + `?` + `From` for clean multi-field parsing

```rust
// Before — Vec allocation just to count pieces; unwrap panics on bad input
// After
pub fn parse_srt_timestamp(s: &str) -> Result<Duration, ParseError> {
    let (h, rest) = s.split_once(':').ok_or(ParseError::BadFormat)?;
    let (m, rest) = rest.split_once(':').ok_or(ParseError::BadFormat)?;
    let (sec, ms) = rest.split_once(',').ok_or(ParseError::BadFormat)?;
    let total_ms = h.parse::<u64>()? * 3_600_000
        + m.parse::<u64>()? * 60_000
        + sec.parse::<u64>()? * 1_000
        + ms.parse::<u64>()?;
    Ok(Duration::from_millis(total_ms))
}
```

`split_once` is allocation-free and gives both halves. With
`impl From<ParseIntError> for ParseError`, `?` propagates both error kinds. The
natural next step is `impl FromStr` on a newtype so callers write `s.parse()`.

### 18. `match` with `|` arms + byte indexing for ASCII

```rust
// Before — Vec<char> allocation to index by position; if/else-if cascade on country
// After
pub fn is_valid_iban(s: &str) -> bool {
    let expected = match s.get(0..2) {
        Some("DE") | Some("GB") => 22,
        Some("FR") => 27,
        Some("CH") => 21,
        Some("NL") => 18,
        Some("AT") => 20,
        _ => return false,
    };
    if s.len() != expected { return false; }
    let b = s.as_bytes();
    b[0].is_ascii_alphabetic() && b[1].is_ascii_alphabetic()
        && b[2].is_ascii_digit() && b[3].is_ascii_digit()
        && s.bytes().all(|c| c.is_ascii_alphanumeric())
}
```

For ASCII data, `as_bytes()` indexing is O(1) with no allocation. `match` with
`|` maps country→length declaratively; `bytes().all(...)` replaces the per-index
loop.

---

## Phase 5 — Domain modeling (delete code by changing types)

See also [domain-modeling.md](domain-modeling.md). This is where the biggest
wins live: the right type makes whole functions vanish.

### 19. `Option<&T>` over `&Option<T>`; `any` over loop-and-return; empty as sentinel

```rust
// Before
if let Some(excluded) = excluded {
    for prefix in excluded { if path.starts_with(prefix) { return true; } }
}
false
// After
let Some(excluded) = excluded else { return false };
excluded.iter().any(|prefix| path.starts_with(prefix))
```

`let ... else` removes the nesting; `iter().any(...)` is "does any element
match?". Better still: take `&HashSet<PathBuf>` and let the empty set mean
"nothing excluded" — the `Option` is redundant.

### 20. Make illegal states unrepresentable — enum over bool soup

```rust
// Before — fn can_access(path, is_authenticated: bool, is_admin: bool)
// (is_authenticated=false, is_admin=true) is a nonsense state the type allows
// After
pub enum UserStatus { Anonymous, Authenticated, Admin }
pub fn can_access(path: &str, status: UserStatus) -> bool {
    match (path, status) {
        (p, _)                 if p.starts_with("/public") => true,
        (p, UserStatus::Admin) if p.starts_with("/admin")  => true,
        (p, _)                 if p.starts_with("/admin")  => false,
        (_, UserStatus::Anonymous) => false,
        _ => true,
    }
}
```

Two booleans (four states) collapse to three real states; the impossible one no
longer exists. The decision becomes one flat `match`. Adding a role is "add a
variant, fix the compiler errors".

### 21. `&[T]` over `&Vec<T>`; build the lookup set once; borrow outputs

```rust
// Before — O(n·m), to_lowercase allocates inside the inner loop, clones outputs
// After
pub fn spell_check<'a>(words: &[&'a str], dict: &HashSet<String>) -> Vec<&'a str> {
    words.iter().copied()
        .filter(|w| !dict.contains(&w.to_ascii_lowercase()))
        .collect()
}
```

The caller builds the (lowercased) `HashSet` once and reuses it. `&[&str]` +
`Vec<&'a str>` avoid per-word clones. API lesson: take the data structure that
fits the algorithm, not the one the caller happens to hold.

### 22. Consume what you own; `match` over `if let` chains; std string helpers

```rust
// Before — index loop, .clone() on owned items, if-let chain on an enum
// After
input.into_iter()
    .map(|(s, cmd)| match cmd {
        Command::Uppercase => s.to_uppercase(),
        Command::Trim => s.trim().to_string(),
        Command::Append(n) => s + &"bar".repeat(n),
    })
    .collect()
```

`into_iter()` consumes (no clones); `match` is exhaustive (new variant → compile
error here); `"bar".repeat(n)` replaces the inner loop; `String + &str` reuses
the buffer.

### 23. Extension traits — add methods to foreign types for left-to-right flow

```rust
// Before — call sites read inside-out: sparkle(&shout(s.trim()))
// After
pub trait FunStr {
    fn shout(&self) -> String;
    fn sparkle(&self) -> String;
}
impl FunStr for str {
    fn shout(&self) -> String { self.to_uppercase() }
    fn sparkle(&self) -> String { format!("✨ {} ✨", self) }
}
// now: "hello".trim().shout().sparkle()
```

The orphan rule lets you `impl Trait for ForeignType` as long as the trait is
yours. This is how `itertools::Itertools`, `anyhow::Context`,
`tokio::AsyncReadExt` work.

### 24. Separate classification from policy

```rust
// Before — one function tangles "what class is this status?" with "what do we do?"
// After: classify first…
enum StatusClass { Informational, Success, Redirect, ClientError, ServerError, Unknown }
impl StatusClass {
    fn classify(status: u16) -> Self {
        match status {
            100..=199 => Self::Informational,
            200..=299 => Self::Success,
            300..=399 => Self::Redirect,
            400..=499 => Self::ClientError,
            500..=599 => Self::ServerError,
            _ => Self::Unknown,
        }
    }
}
// …then dispatch on the class (one arm per policy decision):
match StatusClass::classify(resp.status) {
    ServerError => Action::Retry,
    ClientError if matches!(resp.status, 408 | 429) => Action::Retry,
    ClientError => Action::Fail(format!("client error {}", resp.status)),
    // …
}
```

The classifier is the only place that knows the numeric ranges, and it's reusable
and unit-testable. Range patterns (`100..=199`) replace `>= && <` ladders. A
scoped `use StatusClass::*;` keeps the match arms short. (Real `http::StatusCode`
already gives `.is_success()` etc. — same idea, upstream.)

### 25. Parse the stringly-typed map once into a typed struct (`TryFrom` at the edge)

```rust
// Before — four getters each re-parse the map; "banana" silently becomes 100
// After
pub struct Config { pub port: u16, pub host: String, pub debug: bool, pub max_connections: u32 }
#[derive(Debug)]
pub enum ConfigError { BadValue { key: &'static str, value: String } }

impl TryFrom<&HashMap<String, String>> for Config {
    type Error = ConfigError;
    fn try_from(map: &HashMap<String, String>) -> Result<Self, Self::Error> {
        let parsed = |key, default: &str| -> Result<_, _> {
            let raw = map.get(key).map(String::as_str).unwrap_or(default).to_string();
            raw.parse().map_err(|_| ConfigError::BadValue { key, value: raw })
        };
        Ok(Config {
            port: parsed("PORT", "8080")?,
            max_connections: parsed("MAX_CONNECTIONS", "100")?,
            host: map.get("HOST").cloned().unwrap_or_else(|| "localhost".into()),
            debug: matches!(map.get("DEBUG").map(String::as_str), Some("1" | "true" | "yes")),
        })
    }
}
```

Parse at the boundary; the rest of the program uses `config.port`. Bad input is a
real error, not a silent default. The four getters disappear. (In production:
`serde` + `envy`/`figment`/`config` — this is *why* those crates exist.)

### 26. Capstone — a character-level state machine beats a pile of flag variables

When parsing has context-dependent rules (quotes that protect `#` and `=`,
escapes, multi-line values, line-numbered errors), booleans like `in_quotes`,
`started`, `seen_equals` multiply and drift out of sync. Model the parser state
as an `enum` and drive it one character at a time with `match (state, ch)`:

```rust
enum State { LineStart, InKey, AfterKey, BeforeValue, InBareValue,
             InQuotedValue, InQuotedEscape, AfterQuotedValue, InComment }
```

Every byte is handled in exactly one place, so the rules can't drift; the
end-of-input state determines whether you commit a final value or report an
`UnterminatedQuote`. Errors carry a line number. The naive `split('=')`-and-count
version silently drops `A=b=c` and lets `#` inside quotes start a comment — the
state machine fixes both by construction. (Full worked example:
`solutions/26_env_file_parser.rs` in corrode/refactoring-rust.)

### 27. Capstone — one typed value, one typed command, one typed reply

A "mini-redis" with three sidecar maps (`strings`, `lists`, `hashes`), an
`if/else` ladder over an uppercased command string, and hand-written wire bytes
(`+OK\r\n`) at every return site has three bug classes baked in: the same key can
exist in multiple maps at once, arity is checked three different ways, and one
missing `\r\n` hangs a client. Collapse each axis into a type:

```rust
enum Value { String(String), List(Vec<String>), Hash(HashMap<String, String>) }
// one store: HashMap<String, Value> — a key has exactly one type → WRONGTYPE errors become possible
enum Command { Set { key: String, value: String }, LPush { /*…*/ }, HSet { /*…*/ } }
// Command::parse(args) -> Result<Command, CommandError> does arity once, up front
enum Reply { Ok, Integer(i64), Error(String) }
impl fmt::Display for Reply { /* the ONE place that knows \r\n */ }
```

Dispatch becomes a `match` on `Command`. Adding `GET`/`DEL`/`INCR` is a new enum
variant and a new arm — the compiler shows you every place to update, and the
wire format lives in a single `Display` impl. (Full worked example:
`solutions/27_mini_redis.rs` in corrode/refactoring-rust.)

---

## Anti-patterns to avoid

From the [Rust Design Patterns book](https://rust-unofficial.github.io/patterns/).

### Cloning to satisfy the borrow checker

Reaching for `.clone()` to silence a borrow error hides the real problem and
desyncs two values that look related. Three honest options instead:

- **Restructure** so borrows don't overlap (shorten a borrow's scope, split a
  function, take `&` where you took `&mut`). This is the default.
- **`mem::take` / `mem::replace`** to move a value out of a `&mut` without cloning,
  leaving a default/placeholder behind:

  ```rust
  let owned = std::mem::take(&mut self.buffer);    // buffer now empty, owned is yours
  let prev  = std::mem::replace(&mut self.state, State::Idle);
  ```

- **`Rc<T>` / `Arc<T>`** when you genuinely need shared ownership (clones are then
  cheap refcount bumps, not deep copies).

Cloning *for simplicity in a cold path* is fine (see "Keep it simple" in
SKILL.md); cloning to *dodge* the borrow checker on a hot or central path is the
smell.

### Deref polymorphism

Implementing `Deref` to make a wrapper "inherit" the inner type's methods fakes
inheritance and surprises readers — trait bounds don't transfer, `self` is the
wrong type, and it only works one level deep. Use real composition: implement the
traits you want, write thin facade methods, or use a delegation crate (`delegate`,
`ambassador`). Only smart pointers should implement `Deref`.

### `#![deny(warnings)]` in source

Don't bake it into the crate — it breaks your build whenever a new compiler
version adds a lint or deprecates an API. Enforce `-D warnings` in CI instead (see
the validation loop in SKILL.md).

---

## GoF patterns map to Rust-native constructs

You rarely need the classic OO pattern names in Rust — the language gives you the
mechanism directly:

- **Command** → an `enum` of operations + a `match` dispatcher (exactly catalog
  #27, mini-redis).
- **Strategy** → pass a closure, or a `&dyn Trait` / generic `impl Trait`.
- **Visitor / Fold** → a trait with a method per node type, or an iterator +
  `fold`/`map`.
- **Interpreter** → an `enum` AST + a recursive `eval` over `match`.

Reach for the construct, not the ceremony.

---

## The meta-lessons

1. **Loops and nested matches are usually combinators in disguise.** Reach for
   iterator adapters and `Option`/`Result` methods before writing `mut` state.
2. **Borrow at boundaries.** `&str`/`&[T]`/`Option<&T>`; consume with `into_iter`
   when you own the data; clone last, not first.
3. **Parse, don't validate.** Convert stringly-typed input into types that prove
   the invariants, once, at the edge. Then illegal states can't be constructed.
4. **Make illegal states unrepresentable.** Enums over bool soup; newtypes and
   `NonZero*` over "remember to check"; one typed model over parallel maps.
5. **Errors are values — report them honestly.** `Result` + `?` over `unwrap`;
   custom error enums over discarding the reason with `Option`.
6. **Let clippy and the compiler teach you.** Most of the above is enforceable;
   run `cargo clippy -- -D warnings` and fix, don't `#[allow]`.
7. **Exhaustive matches over `_` on your own enums.** A catch-all silently absorbs
   new variants; listing them makes the compiler flag every place to update.
   Reserve `_` for open/numeric/string matches and foreign types — which is why
   examples 2, 18, and 24 above legitimately use it (they match on numbers and
   strings, not your own closed enum).
