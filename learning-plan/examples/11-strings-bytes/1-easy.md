# Step 11 — Strings, Runes & Bytes · 🟢 Easy

Examples **1–6**: the three lengths of a string, immutability and free substrings, the price of every
conversion, the UTF-8 indexing traps, `+=` against `strings.Builder`, and why comparison is bytes.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic; **Sample output:** ones print timings or memory, which vary between runs.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. One string, three lengths

`🟢 easy` · *UTF-8 encoding*

A Go string is a sequence of **bytes**, conventionally UTF-8, and it has three lengths: `len(s)` counts bytes,
`utf8.RuneCountInString` counts code points (runes), and what a reader calls "a character" is a third thing again.
Seeing the encoding once — 1 to 4 bytes per code point, the lead byte's leading 1s giving the width, every
continuation byte starting `10` — makes every later trap in this lesson obvious. The costs differ too: `len` is
Θ(1) because the byte count sits in the 16-byte header, while counting runes is Θ(n) because widths vary and
every byte has to be decoded.

**Steps:**

1. Encode `A`, `é`, `€` and `😀` with `utf8.EncodeRune` and print each byte in binary.
2. Tabulate bytes, runes and (as typed-in data) what you see for six strings, including `"e\u0301"`.
3. Count the rune decoder's work on a 13,000-byte string: it examines every byte, while `len` examines none.

```go
package main

import (
	"fmt"
	"strings"
	"unicode/utf8"
)

// A Go string is a sequence of BYTES. By convention (and always, for a string
// literal in Go source) those bytes are UTF-8, which spends 1 to 4 bytes on
// each code point. The lead byte says how many:
//
//	0xxxxxxx                              1 byte   U+0000..U+007F (ASCII)
//	110xxxxx 10xxxxxx                     2 bytes  U+0080..U+07FF
//	1110xxxx 10xxxxxx 10xxxxxx            3 bytes  U+0800..U+FFFF
//	11110xxx 10xxxxxx 10xxxxxx 10xxxxxx   4 bytes  U+10000..U+10FFFF
//
// So one string has three lengths:
//
//	len(s)                     bytes       -- read from the header, Theta(1)
//	utf8.RuneCountInString(s)  code points -- must decode every byte, Theta(n)
//	what a reader sees         graphemes   -- neither of the above (example 15)

// countRunes is utf8.RuneCountInString written out, so we can count the work.
// Each step decodes one rune and moves forward by its width.
func countRunes(s string) (runes, examined int) {
	for i := 0; i < len(s); {
		_, w := utf8.DecodeRuneInString(s[i:])
		i += w
		examined += w
		runes++
	}
	return runes, examined
}

// bits prints bytes in binary so the lead/continuation patterns are visible.
func bits(b []byte) string {
	parts := make([]string, len(b))
	for i, x := range b {
		parts[i] = fmt.Sprintf("%08b", x)
	}
	return strings.Join(parts, " ")
}

func main() {
	fmt.Println("Part 1 -- one code point, 1 to 4 bytes:")
	fmt.Printf("  %-4s %-8s %-12s %5s  %s\n", "char", "code pt", "UTF-8 hex", "width", "bytes in binary")
	for _, r := range []rune{'A', 'é', '€', '😀'} {
		buf := make([]byte, 4)
		n := utf8.EncodeRune(buf, r) // writes the UTF-8 bytes, returns how many
		hex := fmt.Sprintf("% x", buf[:n])
		cp := fmt.Sprintf("U+%04X", r)
		fmt.Printf("  %-4s %-8s %-12s %5d  %s\n", string(r), cp, hex, utf8.RuneLen(r), bits(buf[:n]))
	}
	fmt.Println("  the lead byte's leading 1s = the width; every other byte starts 10")

	fmt.Println()
	fmt.Println("Part 2 -- three lengths of one string:")
	// The "you see" column is DATA typed in by a human, not computed:
	// counting what a reader sees needs grapheme segmentation (example 15).
	rows := []struct {
		s     string
		label string // how the string is written in the source
		see   int
	}{
		{"hello", `"hello"`, 5},
		{"héllo", `"héllo"`, 5},
		{"naïve café", `"naïve café"`, 10},
		{"日本語", `"日本語"`, 3},
		{"😀👍", `"😀👍"`, 2},
		{"e\u0301", `"e" + U+0301 (combining acute)`, 1},
	}
	fmt.Printf("  %5s %5s %8s   %s\n", "bytes", "runes", "you see", "string")
	for _, row := range rows {
		fmt.Printf("  %5d %5d %8d   %s\n", len(row.s), utf8.RuneCountInString(row.s), row.see, row.label)
	}
	fmt.Println("  bytes == runes only for ASCII; runes == 'you see' fails for e + U+0301")

	fmt.Println()
	fmt.Println("Part 3 -- what each length costs:")
	big := strings.Repeat("naïve café ", 1000)
	runes, examined := countRunes(big)
	fmt.Printf("  len(big)        = %d   (read from the 16-byte header, 0 bytes examined)\n", len(big))
	fmt.Printf("  rune count      = %d   (%d bytes examined: every byte, len = %d)\n", runes, examined, len(big))
	fmt.Printf("  RuneCountInString agrees: %v\n", utf8.RuneCountInString(big) == runes)
}
```

**Output:**

```
Part 1 -- one code point, 1 to 4 bytes:
  char code pt  UTF-8 hex    width  bytes in binary
  A    U+0041   41               1  01000001
  é    U+00E9   c3 a9            2  11000011 10101001
  €    U+20AC   e2 82 ac         3  11100010 10000010 10101100
  😀    U+1F600  f0 9f 98 80      4  11110000 10011111 10011000 10000000
  the lead byte's leading 1s = the width; every other byte starts 10

Part 2 -- three lengths of one string:
  bytes runes  you see   string
      5     5        5   "hello"
      6     5        5   "héllo"
     12    10       10   "naïve café"
      9     3        3   "日本語"
      8     2        2   "😀👍"
      3     2        1   "e" + U+0301 (combining acute)
  bytes == runes only for ASCII; runes == 'you see' fails for e + U+0301

Part 3 -- what each length costs:
  len(big)        = 13000   (read from the 16-byte header, 0 bytes examined)
  rune count      = 11000   (13000 bytes examined: every byte, len = 13000)
  RuneCountInString agrees: true
```

**Rust:**

```rust
// A Rust `str` is a sequence of BYTES that is ALWAYS valid UTF-8 (the type
// guarantees it -- Go's string only promises it by convention). UTF-8 spends
// 1 to 4 bytes on each code point. The lead byte says how many:
//
//   0xxxxxxx                              1 byte   U+0000..U+007F (ASCII)
//   110xxxxx 10xxxxxx                     2 bytes  U+0080..U+07FF
//   1110xxxx 10xxxxxx 10xxxxxx            3 bytes  U+0800..U+FFFF
//   11110xxx 10xxxxxx 10xxxxxx 10xxxxxx   4 bytes  U+10000..U+10FFFF
//
// So one string has three lengths:
//
//   s.len()            bytes       -- read from the (ptr, len) pair, Theta(1)
//   s.chars().count()  code points -- must decode every byte, Theta(n)
//   what a reader sees graphemes   -- neither of the above (example 15)
//
// A `char` is a Unicode scalar value: a code point that is not a surrogate.
// size_of::<char>() == 4, the same as Go's `rune` (int32).

/// count_runes is `s.chars().count()` written out, so we can count the work.
/// `chars()` decodes one code point per step and moves forward by its width;
/// `len_utf8()` is that width.
fn count_runes(s: &str) -> (usize, usize) {
    let (mut runes, mut examined) = (0, 0);
    for c in s.chars() {
        examined += c.len_utf8();
        runes += 1;
    }
    (runes, examined)
}

/// bits prints bytes in binary so the lead/continuation patterns are visible.
fn bits(b: &[u8]) -> String {
    b.iter()
        .map(|x| format!("{x:08b}"))
        .collect::<Vec<_>>()
        .join(" ")
}

fn main() {
    println!("Part 1 -- one code point, 1 to 4 bytes:");
    println!(
        "  {:<4} {:<8} {:<12} {:>5}  bytes in binary",
        "char", "code pt", "UTF-8 hex", "width"
    );
    for c in ['A', 'é', '€', '😀'] {
        let mut buf = [0u8; 4];
        let enc = c.encode_utf8(&mut buf); // writes the UTF-8 bytes, returns them as &mut str
        let bytes = enc.as_bytes();
        let hex = bytes
            .iter()
            .map(|x| format!("{x:02x}"))
            .collect::<Vec<_>>()
            .join(" ");
        let cp = format!("U+{:04X}", c as u32);
        println!(
            "  {:<4} {:<8} {:<12} {:>5}  {}",
            c,
            cp,
            hex,
            c.len_utf8(),
            bits(bytes)
        );
    }
    println!("  the lead byte's leading 1s = the width; every other byte starts 10");

    println!();
    println!("Part 2 -- three lengths of one string:");
    // The "you see" column is DATA typed in by a human, not computed:
    // counting what a reader sees needs grapheme segmentation (example 15).
    let rows: [(&str, &str, usize); 6] = [
        ("hello", "\"hello\"", 5),
        ("héllo", "\"héllo\"", 5),
        ("naïve café", "\"naïve café\"", 10),
        ("日本語", "\"日本語\"", 3),
        ("😀👍", "\"😀👍\"", 2),
        ("e\u{301}", "\"e\" + U+0301 (combining acute)", 1),
    ];
    println!("  {:>5} {:>5} {:>8}   string", "bytes", "runes", "you see");
    for (s, label, see) in rows {
        println!(
            "  {:>5} {:>5} {:>8}   {}",
            s.len(),
            s.chars().count(),
            see,
            label
        );
    }
    println!("  bytes == runes only for ASCII; runes == 'you see' fails for e + U+0301");

    println!();
    println!("Part 3 -- what each length costs:");
    let big = "naïve café ".repeat(1000);
    let (runes, examined) = count_runes(&big);
    println!(
        "  len(big)        = {}   (a stored length field, 0 bytes examined)",
        big.len()
    );
    println!(
        "  rune count      = {}   ({} bytes examined: every byte, len = {})",
        runes,
        examined,
        big.len()
    );
    println!("  chars().count() agrees: {}", big.chars().count() == runes);
    println!(
        "  size_of::<char>() = {}, the same as Go's rune",
        std::mem::size_of::<char>()
    );
}
```

**Rust output:**

```
Part 1 -- one code point, 1 to 4 bytes:
  char code pt  UTF-8 hex    width  bytes in binary
  A    U+0041   41               1  01000001
  é    U+00E9   c3 a9            2  11000011 10101001
  €    U+20AC   e2 82 ac         3  11100010 10000010 10101100
  😀    U+1F600  f0 9f 98 80      4  11110000 10011111 10011000 10000000
  the lead byte's leading 1s = the width; every other byte starts 10

Part 2 -- three lengths of one string:
  bytes runes  you see   string
      5     5        5   "hello"
      6     5        5   "héllo"
     12    10       10   "naïve café"
      9     3        3   "日本語"
      8     2        2   "😀👍"
      3     2        1   "e" + U+0301 (combining acute)
  bytes == runes only for ASCII; runes == 'you see' fails for e + U+0301

Part 3 -- what each length costs:
  len(big)        = 13000   (a stored length field, 0 bytes examined)
  rune count      = 11000   (13000 bytes examined: every byte, len = 13000)
  chars().count() agrees: true
  size_of::<char>() = 4, the same as Go's rune
```

