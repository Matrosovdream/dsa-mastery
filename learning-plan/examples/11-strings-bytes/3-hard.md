# Step 11 — Strings, Runes & Bytes · 🔴 Hard

Examples **12–16**: the cost model measured, the substring that pins its parent, a zero-allocation
tokenizer, grapheme clusters, and the capstone.

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

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The cost model, measured

`🔴 hard` · *Cost model*

Every string operation has one of three shapes: Θ(1) on the 16-byte header, Θ(n) over the bytes, or Θ(n) plus a
new allocation. You can tell which from what the operation must touch, and this example checks that against the
machine for 16 operations at n = 1,000 / 10,000 / 100,000. Allocation counts come first because they are exact:
`testing.AllocsPerRun` and the runtime's byte counter never move between runs, and bytes that grow 10× with a 10×
input mean the operation copies the input. Time comes second, classified by the ratio test from lesson 03 (t(100k)/t(10k)
≈ 1 means Θ(1), ≈ 10 means Θ(n)). The timing pass runs three times, and a class is printed only if all three agreed.

**Steps:**

1. Build each input once per size as its own heap block (comparing a string with itself would stop at "same pointer"),
   with the difference placed where it costs the most: the last byte, or the needle at the end.
2. Count allocations and bytes per call at all three sizes. Then time every call three times and keep the best.
3. Read the shapes off the tables: header operations stay flat, scans grow 10×, and conversions grow 10× *and* allocate.
   `ToLower` on lowercase input is the odd one: 0 allocations, but still Θ(n).

```go
package main

import (
	"flag"
	"fmt"
	"runtime"
	"runtime/debug"
	"strings"
	"testing"
	"unicode/utf8"
)

// Every string operation has one of three shapes, decided by what it must touch:
//
//	Θ(1)        only the 16-byte header (pointer, length)
//	Θ(n)        reads the bytes, allocates nothing
//	Θ(n) + 1    reads the bytes AND allocates a new block
//
// Instead of memorising which is which, we measure. Two instruments:
//
//  1. ALLOCATIONS (deterministic): testing.AllocsPerRun counts mallocs per
//     call; runtime.MemStats.TotalAlloc gives the bytes. If bytes/op grows 10x
//     when n grows 10x, the operation copies the input. No noise at all.
//  2. TIME (a supplement, noisy): testing.Benchmark at n = 1,000 / 10,000 /
//     100,000. The ratio test (lesson 03 example 4): t(100k)/t(10k) ≈ 1 means
//     Θ(1), ≈ 10 means Θ(n). We run the whole timing pass 3 times, keep the
//     best time of the 3 for each cell, and print a class ONLY if all three
//     passes agreed on it.
//
// Strings are immutable, so no operation below changes its input: building the
// inputs once, outside the timed loop, is correct here. (A benchmark of an
// operation that MUTATES its input -- sorting, in-place reverse -- must rebuild
// the input inside the loop, or every iteration after the first measures a
// different, already-done problem.)

// Sinks: results escape to package level, so nothing is optimised away and no
// short result can hide in a 32-byte stack buffer.
var (
	sinkI   int
	sinkB   byte
	sinkOK  bool
	sinkS   string
	sinkR   []rune
	sinkSS  []string
	sinkMap int
)

// inputs for one size n. Every string is its own heap block: comparing a
// string with ITSELF would let memequal stop at "same pointer".
type inputs struct {
	s       string // n bytes of lowercase words: "alpha beta, gamma delta, ..."
	sameLen string // a copy of s with the LAST byte changed
	shorter string // s minus its last byte
	first   string // s with the FIRST byte changed
	upper   string // s with one uppercase letter, at the end
	hay     string // s with the needle "ZQX" as its last 3 bytes
	m       map[string]int
}

func build(n int) inputs {
	words := "alpha beta, gamma delta, "
	s := strings.Repeat(words, n/len(words)+1)[:n-1] + "z"
	change := func(s string, i int, c byte) string {
		b := []byte(s)
		b[i] = c
		return string(b)
	}
	return inputs{
		s:       s,
		sameLen: change(s, n-1, 'y'),
		shorter: strings.Clone(s[:n-1]),
		first:   change(s, 0, 'b'),
		upper:   change(s, n-1, 'Z'),
		hay:     s[:n-3] + "ZQX",
		m:       map[string]int{strings.Clone(s): 7},
	}
}

type op struct {
	name string
	f    func(in *inputs)
}

var ops = []op{
	{"len(s)", func(in *inputs) { sinkI = len(in.s) }},
	{"s[n/2]", func(in *inputs) { sinkB = in.s[len(in.s)/2] }},
	{"s[n/4 : 3n/4]", func(in *inputs) { n := len(in.s); sinkS = in.s[n/4 : 3*n/4] }},
	{"s == t (same len, last differs)", func(in *inputs) { sinkOK = in.s == in.sameLen }},
	{"s == t (different length)", func(in *inputs) { sinkOK = in.s == in.shorter }},
	{"s < t (byte 0 differs)", func(in *inputs) { sinkOK = in.s < in.first }},
	{"s < t (last byte differs)", func(in *inputs) { sinkOK = in.s < in.sameLen }},
	{"utf8.RuneCountInString(s)", func(in *inputs) { sinkI = utf8.RuneCountInString(in.s) }},
	{"strings.Index(s, \"ZQX\") at end", func(in *inputs) { sinkI = strings.Index(in.hay, "ZQX") }},
	{"m[s] (hash + compare n bytes)", func(in *inputs) { sinkMap = in.m[in.s] }},
	{"strings.ToLower (all lower)", func(in *inputs) { sinkS = strings.ToLower(in.s) }},
	{"strings.ToLower (1 upper)", func(in *inputs) { sinkS = strings.ToLower(in.upper) }},
	{"[]rune(s)", func(in *inputs) { sinkR = []rune(in.s) }},
	{"strings.Split(s, \",\")", func(in *inputs) { sinkSS = strings.Split(in.s, ",") }},
	{"strings.Fields(s)", func(in *inputs) { sinkSS = strings.Fields(in.s) }},
	{"s + \"x\"", func(in *inputs) { sinkS = in.s + "x" }},
}

var sizes = []int{1_000, 10_000, 100_000}

// bytesPerOp: TotalAlloc is the runtime's running sum of allocated bytes
// (rounded up to the allocator's size classes). GC paused while we count.
func bytesPerOp(f func()) uint64 {
	const runs = 200
	f()
	runtime.GC()
	defer debug.SetGCPercent(debug.SetGCPercent(-1))
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	for range runs {
		f()
	}
	runtime.ReadMemStats(&b)
	return (b.TotalAlloc - a.TotalAlloc) / runs
}

func nsPerOp(f func()) float64 {
	r := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			f()
		}
	})
	return float64(r.T.Nanoseconds()) / float64(r.N)
}

// class from the ratio t(100k)/t(10k): a 10x larger input. Θ(n) would be
// exactly 10 on an ideal machine; on a real one 100 KB no longer fits in the
// L1 cache the 10 KB input lived in, so reads get slower per byte and ratios
// of 15-25 are still "linear, plus a cache step" -- Θ(n²) would be ~100.
func class(ratio float64) string {
	switch {
	case ratio < 2.5:
		return "Θ(1)"
	case ratio >= 4 && ratio < 40:
		return "Θ(n)"
	}
	return "?"
}

func main() {
	testing.Init()
	flag.Set("test.benchtime", "25ms") // short runs: 16 ops x 3 sizes x 3 passes

	in := make([]inputs, len(sizes))
	for i, n := range sizes {
		in[i] = build(n)
	}

	// ---- 1. allocations: exact, and the same on every run ----
	fmt.Println("1. Allocations per call (testing.AllocsPerRun) and bytes per call (MemStats.TotalAlloc)")
	fmt.Printf("   %-32s %7s %9s %9s %9s %7s\n", "operation", "allocs", "B @1k", "B @10k", "B @100k", "B ratio")
	allocating := map[string]bool{}
	gLo, gHi := 1e18, 0.0
	for _, o := range ops {
		var allocs [3]float64
		var bytes [3]uint64
		for i := range sizes {
			p := &in[i]
			allocs[i] = testing.AllocsPerRun(200, func() { o.f(p) })
			bytes[i] = bytesPerOp(func() { o.f(p) })
		}
		if allocs[0] != allocs[1] || allocs[1] != allocs[2] {
			panic("allocation count depends on n for " + o.name)
		}
		grows := "-"
		if bytes[1] > 0 {
			g := float64(bytes[2]) / float64(bytes[1])
			grows = fmt.Sprintf("%.1f", g)
			allocating[o.name] = true
			gLo, gHi = min(gLo, g), max(gHi, g)
		}
		fmt.Printf("   %-32s %7.0f %9d %9d %9d %7s\n", o.name, allocs[0], bytes[0], bytes[1], bytes[2], grows)
	}

	fmt.Printf("   %d of %d operations allocate; each makes exactly 1 block, and its bytes grow %.1fx-%.1fx\n",
		len(allocating), len(ops), gLo, gHi)
	fmt.Println("   for a 10x longer input: the new block is (a multiple of) the input. The rest allocate nothing.")

	// ---- 2. time: 3 passes, best of 3 per cell, class only if all 3 agree ----
	const passes = 3
	var t [passes][][3]float64
	for p := range passes {
		t[p] = make([][3]float64, len(ops))
		for k, o := range ops {
			for i := range sizes {
				q := &in[i]
				t[p][k][i] = nsPerOp(func() { o.f(q) })
			}
		}
	}
	fmt.Println()
	fmt.Println("2. Time per call (ns, best of 3 passes) and the ratio test t(100k)/t(10k)")
	fmt.Printf("   %-32s %9s %9s %9s %13s  %s\n", "operation", "n=1k", "n=10k", "n=100k", "ratio (3)", "class")
	agreed := 0
	for k, o := range ops {
		var best [3]float64
		lo, hi := 1e18, 0.0
		classes := map[string]bool{}
		for i := range sizes {
			best[i] = 1e18
			for p := range passes {
				best[i] = min(best[i], t[p][k][i])
			}
		}
		for p := range passes {
			r := t[p][k][2] / t[p][k][1]
			lo, hi = min(lo, r), max(hi, r)
			classes[class(r)] = true
		}
		verdict := "unstable"
		if len(classes) == 1 && !classes["?"] {
			verdict = class(lo)
			if allocating[o.name] {
				verdict += " + alloc"
			}
			agreed++
		}
		fmt.Printf("   %-32s %9.1f %9.1f %9.1f %6.1f-%-6.1f  %s\n",
			o.name, best[0], best[1], best[2], lo, hi, verdict)
	}
	fmt.Printf("\n%d of %d operations got the same class in all %d passes", agreed, len(ops), passes)
	if agreed < len(ops) {
		fmt.Print("; the others print \"unstable\" rather than a guess")
	}
	fmt.Println(".")

	// Two readings of the table, computed from the best times at n = 100k.
	at := func(name string) float64 {
		for k, o := range ops {
			if o.name == name {
				b := 1e18
				for p := range passes {
					b = min(b, t[p][k][2])
				}
				return b
			}
		}
		panic(name)
	}
	fmt.Printf("At n=100k, == on equal lengths took %.0fx as long as on different lengths: the length check comes first.\n",
		at("s == t (same len, last differs)")/at("s == t (different length)"))
	fmt.Printf("ToLower on all-lowercase input: 0 allocations, yet %.0f%% of the time of the 1-uppercase call --\n",
		100*at("strings.ToLower (all lower)")/at("strings.ToLower (1 upper)"))
	fmt.Println("returning s itself saves the copy, not the scan that decides there is nothing to change.")
	fmt.Println("The allocation table is the part to trust: it is exact and does not move between runs.")
}
```

**Sample output:**

```
1. Allocations per call (testing.AllocsPerRun) and bytes per call (MemStats.TotalAlloc)
   operation                         allocs     B @1k    B @10k   B @100k B ratio
   len(s)                                 0         0         0         0       -
   s[n/2]                                 0         0         0         0       -
   s[n/4 : 3n/4]                          0         0         0         0       -
   s == t (same len, last differs)        0         0         0         0       -
   s == t (different length)              0         0         0         0       -
   s < t (byte 0 differs)                 0         0         0         0       -
   s < t (last byte differs)              0         0         0         0       -
   utf8.RuneCountInString(s)              0         0         0         0       -
   strings.Index(s, "ZQX") at end         0         0         0         0       -
   m[s] (hash + compare n bytes)          0         0         0         0       -
   strings.ToLower (all lower)            0         0         0         0       -
   strings.ToLower (1 upper)              1      1024     10240    106496    10.4
   []rune(s)                              1      4096     40960    401408     9.8
   strings.Split(s, ",")                  1      1408     13568    131072     9.7
   strings.Fields(s)                      1      2688     27264    262144     9.6
   s + "x"                                1      1024     10240    106496    10.4
   5 of 16 operations allocate; each makes exactly 1 block, and its bytes grow 9.6x-10.4x
   for a 10x longer input: the new block is (a multiple of) the input. The rest allocate nothing.

2. Time per call (ns, best of 3 passes) and the ratio test t(100k)/t(10k)
   operation                             n=1k     n=10k    n=100k     ratio (3)  class
   len(s)                                 1.8       1.8       1.8    0.4-1.1     Θ(1)
   s[n/2]                                 2.0       2.1       2.0    0.9-1.6     Θ(1)
   s[n/4 : 3n/4]                          2.3       2.6       2.1    0.8-0.9     Θ(1)
   s == t (same len, last differs)       16.0     145.0    2116.7    7.7-14.6    Θ(n)
   s == t (different length)              2.3       2.2       2.2    0.7-1.0     Θ(1)
   s < t (byte 0 differs)                 2.8       2.5       2.7    1.1-1.1     Θ(1)
   s < t (last byte differs)             31.0     249.7    2454.8    9.8-12.4    Θ(n)
   utf8.RuneCountInString(s)            279.1    2580.8   25846.3    5.4-10.0    Θ(n)
   strings.Index(s, "ZQX") at end        14.6     120.9    1022.1    4.7-8.6     Θ(n)
   m[s] (hash + compare n bytes)         19.8     134.4    1790.0   12.0-15.3    Θ(n)
   strings.ToLower (all lower)          542.4    5154.7   51781.0    7.1-10.0    Θ(n)
   strings.ToLower (1 upper)           1152.7   11051.7  110909.6    7.2-10.0    Θ(n) + alloc
   []rune(s)                           1177.1   10499.6  103077.2    9.7-10.1    Θ(n) + alloc
   strings.Split(s, ",")                818.2    8083.7   79513.6    9.7-13.9    Θ(n) + alloc
   strings.Fields(s)                   1233.0   11240.0  130144.0    7.1-11.6    Θ(n) + alloc
   s + "x"                              126.3     943.2    8603.7    5.6-9.1     Θ(n) + alloc

16 of 16 operations got the same class in all 3 passes.
At n=100k, == on equal lengths took 979x as long as on different lengths: the length check comes first.
ToLower on all-lowercase input: 0 allocations, yet 47% of the time of the 1-uppercase call --
returning s itself saves the copy, not the scan that decides there is nothing to change.
The allocation table is the part to trust: it is exact and does not move between runs.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Every string operation has one of three shapes, decided by what it must touch:
//
//   Θ(1)        only the header (&str: pointer + length, 16 bytes)
//   Θ(n)        reads the bytes, allocates nothing
//   Θ(n) + 1    reads the bytes AND allocates a new block
//
// Two instruments, as in the Go version:
//  1. ALLOCATIONS (deterministic): a counting #[global_allocator] stands in for
//     testing.AllocsPerRun. It counts `alloc` and `realloc` separately (a
//     realloc may grow in place, so it is not a copy we can claim), and the
//     bytes requested, exactly -- Go's numbers are rounded to size classes.
//  2. TIME (a supplement, noisy): ns per call at n = 1,000 / 10,000 / 100,000,
//     three passes, best of 3 per cell, and a class only if all 3 passes agree.
//
// No operation below mutates its input (they all take &str), so building the
// inputs once, outside the timed loop, is correct. An in-place operation
// (sort, make_ascii_lowercase) would have to rebuild its input every iteration.

struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0);
static BYTES: AtomicUsize = AtomicUsize::new(0); // bytes requested (alloc size, or realloc's new size)

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        REALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocs, reallocs, bytes requested) for one call of `f`.
fn count(f: impl Fn()) -> (usize, usize, usize) {
    let (a, r, b) = (
        ALLOCS.load(Relaxed),
        REALLOCS.load(Relaxed),
        BYTES.load(Relaxed),
    );
    f();
    (
        ALLOCS.load(Relaxed) - a,
        REALLOCS.load(Relaxed) - r,
        BYTES.load(Relaxed) - b,
    )
}

/// ns per call (the kit's stand-in for testing.Benchmark), with a 25 ms target.
fn ns_per_op(f: impl Fn()) -> f64 {
    let target = Duration::from_millis(25);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

/// Inputs for one size n. Every string is its own heap block, so `==` cannot
/// stop at "same pointer". All ASCII, so every byte offset is a char boundary.
struct Inputs {
    s: String,        // n bytes of lowercase words: "alpha beta, gamma delta, ..."
    same_len: String, // s with the LAST byte changed
    shorter: String,  // s minus its last byte
    first: String,    // s with the FIRST byte changed
    upper: String,    // s with one uppercase letter, at the end
    hay: String,      // s with the needle "ZQX" as its last 3 bytes
    m: HashMap<String, i32>,
}

fn build(n: usize) -> Inputs {
    let words = "alpha beta, gamma delta, ";
    let mut s = words.repeat(n / words.len() + 1);
    s.truncate(n - 1);
    s.push('z');
    let change = |i: usize, c: &str| {
        let mut t = s.clone();
        t.replace_range(i..i + 1, c); // ASCII: i..i+1 is one whole char
        t
    };
    Inputs {
        same_len: change(n - 1, "y"),
        shorter: s[..n - 1].to_owned(),
        first: change(0, "b"),
        upper: change(n - 1, "Z"),
        hay: format!("{}ZQX", &s[..n - 3]),
        m: HashMap::from([(s.clone(), 7)]),
        s,
    }
}

type Op = (&'static str, fn(&Inputs));

#[rustfmt::skip]
const OPS: [Op; 18] = [
    ("s.len()", |i| { black_box(black_box(&i.s).len()); }),
    ("s.as_bytes()[n/2]", |i| { let s = black_box(&i.s); black_box(s.as_bytes()[s.len() / 2]); }),
    ("&s[n/4..3n/4]", |i| { let s = black_box(&i.s); let n = s.len(); black_box(&s[n / 4..3 * n / 4]); }),
    ("s == t (same len, last differs)", |i| { black_box(black_box(&i.s) == black_box(&i.same_len)); }),
    ("s == t (different length)", |i| { black_box(black_box(&i.s) == black_box(&i.shorter)); }),
    ("s < t (byte 0 differs)", |i| { black_box(black_box(&i.s) < black_box(&i.first)); }),
    ("s < t (last byte differs)", |i| { black_box(black_box(&i.s) < black_box(&i.same_len)); }),
    ("s.chars().count()", |i| { black_box(black_box(&i.s).chars().count()); }),
    ("s.find(\"ZQX\") at end", |i| { black_box(black_box(&i.hay).find("ZQX")); }),
    ("m.get(s) (SipHash + compare)", |i| { black_box(black_box(&i.m).get(black_box(i.s.as_str()))); }),
    ("s.to_lowercase() (all lower)", |i| { black_box(black_box(&i.s).to_lowercase()); }),
    ("s.to_lowercase() (1 upper)", |i| { black_box(black_box(&i.upper).to_lowercase()); }),
    ("chars().collect::<Vec<char>>()", |i| { black_box(black_box(&i.s).chars().collect::<Vec<char>>()); }),
    ("split(',').count() (lazy)", |i| { black_box(black_box(&i.s).split(',').count()); }),
    ("split(',').collect::<Vec<&str>>", |i| { black_box(black_box(&i.s).split(',').collect::<Vec<&str>>()); }),
    ("split_whitespace().collect", |i| { black_box(black_box(&i.s).split_whitespace().collect::<Vec<&str>>()); }),
    ("s.to_owned() + \"x\"", |i| { black_box(black_box(i.s.as_str()).to_owned() + "x"); }),
    ("[s, \"x\"].concat()", |i| { black_box([black_box(i.s.as_str()), "x"].concat()); }),
];

const SIZES: [usize; 3] = [1_000, 10_000, 100_000];

/// class from the ratio t(100k)/t(10k): a 10x larger input. Linear work is 10
/// on an ideal machine; leaving the L1 cache pushes it to 15-25, while Θ(n²)
/// would be ~100.
fn class(ratio: f64) -> &'static str {
    if ratio < 2.5 {
        "Θ(1)"
    } else if (4.0..40.0).contains(&ratio) {
        "Θ(n)"
    } else {
        "?"
    }
}

fn main() {
    let ins: Vec<Inputs> = SIZES.iter().map(|&n| build(n)).collect();
    assert!(
        ins.iter()
            .all(|i| i.hay.ends_with("ZQX") && i.hay.len() == i.s.len())
    );

    // ---- 1. allocations: exact, and the same on every run ----
    println!("1. Allocations per call (counting allocator: allocs + reallocs) and bytes requested");
    println!(
        "   {:<32} {:>10} {:>9} {:>9} {:>9} {:>7}",
        "operation", "a + r", "B @1k", "B @10k", "B @100k", "B ratio"
    );
    let mut allocating = [false; OPS.len()];
    let (mut g_lo, mut g_hi) = (f64::MAX, 0.0f64);
    let mut grows_events = Vec::new();
    for (k, (name, f)) in OPS.iter().enumerate() {
        let c: Vec<(usize, usize, usize)> = ins.iter().map(|i| count(|| f(i))).collect();
        let events = |x: &(usize, usize, usize)| format!("{} + {}", x.0, x.1);
        let ev = if c.iter().all(|x| (x.0, x.1) == (c[0].0, c[0].1)) {
            events(&c[0])
        } else {
            grows_events.push(*name);
            format!("{} + {}..{}", c[0].0, c[0].1, c[2].1)
        };
        let ratio = if c[1].2 > 0 {
            let g = c[2].2 as f64 / c[1].2 as f64;
            allocating[k] = true;
            g_lo = g_lo.min(g);
            g_hi = g_hi.max(g);
            format!("{g:.1}")
        } else {
            "-".to_string()
        };
        println!(
            "   {:<32} {:>10} {:>9} {:>9} {:>9} {:>7}",
            name, ev, c[0].2, c[1].2, c[2].2, ratio
        );
    }
    println!(
        "   {} of {} operations allocate; their bytes grow {:.1}x-{:.1}x for a 10x longer input.",
        allocating.iter().filter(|&&a| a).count(),
        OPS.len(),
        g_lo,
        g_hi
    );
    if !grows_events.is_empty() {
        println!(
            "   Reallocations grow with n (\"x..y\" = at 1k..100k) for: {}.",
            grows_events.join(", ")
        );
        println!(
            "   collect() cannot know the piece count up front, so the Vec grows as it fills."
        );
    }

    // ---- 2. time: 3 passes, best of 3 per cell, class only if all 3 agree ----
    const PASSES: usize = 3;
    let mut t = [[[0.0f64; 3]; OPS.len()]; PASSES];
    for pass in t.iter_mut() {
        for (row, (_, f)) in pass.iter_mut().zip(OPS.iter()) {
            for (cell, i) in row.iter_mut().zip(&ins) {
                *cell = ns_per_op(|| f(i));
            }
        }
    }
    let best = |k: usize, j: usize| t.iter().map(|p| p[k][j]).fold(f64::MAX, f64::min);
    println!();
    println!("2. Time per call (ns, best of 3 passes) and the ratio test t(100k)/t(10k)");
    println!(
        "   {:<32} {:>9} {:>9} {:>9} {:>13}  class",
        "operation", "n=1k", "n=10k", "n=100k", "ratio (3)"
    );
    let mut agreed = 0;
    for (k, (name, _)) in OPS.iter().enumerate() {
        let ratios: Vec<f64> = t.iter().map(|p| p[k][2] / p[k][1]).collect();
        let lo = ratios.iter().copied().fold(f64::MAX, f64::min);
        let hi = ratios.iter().copied().fold(0.0, f64::max);
        let first = class(ratios[0]);
        let verdict = if first != "?" && ratios.iter().all(|&r| class(r) == first) {
            agreed += 1;
            if allocating[k] {
                format!("{first} + alloc")
            } else {
                first.to_string()
            }
        } else {
            "unstable".to_string()
        };
        println!(
            "   {:<32} {:>9.1} {:>9.1} {:>9.1} {:>6.1}-{:<6.1}  {}",
            name,
            best(k, 0),
            best(k, 1),
            best(k, 2),
            lo,
            hi,
            verdict
        );
    }
    print!(
        "\n{agreed} of {} operations got the same class in all {PASSES} passes",
        OPS.len()
    );
    if agreed < OPS.len() {
        print!("; the others print \"unstable\" rather than a guess");
    }
    println!(".");

    let at = |name: &str| best(OPS.iter().position(|o| o.0 == name).unwrap(), 2);
    println!(
        "At n=100k, == on equal lengths took {:.0}x as long as on different lengths: the length check comes first.",
        at("s == t (same len, last differs)") / at("s == t (different length)")
    );
    println!(
        "to_lowercase() on all-lowercase input still allocates and copies: {:.0}% of the time of the 1-uppercase call.",
        100.0 * at("s.to_lowercase() (all lower)") / at("s.to_lowercase() (1 upper)")
    );
    println!(
        "The allocation table is the part to trust: it is exact and does not move between runs."
    );
}
```

