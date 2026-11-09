# Step 15 — Binary Search & Its Variants · 🟢 Easy

Examples **1–6**: the invariant and the probe count it implies, the overflow in both languages, lower
and upper bound, which duplicate each standard library returns, and the boundary search with no array.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match character for character; **Sample
output:** ones print timings.

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

## 1. The invariant, written down

`🟢 easy` · *Loop invariant*

The half-open boundary search keeps a window `[lo, hi)` and two promises: everything left of `lo` is
false (`p(lo-1)` false, with `-1` as a sentinel) and `hi` is true (`p(hi)` true, with `n` as a sentinel).
This program checks both promises after every update and shows the window strictly shrinking, which is
the whole correctness proof: once the invariant is written, `lo = mid + 1` and `hi = mid` are the only
updates that keep it. It then counts probes for every possible answer and finds the cost is Θ(log n)
with no early exit: always ⌊log₂(n+1)⌋ or ⌈log₂(n+1)⌉, hit or miss.

**Steps:**

1. Write `firstTrue(n, p)` with `lo, hi := 0, n` and a `check` that reports a broken promise instead of panicking.
2. Trace one search with the invariant checked after each update, then run every answer position 0..n and histogram the probe counts.
3. Compare the probe buckets with `bits.Len`, and watch the closed template's `hi = mid - 1` step to `-1`.

```go
package main

import (
	"fmt"
	"math/bits"
)

// The one template of this lesson: find the FIRST index i in [0, n) where the
// monotone predicate p(i) is true (false...false true...true), or n if none.
//
// Window [lo, hi), and two promises that hold before and after every step:
//   (1) p(lo-1) is false   -- everything left of lo is false (lo-1 = -1 is a sentinel)
//   (2) p(hi)   is true    -- hi is true (hi = n is a sentinel: pretend p(n) == true)
// When lo == hi the two promises meet: lo is the first true. That is the proof.
// The updates are forced by it: p(mid) false -> lo = mid+1 keeps (1);
// p(mid) true -> hi = mid keeps (2). mid in [lo, hi) makes both shrink the window.

// check returns "" if both promises hold, or a description of the broken one.
// (We print it instead of panicking: a broken invariant is a finding, not a crash.)
func check(lo, hi, n int, p func(int) bool) string {
	if lo > 0 && p(lo-1) {
		return fmt.Sprintf("p(lo-1)=p(%d) is true", lo-1)
	}
	if hi < n && !p(hi) {
		return fmt.Sprintf("p(hi)=p(%d) is false", hi)
	}
	return ""
}

// firstTrue is the template; probes counts calls to p made by the search itself.
func firstTrue(n int, p func(int) bool) (ans, probes int) {
	lo, hi := 0, n
	for lo < hi {
		mid := int(uint(lo+hi) >> 1) // rounds down, so lo <= mid < hi
		probes++
		if p(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo, probes
}

func main() {
	// (a) one search, traced, with the invariant checked after every update
	a := make([]int, 16)
	for i := range a {
		a[i] = 3 * i // 0 3 6 ... 45
	}
	n, x := len(a), 20
	p := func(i int) bool { return a[i] >= x }
	fmt.Printf("(a) a = %v, first i with a[i] >= %d\n\n", a, x)
	fmt.Printf("  %-9s %4s %7s %7s   %-8s %s\n", "[lo, hi)", "mid", "a[mid]", "p(mid)", "update", "size -> size, check")
	lo, hi := 0, n
	for lo < hi {
		mid := int(uint(lo+hi) >> 1) // rounds down, so lo <= mid < hi
		window, before, upd := fmt.Sprintf("[%d, %d)", lo, hi), hi-lo, ""
		if p(mid) {
			hi, upd = mid, fmt.Sprintf("hi = %d", mid)
		} else {
			lo, upd = mid+1, fmt.Sprintf("lo = %d", mid+1)
		}
		status := "invariant holds"
		if e := check(lo, hi, n, p); e != "" {
			status = "INVARIANT BROKEN: " + e
		}
		fmt.Printf("  %-9s %4d %7d %7v   %-8s %2d -> %2d, %s\n",
			window, mid, a[mid], p(mid), upd, before, hi-lo, status)
	}
	fmt.Printf("  lo == hi == %d: a[%d] = %d is the first >= %d, a[%d] = %d is not\n\n",
		lo, lo, a[lo], x, lo-1, a[lo-1])

	// (b) every possible answer for n = 1000: the probe count barely moves
	const N = 1000
	hist := map[int]int{}
	for ans := 0; ans <= N; ans++ { // ans == N means "no element is true"
		got, probes := firstTrue(N, func(i int) bool { return i >= ans })
		if got != ans {
			fmt.Println("WRONG ANSWER for", ans)
		}
		hist[probes]++
	}
	floor := bits.Len(uint(N+1)) - 1 // floor(log2(n+1))
	ceil := bits.Len(uint(N))        // ceil(log2(n+1)) for n >= 1
	fmt.Printf("(b) n = %d, all %d answer positions 0..n:\n", N, N+1)
	for k := floor; k <= ceil; k++ {
		fmt.Printf("  %2d probes: %4d answers\n", k, hist[k])
	}
	fmt.Printf("  buckets used: %d; floor(log2(n+1)) = %d, ceil(log2(n+1)) = %d\n\n", len(hist), floor, ceil)

	// (c) the same min/max for a range of n
	fmt.Println("(c) min/max probes over every answer 0..n:")
	fmt.Printf("  %9s %5s %5s %18s %18s\n", "n", "min", "max", "floor(log2(n+1))", "ceil(log2(n+1))")
	oneBucket, allPerfect := []int{}, true
	for _, n := range []int{1, 2, 3, 7, 8, 1023, 1024, 1_000_000} {
		mn, mx := 1<<30, 0
		for ans := 0; ans <= n; ans++ {
			_, probes := firstTrue(n, func(i int) bool { return i >= ans })
			mn, mx = min(mn, probes), max(mx, probes)
		}
		fmt.Printf("  %9d %5d %5d %18d %18d\n", n, mn, mx, bits.Len(uint(n+1))-1, bits.Len(uint(n)))
		perfect := (n+1)&n == 0 // n+1 is a power of two, i.e. n = 2^k - 1
		if (mn == mx) != perfect {
			allPerfect = false
		}
		if mn == mx {
			oneBucket = append(oneBucket, n)
		}
	}
	fmt.Printf("  a single bucket (min == max) at n = %v; exactly the n = 2^k - 1 rows: %v\n\n", oneBucket, allPerfect)

	// (d) the other template: closed window [lo, hi], hi = mid - 1. Search for a
	// value below everything and watch hi walk off the left end.
	y := -5
	lo, hi = 0, n-1
	fmt.Printf("(d) closed template [lo, hi], looking for %d (smaller than every element):\n  hi:", y)
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if a[mid] < y {
			lo = mid + 1
		} else {
			hi = mid - 1 // at mid == 0 this is -1
		}
		fmt.Printf(" %d", hi)
	}
	fmt.Printf("\n  loop ends with lo = %d, hi = %d: fine for a signed int,\n", lo, hi)
	fmt.Println("  but with an unsigned index type that last step is 0 - 1 (see the Rust twin).")
	fmt.Println()
	fmt.Println("the half-open loop never exits early: it cannot know the boundary until the")
	fmt.Println("window is empty, so a hit costs the same probes as a miss.")
}
```

**Output:**

```
(a) a = [0 3 6 9 12 15 18 21 24 27 30 33 36 39 42 45], first i with a[i] >= 20

  [lo, hi)   mid  a[mid]  p(mid)   update   size -> size, check
  [0, 16)      8      24    true   hi = 8   16 ->  8, invariant holds
  [0, 8)       4      12   false   lo = 5    8 ->  3, invariant holds
  [5, 8)       6      18   false   lo = 7    3 ->  1, invariant holds
  [7, 8)       7      21    true   hi = 7    1 ->  0, invariant holds
  lo == hi == 7: a[7] = 21 is the first >= 20, a[6] = 18 is not

(b) n = 1000, all 1001 answer positions 0..n:
   9 probes:   23 answers
  10 probes:  978 answers
  buckets used: 2; floor(log2(n+1)) = 9, ceil(log2(n+1)) = 10

(c) min/max probes over every answer 0..n:
          n   min   max   floor(log2(n+1))    ceil(log2(n+1))
          1     1     1                  1                  1
          2     1     2                  1                  2
          3     2     2                  2                  2
          7     3     3                  3                  3
          8     3     4                  3                  4
       1023    10    10                 10                 10
       1024    10    11                 10                 11
    1000000    19    20                 19                 20
  a single bucket (min == max) at n = [1 3 7 1023]; exactly the n = 2^k - 1 rows: true

(d) closed template [lo, hi], looking for -5 (smaller than every element):
  hi: 6 2 0 -1
  loop ends with lo = 0, hi = -1: fine for a signed int,
  but with an unsigned index type that last step is 0 - 1 (see the Rust twin).

the half-open loop never exits early: it cannot know the boundary until the
window is empty, so a hit costs the same probes as a miss.
```

**Rust:**