> **Go vs Rust:** All counts are identical: 5/6/12/9/8/3 bytes, 13,000 bytes examined for 11,000 code points. The difference is in the type, not the numbers: a Rust `&str` is guaranteed valid UTF-8, so `chars()` never meets a bad byte, while a Go string may hold any bytes and `DecodeRuneInString` must handle them (example 4). `char` is a Unicode scalar value (surrogates excluded), 4 bytes like Go's `rune`; both `{:<4}` and Go's `%-4s` pad by code points, not bytes, which is why the tables line up the same.

**Complexity:** `len(s)` Θ(1) — stored in the 16-byte header · rune count Θ(n) — the 13,000-byte string took 13,000 bytes examined for 11,000 runes · a character (grapheme) is neither: `"e\u0301"` is 3 bytes, 2 runes, 1 on screen (example 15)

---

## 2. Immutable, and why a substring is free

`🟢 easy` · *Immutability*

A Go string is a 16-byte header (pointer, length) over bytes that nobody may change — `s[0] = 'x'` does not
compile. That one rule is why assignment copies 16 bytes and not the data, why `s[i:j]` is Θ(1) with no
allocation (the new header simply points `i` bytes into the same array), and why any goroutine can read any
string without a lock. The other side of the rule: "changing" a string means building a new one, which is Θ(n)
and one allocation even when you change a single byte.

**Steps:**

1. Build a 100-byte string at run time and compare `unsafe.Sizeof` of a string and a `[]byte`.
2. Take `t := s` and `sub := s[10:20]`, print where their data starts relative to `s`, and count allocations.
3. "Edit" with `"X" + s[1:]`: observe a fresh array, one allocation, and every byte copied.

```go
package main

import (
	"fmt"
	"strings"
	"testing"
	"unsafe"
)

// A Go string is a two-word header -- (pointer, length) -- over bytes that
// nobody is allowed to change. The compiler enforces it:
//
//	s[0] = 'x'
//	// ./main.go:5:2: cannot assign to s[0] (neither addressable nor a map index expression)
//
// That one rule buys three things:
//   - assignment copies the 16-byte header, never the bytes;
//   - s[i:j] is a new header pointing i bytes into the SAME bytes: Theta(1), no allocation;
//   - any goroutine can read any string without a lock.
// The price: "changing" a string means building a new one -- Theta(n) and an allocation.

// Results go to package-level sinks so the compiler cannot keep them on the
// stack (escape analysis would otherwise make the allocation counts read 0).
var sinkS string

// offset reports how far into s the bytes of t start (t must share s's bytes
// for the number to mean anything). We print offsets, never raw addresses,
// so the output is the same on every run.
func offset(s, t string) int {
	return int(uintptr(unsafe.Pointer(unsafe.StringData(t))) - uintptr(unsafe.Pointer(unsafe.StringData(s))))
}

// inside reports whether t's bytes start within s's bytes.
func inside(s, t string) bool {
	o := offset(s, t)
	return o >= 0 && o < len(s)
}

func main() {
	s := strings.Repeat("immutable ", 10) // 100 bytes, built at run time
	fmt.Printf("s = %q... (%d bytes)\n", s[:20], len(s))

	fmt.Println()
	fmt.Println("(b) the header:")
	fmt.Printf("  unsafe.Sizeof(s)        = %d bytes (pointer + length)\n", unsafe.Sizeof(s))
	fmt.Printf("  unsafe.Sizeof([]byte{}) = %d bytes (pointer + length + capacity)\n", unsafe.Sizeof([]byte{}))

	fmt.Println()
	fmt.Println("(c) assignment and substrings share the bytes:")
	t := s
	sub := s[10:20]
	fmt.Printf("  t := s          -> data at offset %2d into s\n", offset(s, t))
	fmt.Printf("  sub := s[10:20] -> data at offset %2d into s, sub = %q\n", offset(s, sub), sub)
	allocs := testing.AllocsPerRun(1000, func() { sinkS = s[10:20] })
	fmt.Printf("  allocations per s[10:20]: %.0f   (a new 16-byte header, no bytes copied)\n", allocs)
	allocs = testing.AllocsPerRun(1000, func() { sinkS = s[1:] })
	fmt.Printf("  allocations per s[1:]:    %.0f   (99 bytes long -- the size never matters)\n", allocs)

	fmt.Println()
	fmt.Println("(d) an \"edit\" builds a new string:")
	s2 := "X" + s[1:]
	fmt.Printf("  s2 := \"X\" + s[1:] -> %q...\n", s2[:20])
	fmt.Printf("  s2's bytes inside s's bytes? %v   (a fresh array)\n", inside(s, s2))
	fmt.Printf("  s is unchanged:              %q...\n", s[:20])
	allocs = testing.AllocsPerRun(1000, func() { sinkS = "X" + s[1:] })
	fmt.Printf("  allocations per edit: %.0f, bytes copied: %d (1 + %d) -- Theta(n) to change 1 byte\n",
		allocs, 1+len(s[1:]), len(s[1:]))

	b := []byte(s) // the other way to edit: copy into a mutable []byte ...
	b[0] = 'X'     // ... which CAN be assigned to
	fmt.Printf("  []byte(s) then b[0] = 'X': s still starts %q, b starts %q\n", s[:3], string(b[:3]))

	fmt.Println()
	fmt.Println("(e) sharing has a cost: sub is 10 bytes, but it holds a pointer into s's")
	fmt.Printf("    %d bytes, so the garbage collector keeps all of them alive while sub lives.\n", len(s))
	fmt.Println("    Example 13 measures how much a few tiny substrings can pin.")
}
```

**Output:**

```
s = "immutable immutable "... (100 bytes)

(b) the header:
  unsafe.Sizeof(s)        = 16 bytes (pointer + length)
  unsafe.Sizeof([]byte{}) = 24 bytes (pointer + length + capacity)

(c) assignment and substrings share the bytes:
  t := s          -> data at offset  0 into s
  sub := s[10:20] -> data at offset 10 into s, sub = "immutable "
  allocations per s[10:20]: 0   (a new 16-byte header, no bytes copied)
  allocations per s[1:]:    0   (99 bytes long -- the size never matters)

(d) an "edit" builds a new string:
  s2 := "X" + s[1:] -> "Xmmutable immutable "...
  s2's bytes inside s's bytes? false   (a fresh array)
  s is unchanged:              "immutable immutable "...
  allocations per edit: 1, bytes copied: 100 (1 + 99) -- Theta(n) to change 1 byte
  []byte(s) then b[0] = 'X': s still starts "imm", b starts "Xmm"

(e) sharing has a cost: sub is 10 bytes, but it holds a pointer into s's
    100 bytes, so the garbage collector keeps all of them alive while sub lives.
    Example 13 measures how much a few tiny substrings can pin.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// In Rust, immutability lives in the BORROW, not in the type:
//
//   &str    a shared view (pointer, length) -- nobody may change the bytes
//           while it exists. This is Go's string.
//   String  an owned, growable buffer (pointer, length, capacity) -- its owner
//           may change the bytes in place.
//
// Go's `s[0] = 'x'` does not compile in Rust either, but for a different reason
// (there is no integer indexing on str at all -- example 4):
//
//   error[E0277]: the type `str` cannot be indexed by `{integer}`
//     |     s[0] = b'x';
//     |       ^ string indices are ranges of `usize`
//
// And the thing Go forbids for EVERY string, Rust forbids only while a view is
// alive:
//
//   let r: &str = &s;
//   s.make_ascii_uppercase();
//   println!("{r}");
//   // error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable

// ---- allocation counter: stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes requested

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested, result) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

/// How far into s the bytes of t start (offsets, never raw addresses, so the
/// output is the same on every run).
fn offset(s: &str, t: &str) -> isize {
    t.as_ptr() as isize - s.as_ptr() as isize
}

/// Whether t's bytes start within s's bytes.
fn inside(s: &str, t: &str) -> bool {
    let o = offset(s, t);
    o >= 0 && (o as usize) < s.len()
}

fn main() {
    let mut s = "immutable ".repeat(10); // 100 bytes, an owned String
    println!("s = {:?}... ({} bytes)", &s[..20], s.len());

    println!();
    println!("(b) the headers:");
    println!(
        "  size_of::<&str>()    = {} bytes (pointer + length)          <- Go's string",
        size_of::<&str>()
    );
    println!(
        "  size_of::<String>()  = {} bytes (pointer + length + capacity)",
        size_of::<String>()
    );
    println!(
        "  size_of::<Vec<u8>>() = {} bytes (a String IS a Vec<u8> that holds UTF-8)",
        size_of::<Vec<u8>>()
    );

    println!();
    println!("(c) views share the bytes:");
    {
        let t: &str = &s;
        let sub = &s[10..20];
        println!(
            "  t = &s[..]       -> data at offset {:2} into s",
            offset(&s, t)
        );
        println!(
            "  sub = &s[10..20] -> data at offset {:2} into s, sub = {:?}",
            offset(&s, sub),
            sub
        );
        let (n, _, _) = allocs_of(|| black_box(&s[10..20]).len());
        println!("  allocations per &s[10..20]: {n}   (a new 16-byte view, no bytes copied)");
        let (n, _, _) = allocs_of(|| black_box(&s[1..]).len());
        println!("  allocations per &s[1..]:    {n}   (99 bytes long -- the size never matters)");
        let (n, bytes, copy) = allocs_of(|| s.clone());
        println!(
            "  s.clone() -- Rust's deep copy: {n} allocation, {bytes} bytes (inside s? {})",
            inside(&s, &copy)
        );
    }

    println!();
    println!("(d) an edit can build a new string (Go's only option) ...");
    let (n, bytes, s2) = allocs_of(|| ["X", &s[1..]].concat());
    println!("  s2 = [\"X\", &s[1..]].concat() -> {:?}...", &s2[..20]);
    println!(
        "  s2's bytes inside s's bytes? {}   (a fresh array)",
        inside(&s, &s2)
    );
    println!("  allocations: {n}, bytes: {bytes} -- Theta(n) to change 1 byte");

    println!("... or, because s is an owned String with no view alive, change it IN PLACE:");
    let before = s.as_ptr();
    let (n, _, _) = allocs_of(|| s.replace_range(0..1, "X"));
    println!(
        "  s.replace_range(0..1, \"X\") -> {:?}...  allocations: {n}, same buffer: {}",
        &s[..20],
        s.as_ptr() == before
    );
    let (n, _, _) = allocs_of(|| s.make_ascii_uppercase());
    println!(
        "  s.make_ascii_uppercase()   -> {:?}...  allocations: {n}, same buffer: {}",
        &s[..20],
        s.as_ptr() == before
    );
    println!("  (nothing grew here; a longer replacement may have to reallocate)");

    println!();
    println!("(e) sharing has a cost in Go (a substring pins its parent for the GC).");
    println!("    In Rust a &str borrows its String: the String cannot be dropped or");
    println!("    changed while the view lives -- the compiler checks it. Example 13.");
}
```

**Rust output:**