**Rust sample output:**

```
1. Allocations per call (counting allocator: allocs + reallocs) and bytes requested
   operation                             a + r     B @1k    B @10k   B @100k B ratio
   s.len()                               0 + 0         0         0         0       -
   s.as_bytes()[n/2]                     0 + 0         0         0         0       -
   &s[n/4..3n/4]                         0 + 0         0         0         0       -
   s == t (same len, last differs)       0 + 0         0         0         0       -
   s == t (different length)             0 + 0         0         0         0       -
   s < t (byte 0 differs)                0 + 0         0         0         0       -
   s < t (last byte differs)             0 + 0         0         0         0       -
   s.chars().count()                     0 + 0         0         0         0       -
   s.find("ZQX") at end                  0 + 0         0         0         0       -
   m.get(s) (SipHash + compare)          0 + 0         0         0         0       -
   s.to_lowercase() (all lower)          1 + 0      1000     10000    100000    10.0
   s.to_lowercase() (1 upper)            1 + 0      1000     10000    100000    10.0
   chars().collect::<Vec<char>>()        1 + 2      7028     70028    700028    10.0
   split(',').count() (lazy)             0 + 0         0         0         0       -
   split(',').collect::<Vec<&str>>   1 + 5..11      4032     32704    262080     8.0
   split_whitespace().collect        1 + 6..12      8128     65472    524224     8.0
   s.to_owned() + "x"                    1 + 1      3000     30000    300000    10.0
   [s, "x"].concat()                     1 + 0      1001     10001    100001    10.0
   7 of 18 operations allocate; their bytes grow 8.0x-10.0x for a 10x longer input.
   Reallocations grow with n ("x..y" = at 1k..100k) for: split(',').collect::<Vec<&str>>, split_whitespace().collect.
   collect() cannot know the piece count up front, so the Vec grows as it fills.

2. Time per call (ns, best of 3 passes) and the ratio test t(100k)/t(10k)
   operation                             n=1k     n=10k    n=100k     ratio (3)  class
   s.len()                                0.8       0.8       0.8    0.9-1.0     Θ(1)
   s.as_bytes()[n/2]                      0.8       0.8       0.8    1.0-1.0     Θ(1)
   &s[n/4..3n/4]                          1.1       1.1       1.2    0.6-1.0     Θ(1)
   s == t (same len, last differs)       18.9     186.6    1797.3    9.2-9.7     Θ(n)
   s == t (different length)              1.0       1.0       1.0    1.0-1.0     Θ(1)
   s < t (byte 0 differs)                 1.8       1.8       1.8    0.8-1.0     Θ(1)
   s < t (last byte differs)             19.0     187.7    1811.4    9.3-12.9    Θ(n)
   s.chars().count()                     30.8     318.3    3181.7    9.8-10.1    Θ(n)
   s.find("ZQX") at end                 112.9     929.0    9029.4    8.8-9.8     Θ(n)
   m.get(s) (SipHash + compare)         203.8    2022.4   20066.5    9.9-10.3    Θ(n)
   s.to_lowercase() (all lower)          54.0     289.2    2840.5    9.8-10.0    Θ(n) + alloc
   s.to_lowercase() (1 upper)            57.9     290.9    2815.8    9.2-10.1    Θ(n) + alloc
   chars().collect::<Vec<char>>()       589.4    4805.7   47079.2    8.8-9.9     Θ(n) + alloc
   split(',').count() (lazy)            523.8    5214.6   51608.9    9.6-10.0    Θ(n)
   split(',').collect::<Vec<&str>>     1015.2    8138.0   78794.7    9.7-9.8     Θ(n) + alloc
   split_whitespace().collect          1152.6    9055.1   87006.9    6.8-9.7     Θ(n) + alloc
   s.to_owned() + "x"                    79.1     166.3    1486.0    8.5-9.9     Θ(n) + alloc
   [s, "x"].concat()                     43.2     132.3    1454.0    8.7-11.1    Θ(n) + alloc

18 of 18 operations got the same class in all 3 passes.
At n=100k, == on equal lengths took 1776x as long as on different lengths: the length check comes first.
to_lowercase() on all-lowercase input still allocates and copies: 101% of the time of the 1-uppercase call.
The allocation table is the part to trust: it is exact and does not move between runs.
```

> **Go vs Rust:** Same three shapes and the same class for every operation in both, but the allocation columns differ where the APIs do: `to_lowercase` has no "return `s`" path (1 allocation of exactly n bytes even when nothing changes, where Go's `ToLower` makes 0), `split` is lazy (0 allocations until `collect`, which then reallocates 5 to 11 times because the piece count is unknown, where Go's `Split` counts first and makes one slice), and `s.to_owned() + "x"` is an alloc plus a realloc while `concat`, like Go's `+`, is exactly one. Rust's byte counts are exact (1,001), Go's are rounded to size classes (1,024).
> At n=100k the times differ by implementation, not by shape (best of 3, three runs each): Rust's chunked `to_lowercase` (~2.9 µs) and word-at-a-time `chars().count()` (~3.2 µs) beat Go's byte loops in `ToLower` (52-61 µs all-lower, ~111 µs with the copy) and `RuneCountInString` (~25 µs), while Go's `Index` (~1.0 µs, `IndexByte` skipping) and `m[s]` (~1.8 µs) beat Rust's Two-Way `find` (~9.2 µs) and SipHash `get` (~20 µs).

**Complexity:** `len`, `s[i]`, `s[i:j]`, `==` on different lengths and `<` decided at byte 0 are Θ(1) (flat at ~2 ns) · `==`/`<` to the last byte, `RuneCountInString`, `Index` and `m[s]` are Θ(n) with 0 allocations · `ToLower` with an uppercase letter, `[]rune`, `Split`, `Fields` and `+` are Θ(n) and exactly **1** allocation whose bytes grow ~10× for a 10× input (`[]rune` is 4 bytes per rune: 401,408 B at n=100k) · `ToLower` on lowercase ASCII returns `s`: 0 allocations, still Θ(n) and about half the time of the allocating call · `==` on equal lengths took ~900× longer than on different lengths at n=100k.

---

## 13. The substring that kept its parent alive

`🔴 hard` · *Retention*

Example 2's Θ(1) substring has a price. `line[:16]` is a new header whose pointer lies inside the parent's bytes,
and the garbage collector cannot free part of a block that is still pointed into, so a 16-byte key keeps its whole
one-megabyte line alive. Keep a key from each of 50 such lines and you keep 50 MiB. This is lesson 06's backing-array
leak again, and it is harder to spot here because strings feel like values. `strings.Clone` (or
`string([]byte(sub))`) copies the k bytes, Θ(k), into their own block and lets the parent go. The program measures the
live heap for all four ways of keeping the keys.

**Steps:**

1. Build 50 distinct 1 MiB lines, one allocation each (`Builder.Grow` + `String()`, which does not copy).
2. Keep the first 16 bytes of each line four ways, check which keys point inside a line, then drop the lines and run the GC.
3. Compare the live heap: about 50 MiB for both substring variants (slice and map), about 2 KB for both copies.

```go
package main

import (
	"fmt"
	"runtime"
	"strings"
	"unsafe"
)

// A substring is Θ(1) because it SHARES its parent's bytes: line[:16] is a new
// 16-byte header whose pointer is the parent's pointer. The garbage collector
// only sees that pointer -- it cannot free "the rest" of a block that is still
// pointed into -- so a 16-byte key keeps its whole 1 MB line alive.
// Same leak as lesson 06's backing-array leak (a small slice of a big array),
// in a costume that looks even more innocent: strings are "values".
//
// Experiment: 50 distinct lines of 1 MiB each. Keep the first 16 bytes of each,
// drop the lines, run the GC, and measure the live heap. Four ways to keep.

const (
	numLines = 50
	lineSize = 1 << 20 // 1 MiB
	keyLen   = 16
)

// makeLines builds each line with one allocation: Builder.Grow sizes the
// buffer and String() hands it over without a copy.
func makeLines() []string {
	lines := make([]string, numLines)
	for i := range lines {
		var b strings.Builder
		b.Grow(lineSize)
		fmt.Fprintf(&b, "line-%05d-key| ", i) // 16 bytes: the key we keep
		for j := b.Len(); j < lineSize; j++ {
			b.WriteByte('a' + byte((i+j)%26)) // distinct content per line
		}
		lines[i] = b.String()
	}
	return lines
}

// liveHeap runs the GC twice (the second pass picks up anything the first
// left for sweeping) and returns the bytes in live heap objects.
func liveHeap() uint64 {
	runtime.GC()
	runtime.GC()
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	return m.HeapAlloc
}

// pointsInto reports whether key's bytes are inside line's bytes.
func pointsInto(key, line string) bool {
	k := uintptr(unsafe.Pointer(unsafe.StringData(key)))
	l := uintptr(unsafe.Pointer(unsafe.StringData(line)))
	return k >= l && k < l+uintptr(len(line))
}

type variant struct {
	name string
	keep func(lines []string) (any, []string) // the kept structure, and its keys
}

func main() {
	variants := []variant{
		{"[]string of line[:16]", func(lines []string) (any, []string) {
			keys := make([]string, len(lines))
			for i, l := range lines {
				keys[i] = l[:keyLen]
			}
			return keys, keys
		}},
		{"map[string]struct{} of line[:16]", func(lines []string) (any, []string) {
			set := make(map[string]struct{}, len(lines))
			var keys []string
			for _, l := range lines {
				set[l[:keyLen]] = struct{}{}
			}
			for k := range set {
				keys = append(keys, k) // only to check where they point; dropped below
			}
			return set, keys
		}},
		{"[]string of strings.Clone(line[:16])", func(lines []string) (any, []string) {
			keys := make([]string, len(lines))
			for i, l := range lines {
				keys[i] = strings.Clone(l[:keyLen])
			}
			return keys, keys
		}},
		{"[]string of string([]byte(line[:16]))", func(lines []string) (any, []string) {
			keys := make([]string, len(lines))
			for i, l := range lines {
				keys[i] = string([]byte(l[:keyLen]))
			}
			return keys, keys
		}},
	}

	fmt.Printf("%d lines x %d bytes = %d bytes of text; we keep %d bytes of each (%d bytes of keys)\n\n",
		numLines, lineSize, numLines*lineSize, keyLen, numLines*keyLen)
	fmt.Printf("  %-38s %9s %12s %13s\n", "kept as", "share", "live bytes", "bytes per key")

	results := make([]uint64, len(variants))
	for pass := range 2 { // pass 0 is a warm-up: the runtime's first-use allocations land there
		for v, vr := range variants {
			base := liveHeap()
			lines := makeLines()
			kept, keys := vr.keep(lines)
			shared := 0
			for _, k := range keys {
				for _, l := range lines {
					if pointsInto(k, l) {
						shared++
						break
					}
				}
			}
			keys = nil  // the map variant's helper slice must not count
			lines = nil // drop every reference to the lines we hold ourselves
			live := liveHeap() - base
			runtime.KeepAlive(kept) // the kept structure is alive up to here
			results[v] = live
			if pass == 1 {
				fmt.Printf("  %-38s %5d/%-3d %12d %13d\n",
					vr.name, shared, numLines, live, live/numLines)
			}
			_, _ = keys, lines
		}
	}

	pinned, cloned := results[0], results[2]
	fmt.Printf("\n\"share\" = keys whose pointer lies inside a line (checked before the lines were dropped).\n")
	fmt.Printf("50 substrings pinned %.2f MiB: %.0fx the %d bytes they show.\n",
		float64(pinned)/(1<<20), float64(pinned)/float64(numLines*keyLen), numLines*keyLen)
	fmt.Printf("Cloned keys: %d bytes live, of which the copies are %d and the []string's headers %d;\n",
		cloned, numLines*keyLen, numLines*int(unsafe.Sizeof("")))
	fmt.Println("the rest is size-class rounding and the runtime's own small allocations.")
	fmt.Printf("The substrings kept %.0fx more memory alive than the clones.\n", float64(pinned)/float64(cloned))
	fmt.Println("Live bytes = HeapAlloc after two GCs, minus the same reading taken before the lines were built.")
}
```

