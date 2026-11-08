# Step 01 — Introduction · 🟢 Easy

Examples **1–5**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Everything in this tier is **deterministic** — no clocks, no randomness. Your output should match
these blocks character for character.

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

## 1. Same data, different structure

`🟢 easy` · *Structure vs algorithm*

The single most important idea in the plan, in fifteen lines: the **question doesn't change**, the
**structure does**, and the complexity follows the structure. A slice must look at every element to
answer "is `x` in here?"; a set hashes once and jumps straight to the answer.

**Steps:**

1. Answer the membership question by scanning a slice.
2. Load the same values into a `map[int]struct{}` — Go's idiomatic set (the empty struct costs 0 bytes).
3. Answer the same question with one lookup, and note what you paid to make that possible.

```go
package main

import "fmt"

// contains answers "is x in the data?" by looking at every element: O(n).
func contains(xs []int, x int) bool {
	for _, v := range xs {
		if v == x {
			return true
		}
	}
	return false
}

func main() {
	data := []int{41, 7, 93, 12, 55, 8, 77, 30}

	// Structure 1: a slice. Membership costs a scan.
	fmt.Println("slice contains 55:", contains(data, 55))
	fmt.Println("slice contains 56:", contains(data, 56))

	// Structure 2: the same values in a set. Membership is one hash lookup.
	set := make(map[int]struct{}, len(data))
	for _, v := range data {
		set[v] = struct{}{}
	}
	_, has55 := set[55]
	_, has56 := set[56]
	fmt.Println("set contains 55:  ", has55)
	fmt.Println("set contains 56:  ", has56)

	fmt.Println()
	fmt.Println("same data, same question, different structure:")
	fmt.Println("  slice -> O(n) per lookup, O(0) to prepare")
	fmt.Println("  set   -> O(1) per lookup, O(n) to prepare once")
}
```

**Output:**

```
slice contains 55: true
slice contains 56: false
set contains 55:   true
set contains 56:   false

same data, same question, different structure:
  slice -> O(n) per lookup, O(0) to prepare
  set   -> O(1) per lookup, O(n) to prepare once
```

**Rust:**

```rust
use std::collections::HashSet;

// contains answers "is x in the data?" by looking at every element: O(n).
fn contains(xs: &[i32], x: i32) -> bool {
    for &v in xs {
        if v == x {
            return true;
        }
    }
    false
}

fn main() {
    let data = [41, 7, 93, 12, 55, 8, 77, 30];

    // Structure 1: a slice. Membership costs a scan.
    println!("slice contains 55: {}", contains(&data, 55));
    println!("slice contains 56: {}", contains(&data, 56));

    // Structure 2: the same values in a set. Membership is one hash lookup.
    // HashSet<T> is HashMap<T, ()> underneath -- the same "empty value" trick
    // Go spells map[int]struct{}.
    let set: HashSet<i32> = data.iter().copied().collect();
    println!("set contains 55:   {}", set.contains(&55));
    println!("set contains 56:   {}", set.contains(&56));

    println!();
    println!("same data, same question, different structure:");
    println!("  slice -> O(n) per lookup, O(0) to prepare");
    println!("  set   -> O(1) per lookup, O(n) to prepare once");
}
```

**Rust output:**

```
slice contains 55: true
slice contains 56: false
set contains 55:   true
set contains 56:   false

same data, same question, different structure:
  slice -> O(n) per lookup, O(0) to prepare
  set   -> O(1) per lookup, O(n) to prepare once
```

> **Go vs Rust:** Rust has a real set type, `HashSet<T>` — it is a `HashMap<T, ()>` underneath, the same empty-value trick as Go's `map[int]struct{}`. The default hasher is SipHash-1-3, chosen to resist hash-flooding rather than for raw speed (lesson 10 returns to this).

**Complexity:** slice lookup O(n) · set lookup O(1) average, O(n) worst · set build O(n) time and O(n) space

---

## 2. Count the work, not the clock

`🟢 easy` · *Best vs worst case*

Before any stopwatch: **instrument the loop**. A comparison counter is reproducible on every machine,
immune to CPU noise, and shows the growth rate directly. It also makes the point that "linear search
is O(n)" is only half a sentence — the *same function* is O(1) in its best case.

**Steps:**

1. Return the comparison count alongside the result.
2. Search for the first element (best case) and for an absent element (worst case).
3. Read the two columns: one is flat, one tracks `n` exactly.