```
s = "immutable immutable "... (100 bytes)

(b) the headers:
  size_of::<&str>()    = 16 bytes (pointer + length)          <- Go's string
  size_of::<String>()  = 24 bytes (pointer + length + capacity)
  size_of::<Vec<u8>>() = 24 bytes (a String IS a Vec<u8> that holds UTF-8)

(c) views share the bytes:
  t = &s[..]       -> data at offset  0 into s
  sub = &s[10..20] -> data at offset 10 into s, sub = "immutable "
  allocations per &s[10..20]: 0   (a new 16-byte view, no bytes copied)
  allocations per &s[1..]:    0   (99 bytes long -- the size never matters)
  s.clone() -- Rust's deep copy: 1 allocation, 100 bytes (inside s? false)

(d) an edit can build a new string (Go's only option) ...
  s2 = ["X", &s[1..]].concat() -> "Xmmutable immutable "...
  s2's bytes inside s's bytes? false   (a fresh array)
  allocations: 1, bytes: 100 -- Theta(n) to change 1 byte
... or, because s is an owned String with no view alive, change it IN PLACE:
  s.replace_range(0..1, "X") -> "Xmmutable immutable "...  allocations: 0, same buffer: true
  s.make_ascii_uppercase()   -> "XMMUTABLE IMMUTABLE "...  allocations: 0, same buffer: true
  (nothing grew here; a longer replacement may have to reallocate)

(e) sharing has a cost in Go (a substring pins its parent for the GC).
    In Rust a &str borrows its String: the String cannot be dropped or
    changed while the view lives -- the compiler checks it. Example 13.
```

> **Go vs Rust:** Same offsets (0 and 10), same 0 allocations for any slice length, same 1 allocation and 100 bytes for the copy-edit. What differs is where immutability lives: Go makes every string immutable, so an edit must copy; Rust makes the *view* (`&str`) immutable and the owned `String` mutable, so `replace_range` and `make_ascii_uppercase` changed the bytes in place with 0 allocations — and the borrow checker (E0502) refuses that while any `&str` into it is alive. `let t = s` on a `String` is a move (no bytes copied, `s` unusable after); `clone()` is the explicit deep copy.

**Complexity:** substring Θ(1), 0 allocations whatever its length (offsets 0 and 10 into the parent) · assignment copies the 16-byte header, never the bytes · any edit Θ(n), 1 allocation — 100 bytes copied to change 1

---

## 3. The `string` / `[]byte` / `[]rune` triangle

`🟢 easy` · *Conversions*

Text has three representations in Go: `string` (immutable bytes), `[]byte` (mutable bytes) and `[]rune` (one
`int32` per code point, so the i-th code point is Θ(1) away). Every conversion between them is Θ(n) and allocates,
because each side must not be able to change the other — and `[]rune` costs 4 bytes per code point however short
the UTF-8 was. The compiler skips the copy only in a few temporaries it can prove nobody keeps. Knowing the price
of each edge is how you choose the representation an algorithm should run on, instead of converting back and
forth inside a loop.

**Steps:**

1. Build a 3,000-byte ASCII string and a 6,000-byte mixed one (`"aé€"` × 1,000), both 3,000 runes.
2. Count allocations and bytes per conversion with the runtime's counters, results escaping to package-level sinks.
3. Measure the compiler's free cases — map lookup, comparison, one-byte string — and one that is only half free.

```go
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
	"strings"
	"unicode/utf8"
)

// Three representations of text, three edges between them:
//
//	string  immutable bytes          s[i] is a byte
//	[]byte  mutable bytes            b[i] is a byte you may assign
//	[]rune  one int32 per code point r[i] is the i-th code point, Theta(1)
//
// Every conversion must COPY: a []byte made from a string must not be able to
// change the string, and a string made from a []byte must not change when the
// slice does later. []rune(s) also re-encodes, at 4 bytes per code point, and
// makes two passes (count the runes, then fill -- runtime.stringtoslicerune).
// The compiler skips the copy only where it can prove nobody will notice.

// Sinks: results must escape to the heap, or a short result could live in a
// 32-byte stack buffer and the counts would read 0 for the wrong reason.
var (
	sinkS  string
	sinkB  []byte
	sinkR  []rune
	sinkI  int
	sinkOK bool
)

// measure is what testing.AllocsPerRun and AllocedBytesPerOp compute: the
// runtime's malloc counter and allocated-bytes counter, before and after.
// Go's byte count is rounded up to the allocator's size class (3000 -> 3072).
// The GC is paused while we count, so its own bookkeeping cannot leak in.
func measure(f func()) (allocs, bytes uint64) {
	const runs = 1000
	f() // warm up
	runtime.GC()
	defer debug.SetGCPercent(debug.SetGCPercent(-1))
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	for range runs {
		f()
	}
	runtime.ReadMemStats(&b)
	return (b.Mallocs - a.Mallocs) / runs, (b.TotalAlloc - a.TotalAlloc) / runs
}

func main() {
	ascii := strings.Repeat("abc", 1000) // 3000 bytes, 3000 runes
	mixed := strings.Repeat("aé€", 1000) // 1+2+3 bytes per repeat: 6000 bytes, 3000 runes

	fmt.Printf("ascii: %d bytes, %d runes    mixed: %d bytes, %d runes\n\n",
		len(ascii), utf8.RuneCountInString(ascii), len(mixed), utf8.RuneCountInString(mixed))

	type edge struct {
		name string
		op   func(s string, b []byte, r []rune)
	}
	edges := []edge{
		{"[]byte(s)", func(s string, b []byte, r []rune) { sinkB = []byte(s) }},
		{"string(b)", func(s string, b []byte, r []rune) { sinkS = string(b) }},
		{"[]rune(s)", func(s string, b []byte, r []rune) { sinkR = []rune(s) }},
		{"string(r)", func(s string, b []byte, r []rune) { sinkS = string(r) }},
		{"utf8.RuneCountInString(s)", func(s string, b []byte, r []rune) { sinkI = utf8.RuneCountInString(s) }},
		// 6 and 2502 are multiples of 6, so the cut is between two "aé€" in mixed
		{"s[6:2502]", func(s string, b []byte, r []rune) { sinkS = s[6:2502] }},
	}
	fmt.Printf("  %-27s %18s   %18s\n", "conversion", "ascii allocs/bytes", "mixed allocs/bytes")
	for _, e := range edges {
		fmt.Printf("  %-27s", e.name)
		for i, s := range []string{ascii, mixed} {
			b, r := []byte(s), []rune(s)
			allocs, bytes := measure(func() { e.op(s, b, r) })
			if i > 0 {
				fmt.Print("  ")
			}
			fmt.Printf(" %8d / %7d", allocs, bytes)
		}
		fmt.Println()
	}
	payload := 4 * utf8.RuneCountInString(ascii) // the same 3000 runes in both
	fmt.Printf("  []rune payload: 4 x 3000 runes = %d bytes for both -- %dx ascii's bytes, %dx mixed's\n",
		payload, payload/len(ascii), payload/len(mixed))

	fmt.Println()
	fmt.Println("the conversions the compiler makes free (b holds ascii's 3000 bytes):")
	b := []byte(ascii)
	m := map[string]int{ascii: 42}
	free := []edge{
		{"m[string(b)]", func(string, []byte, []rune) { sinkI = m[string(b)] }},
		{`string(b) == "foo"`, func(string, []byte, []rune) { sinkOK = string(b) == "foo" }},
		{"string(b[:1])", func(string, []byte, []rune) { sinkS = string(b[:1]) }},
		{`"<" + string(b) + ">"`, func(string, []byte, []rune) { sinkS = "<" + string(b) + ">" }},
	}
	for _, e := range free {
		allocs, bytes := measure(func() { e.op("", nil, nil) })
		fmt.Printf("  %-27s %8d / %7d\n", e.name, allocs, bytes)
	}
	fmt.Println("  the map lookup and the comparison only READ the bytes, so no copy is made;")
	fmt.Println("  a 1-byte string points into a static table of all 256 one-byte strings;")
	fmt.Println("  the concatenation allocates its result once, but string(b) inside it is free")
}
```

**Output:**

```
ascii: 3000 bytes, 3000 runes    mixed: 6000 bytes, 3000 runes

  conversion                  ascii allocs/bytes   mixed allocs/bytes
  []byte(s)                          1 /    3072          1 /    6144
  string(b)                          1 /    3072          1 /    6144
  []rune(s)                          1 /   12288          1 /   12288
  string(r)                          1 /    3072          1 /    6144
  utf8.RuneCountInString(s)          0 /       0          0 /       0
  s[6:2502]                          0 /       0          0 /       0
  []rune payload: 4 x 3000 runes = 12000 bytes for both -- 4x ascii's bytes, 2x mixed's

the conversions the compiler makes free (b holds ascii's 3000 bytes):
  m[string(b)]                       0 /       0
  string(b) == "foo"                 0 /       0
  string(b[:1])                      0 /       0
  "<" + string(b) + ">"              1 /    3072
  the map lookup and the comparison only READ the bytes, so no copy is made;
  a 1-byte string points into a static table of all 256 one-byte strings;
  the concatenation allocates its result once, but string(b) inside it is free
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// The same three representations in Rust:
//
//   &str / String     bytes, always valid UTF-8   (Go: string)
//   &[u8] / Vec<u8>   bytes, any bytes             (Go: []byte)
//   Vec<char>         4 bytes per code point       (Go: []rune)
//
// Go's edges always copy, because a Go string must never change. In Rust the
// borrow checker already guarantees that nobody changes bytes while a view is
// alive -- so most edges are BORROWS (look at the same bytes) or MOVES (take
// over the same buffer), and cost no allocation. What is left to pay for is
// UTF-8 VALIDATION: bytes -> str must check every byte, Theta(n), and can fail.

// ---- allocation counter: alloc and realloc counted separately ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0);
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        REALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// "allocs+reallocs / bytes the result holds" while running `f` once. The
/// result is kept alive until after the count, so the live-byte delta is
/// exactly its heap size -- Rust's numbers are exact, not size-class rounded.
fn measure<R>(f: impl FnOnce() -> R) -> String {
    let (a0, r0, l0) = (
        ALLOCS.load(Relaxed),
        REALLOCS.load(Relaxed),
        LIVE.load(Relaxed),
    );
    let r = black_box(f());
    let (a, re, live) = (
        ALLOCS.load(Relaxed) - a0,
        REALLOCS.load(Relaxed) - r0,
        LIVE.load(Relaxed) - l0,
    );
    drop(r);
    format!("{a}+{re} / {live:>5}")
}

fn main() {
    let ascii = "abc".repeat(1000); // 3000 bytes, 3000 chars
    let mixed = "aé€".repeat(1000); // 1+2+3 bytes per repeat: 6000 bytes, 3000 chars

    println!(
        "ascii: {} bytes, {} runes    mixed: {} bytes, {} runes\n",
        ascii.len(),
        ascii.chars().count(),
        mixed.len(),
        mixed.chars().count()
    );

    // One row per edge: (Go spelling it replaces, Rust spelling, the operation).
    type Op = fn(&str, &[u8], &[char]) -> String;
    let edges: [(&str, &str, Op); 10] = [
        ("", "s.as_bytes()", |s, _, _| measure(|| s.as_bytes())),
        ("[]byte(s)", "s.as_bytes().to_vec()", |s, _, _| {
            measure(|| s.as_bytes().to_vec())
        }),
        ("", "String::into_bytes(own)", |s, _, _| {
            let own = s.to_owned();
            measure(move || own.into_bytes()) // gives the SAME buffer back
        }),
        ("", "str::from_utf8(&b)", |_, b, _| {
            measure(|| std::str::from_utf8(b).unwrap()) // validates, borrows
        }),
        ("string(b)", "String::from_utf8(v)", |_, b, _| {
            let v = b.to_vec();
            measure(move || String::from_utf8(v).unwrap()) // validates, takes over v
        }),
        ("[]rune(s)", "s.chars().collect()", |s, _, _| {
            measure(|| s.chars().collect::<Vec<char>>())
        }),
        ("", "  ...with_capacity(count)", |s, _, _| {
            measure(|| {
                let mut v = Vec::with_capacity(s.chars().count()); // pass 1: count
                v.extend(s.chars()); // pass 2: fill -- Go's two passes
                v
            })
        }),
        ("string(r)", "r.iter().collect()", |_, _, r| {
            measure(|| r.iter().collect::<String>())
        }),
        ("RuneCountInString", "s.chars().count()", |s, _, _| {
            measure(|| s.chars().count())
        }),
        // 6 and 2502 are multiples of 6: between two "aé€" in mixed. Byte 5 is
        // inside a '€', and &s[5..] would PANIC (example 4) -- Go would just cut.
        ("s[6:2502]", "&s[6..2502]", |s, _, _| {
            measure(|| &s[6..2502])
        }),
    ];
    println!(
        "  {:<18} {:<27} {:>14}   {:>14}",
        "Go", "Rust", "ascii a+r/bytes", "mixed a+r/bytes"
    );
    let inputs: Vec<(&str, Vec<u8>, Vec<char>)> = [&ascii, &mixed]
        .iter()
        .map(|s| (s.as_str(), s.as_bytes().to_vec(), s.chars().collect()))
        .collect();
    for (go, rust, op) in edges {
        let cells: Vec<String> = inputs.iter().map(|(s, b, r)| op(s, b, r)).collect();
        println!(
            "  {:<18} {:<27} {:>14}   {:>14}",
            go, rust, cells[0], cells[1]
        );
    }
    println!("  a+r = allocations + reallocations; bytes = exactly what the result holds");
    println!(
        "  Vec<char> payload: 4 x 3000 chars = {} bytes for both; collect() guesses",
        4 * ascii.chars().count()
    );
    println!("  its size from the byte length (Chars only knows len/4..len), so it grows");

    println!();
    println!("validation is the price, and it can fail -- the bytes come back unharmed:");
    let bad = vec![b'a', 0xff, b'b'];
    match String::from_utf8(bad) {
        Ok(s) => println!("  unexpected: {s}"),
        Err(e) => println!(
            "  String::from_utf8([61 ff 62]) -> Err({}), bytes back: {:02x?}",
            e.utf8_error(),
            e.into_bytes()
        ),
    }

    println!();
    println!("Go's compiler-elided cases (b holds ascii's 3000 bytes):");
    let b = ascii.as_bytes().to_vec();
    let m: HashMap<String, i32> = HashMap::from([(ascii.clone(), 42)]);
    let cases: [(&str, &str, String); 4] = [
        (
            "m[string(b)]",
            "m.get(str::from_utf8(&b)?)",
            measure(|| *m.get(std::str::from_utf8(&b).unwrap()).unwrap()),
        ),
        (
            "string(b) == \"foo\"",
            "b == b\"foo\"",
            measure(|| b == b"foo"),
        ),
        ("string(b[:1])", "&ascii[..1]", measure(|| &ascii[..1])),
        (
            "\"<\"+string(b)+\">\"",
            "[\"<\", s, \">\"].concat()",
            measure(|| ["<", &ascii, ">"].concat()),
        ),
    ];
    for (go, rust, cell) in cases {
        println!("  {go:<18} {rust:<27} {cell:>14}");
    }
    println!("  in Rust these are not compiler tricks: a lookup by &str, a comparison of");
    println!("  slices and a sub-slice are ordinary borrows; only the concatenation builds");
    println!("  a new value, and concat() sums the lengths first so it allocates once");
}
```