**Sample output:**

```
50 lines x 1048576 bytes = 52428800 bytes of text; we keep 16 bytes of each (800 bytes of keys)

  kept as                                    share   live bytes bytes per key
  []string of line[:16]                     50/50      52430072       1048601
  map[string]struct{} of line[:16]          50/50      52431016       1048620
  []string of strings.Clone(line[:16])       0/50          1832            36
  []string of string([]byte(line[:16]))      0/50          1720            34

"share" = keys whose pointer lies inside a line (checked before the lines were dropped).
50 substrings pinned 50.00 MiB: 65538x the 800 bytes they show.
Cloned keys: 1832 bytes live, of which the copies are 800 and the []string's headers 800;
the rest is size-class rounding and the runtime's own small allocations.
The substrings kept 28619x more memory alive than the clones.
Live bytes = HeapAlloc after two GCs, minus the same reading taken before the lines were built.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashSet;
use std::fmt::Write;
use std::ops::Range;
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Go's substring shares its parent's bytes, and the GC silently keeps the whole
// parent alive. Rust has no GC, so the question becomes: who OWNS the bytes?
//
//   &str    borrows. It can never outlive its String -- the borrow checker
//           refuses -- so a borrowed key keeps the line alive in plain sight:
//           you must keep `lines` yourself, or the program does not compile.
//   String  owns. `to_owned()` copies k bytes into a new block; the line can go.
//
// The leak Rust CAN have is capacity: `line.truncate(16)` keeps the 1 MiB
// buffer and just shortens the length. And Go's model can be written out on
// purpose: an Rc<str> plus a range is exactly "a pointer into a shared parent".
//
// The counting allocator below is exact: live bytes are a running sum of what
// is allocated and not yet freed. No GC, no size classes, no noise.

struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

const NUM_LINES: usize = 50;
const LINE_SIZE: usize = 1 << 20; // 1 MiB
const KEY_LEN: usize = 16;

/// 50 distinct lines, each exactly one 1 MiB allocation (with_capacity, then
/// filled without growing). Every line is ASCII, so byte 16 is a char boundary.
fn make_lines() -> Vec<String> {
    (0..NUM_LINES)
        .map(|i| {
            let mut s = String::with_capacity(LINE_SIZE);
            write!(s, "line-{i:05}-key| ").unwrap(); // 16 bytes: the key we keep
            let start = s.len();
            s.extend((start..LINE_SIZE).map(|j| char::from(b'a' + ((i + j) % 26) as u8)));
            s
        })
        .collect()
}

/// The address range of each line's buffer (a stack array: no heap noise).
fn spans(lines: &[String]) -> [Range<usize>; NUM_LINES] {
    std::array::from_fn(|i| {
        let p = lines[i].as_ptr() as usize;
        p..p + lines[i].capacity()
    })
}

/// How many keys start inside one of the line buffers.
fn shared<'a>(keys: impl Iterator<Item = &'a str>, spans: &[Range<usize>]) -> usize {
    keys.filter(|k| spans.iter().any(|s| s.contains(&(k.as_ptr() as usize))))
        .count()
}

fn row(name: &str, shared: usize, live: usize) {
    println!(
        "  {:<40} {:>5}/{:<3} {:>12} {:>13}",
        name,
        shared,
        NUM_LINES,
        live,
        live / NUM_LINES
    );
}

/// Go's substring, written out: a shared, reference-counted parent + a range.
struct Key {
    parent: Rc<str>,
    range: Range<usize>,
}
impl Key {
    fn as_str(&self) -> &str {
        &self.parent[self.range.clone()] // 0..16 of an ASCII line: on a boundary
    }
}

// What the Go program does -- keep the substrings, drop the lines -- does not
// compile in Rust:
//
//     let keys: Vec<&str>;
//     {
//         let lines = make_lines();
//         keys = lines.iter().map(|l| &l[..16]).collect();
//     } // the lines are dropped here
//     println!("{}", keys.len());
//
//   error[E0597]: `lines` does not live long enough
//     |         let lines = make_lines();
//     |             ----- binding `lines` declared here
//     |         keys = lines.iter().map(|l| &l[..16]).collect();
//     |                ^^^^^ borrowed value does not live long enough
//     |     } // the lines are dropped here
//     |     - `lines` dropped here while still borrowed
//     |     println!("{}", keys.len());
//     |                    ---- borrow later used here
//     = note: `keys` is a collection that stores borrowed references, but `lines`
//             does not live long enough to be stored in it
//
// So the borrowed variants below keep `lines` alive explicitly -- and the
// measurement says what that costs.

fn main() {
    println!(
        "{NUM_LINES} lines x {LINE_SIZE} bytes = {} bytes of text; we keep {KEY_LEN} bytes of each ({} bytes of keys)\n",
        NUM_LINES * LINE_SIZE,
        NUM_LINES * KEY_LEN
    );
    println!(
        "  {:<40} {:>9} {:>12} {:>13}",
        "kept as", "share", "live bytes", "bytes per key"
    );
    let mut results = Vec::with_capacity(6);
    let mut record = |name: &str, s: usize, live: usize| {
        row(name, s, live);
        results.push(live);
    };

    // 1. Vec<&str>: borrows -- `lines` must stay alive as long as `keys`.
    let base = live_bytes();
    let lines = make_lines();
    let keys: Vec<&str> = lines.iter().map(|l| &l[..KEY_LEN]).collect();
    let s = shared(keys.iter().copied(), &spans(&lines));
    record("Vec<&str> (lines kept alive)", s, live_bytes() - base);
    drop(keys);
    drop(lines);

    // 2. HashSet<&str>: same borrow, same rule.
    let base = live_bytes();
    let lines = make_lines();
    let set: HashSet<&str> = lines.iter().map(|l| &l[..KEY_LEN]).collect();
    let s = shared(set.iter().copied(), &spans(&lines));
    record("HashSet<&str> (lines kept alive)", s, live_bytes() - base);
    drop(set);
    drop(lines);

    // 3. Vec<String> via to_owned(): 16 bytes copied, then the lines go.
    let base = live_bytes();
    let lines = make_lines();
    let sp = spans(&lines);
    let keys: Vec<String> = lines.iter().map(|l| l[..KEY_LEN].to_owned()).collect();
    drop(lines);
    let s = shared(keys.iter().map(String::as_str), &sp);
    record(
        "Vec<String>, to_owned() (lines dropped)",
        s,
        live_bytes() - base,
    );
    drop(keys);

    // 4. truncate(16) in place: the length shrinks, the 1 MiB capacity stays.
    let base = live_bytes();
    let lines = make_lines();
    let sp = spans(&lines);
    let keys: Vec<String> = lines
        .into_iter()
        .map(|mut l| {
            l.truncate(KEY_LEN);
            l
        })
        .collect();
    let s = shared(keys.iter().map(String::as_str), &sp);
    let cap = keys[0].capacity();
    record("Vec<String>, truncate(16)", s, live_bytes() - base);

    // 5. ... and shrink_to_fit(): realloc down to 16 bytes, the rest is freed.
    let keys: Vec<String> = keys
        .into_iter()
        .map(|mut k| {
            k.shrink_to_fit();
            k
        })
        .collect();
    let s = shared(keys.iter().map(String::as_str), &sp);
    let cap_after = keys[0].capacity();
    record(
        "Vec<String>, truncate + shrink_to_fit()",
        s,
        live_bytes() - base,
    );
    drop(keys);

    // 6. Go's model on purpose: Rc<str> (the parent, shared) + a range.
    let base = live_bytes();
    let parents: Vec<Rc<str>> = make_lines().into_iter().map(Rc::from).collect();
    let sp: [Range<usize>; NUM_LINES] = std::array::from_fn(|i| {
        let p = parents[i].as_ptr() as usize;
        p..p + parents[i].len()
    });
    let keys: Vec<Key> = parents
        .into_iter() // moves each Rc into its key: the key is now the only owner
        .map(|parent| Key {
            parent,
            range: 0..KEY_LEN,
        })
        .collect();
    let s = shared(keys.iter().map(Key::as_str), &sp);
    record(
        "Vec<Key{Rc<str>, Range}> (Go's model)",
        s,
        live_bytes() - base,
    );
    drop(keys);

    let (pinned, owned) = (results[0], results[2]);
    println!("\n\"share\" = keys whose pointer lies inside a line's buffer.");
    println!(
        "Borrowed keys cost {:.2} MiB because the lines must stay: {:.0}x the {} bytes they show.",
        pinned as f64 / f64::from(1 << 20),
        pinned as f64 / (NUM_LINES * KEY_LEN) as f64,
        NUM_LINES * KEY_LEN
    );
    println!(
        "Owned keys: {owned} bytes live, exactly the copies ({}) plus the Vec's String headers ({}).",
        NUM_LINES * KEY_LEN,
        NUM_LINES * size_of::<String>()
    );
    println!(
        "truncate(16) left capacity {cap}; shrink_to_fit() made it {cap_after}: \
         Rust's quiet retention is capacity, not sharing."
    );
    println!(
        "An Rc<str> per line kept {:.2} MiB alive -- the Go behaviour, but spelled out in the type.",
        results[5] as f64 / f64::from(1 << 20)
    );
}
```

**Rust sample output:**

```
50 lines x 1048576 bytes = 52428800 bytes of text; we keep 16 bytes of each (800 bytes of keys)

  kept as                                      share   live bytes bytes per key
  Vec<&str> (lines kept alive)                50/50      52430800       1048616
  HashSet<&str> (lines kept alive)            50/50      52431096       1048621
  Vec<String>, to_owned() (lines dropped)      0/50          2000            40
  Vec<String>, truncate(16)                   50/50      52430000       1048600
  Vec<String>, truncate + shrink_to_fit()      0/50          2000            40
  Vec<Key{Rc<str>, Range}> (Go's model)       50/50      52431200       1048624

"share" = keys whose pointer lies inside a line's buffer.
Borrowed keys cost 50.00 MiB because the lines must stay: 65538x the 800 bytes they show.
Owned keys: 2000 bytes live, exactly the copies (800) plus the Vec's String headers (1200).
truncate(16) left capacity 1048576; shrink_to_fit() made it 16: Rust's quiet retention is capacity, not sharing.
An Rc<str> per line kept 50.00 MiB alive -- the Go behaviour, but spelled out in the type.
```

> **Go vs Rust:** Go makes retention invisible; Rust makes it a type error. The Go program's "keep the substrings, drop the lines" is E0597 here (copied from a real compile into the comment), so borrowed keys can exist only while `lines` does, and the live count shows what that costs: 52,430,800 bytes, against Go's ~52.43 million with the lines already "dropped". Rust's counts are exact and identical on every run. Owned keys take 2,000 bytes (800 of copies + 50 × 24-byte `String` headers); Go reports ~1.7-2.2 KB, rounded to size classes and measured after a GC.
> What Rust can still do quietly is keep **capacity**: `truncate(16)` leaves each 1 MiB buffer allocated until `shrink_to_fit()`. And `Rc<str>` + a range reproduces Go's sharing on purpose, at the same 50 MiB.

**Complexity:** a substring is Θ(1) and retains its **whole** parent: 50 keys of 16 bytes kept about 52.43 million live bytes (≈ 1,048,600 per key, 50.00 MiB) in the slice and in the map alike · `strings.Clone` and `string([]byte(sub))` are Θ(k) and retain k bytes: about 1.7-2.2 KB live for all 50 keys plus their `[]string` (some runs add ~5 KB of the runtime's own allocations) · the substrings held over 7,000× more memory than the clones in every run.

---

## 14. A tokenizer that never allocates

`🔴 hard` · *Zero-copy parsing*

This is the cost model applied to a real job: parse 100,000 `key=value` log lines and count the `level=` values.
`strings.Split` allocates a slice per line and another per field. Walking the line with `IndexByte` and cutting
substrings allocates nothing, because every token is a view into the input. The `[]rune` version looks like the safe
choice for accented text, and it is the slowest and allocates the most, for no benefit: every byte of a multi-byte
UTF-8 rune is ≥ 0x80, so it can never equal `' '` or `'='`, and a byte scan for ASCII delimiters always cuts on a rune
boundary. All four parsers are Θ(n) in bytes. What separates them is the allocation count, measured exactly, with
the zero asserted.

**Steps:**

1. Generate 100,000 lines from a fixed PCG seed, with accented and CJK users, accented words and an emoji. Check that no
   non-ASCII byte equals a delimiter.
2. Run the four parsers (plus the `FieldsSeq` variant of the zero-copy one), require identical counts, then count
   allocations for one line and for all 100,000.
3. Time three passes per parser. The zero-copy scan is fastest and the `[]rune` scanner slowest, and the program
   prints why `Fields` is not faster than `Split` on this input.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"runtime/debug"
	"strings"
	"testing"
	"time"
	"unicode/utf8"
)

// The job: 100,000 log lines like
//
//	ts=1700000000 level=WARN user=zoë dur=412ms msg="délai dépassé"
//
// and count how many lines have each level= value. Four parsers that agree on
// every count, and differ only in what they ALLOCATE per line:
//
//	A  strings.Split(line, " ") + strings.Split(field, "=")  one slice per line + one per field
//	B  strings.Fields(line) + strings.Cut                    one slice per line
//	C  walk the line with IndexByte, then strings.Cut        nothing: every token is a substring
//	   (C2: the same with the strings.FieldsSeq iterator)
//	D  []rune(line), then string(runes[i:j]) per token       the "Unicode-safe" one: the most
//
// Why C is safe on UTF-8 without decoding anything: every byte of a multi-byte
// UTF-8 sequence is >= 0x80, and ' ' (0x20) and '=' (0x3D) are < 0x80. So a
// byte equal to a delimiter is ALWAYS a real delimiter, never the middle of
// "é" or "李". Cutting at it always lands on a rune boundary.

var (
	levels = []string{"INFO", "WARN", "ERROR", "DEBUG"}
	users  = []string{"alice", "bob", "zoë", "jürgen", "anaïs", "ştefan", "李雷", "olga"}
	words  = []string{"cache", "miss", "délai", "dépassé", "retry", "ok", "naïve", "réseau", "timeout", "😀", "disk", "full"}
)

// makeLines: fixed PCG seed, so every run (and the Rust twin) sees the same text.
func makeLines(n int) []string {
	rng := rand.New(rand.NewPCG(1, 2))
	lines := make([]string, n)
	for i := range lines {
		level := levels[rng.IntN(len(levels))]
		user := users[rng.IntN(len(users))]
		dur := rng.IntN(1000)
		msg := make([]string, 1+rng.IntN(4))
		for j := range msg {
			msg[j] = words[rng.IntN(len(words))]
		}
		lines[i] = fmt.Sprintf(`ts=%d level=%s user=%s dur=%dms msg="%s"`,
			1_700_000_000+i, level, user, dur, strings.Join(msg, " "))
	}
	return lines
}

type counts map[string]int

func newCounts() counts { // pre-seeded: counting an existing key never allocates
	c := counts{}
	for _, l := range levels {
		c[l] = 0
	}
	return c
}

func parseA(line string, c counts) {
	for _, f := range strings.Split(line, " ") {
		kv := strings.Split(f, "=")
		if len(kv) == 2 && kv[0] == "level" {
			c[kv[1]]++
		}
	}
}

func parseB(line string, c counts) {
	for _, f := range strings.Fields(line) {
		if k, v, ok := strings.Cut(f, "="); ok && k == "level" {
			c[v]++
		}
	}
}

func parseC(line string, c counts) {
	for len(line) > 0 {
		f := line
		if i := strings.IndexByte(line, ' '); i >= 0 {
			f, line = line[:i], line[i+1:] // both are views into the same bytes
		} else {
			line = ""
		}
		if k, v, ok := strings.Cut(f, "="); ok && k == "level" {
			c[v]++
		}
	}
}

func parseC2(line string, c counts) {
	for f := range strings.FieldsSeq(line) {
		if k, v, ok := strings.Cut(f, "="); ok && k == "level" {
			c[v]++
		}
	}
}

// parseD: []rune(line) does not escape, but the compiler's stack buffer holds
// only 32 runes, so every line here (50+ runes) gets a heap copy. The key
// string only meets a literal and stays in a stack buffer too (go build
// -gcflags=-m: "string(tok[:j]) does not escape"); the value is passed to the
// map and "escapes to heap": one allocation per key=value field.
func parseD(line string, c counts) {
	r := []rune(line) // 4 bytes per rune
	start := 0
	for i := 0; i <= len(r); i++ {
		if i < len(r) && r[i] != ' ' {
			continue
		}
		tok := r[start:i]
		start = i + 1
		for j, x := range tok {
			if x == '=' {
				k, v := string(tok[:j]), string(tok[j+1:]) // a new string per token
				if k == "level" {
					c[v]++
				}
				break
			}
		}
	}
}

type parser struct {
	name string
	f    func(string, counts)
}

var parsers = []parser{
	{"A  Split + Split", parseA},
	{"B  Fields + Cut", parseB},
	{"C  IndexByte scan + Cut", parseC},
	{"C2 FieldsSeq + Cut", parseC2},
	{"D  []rune + string(runes)", parseD},
}