```go
package main

import "fmt"

// searchCounting returns the index of x plus the number of comparisons it took.
// Counting the work is machine-independent: the numbers are the same everywhere.
func searchCounting(xs []int, x int) (idx, comparisons int) {
	for i, v := range xs {
		comparisons++
		if v == x {
			return i, comparisons
		}
	}
	return -1, comparisons
}

func main() {
	fmt.Printf("%8s %14s %14s\n", "n", "best case", "worst case")
	for _, n := range []int{10, 100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		_, best := searchCounting(xs, 0)   // first element
		_, worst := searchCounting(xs, -1) // absent: every element checked
		fmt.Printf("%8d %14d %14d\n", n, best, worst)
	}

	fmt.Println()
	fmt.Println("best case is flat -> O(1); worst case tracks n exactly -> O(n)")
	fmt.Println("this is why Big-O always names a case: the same code is both")
}
```

**Output:**

```
       n      best case     worst case
      10              1             10
     100              1            100
    1000              1           1000
   10000              1          10000

best case is flat -> O(1); worst case tracks n exactly -> O(n)
this is why Big-O always names a case: the same code is both
```

**Rust:**

```rust
// search_counting returns the index of x plus the number of comparisons it
// took. Counting the work is machine-independent: the numbers are the same
// everywhere. Rust returns Option<usize> where Go returns -1.
fn search_counting(xs: &[i64], x: i64) -> (Option<usize>, u64) {
    let mut comparisons = 0;
    for (i, &v) in xs.iter().enumerate() {
        comparisons += 1;
        if v == x {
            return (Some(i), comparisons);
        }
    }
    (None, comparisons)
}

fn main() {
    println!("{:>8} {:>14} {:>14}", "n", "best case", "worst case");
    for n in [10, 100, 1000, 10000] {
        let xs: Vec<i64> = (0..n).collect();
        let (_, best) = search_counting(&xs, 0); // first element
        let (_, worst) = search_counting(&xs, -1); // absent: every element checked
        println!("{n:>8} {best:>14} {worst:>14}");
    }

    println!();
    println!("best case is flat -> O(1); worst case tracks n exactly -> O(n)");
    println!("this is why Big-O always names a case: the same code is both");
}
```

**Rust output:**

```
       n      best case     worst case
      10              1             10
     100              1            100
    1000              1           1000
   10000              1          10000

best case is flat -> O(1); worst case tracks n exactly -> O(n)
this is why Big-O always names a case: the same code is both
```

> **Go vs Rust:** Rust returns `Option<usize>` where Go returns a `-1` sentinel: "not found" is a different type, so it cannot be mistaken for an index. The comparison counts are identical — they are a property of the algorithm, not the language.

**Complexity:** best O(1) · worst O(n) · average O(n/2) = O(n) · space O(1)

---

## 3. Quadratic vs linear, in operations

`🟢 easy` · *Growth rate — O(n²) vs O(n)*

Two-sum solved twice. Both loops are equally "optimized" — no tricks, no unrolling — and yet at
n=10,000 one does **5,000× more work** than the other. That factor is the algorithm choice, and no
amount of micro-optimization in the brute-force version would close it.

**Steps:**

1. Brute force: check every pair, counting each check.
2. Map version: for each value, ask whether its complement has already been seen.
3. Print the ratio and watch it grow linearly with `n` — that's the signature of O(n²) vs O(n).

```go
package main

import "fmt"

// twoSumBrute checks every pair: O(n^2).
func twoSumBrute(xs []int, target int) (i, j, ops int) {
	for a := 0; a < len(xs); a++ {
		for b := a + 1; b < len(xs); b++ {
			ops++
			if xs[a]+xs[b] == target {
				return a, b, ops
			}
		}
	}
	return -1, -1, ops
}

// twoSumMap remembers what it has already seen: O(n) time, O(n) space.
func twoSumMap(xs []int, target int) (i, j, ops int) {
	seen := make(map[int]int, len(xs))
	for b, v := range xs {
		ops++
		if a, ok := seen[target-v]; ok {
			return a, b, ops
		}
		seen[v] = b
	}
	return -1, -1, ops
}

func main() {
	fmt.Printf("%8s %16s %12s %12s\n", "n", "brute ops", "map ops", "ratio")
	for _, n := range []int{10, 100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		target := 2*n - 3 // the last two elements: the worst case for the brute force

		_, _, bruteOps := twoSumBrute(xs, target)
		_, _, mapOps := twoSumMap(xs, target)
		fmt.Printf("%8d %16d %12d %11.0fx\n", n, bruteOps, mapOps, float64(bruteOps)/float64(mapOps))
	}

	fmt.Println()
	fmt.Println("10x the input costs the brute force ~100x the work, the map ~10x")
	fmt.Println("that gap IS the algorithm choice -- both loops are equally 'optimized'")
}
```