**Rust output:**

```
ascii: 3000 bytes, 3000 runes    mixed: 6000 bytes, 3000 runes

  Go                 Rust                        ascii a+r/bytes   mixed a+r/bytes
                     s.as_bytes()                   0+0 /     0      0+0 /     0
  []byte(s)          s.as_bytes().to_vec()          1+0 /  3000      1+0 /  6000
                     String::into_bytes(own)        0+0 /     0      0+0 /     0
                     str::from_utf8(&b)             0+0 /     0      0+0 /     0
  string(b)          String::from_utf8(v)           0+0 /     0      0+0 /     0
  []rune(s)          s.chars().collect()            1+2 / 12016      1+1 / 12008
                       ...with_capacity(count)      1+0 / 12000      1+0 / 12000
  string(r)          r.iter().collect()             1+0 /  3000      1+1 /  6000
  RuneCountInString  s.chars().count()              0+0 /     0      0+0 /     0
  s[6:2502]          &s[6..2502]                    0+0 /     0      0+0 /     0
  a+r = allocations + reallocations; bytes = exactly what the result holds
  Vec<char> payload: 4 x 3000 chars = 12000 bytes for both; collect() guesses
  its size from the byte length (Chars only knows len/4..len), so it grows

validation is the price, and it can fail -- the bytes come back unharmed:
  String::from_utf8([61 ff 62]) -> Err(invalid utf-8 sequence of 1 bytes from index 1), bytes back: [61, ff, 62]

Go's compiler-elided cases (b holds ascii's 3000 bytes):
  m[string(b)]       m.get(str::from_utf8(&b)?)     0+0 /     0
  string(b) == "foo" b == b"foo"                    0+0 /     0
  string(b[:1])      &ascii[..1]                    0+0 /     0
  "<"+string(b)+">"  ["<", s, ">"].concat()         1+0 /  3002
  in Rust these are not compiler tricks: a lookup by &str, a comparison of
  slices and a sub-slice are ordinary borrows; only the concatenation builds
  a new value, and concat() sums the lengths first so it allocates once
```

> **Go vs Rust:** Go pays one allocation per edge; Rust pays none on five of them, because `as_bytes`, `str::from_utf8` and slicing are borrows and `into_bytes`/`String::from_utf8(v)` are moves of the same buffer. What Rust charges instead is Θ(n) UTF-8 validation on bytes → `str`, which returns `Err` (with the bytes) where Go would accept anything. `collect::<Vec<char>>()` sizes itself from `Chars`'s byte-based hint and reallocated (1+2 and 1+1, ending at 12,016/12,008 bytes); counting first gives exactly 1 allocation and 12,000 bytes, Go's two-pass shape. Go's byte counts are size-class rounded (3,072, 12,288), Rust's are exact.

**Complexity:** each conversion Θ(n) time and 1 allocation (3,000 bytes → a 3,072-byte size class) · `[]rune` costs 4 bytes per rune: 12,000 bytes for 3,000 runes, 4× the ASCII string and 2× the mixed one · `m[string(b)]`, `string(b) == "lit"` and a one-byte `string(b[:1])` are free because the compiler skips the copy; `"<" + string(b) + ">"` pays only for its result

---

## 4. UTF-8 indexing traps

`🟢 easy` · *UTF-8 indexing*

`s[i]` is a byte, `s[:k]` cuts at a byte, and `range` hands you byte offsets — none of them is "the i-th
character", and Go never stops you from mixing them up: the result is silently invalid UTF-8. This example walks
the five traps on `"héllo wörld"` and fixes each one with the property that makes UTF-8 workable: a continuation
byte always looks like `10xxxxxx`, so the start of the current rune is at most 3 steps back (a Θ(1) safe cut),
while the k-th rune can only be found by decoding from the start (Θ(k)).

**Steps:**

1. Index and slice `"héllo wörld"` naively: a byte read as a code point, and a cut through `é`.
2. Fix each trap: decode instead of index, back off with `utf8.RuneStart`, count runes without `[]rune`.
3. Feed `range` an invalid string and watch `U+FFFD` appear while the raw byte stays in the string.

```go
package main

import (
	"fmt"
	"strings"
	"testing"
	"unicode/utf8"
)

// s[i] is a BYTE, s[:k] cuts at a BYTE, and range gives BYTE offsets. None of
// them is "the i-th character", and Go never stops you from mixing them up:
// the result is silently invalid UTF-8. Five traps, each with its fix.

var sinkR rune // results escape, so allocation counts are honest

// runeAt returns the k-th rune by decoding from the start -- Theta(k), no
// allocation -- and how many bytes it had to examine to get there.
func runeAt(s string, k int) (r rune, examined int, ok bool) {
	n := 0
	for i, r := range s {
		if n == k {
			return r, i + utf8.RuneLen(r), true
		}
		n++
	}
	return 0, len(s), false
}

// safeCut returns the longest prefix of s that is at most max bytes and does
// not split a rune: back off while s[cut] is a continuation byte (10xxxxxx).
// A rune is at most 4 bytes, so this takes at most 3 steps.
func safeCut(s string, max int) (prefix string, steps int) {
	if max >= len(s) {
		return s, 0
	}
	cut := max
	for cut > 0 && !utf8.RuneStart(s[cut]) {
		cut--
		steps++
	}
	return s[:cut], steps
}

func main() {
	s := "héllo wörld"
	fmt.Printf("s = %q: len %d bytes, %d runes\n", s, len(s), utf8.RuneCountInString(s))

	fmt.Println("\nTrap 1 -- s[i] is a byte, and string(byte) treats it as a code point:")
	fmt.Printf("  s[1]         = %d (0x%x), the first byte of 'é'\n", s[1], s[1])
	fmt.Printf("  string(s[1]) = %q  <- byte 195 read as code point U+00C3\n", string(s[1]))
	r, w := utf8.DecodeRuneInString(s[1:])
	fmt.Printf("  fix: utf8.DecodeRuneInString(s[1:]) = %q, width %d\n", r, w)

	fmt.Println("\nTrap 2 -- s[:k] cuts at a byte and can split a letter:")
	fmt.Printf("  s[:2] = %q   utf8.ValidString: %v\n", s[:2], utf8.ValidString(s[:2]))
	for _, max := range []int{2, 3, 9} {
		p, steps := safeCut(s, max)
		fmt.Printf("  fix: safeCut(s, %d) = %-9q valid: %v, backed off %d byte(s)\n", max, p, utf8.ValidString(p), steps)
	}

	fmt.Println("\nTrap 3 -- range decodes runes, but i is a BYTE offset:")
	fmt.Print("  i:")
	for i := range s {
		fmt.Printf(" %d", i)
	}
	var skipped []int // byte positions range never lands on: continuation bytes
	for i := 0; i < len(s); i++ {
		if !utf8.RuneStart(s[i]) {
			skipped = append(skipped, i)
		}
	}
	fmt.Printf("   <- skipped %v: the 2nd bytes of é and ö\n", skipped)

	fmt.Println("\nTrap 4 -- the k-th rune has no shortcut, because widths vary:")
	// 120 runes: more than the 32-rune stack buffer the runtime uses for a
	// short []rune that does not escape, so []rune(long) really allocates.
	long := strings.Repeat(s+" ", 10) // 140 bytes, 120 runes
	const k = 100
	r, examined, _ := runeAt(long, k)
	viaSlice := []rune(long)[k]
	fmt.Printf("  long: %d bytes, %d runes, k = %d\n", len(long), utf8.RuneCountInString(long), k)
	fmt.Printf("  runeAt(long, k)  = %q  examined %d bytes, Theta(k)\n", r, examined)
	fmt.Printf("  []rune(long)[k]  = %q  decoded all %d bytes first\n", viaSlice, len(long))
	a1 := testing.AllocsPerRun(1000, func() { sinkR, _, _ = runeAt(long, k) })
	a2 := testing.AllocsPerRun(1000, func() { sinkR = []rune(long)[k] })
	fmt.Printf("  allocations: runeAt %.0f, []rune(long)[k] %.0f (%d bytes of runes for one answer)\n",
		a1, a2, 4*utf8.RuneCountInString(long))

	fmt.Println("\nTrap 5 -- invalid UTF-8 is a legal Go string:")
	bad := "a\xffb"
	fmt.Printf("  bad = %q: len %d, utf8.ValidString %v\n", bad, len(bad), utf8.ValidString(bad))
	for i, r := range bad {
		_, w := utf8.DecodeRuneInString(bad[i:])
		fmt.Printf("  range: i=%d  U+%04X  width %d\n", i, r, w)
	}
	fmt.Printf("  bad[1] is still the raw byte 0x%x; U+FFFD only appears when decoding\n", bad[1])
	fmt.Printf("  fix: validate at the boundary -- strings.ToValidUTF8(bad, \"?\") = %q\n", strings.ToValidUTF8(bad, "?"))
}
```