func main() {
	const n = 100_000
	lines := makeLines(n)
	sample := lines[0]
	total, nonASCII, delim, nonASCIILines := 0, 0, 0, 0
	for _, l := range lines {
		total += len(l)
		if len(l) != utf8.RuneCountInString(l) {
			nonASCIILines++
		}
		for i := 0; i < len(l); i++ {
			if l[i] >= utf8.RuneSelf {
				nonASCII++
				if l[i] == ' ' || l[i] == '=' {
					delim++ // can never happen: shown, not assumed
				}
			}
		}
	}
	fmt.Printf("%d lines, %d bytes; %d bytes belong to multi-byte runes, %d of them equal ' ' or '='\n",
		n, total, nonASCII, delim)
	fmt.Printf("sample line (%d bytes, %d runes, %d fields): %s\n\n",
		len(sample), utf8.RuneCountInString(sample), len(strings.Split(sample, " ")), sample)

	// ---- 1. correctness: all parsers must agree on every count ----
	var want counts
	for _, p := range parsers {
		c := newCounts()
		for _, l := range lines {
			p.f(l, c)
		}
		if want == nil {
			want = c
			fmt.Printf("level counts: INFO %d  WARN %d  ERROR %d  DEBUG %d\n",
				c["INFO"], c["WARN"], c["ERROR"], c["DEBUG"])
		}
		for _, l := range levels {
			if c[l] != want[l] {
				panic(fmt.Sprintf("%s disagrees on %s: %d vs %d", p.name, l, c[l], want[l]))
			}
		}
	}
	fmt.Printf("all %d parsers agree on all %d counts\n\n", len(parsers), len(levels))

	// ---- 2. allocations: exact counts ----
	fmt.Println("allocations (the sample line: testing.AllocsPerRun; all lines: MemStats.Mallocs)")
	fmt.Printf("  %-28s %13s %16s %10s %10s\n", "parser", "sample line", "all 100k lines", "per line", "B/line")
	allocs := map[string]uint64{}
	for _, p := range parsers {
		c := newCounts()
		one := testing.AllocsPerRun(100, func() { p.f(sample, c) })
		runtime.GC()
		old := debug.SetGCPercent(-1) // no GC cycle (and its bookkeeping) inside the count
		var a, b runtime.MemStats
		runtime.ReadMemStats(&a)
		for _, l := range lines {
			p.f(l, c)
		}
		runtime.ReadMemStats(&b)
		debug.SetGCPercent(old)
		m := b.Mallocs - a.Mallocs
		allocs[p.name] = m
		fmt.Printf("  %-28s %13.0f %16d %10.2f %10.1f\n", p.name, one, m,
			float64(m)/n, float64(b.TotalAlloc-a.TotalAlloc)/n)
	}
	for _, name := range []string{"C  IndexByte scan + Cut", "C2 FieldsSeq + Cut"} {
		if allocs[name] != 0 {
			panic(name + " allocated")
		}
	}
	fmt.Println("  assert: C and C2 made 0 allocations over all 100,000 lines -- held")

	// ---- 3. time: one table, best of 3 passes over all lines ----
	const passes = 3
	fmt.Println()
	fmt.Println("time per line (ns): 3 passes over all 100,000 lines")
	fmt.Printf("  %-28s %8s %17s %10s\n", "parser", "best", "range of 3", "vs C")
	best := make([]float64, len(parsers))
	ranges := make([][2]float64, len(parsers))
	for k, p := range parsers {
		lo, hi := 1e18, 0.0
		for range passes {
			c := newCounts()
			t0 := time.Now()
			for _, l := range lines {
				p.f(l, c)
			}
			ns := float64(time.Since(t0).Nanoseconds()) / n
			lo, hi = min(lo, ns), max(hi, ns)
		}
		best[k], ranges[k] = lo, [2]float64{lo, hi}
	}
	for k, p := range parsers {
		fmt.Printf("  %-28s %8.1f %8.1f - %-6.1f %9.1fx\n",
			p.name, best[k], ranges[k][0], ranges[k][1], best[k]/best[2])
	}
	fmt.Printf("\n%d of %d lines contain a multi-byte rune; on those, strings.Fields leaves its ASCII fast path\n",
		nonASCIILines, n)
	fmt.Printf("for FieldsFunc(unicode.IsSpace) -- so B, at %.1f allocations per line, took %.2fx the time of A at %.1f.\n",
		float64(allocs["B  Fields + Cut"])/n, best[1]/best[0], float64(allocs["A  Split + Split"])/n)
	fmt.Printf("D, the []rune parser, made %.1f allocations per line and was %.1fx slower than C, for no gain:\n",
		float64(allocs["D  []rune + string(runes)"])/n, best[4]/best[2])
	fmt.Println("its counts are identical, because no delimiter byte can occur inside a multi-byte rune.")
}
```

**Sample output:**

```
100000 lines, 6725341 bytes; 467366 bytes belong to multi-byte runes, 0 of them equal ' ' or '='
sample line (68 bytes, 66 runes, 7 fields): ts=1700000000 level=INFO user=anaïs dur=784ms msg="délai cache ok"

level counts: INFO 24789  WARN 25082  ERROR 25109  DEBUG 25020
all 5 parsers agree on all 4 counts

allocations (the sample line: testing.AllocsPerRun; all lines: MemStats.Mallocs)
  parser                         sample line   all 100k lines   per line     B/line
  A  Split + Split                         8           749960       7.50      288.0
  B  Fields + Cut                          1           100000       1.00      104.0
  C  IndexByte scan + Cut                  0                0       0.00        0.0
  C2 FieldsSeq + Cut                       0                0       0.00        0.0
  D  []rune + string(runes)                6           600000       6.00      325.3
  assert: C and C2 made 0 allocations over all 100,000 lines -- held

time per line (ns): 3 passes over all 100,000 lines
  parser                           best        range of 3       vs C
  A  Split + Split                228.4    228.4 - 241.2        3.0x
  B  Fields + Cut                 245.2    245.2 - 248.8        3.2x
  C  IndexByte scan + Cut          75.7     75.7 - 79.1         1.0x
  C2 FieldsSeq + Cut              140.4    140.4 - 144.2        1.9x
  D  []rune + string(runes)       368.7    368.7 - 388.8        4.9x

88422 of 100000 lines contain a multi-byte rune; on those, strings.Fields leaves its ASCII fast path
for FieldsFunc(unicode.IsSpace) -- so B, at 1.0 allocations per line, took 1.07x the time of A at 7.5.
D, the []rune parser, made 6.0 allocations per line and was 4.9x slower than C, for no gain:
its counts are identical, because no delimiter byte can occur inside a multi-byte rune.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::Instant;

// The job: 100,000 log lines like
//
//   ts=1700000000 level=WARN user=zoë dur=412ms msg="délai dépassé"
//
// and count how many lines have each level= value. Same four parsers as the Go
// version, Rust spelling:
//
//   A  split(' ').collect::<Vec<&str>>() + split('=').collect()   a Vec per line + one per field
//   B  split_whitespace().collect::<Vec<&str>>() + split_once     a Vec per line
//   C  split_once(' ') in a loop + split_once('=')                nothing: every token is a &str view
//      (C2: the lazy split_ascii_whitespace() iterator)
//   D  chars().collect::<Vec<char>>(), a String per token          the "Unicode-safe" one
//
// In Rust the zero-allocation parser is simply the default: split, split_once
// and split_*whitespace are lazy iterators yielding &str borrowed from the
// line. Allocation only happens where you write collect() or to_string().
//
// Why the byte-level cut is safe on UTF-8: every byte of a multi-byte sequence
// is >= 0x80, and ' ' (0x20) and '=' (0x3D) are < 0x80, so a delimiter byte is
// always on a char boundary. Rust checks it anyway: slicing a &str off a
// boundary panics, and split_once(' ') never does here.

struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0);
static BYTES: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        REALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocs, reallocs, bytes requested) while running `f`.
fn count(f: impl FnOnce()) -> (usize, usize, usize) {
    let (a, r, b) = (
        ALLOCS.load(Relaxed),
        REALLOCS.load(Relaxed),
        BYTES.load(Relaxed),
    );
    f();
    (
        ALLOCS.load(Relaxed) - a,
        REALLOCS.load(Relaxed) - r,
        BYTES.load(Relaxed) - b,
    )
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN, so the same seed
// (1, 2) generates byte-for-byte the same 100,000 lines as the Go program.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
}

const LEVELS: [&str; 4] = ["INFO", "WARN", "ERROR", "DEBUG"];
const USERS: [&str; 8] = [
    "alice", "bob", "zoë", "jürgen", "anaïs", "ştefan", "李雷", "olga",
];
const WORDS: [&str; 12] = [
    "cache",
    "miss",
    "délai",
    "dépassé",
    "retry",
    "ok",
    "naïve",
    "réseau",
    "timeout",
    "😀",
    "disk",
    "full",
];

fn make_lines(n: usize) -> Vec<String> {
    let mut rng = Pcg::new(1, 2);
    (0..n)
        .map(|i| {
            let level = LEVELS[rng.int_n(LEVELS.len())];
            let user = USERS[rng.int_n(USERS.len())];
            let dur = rng.int_n(1000);
            let words = 1 + rng.int_n(4);
            let msg: Vec<&str> = (0..words).map(|_| WORDS[rng.int_n(WORDS.len())]).collect();
            format!(
                "ts={} level={level} user={user} dur={dur}ms msg=\"{}\"",
                1_700_000_000 + i,
                msg.join(" ")
            )
        })
        .collect()
}

type Counts = HashMap<String, u64>;

fn new_counts() -> Counts {
    // pre-seeded: counting an existing key never allocates
    LEVELS.iter().map(|l| (l.to_string(), 0)).collect()
}

fn bump(c: &mut Counts, v: &str) {
    match c.get_mut(v) {
        Some(n) => *n += 1,
        None => {
            c.insert(v.to_owned(), 1);
        }
    }
}

fn parse_a(line: &str, c: &mut Counts) {
    for f in line.split(' ').collect::<Vec<&str>>() {
        let kv: Vec<&str> = f.split('=').collect();
        if kv.len() == 2 && kv[0] == "level" {
            bump(c, kv[1]);
        }
    }
}

fn parse_b(line: &str, c: &mut Counts) {
    for f in line.split_whitespace().collect::<Vec<&str>>() {
        if let Some(("level", v)) = f.split_once('=') {
            bump(c, v);
        }
    }
}

fn parse_c(line: &str, c: &mut Counts) {
    let mut rest = line;
    while !rest.is_empty() {
        // split_once = find the byte + cut: both halves borrow the same bytes
        let (f, r) = rest.split_once(' ').unwrap_or((rest, ""));
        rest = r;
        if let Some(("level", v)) = f.split_once('=') {
            bump(c, v);
        }
    }
}

fn parse_c2(line: &str, c: &mut Counts) {
    for f in line.split_ascii_whitespace() {
        if let Some(("level", v)) = f.split_once('=') {
            bump(c, v);
        }
    }
}

fn parse_d(line: &str, c: &mut Counts) {
    let r: Vec<char> = line.chars().collect(); // 4 bytes per char
    for tok in r.split(|&x| x == ' ') {
        if let Some(j) = tok.iter().position(|&x| x == '=') {
            let k: String = tok[..j].iter().collect(); // a new String per token
            let v: String = tok[j + 1..].iter().collect();
            if k == "level" {
                bump(c, &v);
            }
        }
    }
}