**Output:**

```
       n        brute ops      map ops        ratio
      10               45           10           4x
     100             4950          100          50x
    1000           499500         1000         500x
   10000         49995000        10000        5000x

10x the input costs the brute force ~100x the work, the map ~10x
that gap IS the algorithm choice -- both loops are equally 'optimized'
```

**Rust:**

```rust
use std::collections::HashMap;

// two_sum_brute checks every pair: O(n^2). Returns the pair (if any) and the
// number of pair checks it made.
fn two_sum_brute(xs: &[i64], target: i64) -> (Option<(usize, usize)>, u64) {
    let mut ops = 0;
    for a in 0..xs.len() {
        for b in a + 1..xs.len() {
            ops += 1;
            if xs[a] + xs[b] == target {
                return (Some((a, b)), ops);
            }
        }
    }
    (None, ops)
}

// two_sum_map remembers what it has already seen: O(n) time, O(n) space.
fn two_sum_map(xs: &[i64], target: i64) -> (Option<(usize, usize)>, u64) {
    let mut ops = 0;
    let mut seen: HashMap<i64, usize> = HashMap::with_capacity(xs.len());
    for (b, &v) in xs.iter().enumerate() {
        ops += 1;
        if let Some(&a) = seen.get(&(target - v)) {
            return (Some((a, b)), ops);
        }
        seen.insert(v, b);
    }
    (None, ops)
}

fn main() {
    println!(
        "{:>8} {:>16} {:>12} {:>12}",
        "n", "brute ops", "map ops", "ratio"
    );
    for n in [10i64, 100, 1000, 10000] {
        let xs: Vec<i64> = (0..n).collect();
        let target = 2 * n - 3; // the last two elements: the worst case for the brute force

        let (_, brute_ops) = two_sum_brute(&xs, target);
        let (_, map_ops) = two_sum_map(&xs, target);
        println!(
            "{n:>8} {brute_ops:>16} {map_ops:>12} {:>11.0}x",
            brute_ops as f64 / map_ops as f64
        );
    }

    println!();
    println!("10x the input costs the brute force ~100x the work, the map ~10x");
    println!("that gap IS the algorithm choice -- both loops are equally 'optimized'");
}
```

**Rust output:**

```
       n        brute ops      map ops        ratio
      10               45           10           4x
     100             4950          100          50x
    1000           499500         1000         500x
   10000         49995000        10000        5000x

10x the input costs the brute force ~100x the work, the map ~10x
that gap IS the algorithm choice -- both loops are equally 'optimized'
```

> **Go vs Rust:** Identical output, as it must be: operation counts belong to the algorithm. Both languages show the same 5000× gap at n=10,000.

**Complexity:** brute O(n²) time, O(1) space · map O(n) time, O(n) space — the classic time-for-space trade

---

## 4. What the growth curves actually cost

`🟢 easy` · *Feasible input size*

The growth table stops being abstract when you convert it to wall-clock time. At a billion operations
per second, an O(n log n) algorithm chews through a billion items in half a minute; an O(2ⁿ) one needs
**10¹³ years** at n=100. This is the table that tells you which complexity class you're allowed to ship.

**Steps:**

1. Print log n, n, n log n, n² and 2ⁿ side by side for n up to 10,000.
2. Notice 2ⁿ overflowing a `float64` before n reaches 10,000.
3. Convert three representative workloads into human time.

```go
package main

import (
	"fmt"
	"math"
)

// pow2 formats 2^n, which stops fitting in a float64 above n = 1024.
func pow2(n int) string {
	v := math.Pow(2, float64(n))
	if math.IsInf(v, 1) {
		return "overflows float64"
	}
	return fmt.Sprintf("%.2e", v)
}

func main() {
	fmt.Printf("%8s %10s %12s %14s %14s %20s\n", "n", "log2 n", "n", "n log2 n", "n^2", "2^n")
	for _, n := range []int{1, 10, 100, 1000, 10000} {
		f := float64(n)
		lg := math.Log2(f)
		fmt.Printf("%8d %10.1f %12d %14.0f %14.0f %20s\n", n, lg, n, f*lg, f*f, pow2(n))
	}

	fmt.Println()
	fmt.Println("at 1 billion ops/second:")
	for _, row := range []struct {
		name string
		ops  float64
	}{
		{"n log2 n at n=10^9", 1e9 * math.Log2(1e9)},
		{"n^2       at n=10^6", 1e12},
		{"2^n       at n=100", math.Pow(2, 100)},
	} {
		secs := row.ops / 1e9
		fmt.Printf("  %-20s %10.2e ops -> %s\n", row.name, row.ops, humanTime(secs))
	}
}

func humanTime(secs float64) string {
	switch {
	case secs < 60:
		return fmt.Sprintf("%.1f seconds", secs)
	case secs < 3600:
		return fmt.Sprintf("%.1f minutes", secs/60)
	case secs < 86400*365:
		return fmt.Sprintf("%.1f days", secs/86400)
	default:
		return fmt.Sprintf("%.2e years", secs/(86400*365))
	}
}
```