**Output:**

```
s = "héllo wörld": len 13 bytes, 11 runes

Trap 1 -- s[i] is a byte, and string(byte) treats it as a code point:
  s[1]         = 195 (0xc3), the first byte of 'é'
  string(s[1]) = "Ã"  <- byte 195 read as code point U+00C3
  fix: utf8.DecodeRuneInString(s[1:]) = 'é', width 2

Trap 2 -- s[:k] cuts at a byte and can split a letter:
  s[:2] = "h\xc3"   utf8.ValidString: false
  fix: safeCut(s, 2) = "h"       valid: true, backed off 1 byte(s)
  fix: safeCut(s, 3) = "hé"      valid: true, backed off 0 byte(s)
  fix: safeCut(s, 9) = "héllo w" valid: true, backed off 1 byte(s)

Trap 3 -- range decodes runes, but i is a BYTE offset:
  i: 0 1 3 4 5 6 7 8 10 11 12   <- skipped [2 9]: the 2nd bytes of é and ö

Trap 4 -- the k-th rune has no shortcut, because widths vary:
  long: 140 bytes, 120 runes, k = 100
  runeAt(long, k)  = 'o'  examined 118 bytes, Theta(k)
  []rune(long)[k]  = 'o'  decoded all 140 bytes first
  allocations: runeAt 0, []rune(long)[k] 1 (480 bytes of runes for one answer)

Trap 5 -- invalid UTF-8 is a legal Go string:
  bad = "a\xffb": len 3, utf8.ValidString false
  range: i=0  U+0061  width 1
  range: i=1  U+FFFD  width 1
  range: i=2  U+0062  width 1
  bad[1] is still the raw byte 0xff; U+FFFD only appears when decoding
  fix: validate at the boundary -- strings.ToValidUTF8(bad, "?") = "a?b"
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// The same five traps in Rust -- but Rust refuses each one, at compile time or
// at run time, instead of producing invalid UTF-8:
//   1. s[1] does not compile; you must say whether you want a byte or a char.
//   2. &s[..2] through the middle of a letter PANICS; s.get(..2) returns None.
//   3. char_indices() gives byte offsets, exactly like Go's range.
//   4. chars().nth(k) is Theta(k), like Go -- widths vary in both.
//   5. a &str cannot hold invalid UTF-8 at all.

// ---- allocation counter: stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = black_box(f());
    (ALLOCS.load(Relaxed) - a0, r)
}

/// The k-th char by decoding from the start -- Theta(k), no allocation -- and
/// how many bytes were examined to get there.
fn rune_at(s: &str, k: usize) -> Option<(char, usize)> {
    s.char_indices().nth(k).map(|(i, c)| (c, i + c.len_utf8()))
}

/// The longest prefix of s that is at most max bytes and does not split a char:
/// back off while `max` is not a char boundary (at most 3 steps).
fn safe_cut(s: &str, max: usize) -> (&str, usize) {
    if max >= s.len() {
        return (s, 0);
    }
    let (mut cut, mut steps) = (max, 0);
    while !s.is_char_boundary(cut) {
        cut -= 1;
        steps += 1;
    }
    (&s[..cut], steps)
}

fn main() {
    let s = "héllo wörld";
    println!(
        "s = {:?}: len {} bytes, {} runes",
        s,
        s.len(),
        s.chars().count()
    );

    println!(
        "\nTrap 1 -- there is no s[i]; a byte must be asked for, and `as char` treats it as a code point:"
    );
    // s[1] does not compile:
    //   error[E0277]: the type `str` cannot be indexed by `{integer}`
    //     |     s[1]
    //     |       ^ string indices are ranges of `usize`
    let b = s.as_bytes()[1];
    println!("  s.as_bytes()[1]   = {b} (0x{b:x}), the first byte of 'é'");
    println!(
        "  as_bytes()[1] as char = {:?}  <- byte 195 read as code point U+00C3 (same trap)",
        b as char
    );
    let c = s[1..].chars().next().unwrap();
    println!(
        "  fix: s[1..].chars().next() = {:?}, width {}",
        c,
        c.len_utf8()
    );

    println!("\nTrap 2 -- &s[..k] through a letter does not give bad UTF-8: it PANICS:");
    std::panic::set_hook(Box::new(|_| {})); // silence the default "thread 'main' panicked" message
    let k = black_box(2);
    let caught = std::panic::catch_unwind(|| s[..k].len());
    let _ = std::panic::take_hook(); // restore the default hook
    match caught {
        Ok(n) => println!("  unexpected: {n}"),
        Err(e) => println!(
            "  &s[..2] panicked: {}",
            e.downcast_ref::<String>().map_or("?", |m| m.as_str())
        ),
    }
    println!(
        "  s.get(..2) = {:?}   s.is_char_boundary(2) = {}",
        s.get(..2),
        s.is_char_boundary(2)
    );
    for max in [2, 3, 9] {
        let (p, steps) = safe_cut(s, max);
        println!(
            "  fix: safe_cut(s, {max}) = {:<9} valid: {}, backed off {steps} byte(s)",
            format!("{p:?}"),
            std::str::from_utf8(p.as_bytes()).is_ok() // always: a &str cannot be invalid
        );
    }
    println!(
        "  std does the same: s.floor_char_boundary(2) = {}, (9) = {}",
        s.floor_char_boundary(2),
        s.floor_char_boundary(9)
    );

    println!("\nTrap 3 -- char_indices decodes chars, but i is a BYTE offset:");
    let offsets: Vec<String> = s.char_indices().map(|(i, _)| i.to_string()).collect();
    let skipped: Vec<String> = (0..s.len())
        .filter(|&i| !s.is_char_boundary(i))
        .map(|i| i.to_string())
        .collect();
    println!(
        "  i: {}   <- skipped [{}]: the 2nd bytes of é and ö",
        offsets.join(" "),
        skipped.join(" ")
    );

    println!("\nTrap 4 -- the k-th char has no shortcut, because widths vary:");
    let long = format!("{s} ").repeat(10); // 140 bytes, 120 chars
    let k = 100;
    let (c, examined) = rune_at(&long, k).unwrap();
    let via_vec = long.chars().collect::<Vec<char>>()[k];
    println!(
        "  long: {} bytes, {} runes, k = {k}",
        long.len(),
        long.chars().count()
    );
    println!(
        "  rune_at(long, k) = {c:?}  examined {examined} bytes, Theta(k)  (char_indices().nth(k))"
    );
    println!(
        "  collect::<Vec<char>>()[k] = {via_vec:?}  decoded all {} bytes first",
        long.len()
    );
    let (a1, _) = allocs_of(|| rune_at(&long, k));
    let (a2, _) = allocs_of(|| long.chars().collect::<Vec<char>>()[k]);
    println!(
        "  allocation events: rune_at {a1}, Vec<char> {a2} ({} bytes of chars for one answer)",
        4 * long.chars().count()
    );

    println!("\nTrap 5 -- invalid UTF-8 cannot become a &str:");
    let bad: Vec<u8> = vec![b'a', 0xff, b'b'];
    match std::str::from_utf8(&bad) {
        Ok(t) => println!("  unexpected: {t}"),
        Err(e) => println!("  str::from_utf8([61 ff 62]) = Err({e})"),
    }
    // utf8_chunks is the honest version of Go's range over bad bytes: it says
    // which bytes were valid text and which were not, instead of inventing U+FFFD.
    for chunk in bad.utf8_chunks() {
        println!(
            "  utf8_chunks: valid {:?}, invalid {:02x?}",
            chunk.valid(),
            chunk.invalid()
        );
    }
    let lossy = String::from_utf8_lossy(&bad);
    println!(
        "  fix: from_utf8_lossy = {:?}: len {} -- U+FFFD is now REAL, 3 bytes, not width 1",
        lossy,
        lossy.len()
    );
}
```

**Rust output:**

```
s = "héllo wörld": len 13 bytes, 11 runes

Trap 1 -- there is no s[i]; a byte must be asked for, and `as char` treats it as a code point:
  s.as_bytes()[1]   = 195 (0xc3), the first byte of 'é'
  as_bytes()[1] as char = 'Ã'  <- byte 195 read as code point U+00C3 (same trap)
  fix: s[1..].chars().next() = 'é', width 2

Trap 2 -- &s[..k] through a letter does not give bad UTF-8: it PANICS:
  &s[..2] panicked: end byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)
  s.get(..2) = None   s.is_char_boundary(2) = false
  fix: safe_cut(s, 2) = "h"       valid: true, backed off 1 byte(s)
  fix: safe_cut(s, 3) = "hé"      valid: true, backed off 0 byte(s)
  fix: safe_cut(s, 9) = "héllo w" valid: true, backed off 1 byte(s)
  std does the same: s.floor_char_boundary(2) = 1, (9) = 8

Trap 3 -- char_indices decodes chars, but i is a BYTE offset:
  i: 0 1 3 4 5 6 7 8 10 11 12   <- skipped [2 9]: the 2nd bytes of é and ö

Trap 4 -- the k-th char has no shortcut, because widths vary:
  long: 140 bytes, 120 runes, k = 100
  rune_at(long, k) = 'o'  examined 118 bytes, Theta(k)  (char_indices().nth(k))
  collect::<Vec<char>>()[k] = 'o'  decoded all 140 bytes first
  allocation events: rune_at 0, Vec<char> 3 (480 bytes of chars for one answer)

Trap 5 -- invalid UTF-8 cannot become a &str:
  str::from_utf8([61 ff 62]) = Err(invalid utf-8 sequence of 1 bytes from index 1)
  utf8_chunks: valid "a", invalid [ff]
  utf8_chunks: valid "b", invalid []
  fix: from_utf8_lossy = "a�b": len 5 -- U+FFFD is now REAL, 3 bytes, not width 1
```

> **Go vs Rust:** The numbers that belong to UTF-8 are identical: byte 195 → `'Ã'`, offsets `0 1 3 4 5 6 7 8 10 11 12`, the same safe cuts, 118 bytes examined for k = 100. Where Go silently produced `"h\xc3"`, Rust panicked (caught here with `catch_unwind`, so the default run never crashes) and `get(..2)` returned `None`; `floor_char_boundary` (stable since 1.91) is the std version of the back-off loop. Go's `range` reports `U+FFFD` with width 1 over a raw `0xff`; a `&str` cannot hold `0xff` at all, so `from_utf8` errs and `from_utf8_lossy` writes a real 3-byte `U+FFFD` (len 5). `collect::<Vec<char>>()` took 3 allocation events (1 + 2 reallocs) where Go's `[]rune` took 1.

**Complexity:** byte index Θ(1) · k-th rune Θ(k) — 118 bytes examined for k = 100 with 0 allocations, against 1 allocation and all 140 bytes decoded for `[]rune(long)[k]` · a rune-safe cut Θ(1), at most 3 steps back · Go never validates: `"a\xffb"` is a legal string, and `range` reports `U+FFFD` width 1 while the byte stays `0xff`