type Parser = (&'static str, fn(&str, &mut Counts));

const PARSERS: [Parser; 5] = [
    ("A  split.collect + split.collect", parse_a),
    ("B  split_whitespace.collect", parse_b),
    ("C  split_once scan", parse_c),
    ("C2 split_ascii_whitespace", parse_c2),
    ("D  Vec<char> + String/token", parse_d),
];

fn main() {
    const N: usize = 100_000;
    let lines = make_lines(N);
    let sample = &lines[0];
    let total: usize = lines.iter().map(String::len).sum();
    let non_ascii = lines.iter().flat_map(|l| l.bytes()).filter(|&b| b >= 0x80);
    let (mut non_ascii_bytes, mut delim) = (0, 0);
    for b in non_ascii {
        non_ascii_bytes += 1;
        if b == b' ' || b == b'=' {
            delim += 1; // can never happen: shown, not assumed
        }
    }
    let non_ascii_lines = lines.iter().filter(|l| !l.is_ascii()).count();
    println!(
        "{N} lines, {total} bytes; {non_ascii_bytes} bytes belong to multi-byte runes, {delim} of them equal ' ' or '='"
    );
    println!(
        "sample line ({} bytes, {} chars, {} fields): {sample}\n",
        sample.len(),
        sample.chars().count(),
        sample.split(' ').count()
    );

    // ---- 1. correctness: all parsers must agree on every count ----
    let mut want: Option<Counts> = None;
    for (name, f) in PARSERS {
        let mut c = new_counts();
        for l in &lines {
            f(l, &mut c);
        }
        match &want {
            None => {
                println!(
                    "level counts: INFO {}  WARN {}  ERROR {}  DEBUG {}",
                    c["INFO"], c["WARN"], c["ERROR"], c["DEBUG"]
                );
                want = Some(c);
            }
            Some(w) => assert_eq!(&c, w, "{name} disagrees"),
        }
    }
    println!(
        "all {} parsers agree on all {} counts\n",
        PARSERS.len(),
        LEVELS.len()
    );

    // ---- 2. allocations: exact counts from the counting allocator ----
    println!("allocations (counting allocator: allocs + reallocs; bytes requested)");
    println!(
        "  {:<34} {:>11} {:>16} {:>10} {:>10}",
        "parser", "sample line", "all 100k lines", "per line", "B/line"
    );
    let mut events = [0usize; PARSERS.len()];
    for (k, (name, f)) in PARSERS.iter().enumerate() {
        let mut c = new_counts();
        let one = count(|| f(sample, &mut c));
        let all = count(|| {
            for l in &lines {
                f(l, &mut c);
            }
        });
        events[k] = all.0 + all.1;
        println!(
            "  {:<34} {:>11} {:>16} {:>10.2} {:>10.1}",
            name,
            format!("{} + {}", one.0, one.1),
            format!("{} + {}", all.0, all.1),
            events[k] as f64 / N as f64,
            all.2 as f64 / N as f64
        );
    }
    assert_eq!(events[2], 0, "C allocated");
    assert_eq!(events[3], 0, "C2 allocated");
    println!("  assert: C and C2 made 0 allocations over all 100,000 lines -- held");

    // ---- 3. time: one table, best of 3 passes over all lines ----
    const PASSES: usize = 3;
    println!();
    println!("time per line (ns): 3 passes over all 100,000 lines");
    println!(
        "  {:<34} {:>8} {:>17} {:>10}",
        "parser", "best", "range of 3", "vs C"
    );
    let mut ranges = [(f64::MAX, 0.0f64); PARSERS.len()];
    for (k, (_, f)) in PARSERS.iter().enumerate() {
        for _ in 0..PASSES {
            let mut c = new_counts();
            let t0 = Instant::now();
            for l in &lines {
                f(black_box(l), &mut c);
            }
            black_box(&c);
            let ns = t0.elapsed().as_nanos() as f64 / N as f64;
            ranges[k] = (ranges[k].0.min(ns), ranges[k].1.max(ns));
        }
    }
    let best = |k: usize| ranges[k].0;
    for (k, (name, _)) in PARSERS.iter().enumerate() {
        println!(
            "  {:<34} {:>8.1} {:>8.1} - {:<6.1} {:>9.1}x",
            name,
            best(k),
            ranges[k].0,
            ranges[k].1,
            best(k) / best(2)
        );
    }
    println!(
        "\n{non_ascii_lines} of {N} lines contain a multi-byte char. split_whitespace() tests every char for Unicode \
         whitespace on any input,\nso unlike Go's Fields it has no ASCII fast path to lose: B, at {:.1} allocation events per line, \
         took {:.2}x the time of A at {:.1}.",
        events[1] as f64 / N as f64,
        best(1) / best(0),
        events[0] as f64 / N as f64
    );
    println!(
        "The two zero-allocation parsers differ in time only: C2 (a byte split) took {:.2}x the time of C (split_once).",
        best(3) / best(2)
    );
    println!(
        "D, the Vec<char> parser, made {:.1} allocation events per line and was {:.1}x slower than C, for no gain:",
        events[4] as f64 / N as f64,
        best(4) / best(2)
    );
    println!(
        "its counts are identical, because no delimiter byte can occur inside a multi-byte char."
    );
}
```

**Rust sample output:**

```
100000 lines, 6725341 bytes; 467366 bytes belong to multi-byte runes, 0 of them equal ' ' or '='
sample line (68 bytes, 66 chars, 7 fields): ts=1700000000 level=INFO user=anaïs dur=784ms msg="délai cache ok"

level counts: INFO 24789  WARN 25082  ERROR 25109  DEBUG 25020
all 5 parsers agree on all 4 counts

allocations (counting allocator: allocs + reallocs; bytes requested)
  parser                             sample line   all 100k lines   per line     B/line
  A  split.collect + split.collect         8 + 1  749960 + 100000       8.50      608.0
  B  split_whitespace.collect              1 + 1  100000 + 100000       2.00      192.0
  C  split_once scan                       0 + 0            0 + 0       0.00        0.0
  C2 split_ascii_whitespace                0 + 0            0 + 0       0.00        0.0
  D  Vec<char> + String/token             11 + 2 1100000 + 210364      13.10      586.1
  assert: C and C2 made 0 allocations over all 100,000 lines -- held

time per line (ns): 3 passes over all 100,000 lines
  parser                                 best        range of 3       vs C
  A  split.collect + split.collect      356.3    356.3 - 368.7        2.9x
  B  split_whitespace.collect           191.1    191.1 - 199.2        1.6x
  C  split_once scan                    122.1    122.1 - 127.9        1.0x
  C2 split_ascii_whitespace              89.0     89.0 - 89.5         0.7x
  D  Vec<char> + String/token           508.8    508.8 - 527.6        4.2x

88422 of 100000 lines contain a multi-byte char. split_whitespace() tests every char for Unicode whitespace on any input,
so unlike Go's Fields it has no ASCII fast path to lose: B, at 2.0 allocation events per line, took 0.54x the time of A at 8.5.
The two zero-allocation parsers differ in time only: C2 (a byte split) took 0.73x the time of C (split_once).
D, the Vec<char> parser, made 13.1 allocation events per line and was 4.2x slower than C, for no gain:
its counts are identical, because no delimiter byte can occur inside a multi-byte char.
```

> **Go vs Rust:** The `Pcg` port generates byte-for-byte the same 100,000 lines (6,725,341 bytes, 467,366 non-ASCII, the same level counts), and the zero is asserted by counting in both: C and C2 make 0 allocations over all lines. A makes exactly Go's 749,960 allocations plus 100,000 reallocs (`collect` starts the `Vec<&str>` at 4 and grows it once). B is 1 alloc + 1 realloc per line where Go's `Fields` counts the fields first. D makes 13.1 events per line against Go's 6.0, because a key `String` is always a heap block in Rust, while Go kept the key in a stack buffer. Rust's counts are exact and unrounded.
> In 3 runs each, Rust was slower than Go for C (~122-128 vs ~72 ns), A and D, and faster for B (~200 vs ~230 ns). The order also differs: Rust ran C2 < C < B < A < D, Go ran C < C2 < A < B < D. One plausible part of the gap, not measured: every parser does one map update per line, with SipHash in Rust and Go's faster hash.

**Complexity:** all parsers Θ(n) in bytes · allocations over 100,000 lines (exact, the same every run): A `Split`+`Split` 749,960 (1 + one per field, 7.5/line), B `Fields`+`Cut` 100,000 (1/line), C `IndexByte`+`Cut` and C2 `FieldsSeq`+`Cut` **0**, D `[]rune` 600,000 (the rune copy + one per `key=value` value; the keys stay in a stack buffer) · time per line: C ~72 ns, C2 ~1.9×, A ~2.9×, B ~3.2× (88% of lines push `Fields` onto its Unicode path), D ~5× · byte scanning is UTF-8-safe because ASCII delimiters can't match inside a multi-byte rune: 0 of 467,366 non-ASCII bytes did.

---

## 15. Graphemes: when a rune is not a character

`🔴 hard` · *Graphemes*

A rune is a code point, not a character: `"e\u0301"` is 2 runes, a flag is 2 regional indicators, `"👍🏽"` is a
thumb plus a skin tone, and the family `"👨‍👩‍👧"` is 5 runes and 18 bytes, and each of them is one thing on screen.
What a reader sees is a **grapheme cluster** (Unicode UAX #29), and the standard library does not segment them: there is
nothing for it in `strings`, `unicode` or `unicode/utf8` (checked in the Go source; the same holds for Rust's std). So
reversing or truncating by rune moves accents, drops skin tones, splits families and turns one country's flag into
another's. This example builds a small segmenter from the property tables Go *does* have (`unicode.Mn`, `unicode.Me`,
`unicode.Regional_Indicator`) and it is **explicitly partial**: four of the UAX #29 rules, plus three inputs where the
missing rules make it wrong. Segmentation is one Θ(n) pass that classifies each rune once. Normalization (NFC vs NFD) is
the other half of the story, and std does not do that either.

**Steps:**

1. Tabulate bytes, runes and a hand-typed "you see" count for eight strings, then reverse and truncate them by rune.
2. Write `clusters(s)`: one `range` pass that joins a rune to the current cluster under GB9 (Extend/ZWJ), GB11 (pictograph ZWJ pictograph) or GB12/13 (regional-indicator pairs), and breaks otherwise.
3. Check the cluster count against "you see" on every row, redo reverse/truncate by cluster, then read the three counter-examples and the NFC/NFD comparison.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
	"unicode"
	"unicode/utf8"
)

// A rune is a code point. What a reader calls "a character" is a grapheme
// cluster (Unicode UAX #29): one or more code points that display as one thing.
// Go's standard library does not segment clusters -- there is nothing for it in
// strings, unicode or unicode/utf8 (checked in $GOROOT/src). The unicode
// package does have the property tables a segmenter needs (Mn, Me,
// Regional_Indicator), so this program builds a small segmenter on top of them.
//
// It is DELIBERATELY PARTIAL. It implements four rules of UAX #29 and nothing else:
//   GB9      don't break before Extend or ZWJ
//            (Extend here = Mn, Me -- which includes U+FE0F -- and the skin
//            tones U+1F3FB..U+1F3FF)
//   GB11     don't break in pictograph ZWJ pictograph
//            ("pictograph" here = U+2600..U+27BF and U+1F300..U+1FAFF, a rough
//            stand-in for the Extended_Pictographic property, which Go lacks)
//   GB12/13  regional indicators pair up: two of them make one flag
//   GB999    break everywhere else
// Part 4 shows three inputs where the missing rules make it wrong.

type gcb int

const (
	other gcb = iota
	extend
	zwj
	regional
	pict
)

var classified int // how many times classOf ran: the work counter

func classOf(r rune) gcb {
	classified++
	switch {
	case r == 0x200D:
		return zwj
	case unicode.In(r, unicode.Mn, unicode.Me), r >= 0x1F3FB && r <= 0x1F3FF:
		return extend
	case unicode.Is(unicode.Regional_Indicator, r):
		return regional
	case r >= 0x2600 && r <= 0x27BF, r >= 0x1F300 && r <= 0x1FAFF:
		return pict
	}
	return other
}

// clusters splits s into grapheme clusters (by the subset above). One pass,
// each rune classified exactly once: Theta(n). The pieces are substrings of s,
// so the only allocation is the slice that holds them.
func clusters(s string) []string {
	var out []string
	start, ris := 0, 0 // ris: regional indicators in the current cluster
	var first, prev gcb
	for i, r := range s {
		c := classOf(r)
		join := c == extend || c == zwj || // GB9
			(prev == zwj && first == pict && c == pict) || // GB11
			(prev == regional && c == regional && ris%2 == 1) // GB12/13
		if i > 0 && !join { // GB999: a boundary before r
			out = append(out, s[start:i])
			start, ris = i, 0
		}
		if i == start {
			first = c
		}
		if c == regional {
			ris++
		}
		prev = c
	}
	if start < len(s) {
		out = append(out, s[start:])
	}
	return out
}

func reverseRunes(s string) string {
	r := []rune(s)
	slices.Reverse(r)
	return string(r)
}

func reverseClusters(s string) string {
	c := clusters(s)
	slices.Reverse(c)
	return strings.Join(c, "")
}

// truncRunes keeps the first k runes -- the usual "safe" UTF-8 truncation.
func truncRunes(s string, k int) string {
	for i := range s {
		if k == 0 {
			return s[:i]
		}
		k--
	}
	return s
}

// truncClusters keeps whole clusters while the total stays within k runes.
func truncClusters(s string, k int) string {
	end := 0
	for _, c := range clusters(s) {
		k -= utf8.RuneCountInString(c)
		if k < 0 {
			break
		}
		end += len(c)
	}
	return s[:end]
}

// q is %+q: ASCII-only quoting, so the code points are visible and columns line up.
func q(s string) string { return fmt.Sprintf("%+q", s) }

// letters spells regional indicators as the letters they stand for, paired by cluster.
func letters(s string) string {
	var parts []string
	for _, c := range clusters(s) {
		var b strings.Builder
		for _, r := range c {
			b.WriteRune('A' + r - 0x1F1E6)
		}
		parts = append(parts, b.String())
	}
	return strings.Join(parts, " ")
}

type row struct {
	name, s string
	see     int // what a reader sees -- data, typed in by hand, not computed
}

func main() {
	rows := []row{
		{"e + U+0301 (NFD)", "e\u0301", 1},
		{"U+00E9 (NFC)", "\u00e9", 1},
		{"flag FR", "\U0001F1EB\U0001F1F7", 1},
		{"flags SE FR", "\U0001F1F8\U0001F1EA\U0001F1EB\U0001F1F7", 2},
		{"thumbs + tone", "\U0001F44D\U0001F3FD", 1},
		{"family (ZWJ)", "\U0001F468\u200d\U0001F469\u200d\U0001F467", 1},
		{"keycap 1", "1\ufe0f\u20e3", 1},
		{"noel + U+0308 (NFD)", "noe\u0308l", 4},
	}

	fmt.Println("1. Three lengths ('you see' is data, typed in by hand):")
	fmt.Printf("   %-19s %5s %5s %8s   %s\n", "string", "bytes", "runes", "you see", "code points (%+q)")
	for _, r := range rows {
		fmt.Printf("   %-19s %5d %5d %8d   %-44s %s\n", r.name, len(r.s), utf8.RuneCountInString(r.s), r.see, q(r.s), r.s)
	}

	fmt.Println("\n2. Rune operations break them (keep the first k runes, k = half the runes rounded up):")
	fmt.Printf("   %-19s %-44s %s\n", "string", "reversed by rune", "first k runes")
	for _, r := range rows {
		k := (utf8.RuneCountInString(r.s) + 1) / 2
		fmt.Printf("   %-19s %-44s k=%d %s\n", r.name, q(reverseRunes(r.s)), k, q(truncRunes(r.s, k)))
	}

	fmt.Println("\n3. The partial segmenter (GB9, GB11, GB12/13, GB999):")
	fmt.Printf("   %-19s %8s %8s %-8s %-44s %s\n", "string", "clusters", "you see", "", "reversed by cluster", "whole clusters within k runes")
	totalRunes, matches := 0, 0
	classified = 0
	for _, r := range rows {
		n := len(clusters(r.s))
		totalRunes += utf8.RuneCountInString(r.s)
		mark := "MISMATCH"
		if n == r.see {
			mark, matches = "ok", matches+1
		}
		k := (utf8.RuneCountInString(r.s) + 1) / 2
		fmt.Printf("   %-19s %8d %8d %-8s %-44s %s\n", r.name, n, r.see, mark, q(reverseClusters(r.s)), q(truncClusters(r.s, k)))
	}
	fmt.Printf("   %d of %d rows match what a reader sees; a cluster bigger than k is dropped whole, never cut\n", matches, len(rows))
	se := rows[3].s
	fmt.Printf("   %s as regional letters: %s; by rune reversed %s (ES is Spain); by cluster %s\n",
		rows[3].name, letters(se), letters(reverseRunes(se)), letters(reverseClusters(se)))
	// Count the work of one segmentation pass over every row: one classOf per rune.
	classified = 0
	for _, r := range rows {
		clusters(r.s)
	}
	fmt.Printf("   one pass over all rows: %d runes, %d classifications -- Theta(n)\n", totalRunes, classified)

	fmt.Println("\n4. Where the subset is wrong (expected counts are data, from the UAX #29 rule named):")
	limits := []struct {
		name, s, rule string
		want          int
	}{
		{"CR LF", "\r\n", "GB3", 1},
		{"Hangul L+V", "\u1100\u1161", "GB6", 1},
		{"Devanagari ki", "\u0915\u093f", "GB9a", 1},
	}
	for _, l := range limits {
		fmt.Printf("   %-14s %-16s UAX #29 (%-4s) %d cluster, this subset %d\n", l.name, q(l.s), l.rule, l.want, len(clusters(l.s)))
	}

	fmt.Println("\n5. Equal-looking is not equal: normalization")
	for _, p := range [][2]string{{"\u00e9", "e\u0301"}, {"no\u00ebl", "noe\u0308l"}} {
		a, b := p[0], p[1]
		fmt.Printf("   %-12s vs %-14s  ==%-5v  bytes %d/%d  runes %d/%d  clusters %d/%d\n",
			q(a), q(b), a == b, len(a), len(b),
			utf8.RuneCountInString(a), utf8.RuneCountInString(b), len(clusters(a)), len(clusters(b)))
	}
	fmt.Println("   same clusters, different bytes: comparing user text needs NFC/NFD normalization first.")
	fmt.Println("   Neither standard library normalizes or segments: Go has golang.org/x/text/unicode/norm")
	fmt.Println("   and github.com/rivo/uniseg; Rust has the unicode-normalization and unicode-segmentation crates.")
}
```

**Output:**

```
1. Three lengths ('you see' is data, typed in by hand):
   string              bytes runes  you see   code points (%+q)
   e + U+0301 (NFD)        3     2        1   "e\u0301"                                    é
   U+00E9 (NFC)            2     1        1   "\u00e9"                                     é
   flag FR                 8     2        1   "\U0001f1eb\U0001f1f7"                       🇫🇷
   flags SE FR            16     4        2   "\U0001f1f8\U0001f1ea\U0001f1eb\U0001f1f7"   🇸🇪🇫🇷
   thumbs + tone           8     2        1   "\U0001f44d\U0001f3fd"                       👍🏽
   family (ZWJ)           18     5        1   "\U0001f468\u200d\U0001f469\u200d\U0001f467" 👨‍👩‍👧
   keycap 1                7     3        1   "1\ufe0f\u20e3"                              1️⃣
   noel + U+0308 (NFD)     6     5        4   "noe\u0308l"                                 noël

2. Rune operations break them (keep the first k runes, k = half the runes rounded up):
   string              reversed by rune                             first k runes
   e + U+0301 (NFD)    "\u0301e"                                    k=1 "e"
   U+00E9 (NFC)        "\u00e9"                                     k=1 "\u00e9"
   flag FR             "\U0001f1f7\U0001f1eb"                       k=1 "\U0001f1eb"
   flags SE FR         "\U0001f1f7\U0001f1eb\U0001f1ea\U0001f1f8"   k=2 "\U0001f1f8\U0001f1ea"
   thumbs + tone       "\U0001f3fd\U0001f44d"                       k=1 "\U0001f44d"
   family (ZWJ)        "\U0001f467\u200d\U0001f469\u200d\U0001f468" k=3 "\U0001f468\u200d\U0001f469"
   keycap 1            "\u20e3\ufe0f1"                              k=2 "1\ufe0f"
   noel + U+0308 (NFD) "l\u0308eon"                                 k=3 "noe"

3. The partial segmenter (GB9, GB11, GB12/13, GB999):
   string              clusters  you see          reversed by cluster                          whole clusters within k runes
   e + U+0301 (NFD)           1        1 ok       "e\u0301"                                    ""
   U+00E9 (NFC)               1        1 ok       "\u00e9"                                     "\u00e9"
   flag FR                    1        1 ok       "\U0001f1eb\U0001f1f7"                       ""
   flags SE FR                2        2 ok       "\U0001f1eb\U0001f1f7\U0001f1f8\U0001f1ea"   "\U0001f1f8\U0001f1ea"
   thumbs + tone              1        1 ok       "\U0001f44d\U0001f3fd"                       ""
   family (ZWJ)               1        1 ok       "\U0001f468\u200d\U0001f469\u200d\U0001f467" ""
   keycap 1                   1        1 ok       "1\ufe0f\u20e3"                              ""
   noel + U+0308 (NFD)        4        4 ok       "le\u0308on"                                 "no"
   8 of 8 rows match what a reader sees; a cluster bigger than k is dropped whole, never cut
   flags SE FR as regional letters: SE FR; by rune reversed RF ES (ES is Spain); by cluster FR SE
   one pass over all rows: 24 runes, 24 classifications -- Theta(n)

4. Where the subset is wrong (expected counts are data, from the UAX #29 rule named):
   CR LF          "\r\n"           UAX #29 (GB3 ) 1 cluster, this subset 2
   Hangul L+V     "\u1100\u1161"   UAX #29 (GB6 ) 1 cluster, this subset 2
   Devanagari ki  "\u0915\u093f"   UAX #29 (GB9a) 1 cluster, this subset 2

5. Equal-looking is not equal: normalization
   "\u00e9"     vs "e\u0301"       ==false  bytes 2/3  runes 1/2  clusters 1/1
   "no\u00ebl"  vs "noe\u0308l"    ==false  bytes 5/6  runes 4/5  clusters 4/4
   same clusters, different bytes: comparing user text needs NFC/NFD normalization first.
   Neither standard library normalizes or segments: Go has golang.org/x/text/unicode/norm
   and github.com/rivo/uniseg; Rust has the unicode-normalization and unicode-segmentation crates.
```

**Rust:**

```rust
// A char is a Unicode scalar value. What a reader calls "a character" is a
// grapheme cluster (Unicode UAX #29). Rust's std does not segment clusters; the
// doc comment on `str::chars` says so outright: "Iteration over grapheme
// clusters may be what you actually want. That functionality is not provided by
// Rust's standard library" (alloc/src/string.rs, read in the source).
//
// Unlike Go's `unicode` package, std also exposes no property tables: there is
// no stable `is_mark()` / general-category lookup. core does carry a private
// Grapheme_Extend table (`char::is_grapheme_extender`, not `pub`) -- it is what
// makes `{:?}` escape combining marks (part 5 shows it). So this twin hard-codes
// the code point ranges its test strings need.
//
// It is DELIBERATELY PARTIAL -- the same four rules as the Go program:
//   GB9      don't break before Extend or ZWJ
//            (Extend here = U+0300..036F combining diacritics, U+20D0..20FF
//            marks for symbols, U+FE00..FE0F variation selectors, and the skin
//            tones U+1F3FB..1F3FF -- a subset of Go's unicode.Mn + unicode.Me)
//   GB11     don't break in pictograph ZWJ pictograph
//            ("pictograph" = U+2600..27BF and U+1F300..1FAFF, a rough stand-in
//            for Extended_Pictographic)
//   GB12/13  regional indicators pair up: two of them make one flag
//   GB999    break everywhere else
// Part 4 shows three inputs where the missing rules make it wrong.

#[derive(Clone, Copy, PartialEq)]
enum Gcb {
    Other,
    Extend,
    Zwj,
    Regional,
    Pict,
}

// No mutable globals in safe Rust: the work counter is threaded through as `&mut u64`.
fn class_of(c: char, classified: &mut u64) -> Gcb {
    *classified += 1;
    match c as u32 {
        0x200D => Gcb::Zwj,
        0x0300..=0x036F | 0x20D0..=0x20FF | 0xFE00..=0xFE0F | 0x1F3FB..=0x1F3FF => Gcb::Extend,
        0x1F1E6..=0x1F1FF => Gcb::Regional,
        0x2600..=0x27BF | 0x1F300..=0x1FAFF => Gcb::Pict,
        _ => Gcb::Other,
    }
}

/// Splits `s` into grapheme clusters (by the subset above). One pass, each char
/// classified exactly once: Theta(n). The pieces are `&str` views into `s`, cut at
/// `char_indices` offsets, so every slice is on a char boundary by construction.
fn clusters<'a>(s: &'a str, classified: &mut u64) -> Vec<&'a str> {
    let mut out = Vec::new();
    let (mut start, mut ris) = (0, 0); // ris: regional indicators in the current cluster
    let (mut first, mut prev) = (Gcb::Other, Gcb::Other);
    for (i, ch) in s.char_indices() {
        let c = class_of(ch, classified);
        let join = c == Gcb::Extend
            || c == Gcb::Zwj // GB9
            || (prev == Gcb::Zwj && first == Gcb::Pict && c == Gcb::Pict) // GB11
            || (prev == Gcb::Regional && c == Gcb::Regional && ris % 2 == 1); // GB12/13
        if i > 0 && !join {
            // GB999: a boundary before ch
            out.push(&s[start..i]);
            start = i;
            ris = 0;
        }
        if i == start {
            first = c;
        }
        if c == Gcb::Regional {
            ris += 1;
        }
        prev = c;
    }
    if start < s.len() {
        out.push(&s[start..]);
    }
    out
}

/// Cluster segmentation without the work counter.
fn segs(s: &str) -> Vec<&str> {
    clusters(s, &mut 0)
}

fn reverse_runes(s: &str) -> String {
    s.chars().rev().collect()
}

fn reverse_clusters(s: &str) -> String {
    segs(s).into_iter().rev().collect()
}

/// Keeps the first k chars -- the usual "safe" UTF-8 truncation.
fn trunc_runes(s: &str, k: usize) -> &str {
    match s.char_indices().nth(k) {
        Some((i, _)) => &s[..i],
        None => s,
    }
}

/// Keeps whole clusters while the total stays within k chars.
fn trunc_clusters(s: &str, k: usize) -> &str {
    let (mut end, mut used) = (0, 0);
    for c in segs(s) {
        used += c.chars().count();
        if used > k {
            break;
        }
        end += c.len();
    }
    &s[..end]
}

/// Go's `%+q`: ASCII-only quoting, so the code points are visible and columns line up.
fn q(s: &str) -> String {
    let mut out = String::from("\"");
    for c in s.chars() {
        match c {
            '\x07' => out.push_str("\\a"),
            '\x08' => out.push_str("\\b"),
            '\x0c' => out.push_str("\\f"),
            '\n' => out.push_str("\\n"),
            '\r' => out.push_str("\\r"),
            '\t' => out.push_str("\\t"),
            '\x0b' => out.push_str("\\v"),
            '"' | '\\' => {
                out.push('\\');
                out.push(c);
            }
            ' '..='~' => out.push(c),
            c if (c as u32) < 0x20 || c == '\x7f' => out.push_str(&format!("\\x{:02x}", c as u32)),
            c if (c as u32) < 0x10000 => out.push_str(&format!("\\u{:04x}", c as u32)),
            c => out.push_str(&format!("\\U{:08x}", c as u32)),
        }
    }
    out.push('"');
    out
}

/// Spells regional indicators as the letters they stand for, paired by cluster.
fn letters(s: &str) -> String {
    segs(s)
        .iter()
        .map(|c| {
            c.chars()
                .map(|r| char::from(b'A' + (r as u32 - 0x1F1E6) as u8))
                .collect::<String>()
        })
        .collect::<Vec<_>>()
        .join(" ")
}

struct Row {
    name: &'static str,
    s: &'static str,
    see: usize, // what a reader sees -- data, typed in by hand, not computed
}

fn main() {
    let rows = [
        Row {
            name: "e + U+0301 (NFD)",
            s: "e\u{301}",
            see: 1,
        },
        Row {
            name: "U+00E9 (NFC)",
            s: "\u{e9}",
            see: 1,
        },
        Row {
            name: "flag FR",
            s: "\u{1F1EB}\u{1F1F7}",
            see: 1,
        },
        Row {
            name: "flags SE FR",
            s: "\u{1F1F8}\u{1F1EA}\u{1F1EB}\u{1F1F7}",
            see: 2,
        },
        Row {
            name: "thumbs + tone",
            s: "\u{1F44D}\u{1F3FD}",
            see: 1,
        },
        Row {
            name: "family (ZWJ)",
            s: "\u{1F468}\u{200d}\u{1F469}\u{200d}\u{1F467}",
            see: 1,
        },
        Row {
            name: "keycap 1",
            s: "1\u{fe0f}\u{20e3}",
            see: 1,
        },
        Row {
            name: "noel + U+0308 (NFD)",
            s: "noe\u{308}l",
            see: 4,
        },
    ];
    let half = |s: &str| s.chars().count().div_ceil(2);

    println!("1. Three lengths ('you see' is data, typed in by hand):");
    println!(
        "   {:<19} {:>5} {:>5} {:>8}   code points (%+q)",
        "string", "bytes", "runes", "you see"
    );
    for r in &rows {
        let n = r.s.chars().count();
        println!(
            "   {:<19} {:>5} {n:>5} {:>8}   {:<44} {}",
            r.name,
            r.s.len(),
            r.see,
            q(r.s),
            r.s
        );
    }

    println!(
        "\n2. Rune operations break them (keep the first k runes, k = half the runes rounded up):"
    );
    println!(
        "   {:<19} {:<44} first k runes",
        "string", "reversed by rune"
    );
    for r in &rows {
        let k = half(r.s);
        println!(
            "   {:<19} {:<44} k={k} {}",
            r.name,
            q(&reverse_runes(r.s)),
            q(trunc_runes(r.s, k))
        );
    }

    println!("\n3. The partial segmenter (GB9, GB11, GB12/13, GB999):");
    println!(
        "   {:<19} {:>8} {:>8} {:<8} {:<44} whole clusters within k runes",
        "string", "clusters", "you see", "", "reversed by cluster"
    );
    let mut matches = 0;
    for r in &rows {
        let n = segs(r.s).len();
        let mark = if n == r.see {
            matches += 1;
            "ok"
        } else {
            "MISMATCH"
        };
        let k = half(r.s);
        println!(
            "   {:<19} {n:>8} {:>8} {mark:<8} {:<44} {}",
            r.name,
            r.see,
            q(&reverse_clusters(r.s)),
            q(trunc_clusters(r.s, k))
        );
    }
    println!(
        "   {matches} of {} rows match what a reader sees; a cluster bigger than k is dropped whole, never cut",
        rows.len()
    );
    let se = rows[3].s;
    println!(
        "   {} as regional letters: {}; by rune reversed {} (ES is Spain); by cluster {}",
        rows[3].name,
        letters(se),
        letters(&reverse_runes(se)),
        letters(&reverse_clusters(se))
    );
    // Count the work of one segmentation pass over every row: one class_of per char.
    let mut classified = 0;
    let mut total_runes = 0;
    for r in &rows {
        total_runes += r.s.chars().count();
        clusters(r.s, &mut classified);
    }
    println!(
        "   one pass over all rows: {total_runes} runes, {classified} classifications -- Theta(n)"
    );

    println!(
        "\n4. Where the subset is wrong (expected counts are data, from the UAX #29 rule named):"
    );
    let limits = [
        ("CR LF", "\r\n", "GB3", 1),
        ("Hangul L+V", "\u{1100}\u{1161}", "GB6", 1),
        ("Devanagari ki", "\u{915}\u{93f}", "GB9a", 1),
    ];
    for (name, s, rule, want) in limits {
        println!(
            "   {name:<14} {:<16} UAX #29 ({rule:<4}) {want} cluster, this subset {}",
            q(s),
            segs(s).len()
        );
    }

    println!("\n5. Equal-looking is not equal: normalization");
    for (a, b) in [("\u{e9}", "e\u{301}"), ("no\u{eb}l", "noe\u{308}l")] {
        println!(
            "   {:<12} vs {:<14}  =={:<5}  bytes {}/{}  runes {}/{}  clusters {}/{}",
            q(a),
            q(b),
            a == b,
            a.len(),
            b.len(),
            a.chars().count(),
            b.chars().count(),
            segs(a).len(),
            segs(b).len()
        );
    }
    println!(
        "   same clusters, different bytes: comparing user text needs NFC/NFD normalization first."
    );
    println!(
        "   Neither standard library normalizes or segments: Go has golang.org/x/text/unicode/norm"
    );
    println!(
        "   and github.com/rivo/uniseg; Rust has the unicode-normalization and unicode-segmentation crates."
    );
    // Rust-only line: std knows Grapheme_Extend privately -- {:?} escapes the mark only.
    println!(
        "   (Rust only) {{:?}} prints {:?} for the NFD string but {:?} for NFC: core's private Grapheme_Extend table",
        "e\u{301}", "\u{e9}"
    );
}
```

**Rust output:**

```
1. Three lengths ('you see' is data, typed in by hand):
   string              bytes runes  you see   code points (%+q)
   e + U+0301 (NFD)        3     2        1   "e\u0301"                                    é
   U+00E9 (NFC)            2     1        1   "\u00e9"                                     é
   flag FR                 8     2        1   "\U0001f1eb\U0001f1f7"                       🇫🇷
   flags SE FR            16     4        2   "\U0001f1f8\U0001f1ea\U0001f1eb\U0001f1f7"   🇸🇪🇫🇷
   thumbs + tone           8     2        1   "\U0001f44d\U0001f3fd"                       👍🏽
   family (ZWJ)           18     5        1   "\U0001f468\u200d\U0001f469\u200d\U0001f467" 👨‍👩‍👧
   keycap 1                7     3        1   "1\ufe0f\u20e3"                              1️⃣
   noel + U+0308 (NFD)     6     5        4   "noe\u0308l"                                 noël

2. Rune operations break them (keep the first k runes, k = half the runes rounded up):
   string              reversed by rune                             first k runes
   e + U+0301 (NFD)    "\u0301e"                                    k=1 "e"
   U+00E9 (NFC)        "\u00e9"                                     k=1 "\u00e9"
   flag FR             "\U0001f1f7\U0001f1eb"                       k=1 "\U0001f1eb"
   flags SE FR         "\U0001f1f7\U0001f1eb\U0001f1ea\U0001f1f8"   k=2 "\U0001f1f8\U0001f1ea"
   thumbs + tone       "\U0001f3fd\U0001f44d"                       k=1 "\U0001f44d"
   family (ZWJ)        "\U0001f467\u200d\U0001f469\u200d\U0001f468" k=3 "\U0001f468\u200d\U0001f469"
   keycap 1            "\u20e3\ufe0f1"                              k=2 "1\ufe0f"
   noel + U+0308 (NFD) "l\u0308eon"                                 k=3 "noe"

3. The partial segmenter (GB9, GB11, GB12/13, GB999):
   string              clusters  you see          reversed by cluster                          whole clusters within k runes
   e + U+0301 (NFD)           1        1 ok       "e\u0301"                                    ""
   U+00E9 (NFC)               1        1 ok       "\u00e9"                                     "\u00e9"
   flag FR                    1        1 ok       "\U0001f1eb\U0001f1f7"                       ""
   flags SE FR                2        2 ok       "\U0001f1eb\U0001f1f7\U0001f1f8\U0001f1ea"   "\U0001f1f8\U0001f1ea"
   thumbs + tone              1        1 ok       "\U0001f44d\U0001f3fd"                       ""
   family (ZWJ)               1        1 ok       "\U0001f468\u200d\U0001f469\u200d\U0001f467" ""
   keycap 1                   1        1 ok       "1\ufe0f\u20e3"                              ""
   noel + U+0308 (NFD)        4        4 ok       "le\u0308on"                                 "no"
   8 of 8 rows match what a reader sees; a cluster bigger than k is dropped whole, never cut
   flags SE FR as regional letters: SE FR; by rune reversed RF ES (ES is Spain); by cluster FR SE
   one pass over all rows: 24 runes, 24 classifications -- Theta(n)

4. Where the subset is wrong (expected counts are data, from the UAX #29 rule named):
   CR LF          "\r\n"           UAX #29 (GB3 ) 1 cluster, this subset 2
   Hangul L+V     "\u1100\u1161"   UAX #29 (GB6 ) 1 cluster, this subset 2
   Devanagari ki  "\u0915\u093f"   UAX #29 (GB9a) 1 cluster, this subset 2

5. Equal-looking is not equal: normalization
   "\u00e9"     vs "e\u0301"       ==false  bytes 2/3  runes 1/2  clusters 1/1
   "no\u00ebl"  vs "noe\u0308l"    ==false  bytes 5/6  runes 4/5  clusters 4/4
   same clusters, different bytes: comparing user text needs NFC/NFD normalization first.
   Neither standard library normalizes or segments: Go has golang.org/x/text/unicode/norm
   and github.com/rivo/uniseg; Rust has the unicode-normalization and unicode-segmentation crates.
   (Rust only) {:?} prints "e\u{301}" for the NFD string but "é" for NFC: core's private Grapheme_Extend table
```

> **Go vs Rust:** Parts 1–5 are the Go output byte for byte; the twin adds one Rust-only line. Rust has no `unicode.Mn`/`Me`/`Regional_Indicator` (std exposes no general-category or property lookup), so `class_of` hard-codes the ranges its test strings need — a narrower set than Go's tables, which agree with it on every row here. core does hold a private Grapheme_Extend table (`char::is_grapheme_extender`, not `pub`, read in the source): it is why `{:?}` escapes the combining mark, as the last line shows. The pieces are `&str` cut at `char_indices` offsets, so no slice can land off a char boundary; real segmentation and normalization are the `unicode-segmentation` and `unicode-normalization` crates (not used: std only).

**Complexity:** segmentation Θ(n): one classification per rune (24 runes, 24 classifications over the test set) · clusters ≤ runes ≤ bytes (family: 1 ≤ 5 ≤ 18) · reverse and truncate must work on clusters, not runes · `==` needs normalized input (`"\u00e9"` vs `"e\u0301"`: same 1 cluster, 2 vs 3 bytes, unequal) · Go's and Rust's std do neither, and this four-rule subset gets CR LF, Hangul jamo and Devanagari spacing marks wrong

---

## 16. Capstone: the string problem set, proved and measured

`🔴 hard` · *Capstone*

The lesson's five problems as one small package: `IsAnagram`, `IsPalindrome`, `ReverseWords`, `Compress`/`Decompress`
and `LongestUnique`. Each one is rune-correct for any valid UTF-8, and each has an ASCII fast path, which is honest
only because one `isASCII` byte scan proves that every byte is a whole character. Then a `[128]int` can stand in for a
map, and a byte can stand in for a decoded rune. Correctness is not argued, it is **checked**: 100,000 PCG-seeded random
cases over a mixed alphabet (ASCII, `é`/`É`, `€`, `😀`) are compared against brute-force oracles written the obvious
way on `[]rune` (sort, reverse, `Fields`+`Join`, naive run counting, the Θ(n²) every-start scan), on both paths, plus
three properties. Then each path is measured on 1 MiB inputs. All five are Θ(n), and the table shows what the constant
costs: a map instead of an array, a decode instead of a byte, and an oracle's extra allocations.

**Steps:**

1. Write each function twice (ASCII fast path, rune path) behind an `isASCII` dispatcher, and an oracle for each on `[]rune`.
2. Generate 100,000 cases with `rand.NewPCG(1, 2)`: runs of repeated runes for `Compress`, shuffled partners for `IsAnagram`, mirrored halves with case flips for `IsPalindrome`. Count mismatches per path and check the properties.
3. Time fast path, rune path and oracle on a 1 MiB ASCII and a 1 MiB mixed input, read the computed ratios and allocation counts, then compare the brute force's step count on few-letter text and on text with no repeats.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strconv"
	"strings"
	"testing"
	"time"
	"unicode"
	"unicode/utf8"
)