```rust
use std::collections::BTreeMap;
use std::fmt::Display;

// The one template of this lesson: find the FIRST index i in [0, n) where the
// monotone predicate p(i) is true (false...false true...true), or n if none.
//
// Window [lo, hi), and two promises that hold before and after every step:
//   (1) p(lo-1) is false   -- everything left of lo is false (lo == 0: nothing to check)
//   (2) p(hi)   is true    -- hi is true (hi == n is a sentinel: pretend p(n) == true)
// When lo == hi the two promises meet: lo is the first true. That is the proof.
// The updates are forced by it: p(mid) false -> lo = mid+1 keeps (1);
// p(mid) true -> hi = mid keeps (2). mid in [lo, hi) makes both shrink the window.
//
// In Rust the indexes are usize, and the half-open template is the natural fit:
// it never computes anything below lo, so it never needs a -1 (see part (d)).

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Go's bits.Len: the number of bits needed to write n.
fn bits_len(n: usize) -> u32 {
    usize::BITS - n.leading_zeros()
}

/// None if both promises hold, or a description of the broken one.
/// (Reported instead of panicking: a broken invariant is a finding, not a crash.)
fn check(lo: usize, hi: usize, n: usize, p: impl Fn(usize) -> bool) -> Option<String> {
    if lo > 0 && p(lo - 1) {
        return Some(format!("p(lo-1)=p({}) is true", lo - 1));
    }
    if hi < n && !p(hi) {
        return Some(format!("p(hi)=p({hi}) is false"));
    }
    None
}

/// The template; returns (answer, probes made by the search itself).
fn first_true(n: usize, p: impl Fn(usize) -> bool) -> (usize, u32) {
    let (mut lo, mut hi, mut probes) = (0, n, 0);
    while lo < hi {
        let mid = lo.midpoint(hi); // (lo+hi)/2 rounded down, cannot overflow
        probes += 1;
        if p(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    (lo, probes)
}

fn main() {
    // (a) one search, traced, with the invariant checked after every update
    let a: Vec<i64> = (0..16).map(|i| 3 * i).collect(); // values i64, indexes usize
    let (n, x) = (a.len(), 20);
    let p = |i: usize| a[i] >= x;
    println!("(a) a = {}, first i with a[i] >= {x}\n", list(&a));
    println!(
        "  {:<9} {:>4} {:>7} {:>7}   {:<8} size -> size, check",
        "[lo, hi)", "mid", "a[mid]", "p(mid)", "update"
    );
    let (mut lo, mut hi) = (0, n);
    while lo < hi {
        let mid = lo.midpoint(hi); // rounds down, so lo <= mid < hi
        let (window, before) = (format!("[{lo}, {hi})"), hi - lo);
        let upd = if p(mid) {
            hi = mid;
            format!("hi = {mid}")
        } else {
            lo = mid + 1;
            format!("lo = {}", mid + 1)
        };
        let status = match check(lo, hi, n, p) {
            None => "invariant holds".to_string(),
            Some(e) => format!("INVARIANT BROKEN: {e}"),
        };
        println!(
            "  {:<9} {:>4} {:>7} {:>7}   {:<8} {:>2} -> {:>2}, {}",
            window,
            mid,
            a[mid],
            p(mid),
            upd,
            before,
            hi - lo,
            status
        );
    }
    println!(
        "  lo == hi == {lo}: a[{lo}] = {} is the first >= {x}, a[{}] = {} is not\n",
        a[lo],
        lo - 1,
        a[lo - 1]
    );

    // (b) every possible answer for n = 1000: the probe count barely moves
    const N: usize = 1000;
    let mut hist: BTreeMap<u32, usize> = BTreeMap::new();
    for ans in 0..=N {
        // ans == N means "no element is true"
        let (got, probes) = first_true(N, |i| i >= ans);
        if got != ans {
            println!("WRONG ANSWER for {ans}");
        }
        *hist.entry(probes).or_default() += 1;
    }
    let floor = bits_len(N + 1) - 1; // floor(log2(n+1))
    let ceil = bits_len(N); // ceil(log2(n+1)) for n >= 1
    println!("(b) n = {N}, all {} answer positions 0..n:", N + 1);
    for k in floor..=ceil {
        println!(
            "  {k:>2} probes: {:>4} answers",
            hist.get(&k).copied().unwrap_or(0)
        );
    }
    println!(
        "  buckets used: {}; floor(log2(n+1)) = {floor}, ceil(log2(n+1)) = {ceil}\n",
        hist.len()
    );

    // (c) the same min/max for a range of n
    println!("(c) min/max probes over every answer 0..n:");
    println!(
        "  {:>9} {:>5} {:>5} {:>18} {:>18}",
        "n", "min", "max", "floor(log2(n+1))", "ceil(log2(n+1))"
    );
    let (mut one_bucket, mut all_perfect) = (Vec::new(), true);
    for n in [1, 2, 3, 7, 8, 1023, 1024, 1_000_000] {
        let (mn, mx) = (0..=n)
            .map(|ans| first_true(n, |i| i >= ans).1)
            .fold((u32::MAX, 0), |(mn, mx), p| (mn.min(p), mx.max(p)));
        println!(
            "  {n:>9} {mn:>5} {mx:>5} {:>18} {:>18}",
            bits_len(n + 1) - 1,
            bits_len(n)
        );
        let perfect = (n + 1).is_power_of_two(); // n = 2^k - 1
        if (mn == mx) != perfect {
            all_perfect = false;
        }
        if mn == mx {
            one_bucket.push(n);
        }
    }
    println!(
        "  a single bucket (min == max) at n = {}; exactly the n = 2^k - 1 rows: {all_perfect}\n",
        list(&one_bucket)
    );

    // (d) the other template: closed window [lo, hi], hi = mid - 1. Search for a
    // value below everything. With usize, the step that would make hi = -1 has no
    // value to land on: checked_sub says so with None (a plain `mid - 1` would
    // panic in a debug build and wrap to usize::MAX with -O).
    let y: i64 = -5;
    let (mut lo, mut hi) = (0usize, n - 1);
    print!("(d) closed template [lo, hi], looking for {y} (smaller than every element):\n  hi:");
    loop {
        let mid = lo + (hi - lo) / 2;
        if a[mid] < y {
            lo = mid + 1;
        } else {
            match mid.checked_sub(1) {
                Some(h) => hi = h,
                None => {
                    println!(" then {mid} - 1 = checked_sub -> None");
                    break;
                }
            }
        }
        print!(" {hi}");
        if lo > hi {
            println!();
            break;
        }
    }
    println!(
        "  lo = {lo}: the closed loop stops only when hi < lo, i.e. hi = -1; usize has no -1."
    );
    println!("  The half-open template never computes below lo, so it never meets this.");
    println!();
    println!("the half-open loop never exits early: it cannot know the boundary until the");
    println!("window is empty, so a hit costs the same probes as a miss.");
}
```

**Rust output:**

```
(a) a = [0 3 6 9 12 15 18 21 24 27 30 33 36 39 42 45], first i with a[i] >= 20

  [lo, hi)   mid  a[mid]  p(mid)   update   size -> size, check
  [0, 16)      8      24    true   hi = 8   16 ->  8, invariant holds
  [0, 8)       4      12   false   lo = 5    8 ->  3, invariant holds
  [5, 8)       6      18   false   lo = 7    3 ->  1, invariant holds
  [7, 8)       7      21    true   hi = 7    1 ->  0, invariant holds
  lo == hi == 7: a[7] = 21 is the first >= 20, a[6] = 18 is not

(b) n = 1000, all 1001 answer positions 0..n:
   9 probes:   23 answers
  10 probes:  978 answers
  buckets used: 2; floor(log2(n+1)) = 9, ceil(log2(n+1)) = 10

(c) min/max probes over every answer 0..n:
          n   min   max   floor(log2(n+1))    ceil(log2(n+1))
          1     1     1                  1                  1
          2     1     2                  1                  2
          3     2     2                  2                  2
          7     3     3                  3                  3
          8     3     4                  3                  4
       1023    10    10                 10                 10
       1024    10    11                 10                 11
    1000000    19    20                 19                 20
  a single bucket (min == max) at n = [1 3 7 1023]; exactly the n = 2^k - 1 rows: true

(d) closed template [lo, hi], looking for -5 (smaller than every element):
  hi: 6 2 0 then 0 - 1 = checked_sub -> None
  lo = 0: the closed loop stops only when hi < lo, i.e. hi = -1; usize has no -1.
  The half-open template never computes below lo, so it never meets this.

the half-open loop never exits early: it cannot know the boundary until the
window is empty, so a hit costs the same probes as a miss.
```

> **Go vs Rust:** Parts (a)–(c) are identical: same loop, same probe counts, with `usize::midpoint` standing in for Go's `int(uint(lo+hi) >> 1)`. Part (d) differs on purpose: Go's closed template ends with `hi = -1`, an ordinary `int`; with `usize` that step is `0 - 1`, which `checked_sub` reports as `None` (a plain `-` would panic in a debug build and wrap to `usize::MAX` with `-O`). That is why the half-open template is the default in Rust.

**Complexity:** Θ(log n) time, Θ(1) space · for n = 1000 every one of the 1001 answers costs 9 probes (23 of them) or 10 (978) — ⌊log₂ 1001⌋ and ⌈log₂ 1001⌉; n = 2ᵏ − 1 gives a single bucket, because the loop never exits early.

---

## 2. The overflow in `(lo + hi) / 2`

`🟢 easy` · *Integer overflow*

`mid := (lo + hi) / 2` is wrong as soon as `lo + hi` passes the type's maximum, and Go's spec makes that
silent: signed overflow wraps and never panics. With 64-bit `int` a slice index cannot get there, so this
program goes where the bug does bite: a search over an `int64` **value range** ending at `math.MaxInt64`
(search-on-the-answer), and an `int32` index over a two-billion-element virtual array. The two fixes,
`lo + (hi-lo)/2` and the unsigned shift `int(uint(lo+hi) >> 1)` that Go's own library uses, keep the
search Θ(log n) with the same probe budget, so there is no reason not to use them.

**Steps:**

1. Print the three midpoint formulas for a window just under `MaxInt64` and just under `MaxInt32`.
2. Run the same `firstTrue` loop with the buggy and the fixed midpoint, capped at 200 iterations, and trace the first buggy probes.
3. Search an `int32` index range of 2,000,000,000 and bounds-check each `mid` yourself, so the bug shows up as a classified out-of-range probe instead of a panic.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
)

// The Bloch bug: mid := (lo + hi) / 2 overflows when lo + hi exceeds the type.
// Go's spec: signed integer overflow WRAPS and "does not cause a run-time panic".
// So the bug is silent -- you get a wrong (often negative) mid and carry on.
//
// With a 64-bit int a slice index never gets near 2^62, so slices are safe.
// Two places it bites for real: searching a VALUE range (search-on-the-answer
// over int64, up to MaxInt64), and 32-bit index types past 2^30.