---

## 5. `+=` vs `strings.Builder`, counted

`🟢 easy` · *Concatenation*

`s += x` in Go cannot extend `s` in place — strings are immutable and others may share its bytes — so every
append allocates a new string and copies all of `s` into it: n appends cost about n allocations and Θ(n²) bytes
copied. `strings.Builder` appends into a growable `[]byte` with `append`'s amortized growth (Θ(n) bytes, O(log n)
allocations) and its `String()` hands that buffer over **without a final copy**. Lesson 03 example 13 measured
the per-element growth; this example counts allocations and copied bytes for six ways to build the same string,
and shows the three ways to get down to exactly one allocation.

**Steps:**

1. Build a string of n 10-byte pieces (n = 1,000 / 2,000 / 4,000) six ways, counting allocations per build.
2. Count the bytes each way copies and compare how both numbers change when n doubles.
3. Watch the Builder's `Cap()` grow, confirm `String()` allocates nothing, and trip the copy check.

```go
package main

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

// s += x cannot extend s in place -- s is immutable and other strings may share
// its bytes -- so it allocates len(s)+len(x) bytes and copies BOTH, every time.
// n appends of a fixed piece: n allocations, Theta(n^2) bytes copied.
// strings.Builder appends into a growable []byte (append's amortized growth)
// and String() hands that buffer over as a string WITHOUT copying it.

const piece = "0123456789" // 10 bytes per append

var sinkS string // results escape, so nothing can hide in a stack buffer

// Each builder returns the string and the bytes it copied into new storage
// (the new pieces plus every re-copy of old content), counted as it goes.
type build func(n int) (string, int)

func plusEq(n int) (string, int) {
	s, copied := "", 0
	for range n {
		if s != "" { // "" + piece returns piece itself: nothing copied
			copied += len(s) + len(piece)
		}
		s += piece
	}
	return s, copied
}

func sprintf(n int) (string, int) {
	s, copied := "", 0
	for range n {
		copied += 2 * (len(s) + len(piece)) // at least: into fmt's buffer, then out
		s = fmt.Sprintf("%s%s", s, piece)
	}
	return s, copied
}

func builder(n int, grow bool) (string, int) {
	var b strings.Builder
	if grow {
		b.Grow(n * len(piece)) // one allocation of exactly the final size
	}
	copied := 0
	for range n {
		if b.Len()+len(piece) > b.Cap() { // append must move the old bytes
			copied += b.Len()
		}
		b.WriteString(piece)
		copied += len(piece)
	}
	return b.String(), copied // String() copies nothing
}

var parts []string // built once per n, outside the measurement

func join(n int) (string, int) {
	return strings.Join(parts, ""), n * len(piece) // sums the lengths, Grows once
}

func buffer(n int) (string, int) {
	var b bytes.Buffer
	copied := 0
	for range n {
		if b.Len()+len(piece) > b.Cap() {
			copied += b.Len()
		}
		b.WriteString(piece)
		copied += len(piece)
	}
	return b.String(), copied + b.Len() // Buffer.String() DOES copy
}

func main() {
	methods := []struct {
		name string
		f    build
	}{
		{"s += piece", plusEq},
		{"Sprintf(\"%s%s\")", sprintf},
		{"Builder", func(n int) (string, int) { return builder(n, false) }},
		{"Builder + Grow", func(n int) (string, int) { return builder(n, true) }},
		{"strings.Join", join},
		{"bytes.Buffer", buffer},
	}
	ns := []int{1000, 2000, 4000}
	allocs := make([][]float64, len(methods))
	copied := make([][]int, len(methods))
	for i, m := range methods {
		for _, n := range ns {
			parts = make([]string, n)
			for j := range parts {
				parts[j] = piece
			}
			a := testing.AllocsPerRun(3, func() { sinkS, _ = m.f(n) })
			s, c := m.f(n)
			if len(s) != n*len(piece) {
				panic("wrong length from " + m.name)
			}
			allocs[i] = append(allocs[i], a)
			copied[i] = append(copied[i], c)
		}
	}

	fmt.Println("allocations per whole build (testing.AllocsPerRun):")
	fmt.Printf("  %-18s %9s %9s %9s %8s\n", "method", "n=1000", "n=2000", "n=4000", "x 4k/2k")
	for i, m := range methods {
		a := allocs[i]
		fmt.Printf("  %-18s %9.0f %9.0f %9.0f %7.1fx\n", m.name, a[0], a[1], a[2], a[2]/a[1])
	}
	fmt.Println()
	fmt.Println("bytes copied per whole build (counted; Sprintf is a lower bound):")
	fmt.Printf("  %-18s %9s %9s %9s %8s\n", "method", "n=1000", "n=2000", "n=4000", "x 4k/2k")
	for i, m := range methods {
		c := copied[i]
		fmt.Printf("  %-18s %9d %9d %9d %7.1fx\n", m.name, c[0], c[1], c[2], float64(c[2])/float64(c[1]))
	}
	fmt.Printf("  at n=4000, += copied %.0fx the bytes the Builder did, in %.0fx the allocations\n",
		float64(copied[0][2])/float64(copied[2][2]), allocs[0][2]/allocs[2][2])
	fmt.Println("  (Sprintf's allocation counts vary slightly between runs; likely cause: fmt")
	fmt.Println("  reuses buffers through a sync.Pool, which a garbage collection may empty)")

	fmt.Println()
	fmt.Println("Builder.Cap() while appending 10-byte pieces (it follows append's growth):")
	var b strings.Builder
	caps := []string{}
	for range 1000 {
		before := b.Cap()
		b.WriteString(piece)
		if b.Cap() != before {
			caps = append(caps, fmt.Sprint(b.Cap()))
		}
	}
	fmt.Printf("  %s\n", strings.Join(caps, " "))
	fmt.Printf("  %d growths for %d bytes; String() allocations: %.0f\n",
		len(caps), b.Len(), testing.AllocsPerRun(100, func() { sinkS = b.String() }))

	fmt.Println()
	fmt.Println("why a non-empty Builder must not be copied:")
	func() {
		defer func() { fmt.Printf("  recovered: %v\n", recover()) }()
		var orig strings.Builder
		orig.WriteString("shared")
		dup := orig // copies the header: dup.buf points at orig's bytes
		dup.WriteString("!")
	}()
	fmt.Println("  String() returns the buffer itself, so if dup could append into the shared")
	fmt.Println("  array it could overwrite bytes a string from orig is already using.")
}
```

**Sample output:**

```
allocations per whole build (testing.AllocsPerRun):
  method                n=1000    n=2000    n=4000  x 4k/2k
  s += piece               999      1999      3999     2.0x
  Sprintf("%s%s")         2003      4013      8059     2.0x
  Builder                   15        17        19     1.1x
  Builder + Grow             1         1         1     1.0x
  strings.Join               1         1         1     1.0x
  bytes.Buffer              10        11        12     1.1x

bytes copied per whole build (counted; Sprintf is a lower bound):
  method                n=1000    n=2000    n=4000  x 4k/2k
  s += piece           5004990  20009990  80019990     4.0x
  Sprintf("%s%s")     10010000  40020000 160040000     4.0x
  Builder                44220     82880    153310     1.8x
  Builder + Grow         10000     20000     40000     2.0x
  strings.Join           10000     20000     40000     2.0x
  bytes.Buffer           36280     72660    145420     2.0x
  at n=4000, += copied 522x the bytes the Builder did, in 210x the allocations
  (Sprintf's allocation counts vary slightly between runs; likely cause: fmt
  reuses buffers through a sync.Pool, which a garbage collection may empty)

Builder.Cap() while appending 10-byte pieces (it follows append's growth):
  16 32 64 128 256 512 896 1408 2048 3072 4096 5376 6912 9472 12288
  15 growths for 10000 bytes; String() allocations: 0

why a non-empty Builder must not be copied:
  recovered: strings: illegal use of non-zero Builder copied by value
  String() returns the buffer itself, so if dup could append into the shared
  array it could overwrite bytes a string from orig is already using.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// In Rust the operator is NOT the problem. A String owns its buffer, so:
//
//   s += x       is  s.push_str(x)                     (impl AddAssign<&str>)
//   s = s + x    is  fn add(mut self, x) { self.push_str(x); self }
//                    -- Add takes the left String BY VALUE and reuses its buffer
//
// Both are amortized Theta(1) per byte, like Go's strings.Builder. Because `+`
// moves its left operand, the old name is gone afterwards:
//
//   let b = a + "cd";
//   println!("{a} {b}");
//   // error[E0382]: borrow of moved value: `a`
//   //   `a` moved due to usage in operator
//
// The quadratic Rust spellings are the ones that build a NEW String each time:
// `s = format!("{s}{x}")` and the prepend `s = x.to_owned() + &s`.

const PIECE: &str = "0123456789"; // 10 bytes per append

// ---- allocation counter: alloc and realloc counted separately ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0);
static REQUESTED: AtomicUsize = AtomicUsize::new(0); // bytes asked for (alloc size, realloc new size)

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        REQUESTED.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        REALLOCS.fetch_add(1, Relaxed);
        REQUESTED.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocs, reallocs, bytes requested) for one call of `f`.
fn count(f: impl FnOnce() -> String) -> (usize, usize, usize) {
    let (a0, r0, q0) = (
        ALLOCS.load(Relaxed),
        REALLOCS.load(Relaxed),
        REQUESTED.load(Relaxed),
    );
    let s = black_box(f());
    let c = (
        ALLOCS.load(Relaxed) - a0,
        REALLOCS.load(Relaxed) - r0,
        REQUESTED.load(Relaxed) - q0,
    );
    assert_eq!(s.len() % PIECE.len(), 0);
    c
}

fn plus_eq(n: usize) -> String {
    let mut s = String::new();
    for _ in 0..n {
        s += PIECE;
    }
    s
}

// clippy prefers `s += PIECE`; spelling it `s = s + PIECE` IS the point here.
#[allow(clippy::assign_op_pattern)]
fn plus(n: usize) -> String {
    let mut s = String::new();
    for _ in 0..n {
        s = s + PIECE; // moves s in, gets the same buffer back
    }
    s
}

fn format_loop(n: usize) -> String {
    let mut s = String::new();
    for _ in 0..n {
        s = format!("{s}{PIECE}"); // a fresh String every time, then s is dropped
    }
    s
}

fn prepend(n: usize) -> String {
    let mut s = String::new();
    for _ in 0..n {
        s = PIECE.to_owned() + &s; // new String of 10 bytes, then grows to fit s
    }
    s
}

fn with_capacity(n: usize) -> String {
    let mut s = String::with_capacity(n * PIECE.len()); // Go's Builder.Grow
    for _ in 0..n {
        s.push_str(PIECE);
    }
    s
}

fn main() {
    type Build = fn(usize) -> String;
    let methods: [(&str, Build); 7] = [
        ("s += piece", plus_eq),
        ("s = s + piece", plus),
        ("s = format!(\"{s}{p}\")", format_loop),
        ("s = p.to_owned() + &s", prepend),
        ("with_capacity", with_capacity),
        ("parts.concat()", |n| vec![PIECE; n].concat()),
        ("parts.join(\"\")", |n| vec![PIECE; n].join("")),
    ];
    let ns = [1000, 2000, 4000];
    let mut rows = Vec::new();
    for (name, f) in methods {
        // vec![PIECE; n] (the parts) is built inside the closure for concat/join:
        // subtract its one allocation of n x 16 bytes so only the join is counted.
        let parts_cost = |n: usize| {
            if name.starts_with("parts") {
                (1, n * size_of::<&str>())
            } else {
                (0, 0)
            }
        };
        let cells: Vec<(usize, usize, usize)> = ns
            .iter()
            .map(|&n| {
                let (a, r, q) = count(|| f(n));
                let (pa, pq) = parts_cost(n);
                (a - pa, r, q - pq)
            })
            .collect();
        rows.push((name, cells));
    }

    println!("allocations + reallocations per whole build (counting allocator):");
    println!(
        "  {:<23} {:>11} {:>11} {:>11} {:>8}",
        "method", "n=1000", "n=2000", "n=4000", "x 4k/2k"
    );
    for (name, c) in &rows {
        let cell = |i: usize| format!("{}+{}", c[i].0, c[i].1);
        let total = |i: usize| (c[i].0 + c[i].1) as f64;
        println!(
            "  {:<23} {:>11} {:>11} {:>11} {:>7.1}x",
            name,
            cell(0),
            cell(1),
            cell(2),
            total(2) / total(1)
        );
    }
    println!();
    println!("bytes requested from the allocator per build (a realloc may grow in place,");
    println!("so this is an upper bound on bytes moved, not a count of bytes copied):");
    println!(
        "  {:<23} {:>11} {:>11} {:>11} {:>8}",
        "method", "n=1000", "n=2000", "n=4000", "x 4k/2k"
    );
    for (name, c) in &rows {
        println!(
            "  {:<23} {:>11} {:>11} {:>11} {:>7.1}x",
            name,
            c[0].2,
            c[1].2,
            c[2].2,
            c[2].2 as f64 / c[1].2 as f64
        );
    }
    let (pe, fm) = (&rows[0].1[2], &rows[2].1[2]);
    println!(
        "  at n=4000, format! requested {:.0}x the bytes of +=, in {:.0}x the allocation events",
        fm.2 as f64 / pe.2 as f64,
        (fm.0 + fm.1) as f64 / (pe.0 + pe.1) as f64
    );

    println!();
    println!("String capacity while appending 10-byte pieces (Vec's growth policy):");
    let mut s = String::new();
    let mut caps = Vec::new();
    for _ in 0..1000 {
        let before = s.capacity();
        s += PIECE;
        if s.capacity() != before {
            caps.push(s.capacity().to_string());
        }
    }
    println!("  {}", caps.join(" "));
    println!(
        "  {} growths for {} bytes: max(2 x capacity, what is needed)",
        caps.len(),
        s.len()
    );

    println!();
    println!("+ reuses the left operand's buffer, and copying is explicit:");
    let mut a = String::with_capacity(16);
    a.push_str("ab");
    let p = a.as_ptr();
    let b = a + "cd"; // a is moved into +, and comes back as b
    println!(
        "  (a + \"cd\").as_ptr() == a.as_ptr(): {}   (b = {b:?})",
        b.as_ptr() == p
    );
    let dup = b.clone(); // Go's accidental Builder copy cannot happen: a copy is a clone
    println!(
        "  b.clone() shares b's buffer: {}  -- so there is nothing for a copy check to catch",
        dup.as_ptr() == b.as_ptr()
    );
}
```