// ===== the package: five problems, each rune-correct, with an ASCII fast path =====
// The fast path is honest only when the input is all ASCII: then a byte IS a
// character, and a [128]int array can replace a map. Every exported function
// checks isASCII (one Theta(n) byte scan, no decoding) and picks a path.

func isASCII(s string) bool {
	for i := 0; i < len(s); i++ {
		if s[i] >= utf8.RuneSelf {
			return false
		}
	}
	return true
}

// IsAnagram: same multiset of runes. Equal multisets have equal UTF-8 length,
// so different byte lengths are a Theta(1) "no".
func IsAnagram(a, b string) bool {
	if isASCII(a) && isASCII(b) {
		return anagramASCII(a, b)
	}
	return anagramRunes(a, b)
}

func anagramASCII(a, b string) bool {
	if len(a) != len(b) {
		return false
	}
	var count [128]int
	for i := 0; i < len(a); i++ {
		count[a[i]]++
		count[b[i]]--
	}
	return count == [128]int{}
}

func anagramRunes(a, b string) bool {
	if len(a) != len(b) {
		return false
	}
	count := map[rune]int{}
	for _, r := range a {
		count[r]++
	}
	for _, r := range b {
		count[r]--
		if count[r] < 0 { // b has a rune a lacks; with equal byte lengths, that is the only way to differ
			return false
		}
	}
	return true
}

// IsPalindrome: letters and digits only, case folded, two pointers from both ends.
func IsPalindrome(s string) bool {
	if isASCII(s) {
		return palindromeASCII(s)
	}
	return palindromeRunes(s)
}

func alnum(c byte) bool { return 'a' <= c|0x20 && c|0x20 <= 'z' || '0' <= c && c <= '9' }

func palindromeASCII(s string) bool {
	for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
		for i < j && !alnum(s[i]) {
			i++
		}
		for i < j && !alnum(s[j]) {
			j--
		}
		if s[i]|0x20 != s[j]|0x20 { // |0x20 lowercases a letter and leaves a digit alone
			return false
		}
	}
	return true
}

func palindromeRunes(s string) bool {
	keep := func(r rune) bool { return unicode.IsLetter(r) || unicode.IsDigit(r) }
	for i, j := 0, len(s); i < j; { // decode forward at i and backward at j: no []rune
		r, w := utf8.DecodeRuneInString(s[i:])
		if !keep(r) {
			i += w
			continue
		}
		q, v := utf8.DecodeLastRuneInString(s[:j])
		if !keep(q) {
			j -= v
			continue
		}
		if unicode.ToLower(r) != unicode.ToLower(q) {
			return false
		}
		i, j = i+w, j-v
	}
	return true
}

// ReverseWords: words in reverse order, single spaces, no padding. Scan from the
// end and copy each word once into a pre-grown Builder: 1 allocation.
func ReverseWords(s string) string { return reverseWords(s, isASCII(s)) }

func reverseWords(s string, ascii bool) string {
	// lastSpace: is the character that ends at s[j] a space, and how wide is it?
	// The ASCII set (\t \n \v \f \r ' ') is exactly what strings.Fields uses on ASCII.
	lastSpace := func(j int) (bool, int) {
		if ascii {
			c := s[j-1]
			return c == ' ' || '\t' <= c && c <= '\r', 1
		}
		r, w := utf8.DecodeLastRuneInString(s[:j])
		return unicode.IsSpace(r), w
	}
	var b strings.Builder
	b.Grow(len(s))
	for j := len(s); j > 0; {
		if sp, w := lastSpace(j); sp {
			j -= w
			continue
		}
		i := j
		for i > 0 {
			sp, w := lastSpace(i)
			if sp {
				break
			}
			i -= w
		}
		if b.Len() > 0 {
			b.WriteByte(' ')
		}
		b.WriteString(s[i:j])
		j = i
	}
	return b.String()
}

// Compress: run-length "a2b1c5a3" over runes; the original when not shorter.
func Compress(s string) string { return compress(s, isASCII(s)) }

func compress(s string, ascii bool) string {
	if s == "" {
		return s
	}
	var b strings.Builder
	b.Grow(len(s)) // the one allocation: anything as long as s is thrown away
	var num [20]byte
	next := func(i int) (rune, int) {
		if ascii {
			return rune(s[i]), 1 // fast path: no decoding
		}
		// The rune path costs little extra on ASCII: DecodeRuneInString starts with
		// an inlined "s[0] < RuneSelf" check (unicode/utf8, read in the source).
		return utf8.DecodeRuneInString(s[i:])
	}
	for i := 0; i < len(s); {
		r, w := next(i)
		j, n := i+w, 1
		for j < len(s) {
			r2, w2 := next(j)
			if r2 != r {
				break
			}
			j, n = j+w2, n+1
		}
		digits := strconv.AppendInt(num[:0], int64(n), 10)
		if b.Len()+w+len(digits) >= len(s) {
			return s // not shorter
		}
		b.WriteString(s[i : i+w])
		b.Write(digits)
		i = j
	}
	return b.String()
}

// Decompress: a rune, then an optional count (absent = 1, so an uncompressed
// digit-free string decodes to itself). Pass 1 sizes the output, pass 2 fills it.
func Decompress(e string) string {
	var b strings.Builder
	for pass := range 2 {
		total := 0
		for i := 0; i < len(e); {
			_, w := utf8.DecodeRuneInString(e[i:])
			j, n := i+w, 0
			for ; j < len(e) && '0' <= e[j] && e[j] <= '9'; j++ {
				n = n*10 + int(e[j]-'0')
			}
			if j == i+w {
				n = 1
			}
			if total += w * n; pass == 1 {
				for range n {
					b.WriteString(e[i : i+w])
				}
			}
			i = j
		}
		if pass == 0 {
			b.Grow(total)
		}
	}
	return b.String()
}

// LongestUnique: length in runes of the longest substring without a repeated rune.
func LongestUnique(s string) int {
	if isASCII(s) {
		return longestASCII(s)
	}
	return longestRunes(s)
}