// firstTrue64 returns the first x in [lo, hi) with p(x) true, or hi if none
// (hi is the sentinel "pretend true", as in example 1). buggy picks the midpoint.
// It is capped at limit iterations, and counts mids that fall outside [lo, hi).
func firstTrue64(lo, hi int64, p func(int64) bool, buggy bool, limit int) (ans int64, probes, outside int, done bool) {
	for lo < hi {
		if probes == limit {
			return lo, probes, outside, false
		}
		var mid int64
		if buggy {
			mid = (lo + hi) / 2 // wraps when lo+hi > MaxInt64
		} else {
			mid = lo + (hi-lo)/2 // hi-lo fits: both are in range and lo < hi
		}
		probes++
		if mid < lo || mid >= hi {
			outside++
		}
		if p(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo, probes, outside, true
}

// midpoints of an int32 window: buggy, subtract form, unsigned form.
func mids32(lo, hi int32) (int32, int32, int32) {
	return (lo + hi) / 2, lo + (hi-lo)/2, int32(uint32(lo+hi) >> 1)
}

func main() {
	fmt.Println("(a) three midpoints, near the top of each type")
	lo, hi := int64(math.MaxInt64-3), int64(math.MaxInt64-1)
	fmt.Printf("  int64 lo = %d, hi = %d\n", lo, hi)
	fmt.Printf("    (lo+hi)/2              = %d\n", (lo+hi)/2)
	fmt.Printf("    lo+(hi-lo)/2           = %d\n", lo+(hi-lo)/2)
	fmt.Printf("    int64(uint64(lo+hi)>>1) = %d\n", int64(uint64(lo+hi)>>1))
	l32, h32 := int32(math.MaxInt32-3), int32(math.MaxInt32-1)
	b, s, u := mids32(l32, h32)
	fmt.Printf("  int32 lo = %d, hi = %d\n", l32, h32)
	fmt.Printf("    (lo+hi)/2 = %d, lo+(hi-lo)/2 = %d, int32(uint32(lo+hi)>>1) = %d\n\n", b, s, u)
	// the unsigned trick works because two NON-NEGATIVE values sum to at most
	// 2*Max < 2^64 (2^32): the wrapped bits, read as unsigned, are the true sum.

	fmt.Println("(b) search on the answer: first x in [MaxInt64-100, MaxInt64) with x >= MaxInt64-5")
	target := int64(math.MaxInt64 - 5)
	p := func(x int64) bool { return x >= target }
	const limit = 200
	for _, buggy := range []bool{true, false} {
		name := "fixed lo+(hi-lo)/2"
		if buggy {
			name = "buggy (lo+hi)/2   "
		}
		ans, probes, outside, done := firstTrue64(math.MaxInt64-100, math.MaxInt64, p, buggy, limit)
		verdict := fmt.Sprintf("answer MaxInt64-%d", math.MaxInt64-ans)
		if !done {
			verdict = fmt.Sprintf("did not terminate within %d iterations", limit)
		}
		ok := done && ans == target
		fmt.Printf("  %s %-40s probes %3d, mids outside window %3d, correct: %v\n", name, verdict, probes, outside, ok)
	}
	fmt.Printf("  the window holds 100 values: ceil(log2(101)) = %d probes is the budget\n", bits.Len(100))
	fmt.Println("  the buggy loop's first probes -- hi never moves, lo jumps out of the window:")
	l, h := int64(math.MaxInt64-100), int64(math.MaxInt64)
	for range 4 {
		mid := (l + h) / 2
		fmt.Printf("    lo = %20d  mid = %20d  p(mid) = %v\n", l, mid, p(mid))
		if p(mid) {
			h = mid
		} else {
			l = mid + 1
		}
	}
	fmt.Println()

	fmt.Println("(c) int32 index over a virtual array of 2,000,000,000 (no allocation)")
	const n32 = int32(2_000_000_000)
	want := int32(1_500_000_000)
	for _, form := range []string{"buggy", "subtract", "unsigned"} {
		lo, hi, probes := int32(0), n32, 0
		result := ""
		for lo < hi {
			b, s, u := mids32(lo, hi)
			mid := map[string]int32{"buggy": b, "subtract": s, "unsigned": u}[form]
			probes++
			if mid < 0 || mid >= n32 { // a real slice would panic here: index out of range
				result = fmt.Sprintf("probe %d: mid = %d from lo = %d, hi = %d -> out of range", probes, mid, lo, hi)
				break
			}
			if mid >= want {
				hi = mid
			} else {
				lo = mid + 1
			}
		}
		if result == "" {
			result = fmt.Sprintf("answer %d in %d probes, correct: %v", lo, probes, lo == want)
		}
		fmt.Printf("  %-8s %s\n", form, result)
	}
	fmt.Printf("  floor/ceil(log2(n+1)) = %d/%d: the probe budget for n = 2e9\n\n",
		bits.Len(uint(n32)+1)-1, bits.Len(uint(n32)))

	fmt.Println("the fix costs nothing: same loop, same probe budget. The bug only needs lo + hi")
	fmt.Println("past the type's maximum -- never for a slice index on 64-bit, easily for an int64")
	fmt.Println("value range or an int32 index.")
}
```

**Output:**

```
(a) three midpoints, near the top of each type
  int64 lo = 9223372036854775804, hi = 9223372036854775806
    (lo+hi)/2              = -3
    lo+(hi-lo)/2           = 9223372036854775805
    int64(uint64(lo+hi)>>1) = 9223372036854775805
  int32 lo = 2147483644, hi = 2147483646
    (lo+hi)/2 = -3, lo+(hi-lo)/2 = 2147483645, int32(uint32(lo+hi)>>1) = 2147483645

(b) search on the answer: first x in [MaxInt64-100, MaxInt64) with x >= MaxInt64-5
  buggy (lo+hi)/2    did not terminate within 200 iterations  probes 200, mids outside window 100, correct: false
  fixed lo+(hi-lo)/2 answer MaxInt64-5                        probes   7, mids outside window   0, correct: true
  the window holds 100 values: ceil(log2(101)) = 7 probes is the budget
  the buggy loop's first probes -- hi never moves, lo jumps out of the window:
    lo =  9223372036854775707  mid =                  -51  p(mid) = false
    lo =                  -50  mid =  4611686018427387878  p(mid) = false
    lo =  4611686018427387879  mid = -2305843009213693965  p(mid) = false
    lo = -2305843009213693964  mid =  3458764513820540921  p(mid) = false

(c) int32 index over a virtual array of 2,000,000,000 (no allocation)
  buggy    probe 2: mid = -647483647 from lo = 1000000001, hi = 2000000000 -> out of range
  subtract answer 1500000000 in 30 probes, correct: true
  unsigned answer 1500000000 in 30 probes, correct: true
  floor/ceil(log2(n+1)) = 30/31: the probe budget for n = 2e9

the fix costs nothing: same loop, same probe budget. The bug only needs lo + hi
past the type's maximum -- never for a slice index on 64-bit, easily for an int64
value range or an int32 index.
```

**Rust:**

```rust
// The Bloch bug: mid = (lo + hi) / 2 overflows when lo + hi exceeds the type.
// Go wraps silently. Rust's plain `+` does one of two things:
//   built WITHOUT -O (debug):  panics, "attempt to add with overflow"
//   built WITH -O (release):   wraps, exactly like Go (unless -C overflow-checks=on)
// So the same source fails loudly in tests and silently in production. The idiom
// is to say what you mean: wrapping_add, checked_add, overflowing_add, or
// midpoint. This program never uses a plain `+` that can overflow, so it prints
// the same thing in both builds -- and never panics.

/// Which midpoint formula a search uses.
#[derive(Clone, Copy, PartialEq)]
enum Mid {
    Buggy,    // (lo + hi) / 2, written as wrapping_add: what -O compiles `+` to
    Subtract, // lo + (hi - lo) / 2
    Unsigned, // ((lo + hi) as unsigned) >> 1: Go's library trick
}

/// First x in [lo, hi) with p(x) true, or hi if none, capped at `limit` iterations.
/// Returns (Some(answer) or None if the cap was hit, probes, mids outside [lo, hi)).
fn first_true64(
    mut lo: i64,
    mut hi: i64,
    p: impl Fn(i64) -> bool,
    mid_of: Mid,
    limit: u32,
) -> (Option<i64>, u32, u32) {
    let (mut probes, mut outside) = (0, 0);
    while lo < hi {
        if probes == limit {
            return (None, probes, outside);
        }
        let mid = match mid_of {
            Mid::Buggy => lo.wrapping_add(hi) / 2, // wraps when lo+hi > i64::MAX
            _ => lo + (hi - lo) / 2,               // hi-lo fits: lo < hi, both in range
        };
        probes += 1;
        if mid < lo || mid >= hi {
            outside += 1;
        }
        if p(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    (Some(lo), probes, outside)
}

/// Midpoint of an i32 window by the chosen formula.
fn mid32(lo: i32, hi: i32, form: Mid) -> i32 {
    match form {
        Mid::Buggy => lo.wrapping_add(hi) / 2,
        Mid::Subtract => lo + (hi - lo) / 2,
        Mid::Unsigned => ((lo as u32).wrapping_add(hi as u32) >> 1) as i32,
    }
}

fn bits_len(n: u64) -> u32 {
    u64::BITS - n.leading_zeros()
}

fn main() {
    println!(
        "(build) debug_assertions = {}: a plain lo + hi here would {}\n",
        cfg!(debug_assertions),
        if cfg!(debug_assertions) {
            "panic"
        } else {
            "wrap silently, like Go"
        }
    );

    println!("(a) the midpoint, every way Rust lets you write it, near the top of i64 and i32");
    let (lo, hi) = (i64::MAX - 3, i64::MAX - 1);
    println!("  i64 lo = {lo}, hi = {hi}");
    println!(
        "    lo.wrapping_add(hi) / 2  = {}   <- Go's (lo+hi)/2",
        lo.wrapping_add(hi) / 2
    );
    println!(
        "    lo.checked_add(hi)       = {:?}   <- what a debug build panics on",
        lo.checked_add(hi)
    );
    println!(
        "    lo.overflowing_add(hi)   = {:?}",
        lo.overflowing_add(hi)
    );
    println!("    lo + (hi - lo) / 2       = {}", lo + (hi - lo) / 2);
    println!(
        "    (sum as u64 >> 1) as i64 = {}",
        ((lo as u64).wrapping_add(hi as u64) >> 1) as i64
    );
    println!("    lo.midpoint(hi)          = {}", lo.midpoint(hi));
    let (l32, h32) = (i32::MAX - 3, i32::MAX - 1);
    println!("  i32 lo = {l32}, hi = {h32}");
    println!(
        "    wrapping/2 = {}, subtract = {}, unsigned = {}, checked_add = {:?}, midpoint = {}\n",
        mid32(l32, h32, Mid::Buggy),
        mid32(l32, h32, Mid::Subtract),
        mid32(l32, h32, Mid::Unsigned),
        l32.checked_add(h32),
        l32.midpoint(h32)
    );
    // the unsigned trick works because two NON-NEGATIVE values sum to at most
    // 2*MAX < 2^64 (2^32): the wrapped bits, read as unsigned, are the true sum.
    // midpoint on signed ints rounds toward zero -- the same as /2 for these.

    println!("(b) search on the answer: first x in [MaxInt64-100, MaxInt64) with x >= MaxInt64-5");
    let target = i64::MAX - 5;
    let p = |x: i64| x >= target;
    const LIMIT: u32 = 200;
    for (name, form) in [
        ("buggy (lo+hi)/2   ", Mid::Buggy),
        ("fixed lo+(hi-lo)/2", Mid::Subtract),
    ] {
        let (ans, probes, outside) = first_true64(i64::MAX - 100, i64::MAX, p, form, LIMIT);
        let verdict = match ans {
            Some(a) => format!("answer MaxInt64-{}", i64::MAX - a),
            None => format!("did not terminate within {LIMIT} iterations"),
        };
        let ok = ans == Some(target);
        println!(
            "  {name} {verdict:<40} probes {probes:>3}, mids outside window {outside:>3}, correct: {ok}"
        );
    }
    println!(
        "  the window holds 100 values: ceil(log2(101)) = {} probes is the budget",
        bits_len(100)
    );
    println!("  the buggy loop's first probes -- hi never moves, lo jumps out of the window:");
    let (mut l, mut h) = (i64::MAX - 100, i64::MAX);
    for _ in 0..4 {
        let mid = l.wrapping_add(h) / 2;
        println!("    lo = {l:>20}  mid = {mid:>20}  p(mid) = {}", p(mid));
        if p(mid) {
            h = mid;
        } else {
            l = mid + 1;
        }
    }
    println!();

    println!("(c) int32 index over a virtual array of 2,000,000,000 (no allocation)");
    const N32: i32 = 2_000_000_000;
    let want: i32 = 1_500_000_000;
    for (name, form) in [
        ("buggy", Mid::Buggy),
        ("subtract", Mid::Subtract),
        ("unsigned", Mid::Unsigned),
    ] {
        let (mut lo, mut hi, mut probes) = (0i32, N32, 0);
        let mut result = None;
        while lo < hi {
            let mid = mid32(lo, hi, form);
            probes += 1;
            // To index a real slice the i32 must become a usize: try_from refuses a
            // negative value, so the bug surfaces as an Err here, not a wild read.
            match usize::try_from(mid) {
                Ok(i) if i < N32 as usize => {}
                _ => {
                    result = Some(format!(
                        "probe {probes}: mid = {mid} from lo = {lo}, hi = {hi} -> out of range"
                    ));
                    break;
                }
            }
            if mid >= want {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        let result = result
            .unwrap_or_else(|| format!("answer {lo} in {probes} probes, correct: {}", lo == want));
        println!("  {name:<8} {result}");
    }
    println!(
        "  floor/ceil(log2(n+1)) = {}/{}: the probe budget for n = 2e9\n",
        bits_len(N32 as u64 + 1) - 1,
        bits_len(N32 as u64)
    );

    println!("the fix costs nothing: same loop, same probe budget. The bug only needs lo + hi");
    println!("past the type's maximum -- never for a slice index on 64-bit, easily for an i64");
    println!("value range or an i32 index. In Rust, write the intent: midpoint or checked_add.");
}
```

**Rust output:**

```
(build) debug_assertions = false: a plain lo + hi here would wrap silently, like Go

(a) the midpoint, every way Rust lets you write it, near the top of i64 and i32
  i64 lo = 9223372036854775804, hi = 9223372036854775806
    lo.wrapping_add(hi) / 2  = -3   <- Go's (lo+hi)/2
    lo.checked_add(hi)       = None   <- what a debug build panics on
    lo.overflowing_add(hi)   = (-6, true)
    lo + (hi - lo) / 2       = 9223372036854775805
    (sum as u64 >> 1) as i64 = 9223372036854775805
    lo.midpoint(hi)          = 9223372036854775805
  i32 lo = 2147483644, hi = 2147483646
    wrapping/2 = -3, subtract = 2147483645, unsigned = 2147483645, checked_add = None, midpoint = 2147483645

(b) search on the answer: first x in [MaxInt64-100, MaxInt64) with x >= MaxInt64-5
  buggy (lo+hi)/2    did not terminate within 200 iterations  probes 200, mids outside window 100, correct: false
  fixed lo+(hi-lo)/2 answer MaxInt64-5                        probes   7, mids outside window   0, correct: true
  the window holds 100 values: ceil(log2(101)) = 7 probes is the budget
  the buggy loop's first probes -- hi never moves, lo jumps out of the window:
    lo =  9223372036854775707  mid =                  -51  p(mid) = false
    lo =                  -50  mid =  4611686018427387878  p(mid) = false
    lo =  4611686018427387879  mid = -2305843009213693965  p(mid) = false
    lo = -2305843009213693964  mid =  3458764513820540921  p(mid) = false

(c) int32 index over a virtual array of 2,000,000,000 (no allocation)
  buggy    probe 2: mid = -647483647 from lo = 1000000001, hi = 2000000000 -> out of range
  subtract answer 1500000000 in 30 probes, correct: true
  unsigned answer 1500000000 in 30 probes, correct: true
  floor/ceil(log2(n+1)) = 30/31: the probe budget for n = 2e9

the fix costs nothing: same loop, same probe budget. The bug only needs lo + hi
past the type's maximum -- never for a slice index on 64-bit, easily for an i64
value range or an i32 index. In Rust, write the intent: midpoint or checked_add.
```

> **Go vs Rust:** Parts (b) and (c) are identical to Go, line for line: the buggy search was written with `wrapping_add`, which is exactly what `-O` turns a plain `+` into, so it fails the same way. Built without `-O`, or with `-C overflow-checks=on`, a plain `lo + hi` panics with `attempt to add with overflow` (exit 101). That was a separate scratch run; this program never uses a plain `+` that can overflow. Go has no checked mode at all. Rust also refuses to turn the negative `i32` into an index: `usize::try_from(mid)` returns `Err`.

**Complexity:** the fix is free: same Θ(log n) loop · fixed, the value-range search takes 7 probes (⌈log₂ 101⌉) and the int32 search 30 (budget 30/31) · buggy, the value-range search never terminates within 200 iterations (hi never moves, half the mids land outside the window) and the int32 search produces a negative index on its 2nd probe.

---

## 3. Lower bound and upper bound

`🟢 easy` · *Half-open ranges*

Two predicates on the same loop, `a[i] >= x` (lower bound) and `a[i] > x` (upper bound), answer every
question you can ask about a value in a sorted slice with duplicates: is it present, how many copies
there are, where its run `a[lb:ub]` starts and ends, and where to insert a new copy. Inserting at `ub`
is a **stable** insert, so equal keys stay in arrival order; inserting at `lb` reverses them. Each bound
costs Θ(log n) probes. The insert that follows is the Θ(n) shift from lesson 06, and that shift is the
real cost.

**Steps:**

1. Build `lowerBound` and `upperBound` on the one `firstTrue` template, counting probes.
2. For every target 0..9 print `lb`, `ub`, the count, whether it is present and the run, and cross-check `lb` against `sort.SearchInts` and `slices.BinarySearch`.
3. Insert `key#arrival` records at `ub` and at `lb` and compare the order of the equal keys.

```go
package main

import (
	"fmt"
	"slices"
	"sort"
	"strings"
)

// Two predicates, one loop (example 1's template):
//   lowerBound(x) = first i with a[i] >= x
//   upperBound(x) = first i with a[i] >  x
// Everything else falls out of the pair:
//   present  = lb < n && a[lb] == x
//   count    = ub - lb
//   the run  = a[lb:ub]
//   stable insert position = ub (after the equal keys already there)

// firstTrue is the template; *probes counts the predicate calls.
func firstTrue(n int, p func(int) bool, probes *int) int {
	lo, hi := 0, n
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		*probes++
		if p(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func lowerBound(a []int, x int, probes *int) int {
	return firstTrue(len(a), func(i int) bool { return a[i] >= x }, probes)
}

func upperBound(a []int, x int, probes *int) int {
	return firstTrue(len(a), func(i int) bool { return a[i] > x }, probes)
}

type rec struct{ key, arrival int }

func show(rs []rec) string {
	parts := make([]string, len(rs))
	for i, r := range rs {
		parts[i] = fmt.Sprintf("%d#%d", r.key, r.arrival)
	}
	return strings.Join(parts, " ")
}

func main() {
	a := []int{1, 2, 2, 2, 3, 5, 5, 8}
	fmt.Printf("a = %v (n = %d)\n\n", a, len(a))
	fmt.Printf("  %2s %3s %3s %6s %8s  %-10s %s\n", "x", "lb", "ub", "count", "present", "a[lb:ub]", "probes lb+ub")
	agree := 0
	for x := 0; x <= 9; x++ {
		probes := 0
		lb, ub := lowerBound(a, x, &probes), upperBound(a, x, &probes)
		present := lb < len(a) && a[lb] == x
		fmt.Printf("  %2d %3d %3d %6d %8v  %-10v %d\n", x, lb, ub, ub-lb, present, fmt.Sprint(a[lb:ub]), probes)

		// cross-check lb against the two stdlib lower bounds
		bs, _ := slices.BinarySearch(a, x)
		if lb == sort.SearchInts(a, x) && lb == bs {
			agree++
		}
	}
	fmt.Printf("\n  lb == sort.SearchInts == slices.BinarySearch index: %d of 10 targets\n", agree)
	fmt.Println("  (both are documented lower bounds: on a run of duplicates they give its first index)")
	fmt.Println()

	// Insert records into a sorted slice, keeping it sorted by key.
	// At ub, a new record goes AFTER the equal keys already there: arrival order kept.
	// At lb, it goes BEFORE them: equal keys end up in reverse arrival order.
	keys := []int{5, 2, 8, 2, 5, 2, 1, 5}
	fmt.Printf("insert records key#arrival, keys %v in this order:\n", keys)
	for _, pos := range []string{"ub", "lb"} {
		var rs []rec
		for arrival, k := range keys {
			probes := 0
			p := func(i int) bool { return rs[i].key > k } // upper bound on key
			if pos == "lb" {
				p = func(i int) bool { return rs[i].key >= k } // lower bound on key
			}
			i := firstTrue(len(rs), p, &probes)
			rs = slices.Insert(rs, i, rec{k, arrival}) // Theta(n) shift; the search is Theta(log n)
		}
		stable := slices.IsSortedFunc(rs, func(x, y rec) int {
			if x.key != y.key {
				return x.key - y.key
			}
			return x.arrival - y.arrival
		})
		fmt.Printf("  at %s: %s   equal keys in arrival order: %v\n", pos, show(rs), stable)
	}
	fmt.Println()
	fmt.Println("both bounds are the same loop; only >= vs > differs. The insert itself is the")
	fmt.Println("Theta(n) shift from lesson 06 -- the search that finds the spot is not the cost.")
}
```

**Output:**

```
a = [1 2 2 2 3 5 5 8] (n = 8)

   x  lb  ub  count  present  a[lb:ub]   probes lb+ub
   0   0   0      0    false  []         8
   1   0   1      1     true  [1]        8
   2   1   4      3     true  [2 2 2]    7
   3   4   5      1     true  [3]        6
   4   5   5      0    false  []         6
   5   5   7      2     true  [5 5]      6
   6   7   7      0    false  []         6
   7   7   7      0    false  []         6
   8   7   8      1     true  [8]        6
   9   8   8      0    false  []         6

  lb == sort.SearchInts == slices.BinarySearch index: 10 of 10 targets
  (both are documented lower bounds: on a run of duplicates they give its first index)

insert records key#arrival, keys [5 2 8 2 5 2 1 5] in this order:
  at ub: 1#6 2#1 2#3 2#5 5#0 5#4 5#7 8#2   equal keys in arrival order: true
  at lb: 1#6 2#5 2#3 2#1 5#7 5#4 5#0 8#2   equal keys in arrival order: false

both bounds are the same loop; only >= vs > differs. The insert itself is the
Theta(n) shift from lesson 06 -- the search that finds the spot is not the cost.
```

**Rust:**

```rust
// Two predicates, one loop (example 1's template):
//   lower_bound(x) = first i with a[i] >= x
//   upper_bound(x) = first i with a[i] >  x
// Everything else falls out of the pair:
//   present  = lb < n && a[lb] == x
//   count    = ub - lb
//   the run  = &a[lb..ub]
//   stable insert position = ub (after the equal keys already there)
//
// std has both bounds as one method: partition_point(pred) returns the first index
// where pred is FALSE (the mirror of Go's first-true), so
//   lower bound = a.partition_point(|v| *v < x)
//   upper bound = a.partition_point(|v| *v <= x)
// We hand-write them first so the probe counts can be compared with Go's.

/// The template; `probes` counts the predicate calls.
fn first_true(n: usize, p: impl Fn(usize) -> bool, probes: &mut u32) -> usize {
    let (mut lo, mut hi) = (0, n);
    while lo < hi {
        let mid = lo.midpoint(hi);
        *probes += 1;
        if p(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

fn lower_bound(a: &[i64], x: i64, probes: &mut u32) -> usize {
    first_true(a.len(), |i| a[i] >= x, probes)
}

fn upper_bound(a: &[i64], x: i64, probes: &mut u32) -> usize {
    first_true(a.len(), |i| a[i] > x, probes)
}

/// Go's %v of a slice: [1 2 3]
fn list(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

#[derive(Clone, Copy)]
struct Rec {
    key: i64,
    arrival: usize,
}

fn show(rs: &[Rec]) -> String {
    let parts: Vec<String> = rs
        .iter()
        .map(|r| format!("{}#{}", r.key, r.arrival))
        .collect();
    parts.join(" ")
}

fn main() {
    let a: Vec<i64> = vec![1, 2, 2, 2, 3, 5, 5, 8];
    println!("a = {} (n = {})\n", list(&a), a.len());
    println!(
        "  {:>2} {:>3} {:>3} {:>6} {:>8}  {:<10} probes lb+ub",
        "x", "lb", "ub", "count", "present", "a[lb:ub]"
    );
    let (mut agree_pp, mut agree_bs, mut differ) = (0, 0, Vec::new());
    for x in 0..=9 {
        let mut probes = 0;
        let lb = lower_bound(&a, x, &mut probes);
        let ub = upper_bound(&a, x, &mut probes);
        let present = a.get(lb) == Some(&x); // get() folds the lb < n check in
        println!(
            "  {x:>2} {lb:>3} {ub:>3} {:>6} {present:>8}  {:<10} {probes}",
            ub - lb,
            list(&a[lb..ub])
        );

        // cross-check lb against std: partition_point is a guaranteed lower bound;
        // binary_search returns Ok(i) or Err(i) -- the index either way.
        if lb == a.partition_point(|v| *v < x) {
            agree_pp += 1;
        }
        let bs = a.binary_search(&x).unwrap_or_else(|i| i);
        if lb == bs {
            agree_bs += 1;
        } else {
            differ.push(format!("x = {x}: {:?} vs lb {lb}", a.binary_search(&x)));
        }
    }
    println!("\n  lb == partition_point(|v| *v < x): {agree_pp} of 10 targets");
    println!("  lb == binary_search index:         {agree_bs} of 10 targets");
    for d in &differ {
        println!("    differs at {d}");
    }
    if !differ.is_empty() {
        println!("  (binary_search promises SOME match on duplicates, not the first: example 4)");
    }
    println!();

    // Insert records into a sorted Vec, keeping it sorted by key.
    // At ub, a new record goes AFTER the equal keys already there: arrival order kept.
    // At lb, it goes BEFORE them: equal keys end up in reverse arrival order.
    let keys: [i64; 8] = [5, 2, 8, 2, 5, 2, 1, 5];
    println!(
        "insert records key#arrival, keys {} in this order:",
        list(&keys)
    );
    for pos in ["ub", "lb"] {
        let mut rs: Vec<Rec> = Vec::new();
        for (arrival, &k) in keys.iter().enumerate() {
            let i = if pos == "ub" {
                rs.partition_point(|r| r.key <= k) // upper bound on key
            } else {
                rs.partition_point(|r| r.key < k) // lower bound on key
            };
            rs.insert(i, Rec { key: k, arrival }); // Theta(n) shift; the search is Theta(log n)
        }
        let stable = rs.is_sorted_by_key(|r| (r.key, r.arrival));
        println!(
            "  at {pos}: {}   equal keys in arrival order: {stable}",
            show(&rs)
        );
    }
    println!();
    println!("both bounds are the same loop; only >= vs > differs. The insert itself is the");
    println!("Theta(n) shift from lesson 06 -- the search that finds the spot is not the cost.");
}
```

**Rust output:**

```
a = [1 2 2 2 3 5 5 8] (n = 8)

   x  lb  ub  count  present  a[lb:ub]   probes lb+ub
   0   0   0      0    false  []         8
   1   0   1      1     true  [1]        8
   2   1   4      3     true  [2 2 2]    7
   3   4   5      1     true  [3]        6
   4   5   5      0    false  []         6
   5   5   7      2     true  [5 5]      6
   6   7   7      0    false  []         6
   7   7   7      0    false  []         6
   8   7   8      1     true  [8]        6
   9   8   8      0    false  []         6

  lb == partition_point(|v| *v < x): 10 of 10 targets
  lb == binary_search index:         8 of 10 targets
    differs at x = 2: Ok(3) vs lb 1
    differs at x = 5: Ok(6) vs lb 5
  (binary_search promises SOME match on duplicates, not the first: example 4)

insert records key#arrival, keys [5 2 8 2 5 2 1 5] in this order:
  at ub: 1#6 2#1 2#3 2#5 5#0 5#4 5#7 8#2   equal keys in arrival order: true
  at lb: 1#6 2#5 2#3 2#1 5#7 5#4 5#0 8#2   equal keys in arrival order: false

both bounds are the same loop; only >= vs > differs. The insert itself is the
Theta(n) shift from lesson 06 -- the search that finds the spot is not the cost.
```

> **Go vs Rust:** The table and the probe counts are identical to Go. The cross-check is where they differ: `partition_point(|v| *v < x)` matches `lb` on 10 of 10 targets, but `binary_search` matches on only 8. On the runs of 2s and 5s it returned `Ok(3)` and `Ok(6)`, the last copy in each run, where Go's `slices.BinarySearch` returned the first. The records use `partition_point` with `<=` / `<` for the two bounds and `Vec::insert` for the shift.

**Complexity:** Θ(log n) per bound: 6–8 probes for both bounds together at n = 8, i.e. 3 or 4 each · count and equal range Θ(log n) · the insert is a Θ(n) shift · `lb` matched `sort.SearchInts` and `slices.BinarySearch` on 10 of 10 targets · inserting at `ub` kept arrival order and inserting at `lb` reversed it.

---

## 4. Which duplicate do you get?

`🟢 easy` · *Stdlib contracts on duplicates*

Searching for 1 in a slice that holds four 1s can return any of them, and the two standard libraries
answer differently. Go's `slices.BinarySearch` is documented to return *"the earliest position where
target is found"*: it is a lower bound, so you get the first of the run, by contract. This program shows
that with `slices.BinarySearch`, `sort.Find` and `sort.SearchInts` on the array from Rust's docs, then on
runs of k equal elements, where it also counts comparator calls (the lower-bound loop plus one final
call, Θ(log n)), and finally on records sorted by age. The Rust twin shows the other answer.

**Steps:**

1. Search the doc array `[0,1,1,1,1,2,3,5,…]` for 1, 4, 13 and 100 with the three Go tools.
2. Call `slices.BinarySearchFunc` on k copies of 7 for k = 1..9 with a counting comparator, and compare the calls with `bits.Len(k) + 1`.
3. Search people sorted by age for age 30 and see which person comes back.

```go
package main

import (
	"cmp"
	"fmt"
	"math/bits"
	"slices"
	"sort"
)

// "Find 1" in a slice holding four 1s: which one do you get?
//
// Go's answer is written in the docs of slices.BinarySearch: "the earliest
// position where target is found". It is a lower bound (example 3), so the
// FIRST of a run -- guaranteed, not an accident of today's code.
//
// The API shape is (index, found): index is the match, or the insert position.

type person struct {
	name string
	age  int
}

func main() {
	// (a) the array from Rust's binary_search docs, searched with Go's three tools
	a := []int{0, 1, 1, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55}
	fmt.Printf("(a) a = %v\n", a)
	fmt.Printf("  %6s  %-24s %-18s %s\n", "target", "slices.BinarySearch", "sort.Find", "sort.SearchInts")
	agree := true
	for _, t := range []int{1, 4, 13, 100} {
		i, ok := slices.BinarySearch(a, t)
		fi, fok := sort.Find(len(a), func(k int) int { return cmp.Compare(t, a[k]) })
		si := sort.SearchInts(a, t)
		fmt.Printf("  %6d  %-24s %-18s %d\n", t,
			fmt.Sprintf("(%d, %v)", i, ok), fmt.Sprintf("(%d, %v)", fi, fok), si)
		agree = agree && i == fi && i == si && ok == fok
	}
	first1, _ := slices.BinarySearch(a, 1)
	fmt.Printf("  all three agree on every target: %v; target 1 -> index %d, and a[%d-1] = %d,\n",
		agree, first1, first1, a[first1-1])
	fmt.Println("  so it is the first of the run of 1s. A miss returns the insert position.")
	fmt.Println()

	// (b) k equal elements: index returned and comparator calls.
	// BinarySearchFunc runs the lower-bound loop (one cmp per iteration), then
	// makes ONE more call to decide found: `i < n && cmp(x[i], target) == 0`
	// (src/slices/sort.go -- read in the source).
	fmt.Println("(b) k copies of 7, slices.BinarySearchFunc(x, 7, counting cmp):")
	fmt.Printf("  %2s %-10s %6s %16s\n", "k", "(i, found)", "calls", "bits.Len(k) + 1")
	allFirst, allPredicted := true, true
	for k := 1; k <= 9; k++ {
		x := slices.Repeat([]int{7}, k)
		calls := 0
		i, ok := slices.BinarySearchFunc(x, 7, func(e, t int) int {
			calls++
			return cmp.Compare(e, t)
		})
		// on an all-equal slice every probe says ">=", so the loop only moves
		// hi down: floor(log2 k)+1 = bits.Len(k) probes, then the final call
		predicted := bits.Len(uint(k)) + 1
		fmt.Printf("  %2d %-10s %6d %16d\n", k, fmt.Sprintf("(%d, %v)", i, ok), calls, predicted)
		allFirst = allFirst && i == 0
		allPredicted = allPredicted && calls == predicted
	}
	fmt.Printf("  index 0 (the first copy) for every k: %v; calls == loop probes + 1 final: %v\n\n",
		allFirst, allPredicted)

	// (c) records sorted by age: which person aged 30?
	people := []person{
		{"Ana", 25}, {"Ben", 30}, {"Cleo", 30}, {"Dev", 30}, {"Eli", 41},
	}
	i, ok := slices.BinarySearchFunc(people, 30, func(p person, age int) int {
		return cmp.Compare(p.age, age)
	})
	fmt.Printf("(c) people by age %v\n", people)
	fmt.Printf("  BinarySearchFunc(age 30) -> (%d, %v): %s, the first 30-year-old\n", i, ok, people[i].name)
	fmt.Println()
	fmt.Println("Go returns the first of equal elements, by contract. Code that needs the")
	fmt.Println("whole run uses this index as its start and an upper bound as its end.")
}
```

**Output:**

```
(a) a = [0 1 1 1 1 2 3 5 8 13 21 34 55]
  target  slices.BinarySearch      sort.Find          sort.SearchInts
       1  (1, true)                (1, true)          1
       4  (7, false)               (7, false)         7
      13  (9, true)                (9, true)          9
     100  (13, false)              (13, false)        13
  all three agree on every target: true; target 1 -> index 1, and a[1-1] = 0,
  so it is the first of the run of 1s. A miss returns the insert position.

(b) k copies of 7, slices.BinarySearchFunc(x, 7, counting cmp):
   k (i, found)  calls  bits.Len(k) + 1
   1 (0, true)       2                2
   2 (0, true)       3                3
   3 (0, true)       3                3
   4 (0, true)       4                4
   5 (0, true)       4                4
   6 (0, true)       4                4
   7 (0, true)       4                4
   8 (0, true)       5                5
   9 (0, true)       5                5
  index 0 (the first copy) for every k: true; calls == loop probes + 1 final: true

(c) people by age [{Ana 25} {Ben 30} {Cleo 30} {Dev 30} {Eli 41}]
  BinarySearchFunc(age 30) -> (1, true): Ben, the first 30-year-old

Go returns the first of equal elements, by contract. Code that needs the
whole run uses this index as its start and an upper bound as its end.
```

**Rust:**

```rust
use std::cell::Cell;
use std::fmt;

// "Find 1" in a slice holding four 1s: which one do you get?
//
// Rust's answer is in the docs of binary_search (core/src/slice/mod.rs):
//   "If there are multiple matches, then any one of the matches could be
//    returned. The index is chosen deterministically, but is subject to change
//    in future versions of Rust."
// So: SOME match, no promise which. Today's loop (read in the source) never
// stops on Equal and moves right on Less OR Equal -- `base = if cmp == Greater
// { base } else { mid }` -- so it lands on the LAST of a run. Don't rely on it.
// The guaranteed first is partition_point(|v| *v < x): a lower bound.
//
// The API shape is Result<usize, usize>: Ok(i) a match, Err(i) the insert
// position. Go's (index, found) pair, as a type you have to match on.

const DOC: [&str; 2] = [
    "any one of the matches could be returned. The index is chosen",
    "deterministically, but is subject to change in future versions of Rust.",
];

struct Person {
    name: &'static str,
    age: u32,
}

impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{{{} {}}}", self.name, self.age)
    }
}

/// Go's %v of a slice: [1 2 3]
fn list<T: fmt::Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn ceil_log2(k: usize) -> u32 {
    usize::BITS - (k - 1).leading_zeros() // k >= 1
}

fn main() {
    // (a) the array from the binary_search docs, searched with three std tools
    let a: Vec<i64> = vec![0, 1, 1, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55];
    println!("(a) a = {}", list(&a));
    println!(
        "  {:>6}  {:<16} {:<22} partition_point(<t)",
        "target", "binary_search", "binary_search_by(cmp)"
    );
    for t in [1, 4, 13, 100] {
        let bs = a.binary_search(&t);
        let by = a.binary_search_by(|p| p.cmp(&t)); // what binary_search calls
        let pp = a.partition_point(|v| *v < t);
        println!(
            "  {t:>6}  {:<16} {:<22} {pp}",
            format!("{bs:?}"),
            format!("{by:?}")
        );
    }
    let (bs1, pp1) = (a.binary_search(&1), a.partition_point(|v| *v < 1));
    if let Ok(i) = bs1 {
        let last = a.get(i + 1) != Some(&1);
        println!(
            "  target 1 -> binary_search Ok({i}), and a[{i}+1] = {}: the last of the run: {last}",
            a[i + 1]
        );
    }
    println!(
        "  partition_point gives {pp1}, the first 1 -- Go's answer. A miss is Err(insert position)."
    );
    println!("  the docs: \"{}\n             {}\"\n", DOC[0], DOC[1]);

    // (b) k equal elements: index returned and comparator calls.
    // binary_search_by halves `size` while size > 1 (one call each), then makes
    // ONE final call: ceil(log2 k) + 1 calls for every target -- read in the source.
    println!("(b) k copies of 7, binary_search_by(counting cmp):");
    println!(
        "  {:>2} {:<10} {:>6} {:>18} {:>16}",
        "k", "result", "calls", "ceil(log2 k) + 1", "partition_point"
    );
    let (mut all_last, mut all_predicted) = (true, true);
    for k in 1..=9 {
        let x = vec![7i64; k];
        let calls = Cell::new(0u32); // Cell: the closure borrows it shared
        let r = x.binary_search_by(|e| {
            calls.set(calls.get() + 1);
            e.cmp(&7)
        });
        let predicted = ceil_log2(k) + 1;
        let pp = x.partition_point(|v| *v < 7);
        println!(
            "  {k:>2} {:<10} {:>6} {predicted:>18} {pp:>16}",
            format!("{r:?}"),
            calls.get()
        );
        all_last &= r == Ok(k - 1);
        all_predicted &= calls.get() == predicted;
    }
    println!(
        "  Ok(k-1) (the last copy) for every k: {all_last}; calls == ceil(log2 k) + 1: {all_predicted}\n"
    );

    // (c) records sorted by age: which person aged 30?
    let people = [
        Person {
            name: "Ana",
            age: 25,
        },
        Person {
            name: "Ben",
            age: 30,
        },
        Person {
            name: "Cleo",
            age: 30,
        },
        Person {
            name: "Dev",
            age: 30,
        },
        Person {
            name: "Eli",
            age: 41,
        },
    ];
    println!("(c) people by age {}", list(&people));
    match people.binary_search_by_key(&30, |p| p.age) {
        Ok(i) => println!(
            "  binary_search_by_key(&30, |p| p.age) -> Ok({i}): {}",
            people[i].name
        ),
        Err(i) => println!("  binary_search_by_key -> Err({i}): nobody aged 30"),
    }
    let first = people.partition_point(|p| p.age < 30);
    println!(
        "  partition_point(|p| p.age < 30)      -> {first}: {}, the first 30-year-old",
        people[first].name
    );
    println!();
    println!("Rust returns SOME equal element -- today the last -- and says it may change.");
    println!(
        "When the first one matters, ask for it: partition_point is a guaranteed lower bound."
    );
}
```

**Rust output:**

```
(a) a = [0 1 1 1 1 2 3 5 8 13 21 34 55]
  target  binary_search    binary_search_by(cmp)  partition_point(<t)
       1  Ok(4)            Ok(4)                  1
       4  Err(7)           Err(7)                 7
      13  Ok(9)            Ok(9)                  9
     100  Err(13)          Err(13)                13
  target 1 -> binary_search Ok(4), and a[4+1] = 2: the last of the run: true
  partition_point gives 1, the first 1 -- Go's answer. A miss is Err(insert position).
  the docs: "any one of the matches could be returned. The index is chosen
             deterministically, but is subject to change in future versions of Rust."

(b) k copies of 7, binary_search_by(counting cmp):
   k result      calls   ceil(log2 k) + 1  partition_point
   1 Ok(0)           1                  1                0
   2 Ok(1)           2                  2                0
   3 Ok(2)           3                  3                0
   4 Ok(3)           3                  3                0
   5 Ok(4)           4                  4                0
   6 Ok(5)           4                  4                0
   7 Ok(6)           4                  4                0
   8 Ok(7)           4                  4                0
   9 Ok(8)           5                  5                0
  Ok(k-1) (the last copy) for every k: true; calls == ceil(log2 k) + 1: true

(c) people by age [{Ana 25} {Ben 30} {Cleo 30} {Dev 30} {Eli 41}]
  binary_search_by_key(&30, |p| p.age) -> Ok(3): Dev
  partition_point(|p| p.age < 30)      -> 1: Ben, the first 30-year-old

Rust returns SOME equal element -- today the last -- and says it may change.
When the first one matters, ask for it: partition_point is a guaranteed lower bound.
```

> **Go vs Rust:** Same inputs, different answers. Rust's `binary_search` returned `Ok(4)` for the 1s, `Ok(k-1)` for every run of k copies, and Dev for age 30: the **last** match each time. Go returned index 1, index 0 and Ben, the first. In the source, today's loop moves right on `Less` or `Equal`, and the docs promise only "any one of the matches"; `partition_point` gives Go's answers. The comparator call counts differ too: Rust makes ⌈log₂ k⌉ + 1 calls (1 at k = 1, 4 at k = 8), Go makes ⌊log₂ k⌋ + 2 (2 and 5), because each library runs a differently shaped loop before its one final compare.

**Complexity:** Θ(log n) · `BinarySearchFunc` makes the lower-bound loop's probes plus 1 final compare: 2 calls for k = 1 up to 5 for k = 8–9 equal elements · Go returned index 0, the **first** duplicate, for every k, and Ben, the first 30-year-old. That is guaranteed by the documentation.

---

## 5. `sort.Search` is a predicate search

`🟢 easy` · *Monotone predicates*

`sort.Search(n, f)` never needs an array. It finds the first `i` in `[0, n)` where a monotone
false→true predicate becomes true, and `f` can compute its answer instead of looking anything up. That
one idea covers integer square roots, first-bad-version, the guessing game and lookups in sequences that
are never stored, each in Θ(log n) calls to `f`. It is also the bridge to search-on-the-answer
(examples 8–9). The program counts every call and checks the two promises in the docs: `f` is only
called with `i` in `[0, n)`, and the result is `n`, not −1, when nothing is true.

**Steps:**

1. Wrap each predicate in a counter that records calls and whether every `i` was in `[0, n)`.
2. Use `sort.Search` for `isqrt(10¹⁸)`, first bad version of 1,000,000, and every secret of a 0..99 guessing game; use `sort.Find` on the virtual sequence of squares.
3. Read the call counts against `bits.Len(n)`, and the return value when nothing is true.

```go
package main

import (
	"cmp"
	"fmt"
	"math/bits"
	"sort"
)

// sort.Search(n, f) returns the smallest i in [0, n) with f(i) true, or n if
// there is none -- for ANY f that is false...false true...true over [0, n).
// It never looks at an array; f can compute instead of index. That makes it the
// bridge to "search on the answer" (examples 8-9): the index IS the answer.

// counter wraps a predicate, counting calls and checking every i is in [0, n).
type counter struct {
	n, calls int
	inRange  bool
}

// every counter made, so main can check them all at the end
var made []*counter

func newCounter(n int) *counter {
	c := &counter{n: n, inRange: true}
	made = append(made, c)
	return c
}

func (c *counter) wrap(f func(int) bool) func(int) bool {
	return func(i int) bool {
		c.calls++
		c.inRange = c.inRange && 0 <= i && i < c.n
		return f(i)
	}
}

func main() {
	// (a) integer square root: the first i with i*i > x, minus one.
	// Search over [0, 1e9+1): i*i <= (1e9)^2 = 1e18 fits in int64 (max ~9.2e18).
	fmt.Println("(a) isqrt(x) = sort.Search(1e9+1, i*i > x) - 1")
	for _, x := range []int{1_000_000_000_000_000_000, 999_999_999_999_999_999} {
		c := newCounter(1_000_000_001)
		r := sort.Search(c.n, c.wrap(func(i int) bool { return i*i > x })) - 1
		ok := r*r <= x && x < (r+1)*(r+1)
		fmt.Printf("  isqrt(%d) = %d   r*r <= x < (r+1)^2: %v   calls %d\n", x, r, ok, c.calls)
	}
	fmt.Println()

	// (b) first bad version: an "API" isBad(v), monotone by construction.
	const versions, firstBad = 1_000_000, 777_777
	c := newCounter(versions)
	got := sort.Search(versions, c.wrap(func(v int) bool { return v >= firstBad }))
	fmt.Printf("(b) first bad of %d versions: %d (correct: %v), calls %d, bits.Len(n) = %d\n\n",
		versions, got, got == firstBad, c.calls, bits.Len(versions))

	// (c) "I'm thinking of a number in 0..99" -- each f call is one question
	// "is it <= g?". Play every secret and count questions.
	hist := map[int]int{}
	maxQ, allRight := 0, true
	for secret := 0; secret < 100; secret++ {
		c := newCounter(100)
		g := sort.Search(100, c.wrap(func(g int) bool { return secret <= g }))
		allRight = allRight && g == secret
		hist[c.calls]++
		maxQ = max(maxQ, c.calls)
	}
	fmt.Printf("(c) guessing game, every secret 0..99: all found: %v, max questions %d\n", allRight, maxQ)
	for q := 0; q <= maxQ; q++ {
		if hist[q] > 0 {
			fmt.Printf("  %d questions: %d secrets\n", q, hist[q])
		}
	}
	fmt.Println()

	// (d) sort.Find on a "virtual" sorted sequence: the squares 0, 1, 4, 9, ...
	// cmp(i) compares the target with element i; Find returns the first i with
	// cmp(i) <= 0, and found = i < n && cmp(i) == 0 (one final call to decide).
	fmt.Println("(d) sort.Find over the virtual sequence i*i, i in [0, 1000):")
	for _, target := range []int{144, 150, 998_001, 1_000_000} {
		calls := 0
		i, found := sort.Find(1000, func(i int) int { calls++; return cmp.Compare(target, i*i) })
		fmt.Printf("  %8d -> i = %4d, found %-5v  calls %d\n", target, i, found, calls)
	}
	fmt.Println()

	// nothing true -> n, not -1
	c = newCounter(10)
	none := sort.Search(10, c.wrap(func(int) bool { return false }))
	fmt.Printf("nothing true in [0, 10): Search returns %d (= n, not -1), calls %d\n", none, c.calls)
	all := true
	for _, c := range made {
		all = all && c.inRange
	}
	fmt.Printf("all %d counted searches called f only with i in [0, n): %v\n", len(made), all)
}
```

**Output:**

```
(a) isqrt(x) = sort.Search(1e9+1, i*i > x) - 1
  isqrt(1000000000000000000) = 1000000000   r*r <= x < (r+1)^2: true   calls 29
  isqrt(999999999999999999) = 999999999   r*r <= x < (r+1)^2: true   calls 30

(b) first bad of 1000000 versions: 777777 (correct: true), calls 20, bits.Len(n) = 20

(c) guessing game, every secret 0..99: all found: true, max questions 7
  6 questions: 26 secrets
  7 questions: 74 secrets

(d) sort.Find over the virtual sequence i*i, i in [0, 1000):
       144 -> i =   12, found true   calls 11
       150 -> i =   13, found false  calls 11
    998001 -> i =  999, found true   calls 11
   1000000 -> i = 1000, found false  calls 9

nothing true in [0, 10): Search returns 10 (= n, not -1), calls 3
all 104 counted searches called f only with i in [0, n): true
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::BTreeMap;

// Go's sort.Search(n, f): the smallest i in [0, n) with f(i) true, or n if none,
// for ANY f that is false...false true...true over [0, n). No array needed.
//
// Rust's std has this only for slices (partition_point) -- there is no search
// over an integer range. So we write the eight lines ourselves: example 1's
// template, with usize::midpoint for an overflow-free mid.

/// First i in [0, n) with f(i) true, or n.
fn search(n: usize, mut f: impl FnMut(usize) -> bool) -> usize {
    let (mut lo, mut hi) = (0, n);
    while lo < hi {
        let mid = lo.midpoint(hi);
        if f(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

/// Go's sort.Find in Rust's shape: the first i with cmp(i) != Greater, then one
/// final call to decide. Ok(i) = found at i, Err(i) = not found, would go at i.
fn find(n: usize, mut cmp: impl FnMut(usize) -> Ordering) -> Result<usize, usize> {
    let i = search(n, |i| cmp(i) != Ordering::Greater);
    if i < n && cmp(i) == Ordering::Equal {
        Ok(i)
    } else {
        Err(i)
    }
}

/// Every counted search adds itself here, so main can check them all at the end
/// (Rust has no mutable globals; the tally is passed by &mut instead).
struct Tally {
    searches: u32,
    all_in_range: bool,
}

/// search(n, f) with f counted and every i checked against [0, n).
fn counted(t: &mut Tally, n: usize, f: impl Fn(usize) -> bool) -> (usize, u32) {
    let (mut calls, mut in_range) = (0, true);
    let r = search(n, |i| {
        calls += 1;
        in_range &= i < n; // usize: i >= 0 is guaranteed by the type
        f(i)
    });
    t.searches += 1;
    t.all_in_range &= in_range;
    (r, calls)
}

fn bits_len(n: usize) -> u32 {
    usize::BITS - n.leading_zeros()
}

fn main() {
    let mut t = Tally {
        searches: 0,
        all_in_range: true,
    };

    // (a) integer square root: the first i with i*i > x, minus one.
    // u64: i*i <= (1e9)^2 = 1e18 fits easily (u64 max ~1.8e19).
    println!("(a) isqrt(x) = search(1e9+1, |i| i*i > x) - 1");
    for x in [1_000_000_000_000_000_000u64, 999_999_999_999_999_999] {
        let (i, calls) = counted(&mut t, 1_000_000_001, |i| (i as u64) * (i as u64) > x);
        let r = i as u64 - 1;
        let ok = r * r <= x && x < (r + 1) * (r + 1);
        println!("  isqrt({x}) = {r}   r*r <= x < (r+1)^2: {ok}   calls {calls}");
    }
    println!();

    // (b) first bad version: an "API" is_bad(v), monotone by construction.
    const VERSIONS: usize = 1_000_000;
    const FIRST_BAD: usize = 777_777;
    let (got, calls) = counted(&mut t, VERSIONS, |v| v >= FIRST_BAD);
    println!(
        "(b) first bad of {VERSIONS} versions: {got} (correct: {}), calls {calls}, bits.Len(n) = {}\n",
        got == FIRST_BAD,
        bits_len(VERSIONS)
    );

    // (c) "I'm thinking of a number in 0..99" -- each f call is one question
    // "is it <= g?". Play every secret and count questions.
    let mut hist: BTreeMap<u32, u32> = BTreeMap::new();
    let mut all_right = true;
    for secret in 0..100 {
        let (g, calls) = counted(&mut t, 100, |g| secret <= g);
        all_right &= g == secret;
        *hist.entry(calls).or_default() += 1;
    }
    let max_q = hist.keys().max().copied().unwrap_or(0);
    println!(
        "(c) guessing game, every secret 0..99: all found: {all_right}, max questions {max_q}"
    );
    for (q, secrets) in &hist {
        println!("  {q} questions: {secrets} secrets");
    }
    println!();

    // (d) find on a "virtual" sorted sequence: the squares 0, 1, 4, 9, ...
    println!("(d) find (our sort.Find) over the virtual sequence i*i, i in [0, 1000):");
    for target in [144usize, 150, 998_001, 1_000_000] {
        let mut calls = 0;
        let r = find(1000, |i| {
            calls += 1;
            target.cmp(&(i * i))
        });
        let (i, found) = match r {
            Ok(i) => (i, true),
            Err(i) => (i, false),
        };
        println!("  {target:>8} -> i = {i:>4}, found {found:<5}  calls {calls}");
    }
    println!();

    // nothing true -> n, not -1 (and no Option needed: n is the answer)
    let (none, calls) = counted(&mut t, 10, |_| false);
    println!("nothing true in [0, 10): search returns {none} (= n, not -1), calls {calls}");
    println!(
        "all {} counted searches called f only with i in [0, n): {}",
        t.searches, t.all_in_range
    );
}
```

**Rust output:**

```
(a) isqrt(x) = search(1e9+1, |i| i*i > x) - 1
  isqrt(1000000000000000000) = 1000000000   r*r <= x < (r+1)^2: true   calls 29
  isqrt(999999999999999999) = 999999999   r*r <= x < (r+1)^2: true   calls 30

(b) first bad of 1000000 versions: 777777 (correct: true), calls 20, bits.Len(n) = 20

(c) guessing game, every secret 0..99: all found: true, max questions 7
  6 questions: 26 secrets
  7 questions: 74 secrets

(d) find (our sort.Find) over the virtual sequence i*i, i in [0, 1000):
       144 -> i =   12, found true   calls 11
       150 -> i =   13, found false  calls 11
    998001 -> i =  999, found true   calls 11
   1000000 -> i = 1000, found false  calls 9

nothing true in [0, 10): search returns 10 (= n, not -1), calls 3
all 104 counted searches called f only with i in [0, n): true
```

> **Go vs Rust:** Every answer and every call count matches Go: 29/30, 20, 26 + 74 secrets, 11/11/11/9, 3. Only the labels differ, because Rust's std has no search over an integer range (`partition_point` needs a slice), so `search` and `find` are hand-written. `find` returns `Result<usize, usize>` instead of Go's `(i, found)` pair. With no mutable globals, the in-range tally is passed as `&mut Tally`, and `usize` makes the `i >= 0` half of the range check unnecessary.

**Complexity:** Θ(log n) calls to `f`: 29–30 for the square roots (n ≈ 10⁹), 20 for 1,000,000 versions, at most 7 questions for 100 secrets (26 need 6) · `sort.Find` adds one final call to decide `found` (11 at n = 1000) · nothing true → returns `n` · all 104 counted searches stayed inside `[0, n)`.

---

## 6. Three-way search is not faster

`🟢 easy` · *Comparison counting*

The textbook binary search (`==` → return, `<` → go right, else go left) looks faster because it can stop
early. Counting settles it. It pays up to two element comparisons per step, the early exit saves about
one step and only on hits, and on duplicates it returns whichever copy a probe happened to land on. The
boundary search pays exactly one comparison per step plus one equality check at the end. Both are
Θ(log n). The counts matter whenever a comparison is not a single machine instruction: strings, records,
a comparator function. For two `int`s the compiler may well do `==` and `<` in one compare, so the count
here measures comparator calls, not time.

**Steps:**

1. Write both versions over `n = 1000` values `0, 2, 4, …`, counting element comparisons (each `==` or `<` is one) and loop iterations.
2. Search for every one of the 1000 present values and every one of the 1001 gaps, and tabulate the average and maximum for each.
3. Search a run of five equal values and see which copy each version returns.

```go
package main

import "fmt"

// The textbook binary search exits early:
//     if a[mid] == x { return mid }  else if a[mid] < x { ... } else { ... }
// That looks faster -- it can stop before the window is empty. But it pays up to
// TWO element comparisons per step, the early exit only ever helps on a hit, and
// on duplicates it returns whichever copy it happens to land on.
// The boundary search (example 1) pays ONE comparison per step, plus one
// equality check at the end to decide "found". Count both and see.

// counts: element comparisons (each == or < is one) and loop iterations.
type counts struct{ cmps, steps int }

// threeWay is the closed-window textbook version.
func threeWay(a []int, x int, c *counts) (int, bool) {
	lo, hi := 0, len(a)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		c.steps++
		c.cmps++
		if a[mid] == x {
			return mid, true // the early exit
		}
		c.cmps++
		if a[mid] < x {
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return lo, false
}

// twoWay is the half-open lower bound + one final equality check.
func twoWay(a []int, x int, c *counts) (int, bool) {
	lo, hi := 0, len(a)
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		c.steps++
		c.cmps++
		if a[mid] < x {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	if lo == len(a) {
		return lo, false // nothing to compare: x is bigger than everything
	}
	c.cmps++
	return lo, a[lo] == x
}

// stats accumulates comparisons and steps over many searches.
type stats struct{ cmps, maxCmps, steps, count int }

func (s *stats) add(c counts) {
	s.cmps += c.cmps
	s.maxCmps = max(s.maxCmps, c.cmps)
	s.steps += c.steps
	s.count++
}

func (s stats) avg() float64      { return float64(s.cmps) / float64(s.count) }
func (s stats) avgSteps() float64 { return float64(s.steps) / float64(s.count) }

func main() {
	const n = 1000
	a := make([]int, n)
	for i := range a {
		a[i] = 2 * i // 0 2 4 ... 1998: hits are even, gaps are odd
	}
	type variant struct {
		name string
		f    func([]int, int, *counts) (int, bool)
	}
	variants := []variant{{"three-way (early exit)", threeWay}, {"two-way (lower bound + 1)", twoWay}}
	results := make([][2]stats, len(variants))
	for v, vr := range variants {
		for i := 0; i < n; i++ { // every hit
			var c counts
			if j, ok := vr.f(a, 2*i, &c); !ok || j != i {
				fmt.Println("WRONG on hit", 2*i)
			}
			results[v][0].add(c)
		}
		for i := 0; i <= n; i++ { // every gap: -1, 1, 3, ..., 1999
			var c counts
			if j, ok := vr.f(a, 2*i-1, &c); ok || j != i {
				fmt.Println("WRONG on miss", 2*i-1)
			}
			results[v][1].add(c)
		}
	}

	fmt.Printf("n = %d distinct values; %d successful and %d unsuccessful searches\n\n",
		n, results[0][0].count, results[0][1].count)
	fmt.Printf("  %-26s %9s %9s %10s %10s\n", "element comparisons", "hit avg", "hit max", "miss avg", "miss max")
	for v, vr := range variants {
		h, m := results[v][0], results[v][1]
		fmt.Printf("  %-26s %9.2f %9d %10.2f %10d\n", vr.name, h.avg(), h.maxCmps, m.avg(), m.maxCmps)
	}
	fmt.Printf("\n  %-26s %9s %20s\n", "loop iterations (avg)", "hits", "misses")
	for v, vr := range variants {
		fmt.Printf("  %-26s %9.2f %20.2f\n", vr.name, results[v][0].avgSteps(), results[v][1].avgSteps())
	}
	t, w := results[0], results[1]
	fmt.Println()
	fmt.Printf("  on hits the early exit saves %.2f iterations but costs %.2fx the comparisons;\n",
		w[0].avgSteps()-t[0].avgSteps(), t[0].avg()/w[0].avg())
	fmt.Printf("  on misses it saves %.2f iterations and costs %.2fx the comparisons.\n",
		w[1].avgSteps()-t[1].avgSteps(), t[1].avg()/w[1].avg())
	fmt.Println()

	// duplicates: which copy of a run of 5 does each one return?
	d := []int{1, 2, 3, 3, 3, 3, 3, 4, 5, 6}
	var c counts
	i3, _ := threeWay(d, 3, &c)
	i2, _ := twoWay(d, 3, &c)
	fmt.Printf("duplicates %v, search 3 (copies at 2..6):\n", d)
	fmt.Printf("  three-way returns index %d (wherever a mid happened to land)\n", i3)
	fmt.Printf("  two-way   returns index %d (the first copy, always)\n", i2)
}
```

**Output:**

```
n = 1000 distinct values; 1000 successful and 1001 unsuccessful searches

  element comparisons          hit avg   hit max   miss avg   miss max
  three-way (early exit)         16.97        19      19.95         20
  two-way (lower bound + 1)      10.98        11      10.98         11

  loop iterations (avg)           hits               misses
  three-way (early exit)          8.99                 9.98
  two-way (lower bound + 1)       9.98                 9.98

  on hits the early exit saves 0.99 iterations but costs 1.55x the comparisons;
  on misses it saves 0.00 iterations and costs 1.82x the comparisons.

duplicates [1 2 3 3 3 3 3 4 5 6], search 3 (copies at 2..6):
  three-way returns index 4 (wherever a mid happened to land)
  two-way   returns index 2 (the first copy, always)
```

**Rust:**

```rust
use std::cell::Cell;

// The textbook binary search exits early:
//     if a[mid] == x { return mid } else if a[mid] < x { ... } else { ... }
// That looks faster -- it can stop before the window is empty. But it pays up to
// TWO element comparisons per step, the early exit only ever helps on a hit, and
// on duplicates it returns whichever copy it happens to land on.
// The boundary search (example 1) pays ONE comparison per step, plus one
// equality check at the end to decide "found". Count both and see.
//
// Rust adds a third row: std's binary_search_by, counted through its comparator.
// One call returns an Ordering, so it counts as ONE comparison. Its loop has no
// early exit on Equal, by design (read in the source: the iteration count
// depends only on the length, so the CPU can predict it).

/// Element comparisons (each == or < is one) and loop iterations.
#[derive(Default)]
struct Counts {
    cmps: u32,
    steps: u32,
}

/// The closed-window textbook version. Ok(i) found at i, Err(i) insert position.
/// The window is kept as [lo, hi) internally: a closed `hi = mid - 1` would
/// underflow usize at mid = 0 (example 1). The comparisons are the same.
fn three_way(a: &[i64], x: i64, c: &mut Counts) -> Result<usize, usize> {
    let (mut lo, mut hi) = (0, a.len()); // closed [lo, hi-1]
    while lo < hi {
        let mid = lo + (hi - 1 - lo) / 2; // the closed template's mid
        c.steps += 1;
        c.cmps += 1;
        if a[mid] == x {
            return Ok(mid); // the early exit
        }
        c.cmps += 1;
        if a[mid] < x {
            lo = mid + 1;
        } else {
            hi = mid; // closed: hi-1 = mid-1
        }
    }
    Err(lo)
}

/// The half-open lower bound + one final equality check.
fn two_way(a: &[i64], x: i64, c: &mut Counts) -> Result<usize, usize> {
    let (mut lo, mut hi) = (0, a.len());
    while lo < hi {
        let mid = lo.midpoint(hi);
        c.steps += 1;
        c.cmps += 1;
        if a[mid] < x {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    match a.get(lo) {
        None => Err(lo), // nothing to compare: x is bigger than everything
        Some(&v) => {
            c.cmps += 1;
            if v == x { Ok(lo) } else { Err(lo) }
        }
    }
}

/// std's binary_search_by, with a counting comparator (steps not observable).
fn std_search(a: &[i64], x: i64, c: &mut Counts) -> Result<usize, usize> {
    let calls = Cell::new(0);
    let r = a.binary_search_by(|e| {
        calls.set(calls.get() + 1);
        e.cmp(&x)
    });
    c.cmps += calls.get();
    r
}

/// Accumulates comparisons and steps over many searches.
#[derive(Default, Clone, Copy)]
struct Stats {
    cmps: u32,
    min_cmps: u32,
    max_cmps: u32,
    steps: u32,
    count: u32,
}

impl Stats {
    fn add(&mut self, c: &Counts) {
        self.min_cmps = if self.count == 0 {
            c.cmps
        } else {
            self.min_cmps.min(c.cmps)
        };
        self.cmps += c.cmps;
        self.max_cmps = self.max_cmps.max(c.cmps);
        self.steps += c.steps;
        self.count += 1;
    }
    fn avg(&self) -> f64 {
        f64::from(self.cmps) / f64::from(self.count)
    }
    fn avg_steps(&self) -> f64 {
        f64::from(self.steps) / f64::from(self.count)
    }
}

type Search = fn(&[i64], i64, &mut Counts) -> Result<usize, usize>;

fn main() {
    const N: usize = 1000;
    let a: Vec<i64> = (0..N as i64).map(|i| 2 * i).collect(); // hits even, gaps odd
    let variants: [(&str, Search); 3] = [
        ("three-way (early exit)", three_way),
        ("two-way (lower bound + 1)", two_way),
        ("std binary_search_by", std_search),
    ];
    let mut results = [[Stats::default(); 2]; 3];
    for (v, (_, f)) in variants.iter().enumerate() {
        for i in 0..N {
            // every hit
            let mut c = Counts::default();
            if f(&a, 2 * i as i64, &mut c) != Ok(i) {
                println!("WRONG on hit {}", 2 * i);
            }
            results[v][0].add(&c);
        }
        for i in 0..=N {
            // every gap: -1, 1, 3, ..., 1999
            let mut c = Counts::default();
            if f(&a, 2 * i as i64 - 1, &mut c) != Err(i) {
                println!("WRONG on miss {}", 2 * i as i64 - 1);
            }
            results[v][1].add(&c);
        }
    }

    println!(
        "n = {N} distinct values; {} successful and {} unsuccessful searches\n",
        results[0][0].count, results[0][1].count
    );
    println!(
        "  {:<26} {:>9} {:>9} {:>10} {:>10}",
        "element comparisons", "hit avg", "hit max", "miss avg", "miss max"
    );
    for (v, (name, _)) in variants.iter().enumerate() {
        let [h, m] = results[v];
        println!(
            "  {name:<26} {:>9.2} {:>9} {:>10.2} {:>10}",
            h.avg(),
            h.max_cmps,
            m.avg(),
            m.max_cmps
        );
    }
    println!(
        "\n  {:<26} {:>9} {:>20}",
        "loop iterations (avg)", "hits", "misses"
    );
    for (v, (name, _)) in variants.iter().take(2).enumerate() {
        println!(
            "  {name:<26} {:>9.2} {:>20.2}",
            results[v][0].avg_steps(),
            results[v][1].avg_steps()
        );
    }
    let [t, w, s] = results;
    println!();
    println!(
        "  on hits the early exit saves {:.2} iterations but costs {:.2}x the comparisons;",
        w[0].avg_steps() - t[0].avg_steps(),
        t[0].avg() / w[0].avg()
    );
    println!(
        "  on misses it saves {:.2} iterations and costs {:.2}x the comparisons.",
        w[1].avg_steps() - t[1].avg_steps(),
        t[1].avg() / w[1].avg()
    );
    let (lo, hi) = (
        s[0].min_cmps.min(s[1].min_cmps),
        s[0].max_cmps.max(s[1].max_cmps),
    );
    let ceil_log2 = usize::BITS - (N - 1).leading_zeros();
    println!(
        "  std binary_search_by: {lo}..={hi} comparator calls on every one of the {} searches",
        s[0].count + s[1].count
    );
    println!(
        "  (ceil(log2 {N}) + 1 = {}): hit or miss, no early exit.",
        ceil_log2 + 1
    );
    println!();

    // duplicates: which copy of a run of 5 does each one return?
    let d: [i64; 10] = [1, 2, 3, 3, 3, 3, 3, 4, 5, 6];
    let mut c = Counts::default();
    let i3 = three_way(&d, 3, &mut c);
    let i2 = two_way(&d, 3, &mut c);
    let is = d.binary_search(&3);
    println!("duplicates [1 2 3 3 3 3 3 4 5 6], search 3 (copies at 2..6):");
    println!("  three-way returns {i3:?} (wherever a mid happened to land)");
    println!("  two-way   returns {i2:?} (the first copy, always)");
    let which = match is {
        Ok(6) => "the last copy",
        Ok(2) => "the first copy",
        _ => "a middle copy",
    };
    println!("  std binary_search returns {is:?} ({which}, in this version; not promised)");
}
```

**Rust output:**

```
n = 1000 distinct values; 1000 successful and 1001 unsuccessful searches

  element comparisons          hit avg   hit max   miss avg   miss max
  three-way (early exit)         16.97        19      19.95         20
  two-way (lower bound + 1)      10.98        11      10.98         11
  std binary_search_by           11.00        11      11.00         11

  loop iterations (avg)           hits               misses
  three-way (early exit)          8.99                 9.98
  two-way (lower bound + 1)       9.98                 9.98

  on hits the early exit saves 0.99 iterations but costs 1.55x the comparisons;
  on misses it saves 0.00 iterations and costs 1.82x the comparisons.
  std binary_search_by: 11..=11 comparator calls on every one of the 2001 searches
  (ceil(log2 1000) + 1 = 11): hit or miss, no early exit.

duplicates [1 2 3 3 3 3 3 4 5 6], search 3 (copies at 2..6):
  three-way returns Ok(4) (wherever a mid happened to land)
  two-way   returns Ok(2) (the first copy, always)
  std binary_search returns Ok(6) (the last copy, in this version; not promised)
```

> **Go vs Rust:** The two hand-written rows are identical to Go: 16.97/19.95 against 10.98, and 1.55x and 1.82x. The Rust three-way stores its closed window as `[lo, hi)` so that `hi = mid - 1` cannot underflow `usize`; the mids are the same. The extra row is std's `binary_search_by`, which made exactly 11 comparator calls (⌈log₂ 1000⌉ + 1) on all 2001 searches. It never exits early, by design: one `Ordering` call per step plus one final call. On the run of 3s it returned `Ok(6)`, the last copy, where the two-way search returned the first, as Go does.

**Complexity:** both Θ(log n) · three-way: 16.97 comparisons per hit (max 19) and 19.95 per miss (max 20) · two-way: 10.98 for both (max 11) · the early exit saves 0.99 iterations on hits and none on misses, but pays 1.55x and 1.82x the comparisons · on duplicates three-way returned the middle copy (index 4) and two-way the first (index 2).

---

> Next tier: [🟡 medium](2-medium.md) — rotated arrays, searching the answer instead of the array,
> two kinds of sorted matrix, and a binary search on data that is not sorted.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