**Rust output:**

```
allocations + reallocations per whole build (counting allocator):
  method                       n=1000      n=2000      n=4000  x 4k/2k
  s += piece                     1+10        1+11        1+12     1.1x
  s = s + piece                  1+10        1+11        1+12     1.1x
  s = format!("{s}{p}")      1000+999   2000+1999   4000+3999     2.0x
  s = p.to_owned() + &s      1000+999   2000+1999   4000+3999     2.0x
  with_capacity                   1+0         1+0         1+0     1.0x
  parts.concat()                  1+0         1+0         1+0     1.0x
  parts.join("")                  1+0         1+0         1+0     1.0x

bytes requested from the allocator per build (a realloc may grow in place,
so this is an upper bound on bytes moved, not a count of bytes copied):
  method                       n=1000      n=2000      n=4000  x 4k/2k
  s += piece                    20470       40950       81910     2.0x
  s = s + piece                 20470       40950       81910     2.0x
  s = format!("{s}{p}")      14985010    59970010   239940010     4.0x
  s = p.to_owned() + &s       5014990    20029990    80059990     4.0x
  with_capacity                 10000       20000       40000     2.0x
  parts.concat()                10000       20000       40000     2.0x
  parts.join("")                10000       20000       40000     2.0x
  at n=4000, format! requested 2929x the bytes of +=, in 615x the allocation events

String capacity while appending 10-byte pieces (Vec's growth policy):
  10 20 40 80 160 320 640 1280 2560 5120 10240
  11 growths for 10000 bytes: max(2 x capacity, what is needed)

+ reuses the left operand's buffer, and copying is explicit:
  (a + "cd").as_ptr() == a.as_ptr(): true   (b = "abcd")
  b.clone() shares b's buffer: false  -- so there is nothing for a copy check to catch
```

> **Go vs Rust:** The same operator gives opposite results: Go's `s += piece` made 3,999 allocations at n = 4,000, Rust's made 1 allocation + 12 reallocations, the same as `s = s + piece` — `Add` takes the left `String` by value and `push_str`es into its buffer (the pointer did not change). Rust's quadratic versions are `format!` and the prepend: n allocations + n − 1 reallocations and ×4.0 bytes requested per doubling, like Go's `+=`. `with_capacity`/`concat`/`join` are 1 allocation, as `Grow`/`Join` are in Go. Capacity doubles exactly (10 … 10,240), where Go's Builder follows `append` (… 2048 3072 4096 5376 …); Rust's column is bytes *requested*, not copied, because a `realloc` may grow in place. A Rust copy is an explicit `clone()` into a new buffer, so there is no `copyCheck` to trip.

**Complexity:** `+=` in a loop: n − 1 allocations (the first `"" + piece` is free) and Θ(n²) bytes copied — 80 MB for a 40 KB result at n = 4,000, ×4.0 per doubling · Builder: Θ(n) bytes (×1.8–2.0 per doubling), 15–19 allocations, `String()` 0 · `Grow` / `Join`: exactly 1 allocation · only `Sprintf`'s counts wobble between runs (by about one), which is why this is a sample output

---

## 6. Comparison is bytes: equality, order and case

`🟢 easy` · *Comparison*

`==`, `<`, `strings.Compare`, sorting and map keys all compare **bytes**. That makes them fast and simple — `==`
is `len(a) == len(b) && memequal(a, b)`, so strings of different lengths are unequal in Θ(1), and `<` stops at the
first differing byte, Θ(min(len)) — and it makes them disagree with a human reader. `"\u00e9"` and `"e\u0301"` look
the same and are unequal, `"Éclair"` sorts after `"zebra"`, and case-insensitive comparison is a Unicode problem,
not a byte one: `strings.EqualFold` does it in Θ(n) with no allocation, where two `ToLower` calls may allocate twice.

**Steps:**

1. Compare the two spellings of `é` and sort six words, printing the bytes that decide the result.
2. Count the bytes `==` and `<` examine on three 1,000-byte pairs: differ at byte 0, differ in length, equal.
3. Compare `EqualFold` with `ToLower(a) == ToLower(b)` on allocations, then try the Kelvin sign and `ß`.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
	"testing"
)

// ==, <, strings.Compare, slices.Sort and map keys all compare BYTES.
// That makes them fast and simple -- the compiler turns a == b into
//	len(a) == len(b) && memequal(a, b)
// -- and makes them disagree with a human reader about accents, order and case.

var sinkOK bool

// eqCount is a == b written out: lengths first, then bytes until a difference.
// It returns the answer and how many bytes it had to examine.
func eqCount(a, b string) (bool, int) {
	if len(a) != len(b) {
		return false, 0 // Theta(1): never looks at a byte
	}
	for i := 0; i < len(a); i++ {
		if a[i] != b[i] {
			return false, i + 1
		}
	}
	return true, len(a)
}

// cmpCount is strings.Compare written out: bytes until the first difference,
// then the shorter string is smaller. Theta(min(len(a), len(b))).
func cmpCount(a, b string) (int, int) {
	n := min(len(a), len(b))
	for i := 0; i < n; i++ {
		if a[i] != b[i] {
			if a[i] < b[i] {
				return -1, i + 1
			}
			return 1, i + 1
		}
	}
	switch {
	case len(a) < len(b):
		return -1, n
	case len(a) > len(b):
		return 1, n
	}
	return 0, n
}

func main() {
	fmt.Println("(a) one letter, two spellings:")
	nfc, nfd := "\u00e9", "e\u0301" // precomposed é vs e + combining acute
	fmt.Printf("  NFC U+00E9        bytes % x    len %d\n", nfc, len(nfc))
	fmt.Printf("  NFD e + U+0301    bytes % x len %d\n", nfd, len(nfd))
	fmt.Printf("  both print as %s and %s, and nfc == nfd is %v\n", nfc, nfd, nfc == nfd)

	fmt.Println("\n(b) sorting is byte order, not dictionary order:")
	words := []string{"banana", "Apple", "cherry", "apple", "Éclair", "zebra"}
	slices.Sort(words)
	for _, w := range words {
		fmt.Printf("  %-8s first byte 0x%x\n", w, w[0])
	}
	fmt.Println("  uppercase A-Z is 0x41-0x5a, lowercase a-z is 0x61-0x7a, and every")
	fmt.Println("  non-ASCII letter starts with a byte >= 0xc2 -- so É sorts after z")

	fmt.Println("\n(c) what a comparison costs -- bytes examined:")
	base := strings.Repeat("x", 999)
	pairs := []struct{ name, a, b string }{
		{"differ at byte 0", "a" + base, "b" + base},
		{"lengths 1000 vs 1001", "a" + base, "a" + base + "!"},
		{"equal, 1000 bytes", "a" + base, "a" + base},
	}
	fmt.Printf("  %-22s %12s %12s\n", "pair", "== examined", "< examined")
	for _, p := range pairs {
		eq, eqN := eqCount(p.a, p.b)
		c, cN := cmpCount(p.a, p.b)
		if eq != (p.a == p.b) || c != strings.Compare(p.a, p.b) {
			panic("counting versions disagree with the built-ins")
		}
		fmt.Printf("  %-22s %12d %12d\n", p.name, eqN, cN)
	}
	fmt.Println("  == checks the length first: unequal lengths cost nothing")

	fmt.Println("\n(d) case-insensitive comparison:")
	cases := []struct{ a, b string }{{"hello", "hello"}, {"Hello", "hello"}, {"Hello", "HELLO"}}
	fmt.Printf("  %-16s %6s %18s %30s\n", "pair", "equal", "EqualFold allocs", "ToLower(a)==ToLower(b) allocs")
	for _, c := range cases {
		a, b := c.a, c.b
		fold := testing.AllocsPerRun(100, func() { sinkOK = strings.EqualFold(a, b) })
		lower := testing.AllocsPerRun(100, func() { sinkOK = strings.ToLower(a) == strings.ToLower(b) })
		fmt.Printf("  %-16s %6v %18.0f %30.0f\n", a+" / "+b, strings.EqualFold(a, b), fold, lower)
	}
	fmt.Println("  ToLower returns s itself when it is ASCII with no uppercase -- 0 allocations")
	fmt.Println("\n  case is a Unicode problem, not a byte one:")
	fmt.Printf("  EqualFold(KELVIN SIGN U+212A, \"k\") = %v\n", strings.EqualFold("\u212a", "k"))
	fmt.Printf("  ToUpper(\"ß\") = %q (Go maps rune to rune; German writes SS)\n", strings.ToUpper("ß"))
	fmt.Printf("  EqualFold(\"ß\", \"SS\") = %v\n", strings.EqualFold("ß", "SS"))
}
```

**Output:**

```
(a) one letter, two spellings:
  NFC U+00E9        bytes c3 a9    len 2
  NFD e + U+0301    bytes 65 cc 81 len 3
  both print as é and é, and nfc == nfd is false