func longestASCII(s string) int {
	var last [128]int // last[c] = 1 + index of c's latest occurrence, 0 = unseen
	best, left := 0, 0
	for i := 0; i < len(s); i++ {
		left = max(left, last[s[i]]) // never let left move backwards ("abba")
		last[s[i]] = i + 1
		best = max(best, i+1-left)
	}
	return best
}

func longestRunes(s string) int {
	last := map[rune]int{}
	best, left, i := 0, 0, 0 // i counts runes, not bytes
	for _, r := range s {
		left = max(left, last[r])
		last[r] = i + 1
		best = max(best, i+1-left)
		i++
	}
	return best
}

// ===== the oracles: the obvious way, on []rune =====

func anagramOracle(a, b string) bool {
	x, y := []rune(a), []rune(b)
	slices.Sort(x)
	slices.Sort(y)
	return slices.Equal(x, y)
}

func palindromeOracle(s string) bool {
	var f []rune
	for _, r := range s {
		if unicode.IsLetter(r) || unicode.IsDigit(r) {
			f = append(f, unicode.ToLower(r))
		}
	}
	g := slices.Clone(f)
	slices.Reverse(g)
	return slices.Equal(f, g)
}

func reverseWordsOracle(s string) string {
	w := strings.Fields(s)
	slices.Reverse(w)
	return strings.Join(w, " ")
}

func compressOracle(s string) string {
	r := []rune(s)
	var b strings.Builder
	for i := 0; i < len(r); {
		j := i
		for j < len(r) && r[j] == r[i] {
			j++
		}
		// Fprintf boxes both values into interfaces. Values < 256 come from the
		// runtime's staticuint64s table for free; a rune >= 256 costs an allocation
		// (convT32 in runtime/iface.go, read in the source).
		fmt.Fprintf(&b, "%c%d", r[i], j-i)
		i = j
	}
	if b.Len() >= len(s) {
		return s
	}
	return b.String()
}

var bruteSteps int // inner-loop steps of the brute force

func longestOracle(s string) int {
	r, best := []rune(s), 0
	seen := map[rune]bool{}
	for i := range r { // every start, extended until the first repeat: Theta(n^2) worst case
		clear(seen)
		j := i
		for ; j < len(r) && !seen[r[j]]; j++ {
			seen[r[j]] = true
			bruteSteps++
		}
		best = max(best, j-i)
	}
	return best
}

// ===== proof and measurement =====

var (
	sinkB bool
	sinkS string
	sinkI int
)

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	mixed, ascii := []rune("abcA éÉ€😀"), []rune("abcA ")
	names := []string{"IsAnagram", "IsPalindrome", "ReverseWords", "Compress", "LongestUnique"}
	var bad [5][2]int // mismatches vs oracle: [function][0 fast path on ASCII cases, 1 rune path on all]
	var yes [5]int
	const cases = 100_000
	asciiCases, roundTrip, involution, valid := 0, 0, 0, 0
	for range cases {
		alpha := mixed
		if rng.IntN(2) == 0 {
			alpha = ascii
		}
		n, runny := rng.IntN(41), rng.IntN(2) == 0
		rs := make([]rune, 0, n)
		for len(rs) < n {
			c, k := alpha[rng.IntN(len(alpha))], 1
			if runny {
				k = 1 + rng.IntN(5)
			}
			for ; k > 0 && len(rs) < n; k-- {
				rs = append(rs, c)
			}
		}
		s := string(rs)
		bs := slices.Clone(rs) // anagram partner: a shuffle, a shuffle with one change, or random
		for i := len(bs) - 1; i > 0; i-- {
			j := rng.IntN(i + 1)
			bs[i], bs[j] = bs[j], bs[i]
		}
		switch rng.IntN(3) {
		case 1:
			if len(bs) > 0 {
				bs[rng.IntN(len(bs))] = alpha[rng.IntN(len(alpha))]
			}
		case 2:
			for i := range bs {
				bs[i] = alpha[rng.IntN(len(alpha))]
			}
		}
		b := string(bs)
		p := rs // palindrome candidate: s itself, or its first half mirrored with random case flips
		if rng.IntN(2) == 0 {
			p = slices.Clone(rs[:n-n/2])
			for i := n/2 - 1; i >= 0; i-- {
				r := rs[i]
				if rng.IntN(4) == 0 && unicode.IsLower(r) {
					r = unicode.ToUpper(r) // still a palindrome: the check folds case
				}
				p = append(p, r)
			}
		}
		ps := string(p)

		isA := isASCII(s)
		cmp := func(f int, fast, slow, want any) {
			if isA && fast != want {
				bad[f][0]++
			}
			if slow != want {
				bad[f][1]++
			}
		}
		if isA {
			asciiCases++
		}
		wantA := anagramOracle(s, b)
		cmp(0, IsAnagram(s, b), anagramRunes(s, b), wantA)
		wantP := palindromeOracle(ps)
		cmp(1, IsPalindrome(ps), palindromeRunes(ps), wantP)
		rw := ReverseWords(s)
		cmp(2, rw, reverseWords(s, false), reverseWordsOracle(s))
		c := Compress(s)
		cmp(3, c, compress(s, false), compressOracle(s))
		cmp(4, LongestUnique(s), longestRunes(s), longestOracle(s))
		if wantA {
			yes[0]++
		}
		if wantP {
			yes[1]++
		}
		if c != s {
			yes[3]++
		}
		if Decompress(c) == s {
			roundTrip++
		}
		if ReverseWords(rw) == strings.Join(strings.Fields(s), " ") {
			involution++
		}
		if utf8.ValidString(rw) && utf8.ValidString(c) && utf8.ValidString(Decompress(c)) {
			valid++
		}
	}
	fmt.Printf("Proof: %d random cases (PCG 1,2), 0-40 runes over %q or %q; %d all-ASCII\n", cases, string(mixed), string(ascii), asciiCases)
	fmt.Printf("  %-14s %12s %12s %10s\n", "function", "fast path", "rune path", "oracle yes")
	for f, name := range names {
		yesCol := "-"
		if f != 2 && f != 4 {
			yesCol = strconv.Itoa(yes[f])
		}
		fmt.Printf("  %-14s %12d %12d %10s\n", name, bad[f][0], bad[f][1], yesCol)
	}
	fmt.Printf("  (mismatches vs oracle: fast path on the %d ASCII cases, rune path on all; Compress 'yes' = got shorter)\n", asciiCases)
	fmt.Printf("  Decompress(Compress(s)) == s:                          %d / %d\n", roundTrip, cases)
	fmt.Printf("  ReverseWords(ReverseWords(s)) == Join(Fields(s), \" \"): %d / %d\n", involution, cases)
	fmt.Printf("  outputs that are valid UTF-8:                          %d / %d\n", valid, cases)

	// --- measurement: 1 MiB inputs, each path timed separately ---
	text := func(alpha []rune, bytes int, runny bool) string {
		var b strings.Builder
		for b.Len() < bytes {
			c, k := alpha[rng.IntN(len(alpha))], 1
			if runny {
				k = 1 + rng.IntN(9)
			}
			for range k {
				b.WriteRune(c)
			}
		}
		return b.String()
	}
	revRunes := func(s string) string { r := []rune(s); slices.Reverse(r); return string(r) }
	type input struct{ pal, rot, runs string }
	mk := func(alpha []rune) input {
		half := text(alpha, 1<<19, false)
		return input{half + revRunes(half), revRunes(half) + half, text(alpha, 1<<20, true)}
	}
	ins := [2]input{mk([]rune("abcxyz ")), mk([]rune("abc é€😀"))}
	paths := [5][3]func(in input){ // fast, rune, oracle
		{func(in input) { sinkB = anagramASCII(in.pal, in.rot) }, func(in input) { sinkB = anagramRunes(in.pal, in.rot) }, func(in input) { sinkB = anagramOracle(in.pal, in.rot) }},
		{func(in input) { sinkB = palindromeASCII(in.pal) }, func(in input) { sinkB = palindromeRunes(in.pal) }, func(in input) { sinkB = palindromeOracle(in.pal) }},
		{func(in input) { sinkS = reverseWords(in.pal, true) }, func(in input) { sinkS = reverseWords(in.pal, false) }, func(in input) { sinkS = reverseWordsOracle(in.pal) }},
		{func(in input) { sinkS = compress(in.runs, true) }, func(in input) { sinkS = compress(in.runs, false) }, func(in input) { sinkS = compressOracle(in.runs) }},
		{func(in input) { sinkI = longestASCII(in.pal) }, func(in input) { sinkI = longestRunes(in.pal) }, func(in input) { sinkI = longestOracle(in.pal) }},
	}
	measure := func(f func()) (float64, float64) {
		allocs := testing.AllocsPerRun(1, f)
		best := time.Hour
		for range 5 {
			t := time.Now()
			f()
			best = min(best, time.Since(t))
		}
		return float64(best.Nanoseconds()) / 1e6, allocs
	}
	fmt.Printf("\nMeasure: 1 MiB inputs (ASCII %q, mixed %q); ms/op = best of 5, allocs/op from AllocsPerRun\n", "abcxyz ", "abc é€😀")
	fmt.Printf("  %-14s %-7s %9s %8s %9s %8s\n", "function", "path", "ASCII ms", "allocs", "mixed ms", "allocs")
	var ms [5][3][2]float64
	for f, name := range names {
		for p, label := range []string{"fast", "rune", "oracle"} {
			a, aa := measure(func() { paths[f][p](ins[0]) })
			ms[f][p][0] = a
			mixedCol := fmt.Sprintf("%9s %8s", "n/a", "")
			if p > 0 {
				m, ma := measure(func() { paths[f][p](ins[1]) })
				ms[f][p][1] = m
				mixedCol = fmt.Sprintf("%9.2f %8.0f", m, ma)
			}
			fmt.Printf("  %-14s %-7s %9.2f %8.0f %s\n", name, label, a, aa, mixedCol)
		}
	}
	fmt.Println("  ratios (computed from the table above):")
	for f, name := range names {
		fmt.Printf("  %-14s ASCII: rune/fast %5.1fx, oracle/fast %6.1fx   mixed: oracle/rune %5.1fx\n",
			name, ms[f][1][0]/ms[f][0][0], ms[f][2][0]/ms[f][0][0], ms[f][2][1]/ms[f][1][1])
	}

	// The brute force is in the table because on a small alphabet its window can
	// never exceed the alphabet: it is Theta(n * sigma) here. Its worst case is text
	// with no repeats at all.
	bruteSteps = 0
	longestOracle(ins[0].pal)
	k := utf8.RuneCountInString(ins[0].pal)
	fmt.Printf("\nLongestUnique brute force on the ASCII MiB: %d steps for %d runes = %.1f per rune (alphabet of 7)\n", bruteSteps, k, float64(bruteSteps)/float64(k))
	for _, n := range []int{2000, 4000} {
		r := make([]rune, n)
		for i := range r {
			r[i] = rune(0x4E00 + i) // n distinct CJK ideographs
		}
		bruteSteps = 0
		sinkI = longestOracle(string(r))
		fmt.Printf("  on %d distinct runes: %d steps = n(n+1)/2 -- Theta(n^2); window version: %d steps\n", n, bruteSteps, n)
	}
}
```

**Sample output:**

```
Proof: 100000 random cases (PCG 1,2), 0-40 runes over "abcA éÉ€😀" or "abcA "; 54112 all-ASCII
  function          fast path    rune path oracle yes
  IsAnagram                 0            0      40163
  IsPalindrome              0            0      57855
  ReverseWords              0            0          -
  Compress                  0            0      45188
  LongestUnique             0            0          -
  (mismatches vs oracle: fast path on the 54112 ASCII cases, rune path on all; Compress 'yes' = got shorter)
  Decompress(Compress(s)) == s:                          100000 / 100000
  ReverseWords(ReverseWords(s)) == Join(Fields(s), " "): 100000 / 100000
  outputs that are valid UTF-8:                          100000 / 100000

Measure: 1 MiB inputs (ASCII "abcxyz ", mixed "abc é€😀"); ms/op = best of 5, allocs/op from AllocsPerRun
  function       path     ASCII ms   allocs  mixed ms   allocs
  IsAnagram      fast         1.29        0       n/a         
  IsAnagram      rune        17.47        0     13.27        0
  IsAnagram      oracle      18.91        2     19.84        2
  IsPalindrome   fast         1.01        0       n/a         
  IsPalindrome   rune         2.74        0      6.36        0
  IsPalindrome   oracle       4.17       34      7.15       29
  ReverseWords   fast         2.17        1       n/a         
  ReverseWords   rune         3.88        1      6.21        1
  ReverseWords   oracle       2.92        2      5.61       23
  Compress       fast         2.94        1       n/a         
  Compress       rune         3.12        1      2.46        1
  Compress       oracle       9.02       31      7.04    27605
  LongestUnique  fast         0.60        0       n/a         
  LongestUnique  rune        12.85        0      8.47        0
  LongestUnique  oracle      38.63        1     25.29        1
  ratios (computed from the table above):
  IsAnagram      ASCII: rune/fast  13.6x, oracle/fast   14.7x   mixed: oracle/rune   1.5x
  IsPalindrome   ASCII: rune/fast   2.7x, oracle/fast    4.1x   mixed: oracle/rune   1.1x
  ReverseWords   ASCII: rune/fast   1.8x, oracle/fast    1.3x   mixed: oracle/rune   0.9x
  Compress       ASCII: rune/fast   1.1x, oracle/fast    3.1x   mixed: oracle/rune   2.9x
  LongestUnique  ASCII: rune/fast  21.6x, oracle/fast   64.8x   mixed: oracle/rune   3.0x

LongestUnique brute force on the ASCII MiB: 3165954 steps for 1048576 runes = 3.0 per rune (alphabet of 7)
  on 2000 distinct runes: 2001000 steps = n(n+1)/2 -- Theta(n^2); window version: 2000 steps
  on 4000 distinct runes: 8002000 steps = n(n+1)/2 -- Theta(n^2); window version: 4000 steps
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{HashMap, HashSet};
use std::fmt::Write;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
// Counts alloc and realloc events; exact, not rounded to size classes.
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

// ---- Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN, so this program
// sees exactly the random cases the Go program sees (lesson 01, example 14).
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
}

// ===== the package: five problems, each char-correct, with an ASCII fast path =====
// The fast path is honest only when the input is all ASCII: then a byte IS a
// character, and a [_; 128] array can replace a HashMap. `str::is_ascii` is the
// one Theta(n) byte scan that picks the path.

fn is_anagram(a: &str, b: &str) -> bool {
    if a.is_ascii() && b.is_ascii() {
        anagram_ascii(a, b)
    } else {
        anagram_chars(a, b)
    }
}

// Equal multisets have equal UTF-8 length, so different byte lengths are a Theta(1) "no".
fn anagram_ascii(a: &str, b: &str) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut count = [0i32; 128];
    for (&x, &y) in a.as_bytes().iter().zip(b.as_bytes()) {
        count[x as usize] += 1;
        count[y as usize] -= 1;
    }
    count == [0; 128]
}

fn anagram_chars(a: &str, b: &str) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut count: HashMap<char, i32> = HashMap::new();
    for c in a.chars() {
        *count.entry(c).or_insert(0) += 1;
    }
    for c in b.chars() {
        let n = count.entry(c).or_insert(0);
        *n -= 1;
        if *n < 0 {
            // b has a char a lacks; with equal byte lengths, that is the only way to differ
            return false;
        }
    }
    true
}

/// Letters and digits only, case folded, two pointers from both ends.
fn is_palindrome(s: &str) -> bool {
    if s.is_ascii() {
        palindrome_ascii(s)
    } else {
        palindrome_chars(s)
    }
}

fn palindrome_ascii(s: &str) -> bool {
    let b = s.as_bytes();
    let (mut i, mut j) = (0, b.len());
    while i < j {
        if !b[i].is_ascii_alphanumeric() {
            i += 1;
        } else if !b[j - 1].is_ascii_alphanumeric() {
            j -= 1;
        } else if b[i] | 0x20 != b[j - 1] | 0x20 {
            // |0x20 lowercases a letter and leaves a digit alone
            return false;
        } else {
            i += 1;
            j -= 1;
        }
    }
    true
}

// `chars()` is double-ended: next() decodes from the front, next_back() from the
// back, on the same filtered iterator -- Go's DecodeRune/DecodeLastRune pair, 0 allocations.
fn palindrome_chars(s: &str) -> bool {
    let mut it = s.chars().filter(|c| c.is_alphanumeric());
    while let Some(x) = it.next() {
        match it.next_back() {
            None => return true, // x was the middle one
            Some(y) if !x.to_lowercase().eq(y.to_lowercase()) => return false,
            Some(_) => {}
        }
    }
    true
}

/// Words in reverse order, single spaces, no padding. Scan from the end and copy
/// each word once into a pre-sized String: 1 allocation.
fn reverse_words(s: &str, ascii: bool) -> String {
    // last_space: is the character that ends at byte j a space, and how wide is it?
    // `is_ascii_whitespace` leaves out \x0b, so the set is spelled out: it matches
    // `split_whitespace` (and Go's strings.Fields) on ASCII.
    let last_space = |j: usize| -> (bool, usize) {
        if ascii {
            let c = s.as_bytes()[j - 1];
            (c == b' ' || (b'\t'..=b'\r').contains(&c), 1)
        } else {
            let c = s[..j].chars().next_back().unwrap(); // j is always a char boundary
            (c.is_whitespace(), c.len_utf8())
        }
    };
    let mut b = String::with_capacity(s.len());
    let mut j = s.len();
    while j > 0 {
        let (sp, w) = last_space(j);
        if sp {
            j -= w;
            continue;
        }
        let mut i = j;
        while i > 0 {
            let (sp, w) = last_space(i);
            if sp {
                break;
            }
            i -= w;
        }
        if !b.is_empty() {
            b.push(' ');
        }
        b.push_str(&s[i..j]);
        j = i;
    }
    b
}

fn compress_str(s: &str) -> String {
    compress(s, s.is_ascii())
}

/// Run-length "a2b1c5a3" over chars; the original when not shorter.
fn compress(s: &str, ascii: bool) -> String {
    if s.is_empty() {
        return String::new();
    }
    let mut b = String::with_capacity(s.len()); // the one allocation
    let next = |i: usize| -> (char, usize) {
        if ascii {
            (s.as_bytes()[i] as char, 1) // fast path: no decoding
        } else {
            let c = s[i..].chars().next().unwrap();
            (c, c.len_utf8())
        }
    };
    let mut i = 0;
    while i < s.len() {
        let (r, w) = next(i);
        let (mut j, mut n) = (i + w, 1usize);
        while j < s.len() {
            let (r2, w2) = next(j);
            if r2 != r {
                break;
            }
            j += w2;
            n += 1;
        }
        let digits = n.ilog10() as usize + 1;
        if b.len() + w + digits >= s.len() {
            return s.to_string(); // not shorter
        }
        b.push_str(&s[i..i + w]);
        write!(b, "{n}").unwrap(); // fmt::Write into the String: no allocation of its own
        i = j;
    }
    b
}