**Output:**

```
       n     log2 n            n       n log2 n            n^2                  2^n
       1        0.0            1              0              1             2.00e+00
      10        3.3           10             33            100             1.02e+03
     100        6.6          100            664          10000             1.27e+30
    1000       10.0         1000           9966        1000000            1.07e+301
   10000       13.3        10000         132877      100000000    overflows float64

at 1 billion ops/second:
  n log2 n at n=10^9     2.99e+10 ops -> 29.9 seconds
  n^2       at n=10^6    1.00e+12 ops -> 16.7 minutes
  2^n       at n=100     1.27e+30 ops -> 4.02e+13 years
```

**Rust:**

```rust
// Go's %e and Rust's {:e} disagree on spelling: Go prints "1.02e+03", Rust
// prints "1.02e3". go_exp rebuilds Go's form (explicit sign, 2+ exponent
// digits) so the table below reads the same in both languages.
fn go_exp(v: f64, prec: usize) -> String {
    let s = format!("{v:.prec$e}");
    let (mantissa, exp) = s.split_once('e').unwrap();
    let exp: i32 = exp.parse().unwrap();
    format!(
        "{mantissa}e{}{:02}",
        if exp < 0 { '-' } else { '+' },
        exp.abs()
    )
}

// pow2 formats 2^n, which stops fitting in an f64 above n = 1024.
fn pow2(n: u32) -> String {
    let v = 2f64.powf(f64::from(n));
    if v.is_infinite() {
        return "overflows f64".to_string();
    }
    go_exp(v, 2)
}

fn human_time(secs: f64) -> String {
    if secs < 60.0 {
        format!("{secs:.1} seconds")
    } else if secs < 3600.0 {
        format!("{:.1} minutes", secs / 60.0)
    } else if secs < 86400.0 * 365.0 {
        format!("{:.1} days", secs / 86400.0)
    } else {
        format!("{} years", go_exp(secs / (86400.0 * 365.0), 2))
    }
}

fn main() {
    println!(
        "{:>8} {:>10} {:>12} {:>14} {:>14} {:>20}",
        "n", "log2 n", "n", "n log2 n", "n^2", "2^n"
    );
    for n in [1u32, 10, 100, 1000, 10000] {
        let f = f64::from(n);
        let lg = f.log2();
        println!(
            "{n:>8} {lg:>10.1} {n:>12} {:>14.0} {:>14.0} {:>20}",
            f * lg,
            f * f,
            pow2(n)
        );
    }

    println!();
    println!("at 1 billion ops/second:");
    for (name, ops) in [
        ("n log2 n at n=10^9", 1e9 * 1e9f64.log2()),
        ("n^2       at n=10^6", 1e12),
        ("2^n       at n=100", 2f64.powf(100.0)),
    ] {
        let secs = ops / 1e9;
        println!(
            "  {:<20} {:>10} ops -> {}",
            name,
            go_exp(ops, 2),
            human_time(secs)
        );
    }
}
```

**Rust output:**

```
       n     log2 n            n       n log2 n            n^2                  2^n
       1        0.0            1              0              1             2.00e+00
      10        3.3           10             33            100             1.02e+03
     100        6.6          100            664          10000             1.27e+30
    1000       10.0         1000           9966        1000000            1.07e+301
   10000       13.3        10000         132877      100000000        overflows f64

at 1 billion ops/second:
  n log2 n at n=10^9     2.99e+10 ops -> 29.9 seconds
  n^2       at n=10^6    1.00e+12 ops -> 16.7 minutes
  2^n       at n=100     1.27e+30 ops -> 4.02e+13 years
```

> **Go vs Rust:** Go's `%e` and Rust's `{:e}` spell exponents differently (`1.02e+03` vs `1.02e3`), so `go_exp` rebuilds Go's form to keep the table comparable. The one deliberate difference in the output is the last cell: Rust names its type `f64`.

**Complexity:** the program itself is O(1) — it's a lookup table for every complexity you'll meet later

---

## 5. Trading space for time

`🟢 easy` · *The trade-off triangle*