(b) sorting is byte order, not dictionary order:
  Apple    first byte 0x41
  apple    first byte 0x61
  banana   first byte 0x62
  cherry   first byte 0x63
  zebra    first byte 0x7a
  Éclair   first byte 0xc3
  uppercase A-Z is 0x41-0x5a, lowercase a-z is 0x61-0x7a, and every
  non-ASCII letter starts with a byte >= 0xc2 -- so É sorts after z

(c) what a comparison costs -- bytes examined:
  pair                    == examined   < examined
  differ at byte 0                  1            1
  lengths 1000 vs 1001              0         1000
  equal, 1000 bytes              1000         1000
  == checks the length first: unequal lengths cost nothing

(d) case-insensitive comparison:
  pair              equal   EqualFold allocs  ToLower(a)==ToLower(b) allocs
  hello / hello      true                  0                              0
  Hello / hello      true                  0                              1
  Hello / HELLO      true                  0                              2
  ToLower returns s itself when it is ASCII with no uppercase -- 0 allocations

  case is a Unicode problem, not a byte one:
  EqualFold(KELVIN SIGN U+212A, "k") = true
  ToUpper("ß") = "ß" (Go maps rune to rune; German writes SS)
  EqualFold("ß", "SS") = false
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cmp::Ordering;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Rust compares strings by BYTES too: `PartialEq` for str is
// `self.as_bytes() == other.as_bytes()` (slice equality checks the length
// first), and `Ord` is byte-wise lexicographic. So sorting, BTreeMap keys and
// `<` agree with Go exactly. Case is where the two standard libraries differ.

// ---- allocation counter: stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

fn allocs_of<R>(f: impl FnOnce() -> R) -> usize {
    let a0 = ALLOCS.load(Relaxed);
    black_box(f());
    ALLOCS.load(Relaxed) - a0
}

/// a == b written out: lengths first, then bytes until a difference.
/// Returns the answer and how many bytes were examined.
fn eq_count(a: &str, b: &str) -> (bool, usize) {
    if a.len() != b.len() {
        return (false, 0); // Theta(1): never looks at a byte
    }
    match a.bytes().zip(b.bytes()).position(|(x, y)| x != y) {
        Some(i) => (false, i + 1),
        None => (true, a.len()),
    }
}

/// a.cmp(b) written out: bytes until the first difference, then the shorter
/// string is smaller. Theta(min(a.len(), b.len())).
fn cmp_count(a: &str, b: &str) -> (Ordering, usize) {
    let n = a.len().min(b.len());
    match a.bytes().zip(b.bytes()).position(|(x, y)| x != y) {
        Some(i) => (a.as_bytes()[i].cmp(&b.as_bytes()[i]), i + 1),
        None => (a.len().cmp(&b.len()), n),
    }
}

/// Code points of a string, U+XXXX each -- Rust's {:?} and Go's %q escape
/// differently, so text whose bytes matter is printed by hand.
fn code_points(s: &str) -> String {
    s.chars()
        .map(|c| format!("U+{:04X}", c as u32))
        .collect::<Vec<_>>()
        .join(" ")
}

fn hex(s: &str) -> String {
    s.bytes()
        .map(|b| format!("{b:02x}"))
        .collect::<Vec<_>>()
        .join(" ")
}

/// A case-insensitive comparison that allocates nothing: lowercase both char
/// streams lazily and compare them. (Lowercasing is close to, but not exactly,
/// Unicode case folding, which is what Go's EqualFold implements.)
fn eq_lower_iter(a: &str, b: &str) -> bool {
    a.chars()
        .flat_map(char::to_lowercase)
        .eq(b.chars().flat_map(char::to_lowercase))
}

fn main() {
    println!("(a) one letter, two spellings:");
    let (nfc, nfd) = ("\u{e9}", "e\u{301}"); // precomposed é vs e + combining acute
    println!(
        "  NFC U+00E9        bytes {}    len {}",
        hex(nfc),
        nfc.len()
    );
    println!("  NFD e + U+0301    bytes {} len {}", hex(nfd), nfd.len());
    println!(
        "  both print as {nfc} and {nfd}, and nfc == nfd is {}",
        nfc == nfd
    );

    println!("\n(b) sorting is byte order, not dictionary order:");
    let mut words = ["banana", "Apple", "cherry", "apple", "Éclair", "zebra"];
    words.sort(); // Ord for &str: byte-wise
    for w in words {
        println!("  {w:<8} first byte 0x{:x}", w.as_bytes()[0]);
    }
    println!("  uppercase A-Z is 0x41-0x5a, lowercase a-z is 0x61-0x7a, and every");
    println!("  non-ASCII letter starts with a byte >= 0xc2 -- so É sorts after z");

    println!("\n(c) what a comparison costs -- bytes examined:");
    let base = "x".repeat(999);
    let pairs = [
        ("differ at byte 0", format!("a{base}"), format!("b{base}")),
        (
            "lengths 1000 vs 1001",
            format!("a{base}"),
            format!("a{base}!"),
        ),
        ("equal, 1000 bytes", format!("a{base}"), format!("a{base}")),
    ];
    println!(
        "  {:<22} {:>12} {:>12}",
        "pair", "== examined", "< examined"
    );
    for (name, a, b) in &pairs {
        let (eq, eq_n) = eq_count(a, b);
        let (c, c_n) = cmp_count(a, b);
        assert!(
            eq == (a == b) && c == a.cmp(b),
            "counting versions disagree with the built-ins"
        );
        println!("  {name:<22} {eq_n:>12} {c_n:>12}");
    }
    println!("  == checks the length first: unequal lengths cost nothing");

    println!("\n(d) case-insensitive comparison -- std has no EqualFold:");
    println!(
        "  {:<16} {:>18} {:>22} {:>22}",
        "pair", "eq_ignore_ascii", "lowercase iterators", "to_lowercase() =="
    );
    for (a, b) in [("hello", "hello"), ("Hello", "hello"), ("Hello", "HELLO")] {
        let ascii = allocs_of(|| a.eq_ignore_ascii_case(b));
        let iter = allocs_of(|| eq_lower_iter(a, b));
        let owned = allocs_of(|| a.to_lowercase() == b.to_lowercase());
        let cell = |ok: bool, n: usize| format!("{ok}, {n} allocs");
        println!(
            "  {:<16} {:>18} {:>22} {:>22}",
            format!("{a} / {b}"),
            cell(a.eq_ignore_ascii_case(b), ascii),
            cell(eq_lower_iter(a, b), iter),
            cell(a.to_lowercase() == b.to_lowercase(), owned)
        );
    }
    println!("  to_lowercase() always builds a new String -- no \"return s\" fast path");

    println!("\n  case is a Unicode problem, not a byte one:");
    let kelvin = "\u{212a}";
    println!(
        "  KELVIN SIGN U+212A vs \"k\": to_lowercase() == {}, eq_ignore_ascii_case == {}",
        kelvin.to_lowercase() == "k",
        kelvin.eq_ignore_ascii_case("k")
    );
    println!(
        "  \"ß\".to_uppercase() = {:?} (full mapping: 1 char becomes 2; Go keeps \"ß\")",
        "ß".to_uppercase()
    );
    println!(
        "  so \"ß\" vs \"SS\": equal by to_uppercase() {}, by to_lowercase() {}, by lowercase iterators {}",
        "ß".to_uppercase() == "SS".to_uppercase(),
        "ß".to_lowercase() == "SS".to_lowercase(),
        eq_lower_iter("ß", "SS")
    );
    let dotted = "\u{130}".to_lowercase(); // İ, Turkish capital dotted I
    println!(
        "  \"İ\".to_lowercase() = {} -- {} chars, {} bytes, from 1 char of {} bytes",
        code_points(&dotted),
        dotted.chars().count(),
        dotted.len(),
        "\u{130}".len()
    );
}
```

**Rust output:**

```
(a) one letter, two spellings:
  NFC U+00E9        bytes c3 a9    len 2
  NFD e + U+0301    bytes 65 cc 81 len 3
  both print as é and é, and nfc == nfd is false

(b) sorting is byte order, not dictionary order:
  Apple    first byte 0x41
  apple    first byte 0x61
  banana   first byte 0x62
  cherry   first byte 0x63
  zebra    first byte 0x7a
  Éclair   first byte 0xc3
  uppercase A-Z is 0x41-0x5a, lowercase a-z is 0x61-0x7a, and every
  non-ASCII letter starts with a byte >= 0xc2 -- so É sorts after z

(c) what a comparison costs -- bytes examined:
  pair                    == examined   < examined
  differ at byte 0                  1            1
  lengths 1000 vs 1001              0         1000
  equal, 1000 bytes              1000         1000
  == checks the length first: unequal lengths cost nothing

(d) case-insensitive comparison -- std has no EqualFold:
  pair                eq_ignore_ascii    lowercase iterators      to_lowercase() ==
  hello / hello        true, 0 allocs         true, 0 allocs         true, 2 allocs
  Hello / hello        true, 0 allocs         true, 0 allocs         true, 2 allocs
  Hello / HELLO        true, 0 allocs         true, 0 allocs         true, 2 allocs
  to_lowercase() always builds a new String -- no "return s" fast path

  case is a Unicode problem, not a byte one:
  KELVIN SIGN U+212A vs "k": to_lowercase() == true, eq_ignore_ascii_case == false
  "ß".to_uppercase() = "SS" (full mapping: 1 char becomes 2; Go keeps "ß")
  so "ß" vs "SS": equal by to_uppercase() true, by to_lowercase() false, by lowercase iterators false
  "İ".to_lowercase() = U+0069 U+0307 -- 2 chars, 3 bytes, from 1 char of 2 bytes
```

> **Go vs Rust:** Parts (a)–(c) are identical: the same bytes, the same sorted order (`Éclair` last at `0xc3`) and the same bytes examined (1 / 0 / 1,000 for `==`), because `str`'s `PartialEq` and `Ord` are byte-wise like Go's. Case differs. Rust's std has no `EqualFold`: `eq_ignore_ascii_case` is ASCII-only (false for the Kelvin sign), `to_lowercase() ==` took 2 allocations on every pair where Go's `ToLower` took 0, 1 or 2 (it has no "return `s`" fast path), and comparing lazily lowercased `chars()` gave 0 allocations. Rust maps case fully: `"ß".to_uppercase()` is `"SS"` (Go keeps `"ß"`), so `ß`/`SS` is equal by uppercasing and unequal by lowercasing.

**Complexity:** `==` Θ(1) on different lengths (0 bytes examined), else Θ(n) worst (1,000 for an equal pair) · `<` Θ(min(len)) — 1,000 bytes for a pair that only differs in length · `EqualFold` Θ(n), 0 allocations · `ToLower` + `==` Θ(n) and 0, 1 or 2 allocations (0 only when both sides are already lowercase ASCII)

---

> Next tier: [🟡 medium](2-medium.md) — the string problem set: anagram, palindrome, reverse words,
> compress, and the longest substring without repeats.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