/// A char, then an optional count (absent = 1, so an uncompressed digit-free
/// string decodes to itself). Pass 1 sizes the output, pass 2 fills it.
fn decompress(e: &str) -> String {
    let mut b = String::new();
    for pass in 0..2 {
        let mut total = 0;
        let mut it = e.char_indices().peekable();
        while let Some((_, c)) = it.next() {
            let mut n = 0;
            let mut digits = false;
            while let Some(d) = it.peek().and_then(|&(_, d)| d.to_digit(10)) {
                n = n * 10 + d as usize;
                digits = true;
                it.next();
            }
            if !digits {
                n = 1;
            }
            total += c.len_utf8() * n;
            if pass == 1 {
                b.extend(std::iter::repeat_n(c, n));
            }
        }
        if pass == 0 {
            b.reserve_exact(total);
        }
    }
    b
}

/// Length in chars of the longest substring without a repeated char.
fn longest_unique(s: &str) -> usize {
    if s.is_ascii() {
        longest_ascii(s)
    } else {
        longest_chars(s)
    }
}

fn longest_ascii(s: &str) -> usize {
    let mut last = [0usize; 128]; // 1 + index of the latest occurrence, 0 = unseen
    let (mut best, mut left) = (0, 0);
    for (i, &c) in s.as_bytes().iter().enumerate() {
        left = left.max(last[c as usize]); // never let left move backwards ("abba")
        last[c as usize] = i + 1;
        best = best.max(i + 1 - left);
    }
    best
}

fn longest_chars(s: &str) -> usize {
    let mut last: HashMap<char, usize> = HashMap::new();
    let (mut best, mut left) = (0, 0);
    for (i, c) in s.chars().enumerate() {
        // i counts chars, not bytes
        left = left.max(last.get(&c).copied().unwrap_or(0));
        last.insert(c, i + 1);
        best = best.max(i + 1 - left);
    }
    best
}

// ===== the oracles: the obvious way, on Vec<char> =====

fn anagram_oracle(a: &str, b: &str) -> bool {
    let mut x: Vec<char> = a.chars().collect();
    let mut y: Vec<char> = b.chars().collect();
    x.sort();
    y.sort();
    x == y
}

fn palindrome_oracle(s: &str) -> bool {
    let f: Vec<char> = s
        .chars()
        .filter(|c| c.is_alphanumeric())
        .flat_map(char::to_lowercase)
        .collect();
    let g: Vec<char> = f.iter().rev().copied().collect();
    f == g
}

fn reverse_words_oracle(s: &str) -> String {
    let mut w: Vec<&str> = s.split_whitespace().collect();
    w.reverse();
    w.join(" ")
}

fn compress_oracle(s: &str) -> String {
    let r: Vec<char> = s.chars().collect();
    let mut b = String::new();
    let mut i = 0;
    while i < r.len() {
        let mut j = i;
        while j < r.len() && r[j] == r[i] {
            j += 1;
        }
        // write! formats straight into the String: no boxing, unlike Go's Fprintf.
        write!(b, "{}{}", r[i], j - i).unwrap();
        i = j;
    }
    if b.len() >= s.len() { s.to_string() } else { b }
}

/// Every start, extended until the first repeat: Theta(n^2) worst case.
/// `steps` counts inner-loop steps (no mutable globals in safe Rust).
fn longest_oracle(s: &str, steps: &mut u64) -> usize {
    let r: Vec<char> = s.chars().collect();
    let mut best = 0;
    let mut seen = HashSet::new();
    for i in 0..r.len() {
        seen.clear();
        let mut j = i;
        while j < r.len() && seen.insert(r[j]) {
            j += 1;
            *steps += 1;
        }
        best = best.max(j - i);
    }
    best
}

// ===== proof and measurement =====

fn main() {
    let mut rng = Pcg { hi: 1, lo: 2 };
    let mixed: Vec<char> = "abcA éÉ€😀".chars().collect();
    let ascii: Vec<char> = "abcA ".chars().collect();
    let names = [
        "IsAnagram",
        "IsPalindrome",
        "ReverseWords",
        "Compress",
        "LongestUnique",
    ];
    let mut bad = [[0usize; 2]; 5]; // mismatches vs oracle: [function][0 fast path on ASCII cases, 1 char path on all]
    let mut yes = [0usize; 5];
    const CASES: usize = 100_000;
    let (mut ascii_cases, mut round_trip, mut involution, mut valid) = (0, 0, 0, 0);
    for _ in 0..CASES {
        let alpha = if rng.int_n(2) == 0 { &ascii } else { &mixed };
        let n = rng.int_n(41);
        let runny = rng.int_n(2) == 0;
        let mut rs: Vec<char> = Vec::with_capacity(n);
        while rs.len() < n {
            let c = alpha[rng.int_n(alpha.len())];
            let mut k = if runny { 1 + rng.int_n(5) } else { 1 };
            while k > 0 && rs.len() < n {
                rs.push(c);
                k -= 1;
            }
        }
        let s: String = rs.iter().collect();
        // anagram partner: a shuffle, a shuffle with one change, or random
        let mut bs = rs.clone();
        for i in (1..bs.len()).rev() {
            let j = rng.int_n(i + 1);
            bs.swap(i, j);
        }
        match rng.int_n(3) {
            1 if !bs.is_empty() => {
                let i = rng.int_n(bs.len());
                bs[i] = alpha[rng.int_n(alpha.len())];
            }
            2 => bs
                .iter_mut()
                .for_each(|c| *c = alpha[rng.int_n(alpha.len())]),
            _ => {}
        }
        let b: String = bs.iter().collect();
        // palindrome candidate: s itself, or its first half mirrored with random case flips
        let mut p = rs.clone();
        if rng.int_n(2) == 0 {
            p.truncate(n - n / 2);
            for i in (0..n / 2).rev() {
                let mut r = rs[i];
                if rng.int_n(4) == 0 && r.is_lowercase() {
                    r = r.to_uppercase().next().unwrap(); // still a palindrome: the check folds case
                }
                p.push(r);
            }
        }
        let ps: String = p.iter().collect();

        let is_a = s.is_ascii();
        ascii_cases += usize::from(is_a);
        let mut cmp = |f: usize, fast: bool, slow: bool| {
            // fast/slow: does that path agree with the oracle?
            bad[f][0] += usize::from(is_a && !fast);
            bad[f][1] += usize::from(!slow);
        };
        let want_a = anagram_oracle(&s, &b);
        cmp(
            0,
            is_anagram(&s, &b) == want_a,
            anagram_chars(&s, &b) == want_a,
        );
        let want_p = palindrome_oracle(&ps);
        cmp(
            1,
            is_palindrome(&ps) == want_p,
            palindrome_chars(&ps) == want_p,
        );
        let rw = reverse_words(&s, s.is_ascii());
        let want = reverse_words_oracle(&s);
        cmp(2, rw == want, reverse_words(&s, false) == want);
        let c = compress_str(&s);
        let want = compress_oracle(&s);
        cmp(3, c == want, compress(&s, false) == want);
        let want = longest_oracle(&s, &mut 0);
        cmp(4, longest_unique(&s) == want, longest_chars(&s) == want);
        yes[0] += usize::from(want_a);
        yes[1] += usize::from(want_p);
        yes[3] += usize::from(c != s);
        round_trip += usize::from(decompress(&c) == s);
        let fields = s.split_whitespace().collect::<Vec<_>>().join(" ");
        involution += usize::from(reverse_words(&rw, rw.is_ascii()) == fields);
        // Always true in Rust: a String cannot hold invalid UTF-8. Checked anyway, like Go.
        let ok = |x: &str| std::str::from_utf8(x.as_bytes()).is_ok();
        valid += usize::from(ok(&rw) && ok(&c) && ok(&decompress(&c)));
    }
    let mixed_s: String = mixed.iter().collect();
    let ascii_s: String = ascii.iter().collect();
    println!(
        "Proof: {CASES} random cases (PCG 1,2), 0-40 runes over {mixed_s:?} or {ascii_s:?}; {ascii_cases} all-ASCII"
    );
    println!(
        "  {:<14} {:>12} {:>12} {:>10}",
        "function", "fast path", "rune path", "oracle yes"
    );
    for (f, name) in names.iter().enumerate() {
        let yes_col = if f != 2 && f != 4 {
            yes[f].to_string()
        } else {
            "-".to_string()
        };
        println!(
            "  {name:<14} {:>12} {:>12} {yes_col:>10}",
            bad[f][0], bad[f][1]
        );
    }
    println!(
        "  (mismatches vs oracle: fast path on the {ascii_cases} ASCII cases, rune path on all; Compress 'yes' = got shorter)"
    );
    println!("  Decompress(Compress(s)) == s:                          {round_trip} / {CASES}");
    println!("  ReverseWords(ReverseWords(s)) == Join(Fields(s), \" \"): {involution} / {CASES}");
    println!("  outputs that are valid UTF-8:                          {valid} / {CASES}");

    // --- measurement: 1 MiB inputs, each path timed separately ---
    let mut text = |alpha: &[char], bytes: usize, runny: bool| -> String {
        let mut b = String::new();
        while b.len() < bytes {
            let c = alpha[rng.int_n(alpha.len())];
            let k = if runny { 1 + rng.int_n(9) } else { 1 };
            b.extend(std::iter::repeat_n(c, k));
        }
        b
    };
    let rev = |s: &str| s.chars().rev().collect::<String>();
    struct Input {
        pal: String,
        rot: String,
        runs: String,
    }
    let mut mk = |alpha: &str| {
        let a: Vec<char> = alpha.chars().collect();
        let half = text(&a, 1 << 19, false);
        let runs = text(&a, 1 << 20, true);
        Input {
            pal: half.clone() + &rev(&half),
            rot: rev(&half) + &half,
            runs,
        }
    };
    let ins = [mk("abcxyz "), mk("abc é€😀")];
    // One closure per (function, path); black_box keeps the result from being optimised away.
    type Path = fn(&Input);
    let paths: [[Path; 3]; 5] = [
        [
            |i| _ = black_box(anagram_ascii(&i.pal, &i.rot)),
            |i| _ = black_box(anagram_chars(&i.pal, &i.rot)),
            |i| _ = black_box(anagram_oracle(&i.pal, &i.rot)),
        ],
        [
            |i| _ = black_box(palindrome_ascii(&i.pal)),
            |i| _ = black_box(palindrome_chars(&i.pal)),
            |i| _ = black_box(palindrome_oracle(&i.pal)),
        ],
        [
            |i| _ = black_box(reverse_words(&i.pal, true)),
            |i| _ = black_box(reverse_words(&i.pal, false)),
            |i| _ = black_box(reverse_words_oracle(&i.pal)),
        ],
        [
            |i| _ = black_box(compress(&i.runs, true)),
            |i| _ = black_box(compress(&i.runs, false)),
            |i| _ = black_box(compress_oracle(&i.runs)),
        ],
        [
            |i| _ = black_box(longest_ascii(&i.pal)),
            |i| _ = black_box(longest_chars(&i.pal)),
            |i| _ = black_box(longest_oracle(&i.pal, &mut 0)),
        ],
    ];
    let measure = |f: Path, input: &Input| -> (f64, usize) {
        let a0 = ALLOCS.load(Relaxed);
        f(input);
        let allocs = ALLOCS.load(Relaxed) - a0;
        let mut best = Duration::MAX;
        for _ in 0..5 {
            let t = Instant::now();
            f(input);
            best = best.min(t.elapsed());
        }
        (best.as_nanos() as f64 / 1e6, allocs)
    };
    println!(
        "\nMeasure: 1 MiB inputs (ASCII {:?}, mixed {:?}); ms/op = best of 5, allocs/op from the counting allocator",
        "abcxyz ", "abc é€😀"
    );
    println!(
        "  {:<14} {:<7} {:>9} {:>8} {:>9} {:>8}",
        "function", "path", "ASCII ms", "allocs", "mixed ms", "allocs"
    );
    let mut ms = [[[0f64; 2]; 3]; 5];
    for (f, name) in names.iter().enumerate() {
        for (p, label) in ["fast", "rune", "oracle"].iter().enumerate() {
            let (a, aa) = measure(paths[f][p], &ins[0]);
            ms[f][p][0] = a;
            let mixed_col = if p > 0 {
                let (m, ma) = measure(paths[f][p], &ins[1]);
                ms[f][p][1] = m;
                format!("{m:>9.2} {ma:>8}")
            } else {
                format!("{:>9} {:>8}", "n/a", "")
            };
            println!("  {name:<14} {label:<7} {a:>9.2} {aa:>8} {mixed_col}");
        }
    }
    println!("  ratios (computed from the table above):");
    for (f, name) in names.iter().enumerate() {
        let m = &ms[f];
        println!(
            "  {name:<14} ASCII: rune/fast {:>5.1}x, oracle/fast {:>6.1}x   mixed: oracle/rune {:>5.1}x",
            m[1][0] / m[0][0],
            m[2][0] / m[0][0],
            m[2][1] / m[1][1]
        );
    }

    // The brute force is in the table because on a small alphabet its window can
    // never exceed the alphabet: it is Theta(n * sigma) here. Its worst case is text
    // with no repeats at all.
    let mut steps = 0;
    longest_oracle(&ins[0].pal, &mut steps);
    let k = ins[0].pal.chars().count();
    println!(
        "\nLongestUnique brute force on the ASCII MiB: {steps} steps for {k} runes = {:.1} per rune (alphabet of 7)",
        steps as f64 / k as f64
    );
    for n in [2000u32, 4000] {
        // n distinct CJK ideographs
        let s: String = (0..n)
            .map(|i| char::from_u32(0x4E00 + i).unwrap())
            .collect();
        let mut steps = 0;
        black_box(longest_oracle(&s, &mut steps));
        println!(
            "  on {n} distinct runes: {steps} steps = n(n+1)/2 -- Theta(n^2); window version: {n} steps"
        );
    }
}
```

**Rust sample output:**

```
Proof: 100000 random cases (PCG 1,2), 0-40 runes over "abcA éÉ€😀" or "abcA "; 54112 all-ASCII
  function          fast path    rune path oracle yes
  IsAnagram                 0            0      40163
  IsPalindrome              0            0      57855
  ReverseWords              0            0          -
  Compress                  0            0      45188
  LongestUnique             0            0          -
  (mismatches vs oracle: fast path on the 54112 ASCII cases, rune path on all; Compress 'yes' = got shorter)
  Decompress(Compress(s)) == s:                          100000 / 100000
  ReverseWords(ReverseWords(s)) == Join(Fields(s), " "): 100000 / 100000
  outputs that are valid UTF-8:                          100000 / 100000

Measure: 1 MiB inputs (ASCII "abcxyz ", mixed "abc é€😀"); ms/op = best of 5, allocs/op from the counting allocator
  function       path     ASCII ms   allocs  mixed ms   allocs
  IsAnagram      fast         1.24        0       n/a         
  IsAnagram      rune         9.74        2      8.04        2
  IsAnagram      oracle       5.73        8      6.21        8
  IsPalindrome   fast         0.78        0       n/a         
  IsPalindrome   rune         3.15        0      6.46        0
  IsPalindrome   oracle       5.29       20      7.91       19
  ReverseWords   fast         1.77        1       n/a         
  ReverseWords   rune         4.06        1      6.64        1
  ReverseWords   oracle       3.66       17      3.72       17
  Compress       fast         3.08        1       n/a         
  Compress       rune         3.38        1      2.65        1
  Compress       oracle       3.69       20      2.99       20
  LongestUnique  fast         0.59        0       n/a         
  LongestUnique  rune         8.47        3      5.74        3
  LongestUnique  oracle      33.68        6     21.84        6
  ratios (computed from the table above):
  IsAnagram      ASCII: rune/fast   7.8x, oracle/fast    4.6x   mixed: oracle/rune   0.8x
  IsPalindrome   ASCII: rune/fast   4.0x, oracle/fast    6.8x   mixed: oracle/rune   1.2x
  ReverseWords   ASCII: rune/fast   2.3x, oracle/fast    2.1x   mixed: oracle/rune   0.6x
  Compress       ASCII: rune/fast   1.1x, oracle/fast    1.2x   mixed: oracle/rune   1.1x
  LongestUnique  ASCII: rune/fast  14.4x, oracle/fast   57.3x   mixed: oracle/rune   3.8x

LongestUnique brute force on the ASCII MiB: 3165954 steps for 1048576 runes = 3.0 per rune (alphabet of 7)
  on 2000 distinct runes: 2001000 steps = n(n+1)/2 -- Theta(n^2); window version: 2000 steps
  on 4000 distinct runes: 8002000 steps = n(n+1)/2 -- Theta(n^2); window version: 4000 steps
```

> **Go vs Rust:** The proof section is the Go output byte for byte (54,112 ASCII cases, 40,163 anagrams, 45,188 shorter encodings, 0 mismatches), because the PCG port feeds both programs the same 100,000 cases; the brute-force step counts match too. The allocation counts differ for language reasons: Rust's `HashMap` grows (2–3 allocations where Go's small map made 0), `chars().collect()` grows a `Vec` from a low size hint (8 for the anagram oracle's two `Vec`s), and `write!` formats straight into the `String`, so the Compress oracle makes 20 allocations on mixed text where Go's `Fprintf` made 27,605.
> In Rust's runs the sort-based anagram oracle beat the SipHash `HashMap` path on both inputs, so the oracle is not the slow one here. A `String` cannot hold invalid UTF-8, so the validity line is true by type; it is checked anyway to match.

**Complexity:** all five Θ(n) (the sort oracle Θ(n log n); the brute force Θ(n²) worst case: 8,002,000 steps for 4,000 distinct runes, but only ~3 steps per rune on a 7-letter alphabet, which is why it fits in the table) · **0 mismatches** on both paths over 100,000 cases, with 100,000/100,000 round-trips, word-reversal involutions and valid-UTF-8 outputs · fast paths allocate 0 (1 for the string-building ones) · over ten runs the rune path cost about an order of magnitude more than the fast path (≈9–25×) for `IsAnagram` and `LongestUnique`, where a map replaces a `[128]int`; 2–3.5× for `IsPalindrome`; and only 1.0–1.6× for `Compress`, whose decode is mostly the inlined ASCII check · the `Fields`+`Join` oracle stayed within ±30% of the `ReverseWords` fast path · the oracles made 2–34 allocations, and 27,605 for `Compress` on mixed text (`Fprintf` boxes every rune ≥ 256)

---

> Next lesson: [12 — Elementary Sorts](../../12-elementary-sorts.md) — Part 3 begins, and counting
> comparisons becomes the whole method.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