The trade at the heart of dynamic programming, visible in one counter. Naive Fibonacci recomputes the
same subproblems exponentially often — ~30 million calls at n=35. Handing it a `map` to remember
answers drops that to **69**. You spent O(n) space and bought O(2ⁿ) → O(n) time.

**Steps:**

1. Count calls in the naive recursion.
2. Count calls in the memoized one (the extra calls are the memo *hits*).
3. Watch the naive column explode while the memo column grows linearly.

```go
package main

import "fmt"

var naiveCalls int

// fibNaive recomputes the same subproblems over and over: O(2^n) calls.
func fibNaive(n int) int {
	naiveCalls++
	if n < 2 {
		return n
	}
	return fibNaive(n-1) + fibNaive(n-2)
}

var memoCalls int

// fibMemo spends O(n) space to remember each answer once: O(n) calls.
func fibMemo(n int, memo map[int]int) int {
	memoCalls++
	if n < 2 {
		return n
	}
	if v, ok := memo[n]; ok {
		return v
	}
	v := fibMemo(n-1, memo) + fibMemo(n-2, memo)
	memo[n] = v
	return v
}

func main() {
	fmt.Printf("%4s %16s %12s %12s\n", "n", "naive calls", "memo calls", "fib(n)")
	for _, n := range []int{5, 10, 20, 30, 35} {
		naiveCalls, memoCalls = 0, 0
		got := fibNaive(n)
		fibMemo(n, make(map[int]int, n))
		fmt.Printf("%4d %16d %12d %12d\n", n, naiveCalls, memoCalls, got)
	}

	fmt.Println()
	fmt.Println("the memo buys a drop from O(2^n) to O(n) calls by spending O(n) space")
	fmt.Println("time and space are a currency pair -- you trade one for the other")
}
```

**Output:**

```
   n      naive calls   memo calls       fib(n)
   5               15            9            5
  10              177           19           55
  20            21891           39         6765
  30          2692537           59       832040
  35         29860703           69      9227465

the memo buys a drop from O(2^n) to O(n) calls by spending O(n) space
time and space are a currency pair -- you trade one for the other
```

**Rust:**

```rust
use std::collections::HashMap;

// fib_naive recomputes the same subproblems over and over: O(2^n) calls.
// Go keeps the counter in a package-level variable; Rust has no mutable
// globals without unsafe or atomics, so the counter is passed down instead.
fn fib_naive(n: u32, calls: &mut u64) -> u64 {
    *calls += 1;
    if n < 2 {
        return u64::from(n);
    }
    fib_naive(n - 1, calls) + fib_naive(n - 2, calls)
}

// fib_memo spends O(n) space to remember each answer once: O(n) calls.
fn fib_memo(n: u32, memo: &mut HashMap<u32, u64>, calls: &mut u64) -> u64 {
    *calls += 1;
    if n < 2 {
        return u64::from(n);
    }
    if let Some(&v) = memo.get(&n) {
        return v;
    }
    let v = fib_memo(n - 1, memo, calls) + fib_memo(n - 2, memo, calls);
    memo.insert(n, v);
    v
}

fn main() {
    println!(
        "{:>4} {:>16} {:>12} {:>12}",
        "n", "naive calls", "memo calls", "fib(n)"
    );
    for n in [5, 10, 20, 30, 35] {
        let (mut naive_calls, mut memo_calls) = (0, 0);
        let got = fib_naive(n, &mut naive_calls);
        fib_memo(n, &mut HashMap::with_capacity(n as usize), &mut memo_calls);
        println!("{n:>4} {naive_calls:>16} {memo_calls:>12} {got:>12}");
    }

    println!();
    println!("the memo buys a drop from O(2^n) to O(n) calls by spending O(n) space");
    println!("time and space are a currency pair -- you trade one for the other");
}
```

**Rust output:**

```
   n      naive calls   memo calls       fib(n)
   5               15            9            5
  10              177           19           55
  20            21891           39         6765
  30          2692537           59       832040
  35         29860703           69      9227465

the memo buys a drop from O(2^n) to O(n) calls by spending O(n) space
time and space are a currency pair -- you trade one for the other
```

> **Go vs Rust:** Go keeps the call counters in package-level variables; Rust has no mutable globals without `unsafe` or atomics, so the counter is threaded through as `&mut u64`. Same counts, and the same 29,860,703 naive calls at n=35.

**Complexity:** naive O(φⁿ) ≈ O(1.618ⁿ) time, O(n) stack · memo O(n) time, O(n) space — revisited in full at [30 — DP I](../../30-dp-one-dimensional.md)

---

> Next tier: [🟡 medium](2-medium.md) — where the clock comes out.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
