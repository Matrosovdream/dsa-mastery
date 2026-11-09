# Step 12 — Elementary Sorts · 🟢 Easy

Examples **1–6**: bubble, selection and insertion sort with every operation counted, the invariant
each one maintains, which of them are stable, and the inversion count that insertion sort pays.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match your run character for character;
**Sample output:** blocks contain timings, which vary by machine.

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

## 1. Bubble sort, and the early exit

`🟢 easy` · *Adjacent swaps*

Bubble sort swaps adjacent out-of-order pairs until none are left, and it is the right place to start because
every cost in it is visible: each pass carries the largest remaining element to the end, so after pass k the k
largest are final and the next pass can stop k places earlier. With that shrinking bound alone it always makes
n(n−1)/2 comparisons; an early-exit flag (stop after a pass with no swaps) makes sorted input cost n−1. What the
flag cannot fix is the turtle: a small element near the end moves left one place per pass, so a slice one rotation
from sorted can still cost Θ(n²).

**Steps:**

1. Write one bubble sort with a `passes / comparisons / swaps` counter and a switch for the early-exit flag.
2. Trace the fixed input pass by pass, with the final suffix after the `|`.
3. Run five inputs with and without the flag, then race the turtle against the rabbit at n = 1000.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
)

// Bubble sort walks the slice and swaps every adjacent pair that is out of
// order. One walk is a "pass". Invariant: after pass k, the k largest elements
// sit in their final places at the end -- each pass carries the largest
// remaining element all the way right, like a bubble rising.
//
// That invariant gives the first optimisation for free: pass k only needs to
// walk a[0 .. n-k], because the suffix is already final (the "shrinking bound").
// The second optimisation is the early-exit flag: a pass with no swaps proves
// the slice is sorted, so stop.

type stats struct{ passes, comps, swaps int }

// bubble sorts a in place and counts. earlyExit switches the flag on;
// trace prints the slice after every pass with the final suffix after "|".
func bubble(a []int, earlyExit, trace bool) stats {
	var s stats
	for end := len(a) - 1; end > 0; end-- { // a[end+1:] is final
		s.passes++
		c, w := 0, 0
		for j := 0; j < end; j++ {
			c++
			if a[j] > a[j+1] { // strict > : equal neighbours never swap (stable)
				a[j], a[j+1] = a[j+1], a[j]
				w++
			}
		}
		s.comps += c
		s.swaps += w
		if trace {
			fmt.Printf("  pass %d: %-22s  comparisons %d, swaps %d\n", s.passes, marked(a, end), c, w)
		}
		if earlyExit && w == 0 {
			break // no swap in a whole pass: sorted
		}
	}
	return s
}

// marked prints a with a "|" before the final suffix a[from:].
func marked(a []int, from int) string {
	var b strings.Builder
	for i, v := range a {
		if i == from {
			b.WriteString("| ")
		}
		fmt.Fprintf(&b, "%d ", v)
	}
	return strings.TrimSpace(b.String())
}

// inversions counts pairs i < j with a[i] > a[j] (brute force, O(n^2)).
func inversions(a []int) int {
	inv := 0
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i] > a[j] {
				inv++
			}
		}
	}
	return inv
}

func seq(lo, hi int) []int { // lo, lo+1, ..., hi
	var s []int
	for v := lo; v <= hi; v++ {
		s = append(s, v)
	}
	return s
}

func main() {
	fixed := []int{5, 1, 4, 2, 8, 0, 3, 7}

	fmt.Println("trace, early exit on, input", fixed)
	a := slices.Clone(fixed)
	s := bubble(a, true, true)
	fmt.Printf("  total: %d passes, %d comparisons, %d swaps, sorted: %v\n",
		s.passes, s.comps, s.swaps, slices.IsSorted(a))
	fmt.Println("  the 8 rose to the end in pass 1; the 0 moved LEFT only one place per pass.")

	fmt.Println()
	fmt.Println("n = 8: without the flag (shrinking bound only) vs with it")
	fmt.Printf("  %-16s %4s | %6s %5s %5s | %6s %5s %5s\n",
		"input", "inv", "passes", "comps", "swaps", "passes", "comps", "swaps")
	allPlain, allInv := true, true // checked, not assumed
	for _, in := range inputs(8) {
		p, f := slices.Clone(in.a), slices.Clone(in.a)
		sp, sf := bubble(p, false, false), bubble(f, true, false)
		inv := inversions(in.a)
		fmt.Printf("  %-16s %4d | %6d %5d %5d | %6d %5d %5d\n", in.name, inv,
			sp.passes, sp.comps, sp.swaps, sf.passes, sf.comps, sf.swaps)
		allPlain = allPlain && sp.comps == 8*7/2
		allInv = allInv && sp.swaps == inv && sf.swaps == inv
	}
	fmt.Printf("  every row costs n(n-1)/2 = %d comparisons without the flag: %v\n", 8*7/2, allPlain)
	fmt.Printf("  swaps = inversions on every row, flag or not: %v\n", allInv)
	fmt.Println("  the flag only cuts the passes after the first swap-free one; the swaps never change.")

	const big = 1000
	fmt.Println()
	fmt.Printf("n = %d, early exit on: the turtle against the rabbit\n", big)
	for _, in := range inputs(big)[3:] { // skip sorted, reversed, fixed
		a := slices.Clone(in.a)
		s := bubble(a, true, false)
		fmt.Printf("  %-16s inversions %4d, passes %4d, comparisons %6d, swaps %4d\n",
			in.name, inversions(in.a), s.passes, s.comps, s.swaps)
	}
	fmt.Println("  same distance from sorted (one rotation, n-1 inversions), opposite cost:")
	fmt.Println("  a large element moves right n-1 places in ONE pass, a small one moves left ONE place per pass.")
}

type input struct {
	name string
	a    []int
}

// inputs builds the test inputs; all but "fixed input" have length n.
func inputs(n int) []input {
	rev := seq(1, n)
	slices.Reverse(rev)
	return []input{
		{"sorted", seq(1, n)},
		{"reversed", rev},
		{"fixed input", []int{5, 1, 4, 2, 8, 0, 3, 7}},
		{"turtle [2..n 1]", append(seq(2, n), 1)},
		{"rabbit [n 1..]", append([]int{n}, seq(1, n-1)...)},
	}
}
```

**Output:**

```
trace, early exit on, input [5 1 4 2 8 0 3 7]
  pass 1: 1 4 2 5 0 3 7 | 8       comparisons 7, swaps 6
  pass 2: 1 2 4 0 3 5 | 7 8       comparisons 6, swaps 3
  pass 3: 1 2 0 3 4 | 5 7 8       comparisons 5, swaps 2
  pass 4: 1 0 2 3 | 4 5 7 8       comparisons 4, swaps 1
  pass 5: 0 1 2 | 3 4 5 7 8       comparisons 3, swaps 1
  pass 6: 0 1 | 2 3 4 5 7 8       comparisons 2, swaps 0
  total: 6 passes, 27 comparisons, 13 swaps, sorted: true
  the 8 rose to the end in pass 1; the 0 moved LEFT only one place per pass.

n = 8: without the flag (shrinking bound only) vs with it
  input             inv | passes comps swaps | passes comps swaps
  sorted              0 |      7    28     0 |      1     7     0
  reversed           28 |      7    28    28 |      7    28    28
  fixed input        13 |      7    28    13 |      6    27    13
  turtle [2..n 1]     7 |      7    28     7 |      7    28     7
  rabbit [n 1..]      7 |      7    28     7 |      2    13     7
  every row costs n(n-1)/2 = 28 comparisons without the flag: true
  swaps = inversions on every row, flag or not: true
  the flag only cuts the passes after the first swap-free one; the swaps never change.

n = 1000, early exit on: the turtle against the rabbit
  turtle [2..n 1]  inversions  999, passes  999, comparisons 499500, swaps  999
  rabbit [n 1..]   inversions  999, passes    2, comparisons   1997, swaps  999
  same distance from sorted (one rotation, n-1 inversions), opposite cost:
  a large element moves right n-1 places in ONE pass, a small one moves left ONE place per pass.
```

**Rust:**

```rust
// Bubble sort walks the slice and swaps every adjacent pair that is out of
// order. One walk is a "pass". Invariant: after pass k, the k largest elements
// sit in their final places at the end -- each pass carries the largest
// remaining element all the way right, like a bubble rising.
//
// That invariant gives the first optimisation for free: pass k only needs to
// walk a[0 .. n-k], because the suffix is already final (the "shrinking bound").
// The second optimisation is the early-exit flag: a pass with no swaps proves
// the slice is sorted, so stop.

#[derive(Default, Clone, Copy)]
struct Stats {
    passes: usize,
    comps: usize,
    swaps: usize,
}

/// Sorts `a` in place and counts. `early_exit` switches the flag on;
/// `trace` prints the slice after every pass with the final suffix after "|".
fn bubble(a: &mut [i64], early_exit: bool, trace: bool) -> Stats {
    let mut s = Stats::default();
    // `(1..len).rev()` is Go's `for end := len(a)-1; end > 0; end--`;
    // on an empty slice the range is simply empty (no usize underflow).
    for end in (1..a.len()).rev() {
        // a[end+1..] is final
        s.passes += 1;
        let (mut c, mut w) = (0, 0);
        for j in 0..end {
            c += 1;
            if a[j] > a[j + 1] {
                // strict > : equal neighbours never swap (stable)
                a.swap(j, j + 1); // slice::swap -- no temporary to write by hand
                w += 1;
            }
        }
        s.comps += c;
        s.swaps += w;
        if trace {
            println!(
                "  pass {}: {:<22}  comparisons {c}, swaps {w}",
                s.passes,
                marked(a, end)
            );
        }
        if early_exit && w == 0 {
            break; // no swap in a whole pass: sorted
        }
    }
    s
}

/// `a` with a "|" before the final suffix a[from..].
fn marked(a: &[i64], from: usize) -> String {
    let mut parts: Vec<String> = a.iter().map(i64::to_string).collect();
    parts.insert(from, "|".to_string());
    parts.join(" ")
}

/// Pairs i < j with a[i] > a[j] (brute force, O(n^2)).
fn inversions(a: &[i64]) -> usize {
    a.iter()
        .enumerate()
        .map(|(i, x)| a[i + 1..].iter().filter(|y| x > y).count())
        .sum()
}

/// Go's %v of a slice: [1 2 3]
fn go_list(a: &[i64]) -> String {
    let parts: Vec<String> = a.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

/// The test inputs; all but "fixed input" have length n.
fn inputs(n: i64) -> Vec<(&'static str, Vec<i64>)> {
    vec![
        ("sorted", (1..=n).collect()),
        ("reversed", (1..=n).rev().collect()),
        ("fixed input", vec![5, 1, 4, 2, 8, 0, 3, 7]),
        ("turtle [2..n 1]", (2..=n).chain([1]).collect()),
        ("rabbit [n 1..]", [n].into_iter().chain(1..n).collect()),
    ]
}

fn main() {
    let fixed = [5, 1, 4, 2, 8, 0, 3, 7];

    println!("trace, early exit on, input {}", go_list(&fixed));
    let mut a = fixed.to_vec();
    let s = bubble(&mut a, true, true);
    // is_sorted() is std on slices (Go: slices.IsSorted); it is the windows(2)
    // check `a.windows(2).all(|w| w[0] <= w[1])` written for you.
    println!(
        "  total: {} passes, {} comparisons, {} swaps, sorted: {}",
        s.passes,
        s.comps,
        s.swaps,
        a.is_sorted()
    );
    println!("  the 8 rose to the end in pass 1; the 0 moved LEFT only one place per pass.");

    println!();
    println!("n = 8: without the flag (shrinking bound only) vs with it");
    println!(
        "  {:<16} {:>4} | {:>6} {:>5} {:>5} | {:>6} {:>5} {:>5}",
        "input", "inv", "passes", "comps", "swaps", "passes", "comps", "swaps"
    );
    let (mut all_plain, mut all_inv) = (true, true); // checked, not assumed
    for (name, input) in inputs(8) {
        let (mut p, mut f) = (input.clone(), input.clone());
        let (sp, sf) = (bubble(&mut p, false, false), bubble(&mut f, true, false));
        let inv = inversions(&input);
        println!(
            "  {name:<16} {inv:>4} | {:>6} {:>5} {:>5} | {:>6} {:>5} {:>5}",
            sp.passes, sp.comps, sp.swaps, sf.passes, sf.comps, sf.swaps
        );
        all_plain &= sp.comps == 8 * 7 / 2;
        all_inv &= sp.swaps == inv && sf.swaps == inv;
    }
    println!(
        "  every row costs n(n-1)/2 = {} comparisons without the flag: {all_plain}",
        8 * 7 / 2
    );
    println!("  swaps = inversions on every row, flag or not: {all_inv}");
    println!(
        "  the flag only cuts the passes after the first swap-free one; the swaps never change."
    );

    let big = 1000;
    println!();
    println!("n = {big}, early exit on: the turtle against the rabbit");
    for (name, input) in inputs(big).into_iter().skip(3) {
        // skip sorted, reversed, fixed
        let mut a = input.clone();
        let s = bubble(&mut a, true, false);
        println!(
            "  {name:<16} inversions {:>4}, passes {:>4}, comparisons {:>6}, swaps {:>4}",
            inversions(&input),
            s.passes,
            s.comps,
            s.swaps
        );
    }
    println!("  same distance from sorted (one rotation, n-1 inversions), opposite cost:");
    println!(
        "  a large element moves right n-1 places in ONE pass, a small one moves left ONE place per pass."
    );
}
```

**Rust output:**

```
trace, early exit on, input [5 1 4 2 8 0 3 7]
  pass 1: 1 4 2 5 0 3 7 | 8       comparisons 7, swaps 6
  pass 2: 1 2 4 0 3 5 | 7 8       comparisons 6, swaps 3
  pass 3: 1 2 0 3 4 | 5 7 8       comparisons 5, swaps 2
  pass 4: 1 0 2 3 | 4 5 7 8       comparisons 4, swaps 1
  pass 5: 0 1 2 | 3 4 5 7 8       comparisons 3, swaps 1
  pass 6: 0 1 | 2 3 4 5 7 8       comparisons 2, swaps 0
  total: 6 passes, 27 comparisons, 13 swaps, sorted: true
  the 8 rose to the end in pass 1; the 0 moved LEFT only one place per pass.

n = 8: without the flag (shrinking bound only) vs with it
  input             inv | passes comps swaps | passes comps swaps
  sorted              0 |      7    28     0 |      1     7     0
  reversed           28 |      7    28    28 |      7    28    28
  fixed input        13 |      7    28    13 |      6    27    13
  turtle [2..n 1]     7 |      7    28     7 |      7    28     7
  rabbit [n 1..]      7 |      7    28     7 |      2    13     7
  every row costs n(n-1)/2 = 28 comparisons without the flag: true
  swaps = inversions on every row, flag or not: true
  the flag only cuts the passes after the first swap-free one; the swaps never change.

n = 1000, early exit on: the turtle against the rabbit
  turtle [2..n 1]  inversions  999, passes  999, comparisons 499500, swaps  999
  rabbit [n 1..]   inversions  999, passes    2, comparisons   1997, swaps  999
  same distance from sorted (one rotation, n-1 inversions), opposite cost:
  a large element moves right n-1 places in ONE pass, a small one moves left ONE place per pass.
```

> **Go vs Rust:** Identical output, diffed line by line against the Go run: every count is the algorithm's own. `a.swap(j, j + 1)` is Go's `a[j], a[j+1] = a[j+1], a[j]`, and `is_sorted()` is std on slices just as `slices.IsSorted` is in Go (it is the `windows(2)` check written for you). The outer loop `for end in (1..a.len()).rev()` cannot underflow on an empty slice, where Go's `len(a) - 1` is a signed `-1`.

**Complexity:** best Θ(n) with the flag (n−1 comparisons, 0 swaps on sorted input) · worst Θ(n²): n(n−1)/2 comparisons and swaps (28 and 28 for reversed n = 8) · swaps = inversions on every input, flag or not · stable (strict `>`), in place · a turtle costs one pass per position: 999 passes at n = 1000, against 2 for the rabbit

---

## 2. Selection sort never adapts

`🟢 easy` · *Non-adaptive, fewest writes*

Selection sort finds the minimum of the unsorted suffix and swaps it to the front, so its invariant is the
strongest of the three: the prefix holds the smallest elements, sorted, in their final places. Finding a minimum
means reading the whole suffix every time, so the cost is blind to the input — exactly n(n−1)/2 comparisons on
sorted, reversed and all-equal data alike, Θ(n²) best, average and worst. Its one virtue is the write count: at
most one swap per position, n−1 in total, which is why it is the sort to know when a move costs far more than a
comparison. (It is also not stable — that is example 5.)

**Steps:**

1. Write selection sort with a strict `<` scan (the first minimum wins) and count only swaps where `i != min`.
2. Trace the fixed input from example 1: the minimum found, its index, and the final prefix before the `|`.
3. Run four n = 1000 inputs through selection and through bubble sort with the early exit, and compare writes.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
)

// Selection sort: find the minimum of the unsorted suffix a[i:], swap it to
// a[i], repeat. Invariant: a[:i] holds the i smallest elements, sorted, in their
// FINAL places -- the strongest invariant of the three elementary sorts.
//
// The price of that invariant: finding a minimum means looking at every element
// of the suffix, whatever the data looks like. So the comparison count is fixed
// by n alone. The reward: at most one swap per position, n-1 at most.
//
// A "write" here is one slot assignment; a swap is 2 writes.

type stats struct{ comps, swaps int }

func (s stats) writes() int { return 2 * s.swaps }

func selection(a []int, trace bool) stats {
	var s stats
	for i := 0; i < len(a)-1; i++ {
		m := i
		for j := i + 1; j < len(a); j++ {
			s.comps++
			if a[j] < a[m] { // strict < : the FIRST minimum wins
				m = j
			}
		}
		if m != i { // swapping a slot with itself is not a write
			a[i], a[m] = a[m], a[i]
			s.swaps++
		}
		if trace {
			act := fmt.Sprintf("swap a[%d], a[%d]", i, m)
			if m == i {
				act = "already in place"
			}
			fmt.Printf("  step %d: min %d at index %d, %-16s -> %-22s  comparisons %d\n",
				i, a[i], m, act, marked(a, i+1), len(a)-1-i)
		}
	}
	return s
}

// bubble with the early-exit flag (example 1), recomputed here for the
// side-by-side table.
func bubble(a []int) stats {
	var s stats
	for end := len(a) - 1; end > 0; end-- {
		swapped := false
		for j := 0; j < end; j++ {
			s.comps++
			if a[j] > a[j+1] {
				a[j], a[j+1] = a[j+1], a[j]
				s.swaps++
				swapped = true
			}
		}
		if !swapped {
			break
		}
	}
	return s
}

// marked prints a with a "|" after the final prefix a[:upto].
func marked(a []int, upto int) string {
	var b strings.Builder
	for i, v := range a {
		if i == upto {
			b.WriteString("| ")
		}
		fmt.Fprintf(&b, "%d ", v)
	}
	return strings.TrimSpace(b.String())
}

func main() {
	fixed := []int{5, 1, 4, 2, 8, 0, 3, 7}
	fmt.Println("trace, input", fixed, "(final prefix before the |)")
	a := slices.Clone(fixed)
	s := selection(a, true)
	fmt.Printf("  total: %d comparisons, %d swaps, sorted: %v\n", s.comps, s.swaps, slices.IsSorted(a))

	const n = 1000
	sorted, reversed, equal, pipe := make([]int, n), make([]int, n), make([]int, n), make([]int, n)
	for i := range n {
		sorted[i] = i
		reversed[i] = n - 1 - i
		equal[i] = 7
		pipe[i] = min(i, n-1-i) // organ pipe: 0 1 2 ... 499 499 ... 2 1 0
	}
	fmt.Println()
	fmt.Printf("n = %d, selection vs bubble (early exit); writes = 2 per swap\n", n)
	fmt.Printf("  %-12s | %8s %6s %7s | %8s %6s %7s\n",
		"input", "comps", "swaps", "writes", "comps", "swaps", "writes")
	same := true
	var revSel, revBub stats
	for _, in := range []struct {
		name string
		a    []int
	}{{"sorted", sorted}, {"reversed", reversed}, {"all equal", equal}, {"organ pipe", pipe}} {
		x, y := slices.Clone(in.a), slices.Clone(in.a)
		sel, bub := selection(x, false), bubble(y)
		fmt.Printf("  %-12s | %8d %6d %7d | %8d %6d %7d\n", in.name,
			sel.comps, sel.swaps, sel.writes(), bub.comps, bub.swaps, bub.writes())
		if in.name == "reversed" {
			revSel, revBub = sel, bub
		}
		same = same && sel.comps == n*(n-1)/2 && sel.swaps <= n-1
	}
	fmt.Printf("  selection: n(n-1)/2 = %d comparisons and <= n-1 = %d swaps on every row: %v\n",
		n*(n-1)/2, n-1, same)
	fmt.Println("  it never adapts: sorted input costs as many comparisons as reversed input.")
	fmt.Println("  bubble adapts (sorted: n-1 comparisons) but pays a swap per inversion.")
	fmt.Printf("  the write gap on reversed input: %d vs %d writes, %dx fewer for selection.\n",
		revSel.writes(), revBub.writes(), revBub.writes()/revSel.writes())
}
```

**Output:**

```
trace, input [5 1 4 2 8 0 3 7] (final prefix before the |)
  step 0: min 0 at index 5, swap a[0], a[5]  -> 0 | 1 4 2 8 5 3 7       comparisons 7
  step 1: min 1 at index 1, already in place -> 0 1 | 4 2 8 5 3 7       comparisons 6
  step 2: min 2 at index 3, swap a[2], a[3]  -> 0 1 2 | 4 8 5 3 7       comparisons 5
  step 3: min 3 at index 6, swap a[3], a[6]  -> 0 1 2 3 | 8 5 4 7       comparisons 4
  step 4: min 4 at index 6, swap a[4], a[6]  -> 0 1 2 3 4 | 5 8 7       comparisons 3
  step 5: min 5 at index 5, already in place -> 0 1 2 3 4 5 | 8 7       comparisons 2
  step 6: min 7 at index 7, swap a[6], a[7]  -> 0 1 2 3 4 5 7 | 8       comparisons 1
  total: 28 comparisons, 5 swaps, sorted: true

n = 1000, selection vs bubble (early exit); writes = 2 per swap
  input        |    comps  swaps  writes |    comps  swaps  writes
  sorted       |   499500      0       0 |      999      0       0
  reversed     |   499500    500    1000 |   499500 499500  999000
  all equal    |   499500      0       0 |      999      0       0
  organ pipe   |   499500    994    1988 |   499500 249500  499000
  selection: n(n-1)/2 = 499500 comparisons and <= n-1 = 999 swaps on every row: true
  it never adapts: sorted input costs as many comparisons as reversed input.
  bubble adapts (sorted: n-1 comparisons) but pays a swap per inversion.
  the write gap on reversed input: 1000 vs 999000 writes, 999x fewer for selection.
```

**Rust:**

```rust
// Selection sort: find the minimum of the unsorted suffix a[i..], swap it to
// a[i], repeat. Invariant: a[..i] holds the i smallest elements, sorted, in their
// FINAL places -- the strongest invariant of the three elementary sorts.
//
// The price of that invariant: finding a minimum means looking at every element
// of the suffix, whatever the data looks like. So the comparison count is fixed
// by n alone. The reward: at most one swap per position, n-1 at most.
//
// A "write" here is one slot assignment; a swap is 2 writes.

#[derive(Default, Clone, Copy)]
struct Stats {
    comps: usize,
    swaps: usize,
}

impl Stats {
    fn writes(self) -> usize {
        2 * self.swaps
    }
}

fn selection(a: &mut [i64], trace: bool) -> Stats {
    let mut s = Stats::default();
    for i in 0..a.len().saturating_sub(1) {
        // The minimum of the suffix, by iterator. Iterator::min_by returns the
        // FIRST of several equal minima (documented) -- the same element Go's
        // strict `<` scan keeps. It calls the comparator len-1 times, so the
        // closure can count comparisons. (max_by would return the LAST maximum:
        // a descending variant built on it would pick a different element.)
        let m = i + a[i..]
            .iter()
            .enumerate()
            .min_by(|x, y| {
                s.comps += 1;
                x.1.cmp(y.1)
            })
            .map(|(k, _)| k)
            .expect("suffix of length >= 2 is not empty");
        if m != i {
            // swapping a slot with itself is not a write
            a.swap(i, m);
            s.swaps += 1;
        }
        if trace {
            let act = if m == i {
                "already in place".to_string()
            } else {
                format!("swap a[{i}], a[{m}]")
            };
            println!(
                "  step {i}: min {} at index {m}, {act:<16} -> {:<22}  comparisons {}",
                a[i],
                marked(a, i + 1),
                a.len() - 1 - i
            );
        }
    }
    s
}

/// bubble with the early-exit flag (example 1), recomputed here for the
/// side-by-side table.
fn bubble(a: &mut [i64]) -> Stats {
    let mut s = Stats::default();
    for end in (1..a.len()).rev() {
        let mut swapped = false;
        for j in 0..end {
            s.comps += 1;
            if a[j] > a[j + 1] {
                a.swap(j, j + 1);
                s.swaps += 1;
                swapped = true;
            }
        }
        if !swapped {
            break;
        }
    }
    s
}

/// `a` with a "|" after the final prefix a[..upto].
fn marked(a: &[i64], upto: usize) -> String {
    let mut parts: Vec<String> = a.iter().map(i64::to_string).collect();
    parts.insert(upto, "|".to_string());
    parts.join(" ")
}

/// Go's %v of a slice: [1 2 3]
fn go_list(a: &[i64]) -> String {
    let parts: Vec<String> = a.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let fixed = [5, 1, 4, 2, 8, 0, 3, 7];
    println!(
        "trace, input {} (final prefix before the |)",
        go_list(&fixed)
    );
    let mut a = fixed.to_vec();
    let s = selection(&mut a, true);
    println!(
        "  total: {} comparisons, {} swaps, sorted: {}",
        s.comps,
        s.swaps,
        a.is_sorted()
    );

    const N: i64 = 1000;
    let n = N as usize;
    let inputs: [(&str, Vec<i64>); 4] = [
        ("sorted", (0..N).collect()),
        ("reversed", (0..N).rev().collect()),
        ("all equal", vec![7; n]),
        // organ pipe: 0 1 2 ... 499 499 ... 2 1 0
        ("organ pipe", (0..N).map(|i| i.min(N - 1 - i)).collect()),
    ];
    println!();
    println!("n = {n}, selection vs bubble (early exit); writes = 2 per swap");
    println!(
        "  {:<12} | {:>8} {:>6} {:>7} | {:>8} {:>6} {:>7}",
        "input", "comps", "swaps", "writes", "comps", "swaps", "writes"
    );
    let mut same = true;
    let (mut rev_sel, mut rev_bub) = (Stats::default(), Stats::default());
    for (name, input) in &inputs {
        let (mut x, mut y) = (input.clone(), input.clone());
        let (sel, bub) = (selection(&mut x, false), bubble(&mut y));
        println!(
            "  {name:<12} | {:>8} {:>6} {:>7} | {:>8} {:>6} {:>7}",
            sel.comps,
            sel.swaps,
            sel.writes(),
            bub.comps,
            bub.swaps,
            bub.writes()
        );
        if *name == "reversed" {
            (rev_sel, rev_bub) = (sel, bub);
        }
        same &= sel.comps == n * (n - 1) / 2 && sel.swaps < n;
    }
    println!(
        "  selection: n(n-1)/2 = {} comparisons and <= n-1 = {} swaps on every row: {same}",
        n * (n - 1) / 2,
        n - 1
    );
    println!("  it never adapts: sorted input costs as many comparisons as reversed input.");
    println!("  bubble adapts (sorted: n-1 comparisons) but pays a swap per inversion.");
    println!(
        "  the write gap on reversed input: {} vs {} writes, {}x fewer for selection.",
        rev_sel.writes(),
        rev_bub.writes(),
        rev_bub.writes() / rev_sel.writes()
    );
}
```

**Rust output:**

```
trace, input [5 1 4 2 8 0 3 7] (final prefix before the |)
  step 0: min 0 at index 5, swap a[0], a[5]  -> 0 | 1 4 2 8 5 3 7       comparisons 7
  step 1: min 1 at index 1, already in place -> 0 1 | 4 2 8 5 3 7       comparisons 6
  step 2: min 2 at index 3, swap a[2], a[3]  -> 0 1 2 | 4 8 5 3 7       comparisons 5
  step 3: min 3 at index 6, swap a[3], a[6]  -> 0 1 2 3 | 8 5 4 7       comparisons 4
  step 4: min 4 at index 6, swap a[4], a[6]  -> 0 1 2 3 4 | 5 8 7       comparisons 3
  step 5: min 5 at index 5, already in place -> 0 1 2 3 4 5 | 8 7       comparisons 2
  step 6: min 7 at index 7, swap a[6], a[7]  -> 0 1 2 3 4 5 7 | 8       comparisons 1
  total: 28 comparisons, 5 swaps, sorted: true

n = 1000, selection vs bubble (early exit); writes = 2 per swap
  input        |    comps  swaps  writes |    comps  swaps  writes
  sorted       |   499500      0       0 |      999      0       0
  reversed     |   499500    500    1000 |   499500 499500  999000
  all equal    |   499500      0       0 |      999      0       0
  organ pipe   |   499500    994    1988 |   499500 249500  499000
  selection: n(n-1)/2 = 499500 comparisons and <= n-1 = 999 swaps on every row: true
  it never adapts: sorted input costs as many comparisons as reversed input.
  bubble adapts (sorted: n-1 comparisons) but pays a swap per inversion.
  the write gap on reversed input: 1000 vs 999000 writes, 999x fewer for selection.
```

> **Go vs Rust:** Identical output, diffed against the Go run. The inner scan is `a[i..].iter().enumerate().min_by(..)`, whose doc promises the **first** of several equal minima, the same element as Go's strict `<` loop; the comparator runs len−1 times, so counting inside the closure gives the same 499,500. `max_by`/`max_by_key` return the **last** maximum (read in the source, not run here), so a descending variant built on them would pick a different element among ties.

**Complexity:** Θ(n²) best, average and worst — exactly n(n−1)/2 comparisons (499,500 at n = 1000 on all four inputs) · at most n−1 swaps (500 on reversed input, 0 on sorted and all-equal) · 999x fewer writes than bubble sort on reversed input · not stable · not adaptive

---

## 3. Insertion sort: shift, don't swap

`🟢 easy` · *The one that matters*

Insertion sort grows a sorted prefix: take the next element, walk it left past every larger one, put it down.
There are two ways to write the walk. The swap version exchanges neighbours (two writes per step) and is what Go's
`slices.Sort` runs on ranges of ≤ 12; the shift version holds the element in a variable, slides larger ones right
into the hole (one write per step) and drops it in at the end, which is what Rust's std runs on slices of ≤ 20.
Both ask exactly the same questions in the same order, so the comparisons are identical and only the writes
change. The cost is n−1 comparisons on sorted input and n(n−1)/2 on reversed input: Θ(n) best, Θ(n²) worst, and
in between it depends on how far the elements have to travel.

**Steps:**

1. Write the swap version and the shift version, both with a strict `<` and a `comparisons / writes` counter.
2. Trace the shift version on the fixed input with the sorted prefix before the `|`.
3. Run four n = 1000 inputs through both and check which counts agree, which differ, and by how much.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
)

// Insertion sort grows a sorted prefix a[:i]. To add a[i], walk left past
// every larger element and put a[i] just after the first one that is not
// larger. Invariant: a[:i] is sorted and is a permutation of the ORIGINAL
// a[:i] -- not final, later elements can still land inside it.
//
// Two ways to write the walk:
//   swap  version: swap a[j] with a[j-1] while a[j] < a[j-1]
//                  (this is what Go's slices.Sort runs on ranges <= 12)
//   shift version: hold x = a[i], shift larger elements one place right,
//                  drop x into the hole (what Rust's std runs on len <= 20)
// Both ask exactly the same questions in the same order. They differ in writes.
//
// A "write" is one slot assignment: a swap = 2, a shift step = 1, dropping
// the held element into the hole = 1 (skipped when it did not move).

type stats struct{ comps, writes, shifts int }

func insertionSwap(a []int) stats {
	var s stats
	for i := 1; i < len(a); i++ {
		for j := i; j > 0; j-- {
			s.comps++
			if !(a[j] < a[j-1]) { // strict < : equal keys stay in order (stable)
				break
			}
			a[j], a[j-1] = a[j-1], a[j]
			s.writes += 2
		}
	}
	return s
}

// insertionShift sorts a; trace prints the prefix after each insertion.
func insertionShift(a []int, trace bool) stats {
	var s stats
	for i := 1; i < len(a); i++ {
		x := a[i] // hold it: a[i] is now a hole we may overwrite
		j := i
		for j > 0 {
			s.comps++
			if !(x < a[j-1]) {
				break
			}
			a[j] = a[j-1] // shift right into the hole; the hole moves left
			s.writes++
			s.shifts++
			j--
		}
		if j != i {
			a[j] = x // drop x into the hole
			s.writes++
		}
		if trace {
			fmt.Printf("  insert %d: shifts %d -> %s\n", x, i-j, bracket(a, i+1))
		}
	}
	return s
}

// bracket prints a as [sorted prefix | rest].
func bracket(a []int, upto int) string {
	parts := make([]string, 0, len(a)+1)
	for i, v := range a {
		if i == upto {
			parts = append(parts, "|")
		}
		parts = append(parts, fmt.Sprint(v))
	}
	return "[" + strings.Join(parts, " ") + "]"
}

func main() {
	fixed := []int{5, 1, 4, 2, 8, 0, 3, 7}
	fmt.Println("trace (shift version), start", bracket(fixed, 1))
	a := slices.Clone(fixed)
	s := insertionShift(a, true)
	fmt.Printf("  total: %d comparisons, %d writes\n", s.comps, s.writes)

	const n = 1000
	sorted, reversed := make([]int, n), make([]int, n)
	tiled, blocks := make([]int, n), make([]int, n)
	for i := range n {
		sorted[i] = i
		reversed[i] = n - 1 - i
		tiled[i] = fixed[i%8]            // 5 1 4 2 8 0 3 7 5 1 4 2 ... (ties)
		blocks[i] = fixed[i%8] + 9*(i/8) // the same shape, each block above the last
	}
	fmt.Println()
	fmt.Printf("n = %d: same comparisons, different writes\n", n)
	fmt.Printf("  %-14s %8s | %8s %8s | %s\n", "input", "comps", "swap wr", "shift wr", "results equal")
	rows := []struct {
		name string
		a    []int
	}{{"sorted", sorted}, {"reversed", reversed}, {"tiled pattern", tiled}, {"tiled blocks", blocks}}
	var got []stats
	sameComps, twice := true, true
	for _, in := range rows {
		x, y := slices.Clone(in.a), slices.Clone(in.a)
		sw, sh := insertionSwap(x), insertionShift(y, false)
		sameComps = sameComps && sw.comps == sh.comps
		twice = twice && sw.writes == 2*sh.shifts
		fmt.Printf("  %-14s %8d | %8d %8d | %v\n", in.name, sw.comps, sw.writes, sh.writes, slices.Equal(x, y))
		got = append(got, sh)
	}
	fmt.Println()
	fmt.Printf("  the two versions made the same comparisons on every row: %v\n", sameComps)
	fmt.Printf("  sorted:   n-1 = %d comparisons, 0 writes: %v\n", n-1, got[0].comps == n-1 && got[0].writes == 0)
	fmt.Printf("  reversed: n(n-1)/2 = %d comparisons: %v\n", n*(n-1)/2, got[1].comps == n*(n-1)/2)
	fmt.Printf("  swap writes = 2 x shift steps on every row: %v (the rest of the shift writes are drops)\n", twice)
	fmt.Println("  tiled blocks: the same shape per block, but every element is < 8 places from home:")
	fmt.Printf("    %d comparisons (%.1f per element) vs %d for the plain tiled pattern -- %.0fx fewer.\n",
		got[3].comps, float64(got[3].comps)/n, got[2].comps, float64(got[2].comps)/float64(got[3].comps))
}
```

**Output:**

```
trace (shift version), start [5 | 1 4 2 8 0 3 7]
  insert 1: shifts 1 -> [1 5 | 4 2 8 0 3 7]
  insert 4: shifts 1 -> [1 4 5 | 2 8 0 3 7]
  insert 2: shifts 2 -> [1 2 4 5 | 8 0 3 7]
  insert 8: shifts 0 -> [1 2 4 5 8 | 0 3 7]
  insert 0: shifts 5 -> [0 1 2 4 5 8 | 3 7]
  insert 3: shifts 3 -> [0 1 2 3 4 5 8 | 7]
  insert 7: shifts 1 -> [0 1 2 3 4 5 7 8]
  total: 18 comparisons, 19 writes

n = 1000: same comparisons, different writes
  input             comps |  swap wr shift wr | results equal
  sorted              999 |        0        0 | true
  reversed         499500 |   999000   500499 | true
  tiled pattern    219622 |   437250   219499 | true
  tiled blocks       2622 |     3250     2375 | true

  the two versions made the same comparisons on every row: true
  sorted:   n-1 = 999 comparisons, 0 writes: true
  reversed: n(n-1)/2 = 499500 comparisons: true
  swap writes = 2 x shift steps on every row: true (the rest of the shift writes are drops)
  tiled blocks: the same shape per block, but every element is < 8 places from home:
    2622 comparisons (2.6 per element) vs 219622 for the plain tiled pattern -- 84x fewer.
```

**Rust:**

```rust
// Insertion sort grows a sorted prefix a[..i]. To add a[i], walk left past
// every larger element and put a[i] just after the first one that is not
// larger. Invariant: a[..i] is sorted and is a permutation of the ORIGINAL
// a[..i] -- not final, later elements can still land inside it.
//
// Two ways to write the walk:
//   swap  version: swap a[j] with a[j-1] while a[j] < a[j-1]
//                  (this is what Go's slices.Sort runs on ranges <= 12)
//   shift version: hold x = a[i], shift larger elements one place right,
//                  drop x into the hole (what Rust's std runs on len <= 20)
// Both ask exactly the same questions in the same order. They differ in writes.
//
// A "write" is one slot assignment: a swap = 2, a shift step = 1, dropping
// the held element into the hole = 1 (skipped when it did not move).
//
// Rust note: "hold x and leave a hole" needs T: Copy in safe Rust -- for a
// String, `let x = a[i]` would MOVE out of the slice, which the borrow checker
// refuses (a slice may never contain a moved-out slot). std does it with
// unsafe ptr::copy_nonoverlapping plus a CopyOnDrop guard that refills the hole
// if the comparator panics (`insert_tail` in core/src/slice/sort/shared/
// smallsort.rs). The safe spelling for any T is `a[j..=i].rotate_right(1)`.

#[derive(Default, Clone, Copy)]
struct Stats {
    comps: usize,
    writes: usize,
    shifts: usize,
}

fn insertion_swap(a: &mut [i64]) -> Stats {
    let mut s = Stats::default();
    for i in 1..a.len() {
        for j in (1..=i).rev() {
            s.comps += 1;
            if a[j] >= a[j - 1] {
                // i.e. !(a[j] < a[j-1]) -- strict < : equal keys stay in order (stable)
                break;
            }
            a.swap(j, j - 1);
            s.writes += 2;
        }
    }
    s
}

/// The shift (hole) version, for Copy elements. `trace` prints the prefix
/// after each insertion.
fn insertion_shift<T: Copy + Ord + std::fmt::Display>(a: &mut [T], trace: bool) -> Stats {
    let mut s = Stats::default();
    for i in 1..a.len() {
        let x = a[i]; // a copy: a[i] is now a hole we may overwrite
        let mut j = i;
        while j > 0 {
            s.comps += 1;
            if x >= a[j - 1] {
                break;
            }
            a[j] = a[j - 1]; // shift right into the hole; the hole moves left
            s.writes += 1;
            s.shifts += 1;
            j -= 1;
        }
        if j != i {
            a[j] = x; // drop x into the hole
            s.writes += 1;
        }
        if trace {
            println!("  insert {x}: shifts {} -> {}", i - j, bracket(a, i + 1));
        }
    }
    s
}

/// The safe generic spelling: same walk (same comparisons), then one
/// rotate_right(1) of a[j..=i] moves a[i] to j and everything else one right.
/// Works for any T: Ord, e.g. String, with no Copy and no unsafe.
fn insertion_rotate<T: Ord>(a: &mut [T]) -> usize {
    let mut comps = 0;
    for i in 1..a.len() {
        let mut j = i;
        while j > 0 {
            comps += 1;
            if a[i] >= a[j - 1] {
                break;
            }
            j -= 1;
        }
        a[j..=i].rotate_right(1);
    }
    comps
}

/// `a` as [sorted prefix | rest].
fn bracket<T: std::fmt::Display>(a: &[T], upto: usize) -> String {
    let mut parts: Vec<String> = a.iter().map(T::to_string).collect();
    if upto < a.len() {
        parts.insert(upto, "|".to_string());
    }
    format!("[{}]", parts.join(" "))
}

fn main() {
    let fixed: [i64; 8] = [5, 1, 4, 2, 8, 0, 3, 7];
    println!("trace (shift version), start {}", bracket(&fixed, 1));
    let mut a = fixed;
    let s = insertion_shift(&mut a, true);
    println!("  total: {} comparisons, {} writes", s.comps, s.writes);

    const N: usize = 1000;
    let rows: [(&str, Vec<i64>); 4] = [
        ("sorted", (0..N as i64).collect()),
        ("reversed", (0..N as i64).rev().collect()),
        // 5 1 4 2 8 0 3 7 5 1 4 2 ... (ties)
        ("tiled pattern", (0..N).map(|i| fixed[i % 8]).collect()),
        // the same shape, each block above the last
        (
            "tiled blocks",
            (0..N).map(|i| fixed[i % 8] + 9 * (i / 8) as i64).collect(),
        ),
    ];
    println!();
    println!("n = {N}: same comparisons, different writes");
    println!(
        "  {:<14} {:>8} | {:>8} {:>8} | results equal",
        "input", "comps", "swap wr", "shift wr"
    );
    let mut got = Vec::new();
    let (mut same_comps, mut twice) = (true, true);
    for (name, input) in &rows {
        let (mut x, mut y) = (input.clone(), input.clone());
        let (sw, sh) = (insertion_swap(&mut x), insertion_shift(&mut y, false));
        same_comps &= sw.comps == sh.comps;
        twice &= sw.writes == 2 * sh.shifts;
        println!(
            "  {name:<14} {:>8} | {:>8} {:>8} | {}",
            sw.comps,
            sw.writes,
            sh.writes,
            x == y
        );
        got.push(sh);
    }
    println!();
    println!("  the two versions made the same comparisons on every row: {same_comps}");
    println!(
        "  sorted:   n-1 = {} comparisons, 0 writes: {}",
        N - 1,
        got[0].comps == N - 1 && got[0].writes == 0
    );
    println!(
        "  reversed: n(n-1)/2 = {} comparisons: {}",
        N * (N - 1) / 2,
        got[1].comps == N * (N - 1) / 2
    );
    println!(
        "  swap writes = 2 x shift steps on every row: {twice} (the rest of the shift writes are drops)"
    );
    println!(
        "  tiled blocks: the same shape per block, but every element is < 8 places from home:"
    );
    println!(
        "    {} comparisons ({:.1} per element) vs {} for the plain tiled pattern -- {:.0}x fewer.",
        got[3].comps,
        got[3].comps as f64 / N as f64,
        got[2].comps,
        got[2].comps as f64 / got[3].comps as f64
    );

    // Rust only: the rotate_right spelling on a type that is not Copy.
    let words: Vec<String> = rows[2].1.iter().map(|v| format!("k{v}")).collect();
    let mut by_rotate = words.clone();
    let rc = insertion_rotate(&mut by_rotate);
    let mut by_std = words;
    by_std.sort(); // stable std sort as the oracle
    println!(
        "  rotate_right version on {N} Strings (not Copy): {rc} comparisons (same as the tiled pattern row: {}), order matches sort(): {}",
        rc == got[2].comps,
        by_rotate == by_std
    );
}
```

**Rust output:**

```
trace (shift version), start [5 | 1 4 2 8 0 3 7]
  insert 1: shifts 1 -> [1 5 | 4 2 8 0 3 7]
  insert 4: shifts 1 -> [1 4 5 | 2 8 0 3 7]
  insert 2: shifts 2 -> [1 2 4 5 | 8 0 3 7]
  insert 8: shifts 0 -> [1 2 4 5 8 | 0 3 7]
  insert 0: shifts 5 -> [0 1 2 4 5 8 | 3 7]
  insert 3: shifts 3 -> [0 1 2 3 4 5 8 | 7]
  insert 7: shifts 1 -> [0 1 2 3 4 5 7 8]
  total: 18 comparisons, 19 writes

n = 1000: same comparisons, different writes
  input             comps |  swap wr shift wr | results equal
  sorted              999 |        0        0 | true
  reversed         499500 |   999000   500499 | true
  tiled pattern    219622 |   437250   219499 | true
  tiled blocks       2622 |     3250     2375 | true

  the two versions made the same comparisons on every row: true
  sorted:   n-1 = 999 comparisons, 0 writes: true
  reversed: n(n-1)/2 = 499500 comparisons: true
  swap writes = 2 x shift steps on every row: true (the rest of the shift writes are drops)
  tiled blocks: the same shape per block, but every element is < 8 places from home:
    2622 comparisons (2.6 per element) vs 219622 for the plain tiled pattern -- 84x fewer.
  rotate_right version on 1000 Strings (not Copy): 219622 comparisons (same as the tiled pattern row: true), order matches sort(): true
```

> **Go vs Rust:** Lines 1–23 are identical to the Go output (diffed); the last line is Rust-only. The hole needs `T: Copy` in safe Rust: `let x = a[i]` on a `String` would move out of the slice, which the compiler refuses. So the twin adds the safe generic spelling, the same walk followed by `a[j..=i].rotate_right(1)`, and runs it on 1000 `String`s: it makes the same 219,622 comparisons. std itself shifts through a real hole with `ptr::copy_nonoverlapping` and a `CopyOnDrop` guard that refills it if the comparator panics (`insert_tail` in `core/src/slice/sort/shared/smallsort.rs`, read in the source, not run). Like the twin, it skips the hold when the element is already in place.

**Complexity:** best Θ(n) (999 comparisons, 0 writes on sorted n = 1000) · worst Θ(n²) (n(n−1)/2 = 499,500 on reversed) · the swap version writes 2 per step, the shift version 1 per step + 1 drop per element that moved (999,000 vs 500,499 writes on reversed input), with identical comparisons · stable (strict `<`), in place, online · elements < 8 places from home: 2,622 comparisons instead of 219,622

---

## 4. Loop invariants, checked at every step

`🟢 easy` · *Invariants*

Each of the three sorts is correct because of one sentence that is true before its loop, stays true after every
iteration and, when the loop ends, says the slice is sorted: bubble keeps a sorted suffix that is ≥ everything
before it, selection a sorted prefix that is ≤ everything after it, insertion a sorted prefix that is a
permutation of the original prefix. Written as a checker and called after every outer iteration, that sentence
turns the initialization / maintenance / termination proof into something you can run. A planted off-by-one is
then caught at the exact step, with the clause that broke. Checking costs Θ(n) per step, so Θ(n²) per sort on top
of the sort: a debugging tool, not production code.

**Steps:**

1. Write the three sorts with an `off` parameter (0 = correct, 1 = one planted off-by-one) and a `chk` callback.
2. Write each invariant as a function that returns `nil` or an error naming the failed clause.
3. Run 1000 seeded random inputs with ties through all six variants and read where each bug was caught.

```go
package main

import (
	"errors"
	"fmt"
	"math/rand/v2"
	"slices"
)

// A loop invariant is true before the loop (initialization), stays true after
// every iteration (maintenance), and at the end implies the result
// (termination). Each sort below calls chk before its loop and after every
// outer iteration, passing b, the boundary that iteration has just moved:
//   bubble    : a[b:] sorted and >= everything in a[:b]    (suffix final)
//   selection : a[:b] sorted and <= everything in a[b:]    (prefix final)
//   insertion : a[:b] sorted, a permutation of orig[:b]    (prefix sorted, NOT final)
// off = 0 is the correct sort; off = 1 plants one off-by-one.

type check func(a []int, step, b int)

func bubble(a []int, off int, chk check) {
	chk(a, 0, len(a))
	for end := len(a) - 1; end > 0; end-- {
		for j := 0; j < end-off; j++ { // BUG with off = 1: stops one pair early
			if a[j] > a[j+1] {
				a[j], a[j+1] = a[j+1], a[j]
			}
		}
		chk(a, len(a)-end, end)
	}
}

func selection(a []int, off int, chk check) {
	chk(a, 0, 0)
	for i := 0; i < len(a)-1; i++ {
		m := i
		for j := i + 1 + off; j < len(a); j++ { // BUG with off = 1: never looks at a[i+1]
			if a[j] < a[m] {
				m = j
			}
		}
		a[i], a[m] = a[m], a[i]
		chk(a, i+1, i+1)
	}
}

func insertion(a []int, off int, chk check) {
	chk(a, 0, min(1, len(a)))
	for i := 1; i < len(a); i++ {
		x, j := a[i], i
		for ; j > off && x < a[j-1]; j-- { // BUG with off = 1: j > 1 never compares a[0]
			a[j] = a[j-1]
		}
		a[j] = x
		chk(a, i, i+1)
	}
}

// The checkers: nil, or an error naming the clause that failed.
// split reports whether every element of lo is <= every element of hi.
func split(lo, hi []int) bool {
	return len(lo) == 0 || len(hi) == 0 || slices.Max(lo) <= slices.Min(hi)
}

func bubbleInv(a, _ []int, b int) error {
	if !slices.IsSorted(a[b:]) {
		return errors.New("suffix a[b:] is not sorted")
	}
	if !split(a[:b], a[b:]) {
		return errors.New("suffix a[b:] is not >= everything before it")
	}
	return nil
}

func selectionInv(a, _ []int, b int) error {
	if !slices.IsSorted(a[:b]) {
		return errors.New("prefix a[:b] is not sorted")
	}
	if !split(a[:b], a[b:]) {
		return errors.New("prefix a[:b] is not <= everything after it")
	}
	return nil
}

func insertionInv(a, orig []int, b int) error {
	if !slices.IsSorted(a[:b]) {
		return errors.New("prefix a[:b] is not sorted")
	}
	got, want := slices.Clone(a[:b]), slices.Clone(orig[:b])
	slices.Sort(got)
	slices.Sort(want)
	if !slices.Equal(got, want) { // same multiset = a permutation
		return errors.New("a[:b] is not a permutation of the original a[:b]")
	}
	return nil
}

type sorter struct {
	name string
	sort func([]int, int, check)
	inv  func(a, orig []int, b int) error
}

// run sorts a copy of in, checking after every step; it never stops early,
// so the final output can be inspected too. fail is the FIRST failure.
func run(s sorter, in []int, off int) (passed int, out []int, fail string) {
	out = slices.Clone(in)
	s.sort(out, off, func(a []int, step, b int) {
		if err := s.inv(a, in, b); err == nil {
			passed++
		} else if fail == "" {
			fail = fmt.Sprintf("after step %d (b = %d): %v\n      array: %v", step, b, err, a)
		}
	})
	return passed, out, fail
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	trials := make([][]int, 1000)
	for t := range trials {
		trials[t] = make([]int, r.IntN(31)) // n in [0, 30]
		for i := range trials[t] {
			trials[t][i] = r.IntN(10) // values in [0, 10): ties are common
		}
	}
	sorts := []sorter{{"bubble", bubble, bubbleInv},
		{"selection", selection, selectionInv}, {"insertion", insertion, insertionInv}}

	missed := 0 // unsorted outputs that no check flagged
	for off, title := range []string{"correct sorts", "one planted off-by-one each"} {
		fmt.Printf("%s, %d random inputs (n in [0, 30], values in [0, 10))\n", title, len(trials))
		for _, s := range sorts {
			passed, failedTrials, unsorted, first, firstFail := 0, 0, 0, -1, ""
			for t, in := range trials {
				p, out, fail := run(s, in, off)
				passed += p
				if !slices.IsSorted(out) {
					unsorted++
					if fail == "" {
						missed++
					}
				}
				if fail != "" {
					failedTrials++
					if first < 0 {
						first, firstFail = t, fail
					}
				}
			}
			fmt.Printf("  %-9s  checks passed %5d | trials with a failed check %3d | unsorted outputs %3d\n",
				s.name, passed, failedTrials, unsorted)
			if first >= 0 {
				fmt.Printf("    first failure: trial %d, input %v\n    %s\n", first, trials[first], firstFail)
			}
		}
		fmt.Println()
	}
	fmt.Printf("unsorted outputs that no check flagged: %d -- each bug was caught at a named step,\n", missed)
	fmt.Println("with the clause that broke, without looking at the final output.")
}
```

**Output:**

```
correct sorts, 1000 random inputs (n in [0, 30], values in [0, 10))
  bubble     checks passed 15281 | trials with a failed check   0 | unsorted outputs   0
  selection  checks passed 15281 | trials with a failed check   0 | unsorted outputs   0
  insertion  checks passed 15281 | trials with a failed check   0 | unsorted outputs   0

one planted off-by-one each, 1000 random inputs (n in [0, 30], values in [0, 10))
  bubble     checks passed  2662 | trials with a failed check 784 | unsorted outputs 784
    first failure: trial 0, input [6 7 7 2 0 4 4 1 1 6 4 9 1 3 7 7 5 0 3 0 6 2 0]
    after step 1 (b = 22): suffix a[b:] is not >= everything before it
      array: [6 7 2 0 4 4 1 1 6 4 7 1 3 7 7 5 0 3 0 6 2 9 0]
  selection  checks passed 13090 | trials with a failed check 463 | unsorted outputs 463
    first failure: trial 1, input [7 7 8 9 4 5 6 5 0 6]
    after step 8 (b = 8): prefix a[:b] is not <= everything after it
      array: [0 4 5 5 6 6 7 8 7 9]
  insertion  checks passed  4104 | trials with a failed check 770 | unsorted outputs 770
    first failure: trial 0, input [6 7 7 2 0 4 4 1 1 6 4 9 1 3 7 7 5 0 3 0 6 2 0]
    after step 3 (b = 4): prefix a[:b] is not sorted
      array: [6 2 7 7 0 4 4 1 1 6 4 9 1 3 7 7 5 0 3 0 6 2 0]

unsorted outputs that no check flagged: 0 -- each bug was caught at a named step,
with the clause that broke, without looking at the final output.
```

**Rust:**

```rust
// A loop invariant is true before the loop (initialization), stays true after
// every iteration (maintenance), and at the end implies the result
// (termination). Each sort below calls chk before its loop and after every
// outer iteration, passing b, the boundary that iteration has just moved:
//   bubble    : a[b..] sorted and >= everything in a[..b]   (suffix final)
//   selection : a[..b] sorted and <= everything in a[b..]   (prefix final)
//   insertion : a[..b] sorted, a permutation of orig[..b]   (prefix sorted, NOT final)
// off = 0 is the correct sort; off = 1 plants one off-by-one.
//
// Rust note: the obvious tool, debug_assert!, is compiled OUT by `rustc -O`
// (how this program is built), and assert! would panic on the first broken
// step. So each checker is a plain function returning Result<(), String>, and
// the caller decides what a failure means.
//
// The checker cannot capture `a` while the sort holds `&mut a` (the borrow
// checker refuses a closure that reads what is being mutated), so the sort
// hands the slice to the callback at each step: chk(a, step, b).

type Check<'a> = &'a mut dyn FnMut(&[i64], usize, usize);

fn bubble(a: &mut [i64], off: usize, chk: Check) {
    chk(a, 0, a.len());
    for end in (1..a.len()).rev() {
        for j in 0..end - off {
            // BUG with off = 1: stops one pair early
            if a[j] > a[j + 1] {
                a.swap(j, j + 1);
            }
        }
        chk(a, a.len() - end, end);
    }
}

fn selection(a: &mut [i64], off: usize, chk: Check) {
    chk(a, 0, 0);
    for i in 0..a.len().saturating_sub(1) {
        let mut m = i;
        for j in i + 1 + off..a.len() {
            // BUG with off = 1: never looks at a[i+1]
            if a[j] < a[m] {
                m = j;
            }
        }
        a.swap(i, m);
        chk(a, i + 1, i + 1);
    }
}

fn insertion(a: &mut [i64], off: usize, chk: Check) {
    chk(a, 0, a.len().min(1));
    for i in 1..a.len() {
        let (x, mut j) = (a[i], i);
        while j > off && x < a[j - 1] {
            // BUG with off = 1: j > 1 never compares a[0]
            a[j] = a[j - 1];
            j -= 1;
        }
        a[j] = x;
        chk(a, i, i + 1);
    }
}

// The checkers: Ok(()), or Err naming the clause that failed.
/// Every element of lo is <= every element of hi (true if either is empty).
fn split(lo: &[i64], hi: &[i64]) -> bool {
    match (lo.iter().max(), hi.iter().min()) {
        (Some(x), Some(y)) => x <= y,
        _ => true,
    }
}

fn bubble_inv(a: &[i64], _: &[i64], b: usize) -> Result<(), String> {
    if !a[b..].is_sorted() {
        return Err("suffix a[b:] is not sorted".into());
    }
    if !split(&a[..b], &a[b..]) {
        return Err("suffix a[b:] is not >= everything before it".into());
    }
    Ok(())
}

fn selection_inv(a: &[i64], _: &[i64], b: usize) -> Result<(), String> {
    if !a[..b].is_sorted() {
        return Err("prefix a[:b] is not sorted".into());
    }
    if !split(&a[..b], &a[b..]) {
        return Err("prefix a[:b] is not <= everything after it".into());
    }
    Ok(())
}

fn insertion_inv(a: &[i64], orig: &[i64], b: usize) -> Result<(), String> {
    if !a[..b].is_sorted() {
        return Err("prefix a[:b] is not sorted".into());
    }
    let (mut got, mut want) = (a[..b].to_vec(), orig[..b].to_vec());
    got.sort_unstable();
    want.sort_unstable();
    if got != want {
        // same multiset = a permutation
        return Err("a[:b] is not a permutation of the original a[:b]".into());
    }
    Ok(())
}

/// A checker: (array now, original input, boundary b) -> Ok or the failed clause.
type Invariant = fn(&[i64], &[i64], usize) -> Result<(), String>;

struct Sorter {
    name: &'static str,
    sort: fn(&mut [i64], usize, Check),
    inv: Invariant,
}

/// Sorts a copy of `input`, checking after every step; never stops early, so
/// the final output can be inspected too. The failure is the FIRST one.
fn run(s: &Sorter, input: &[i64], off: usize) -> (usize, Vec<i64>, Option<String>) {
    let mut out = input.to_vec();
    let (mut passed, mut fail) = (0, None);
    (s.sort)(
        &mut out,
        off,
        &mut |a, step, b| match (s.inv)(a, input, b) {
            Ok(()) => passed += 1,
            Err(e) => {
                fail.get_or_insert_with(|| {
                    format!(
                        "after step {step} (b = {b}): {e}\n      array: {}",
                        go_list(a)
                    )
                });
            }
        },
    );
    (passed, out, fail)
}

/// Go's %v of a slice: [1 2 3]
fn go_list(a: &[i64]) -> String {
    let parts: Vec<String> = a.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN (lesson 01, example 14), so the same seed gives Go's exact inputs.
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
        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

fn main() {
    let mut r = Pcg::new(1, 2);
    let trials: Vec<Vec<i64>> = (0..1000)
        .map(|_| {
            let n = r.int_n(31); // n in [0, 30]
            // values in [0, 10): ties are common
            (0..n).map(|_| r.int_n(10) as i64).collect()
        })
        .collect();
    let sorts = [
        Sorter {
            name: "bubble",
            sort: bubble,
            inv: bubble_inv,
        },
        Sorter {
            name: "selection",
            sort: selection,
            inv: selection_inv,
        },
        Sorter {
            name: "insertion",
            sort: insertion,
            inv: insertion_inv,
        },
    ];

    let mut missed = 0; // unsorted outputs that no check flagged
    for (off, title) in ["correct sorts", "one planted off-by-one each"]
        .into_iter()
        .enumerate()
    {
        println!(
            "{title}, {} random inputs (n in [0, 30], values in [0, 10))",
            trials.len()
        );
        for s in &sorts {
            let (mut passed, mut failed_trials, mut unsorted) = (0, 0, 0);
            let mut first: Option<(usize, String)> = None;
            for (t, input) in trials.iter().enumerate() {
                let (p, out, fail) = run(s, input, off);
                passed += p;
                if !out.is_sorted() {
                    unsorted += 1;
                    if fail.is_none() {
                        missed += 1;
                    }
                }
                if let Some(f) = fail {
                    failed_trials += 1;
                    first.get_or_insert((t, f));
                }
            }
            println!(
                "  {:<9}  checks passed {passed:>5} | trials with a failed check {failed_trials:>3} | unsorted outputs {unsorted:>3}",
                s.name
            );
            if let Some((t, f)) = first {
                println!(
                    "    first failure: trial {t}, input {}\n    {f}",
                    go_list(&trials[t])
                );
            }
        }
        println!();
    }
    println!(
        "unsorted outputs that no check flagged: {missed} -- each bug was caught at a named step,"
    );
    println!("with the clause that broke, without looking at the final output.");
}
```

**Rust output:**

```
correct sorts, 1000 random inputs (n in [0, 30], values in [0, 10))
  bubble     checks passed 15281 | trials with a failed check   0 | unsorted outputs   0
  selection  checks passed 15281 | trials with a failed check   0 | unsorted outputs   0
  insertion  checks passed 15281 | trials with a failed check   0 | unsorted outputs   0

one planted off-by-one each, 1000 random inputs (n in [0, 30], values in [0, 10))
  bubble     checks passed  2662 | trials with a failed check 784 | unsorted outputs 784
    first failure: trial 0, input [6 7 7 2 0 4 4 1 1 6 4 9 1 3 7 7 5 0 3 0 6 2 0]
    after step 1 (b = 22): suffix a[b:] is not >= everything before it
      array: [6 7 2 0 4 4 1 1 6 4 7 1 3 7 7 5 0 3 0 6 2 9 0]
  selection  checks passed 13090 | trials with a failed check 463 | unsorted outputs 463
    first failure: trial 1, input [7 7 8 9 4 5 6 5 0 6]
    after step 8 (b = 8): prefix a[:b] is not <= everything after it
      array: [0 4 5 5 6 6 7 8 7 9]
  insertion  checks passed  4104 | trials with a failed check 770 | unsorted outputs 770
    first failure: trial 0, input [6 7 7 2 0 4 4 1 1 6 4 9 1 3 7 7 5 0 3 0 6 2 0]
    after step 3 (b = 4): prefix a[:b] is not sorted
      array: [6 2 7 7 0 4 4 1 1 6 4 9 1 3 7 7 5 0 3 0 6 2 0]

unsorted outputs that no check flagged: 0 -- each bug was caught at a named step,
with the clause that broke, without looking at the final output.
```

> **Go vs Rust:** Identical output, diffed against the Go run. The `Pcg` port of `math/rand/v2` gives both programs the same 1000 inputs, so the counts (15,281 / 2,662 / 13,090 / 4,104) and the first failing trials agree. `debug_assert!` would be the reflex in Rust, but `rustc -O` compiles it out, and `assert!` would panic at the first broken step, so each checker returns `Result<(), String>`. The callback receives the slice (`chk(a, step, b)`) instead of capturing it, because a closure cannot read `a` while the sort holds `&mut a`.

**Complexity:** the check costs Θ(n) per step, Θ(n²) per sort — a debugging tool, not production · the sorts themselves are unchanged · 15,281 checks passed for each correct sort over 1000 inputs · each planted bug failed its invariant in every trial whose output came out unsorted (784, 463 and 770 of 1000), and the bubble bug was caught after the first pass

---

## 5. Stability, and the one-character bug

`🟢 easy` · *Stable vs unstable*

A sort is stable if equal keys come out in the order they went in. Sorting plain integers hides the property, so
here each element is a `(key, tag)` record compared by key only. Bubble and insertion sort are stable because
they move an element only past a strictly larger neighbour. Selection sort is not: its long-distance swap can jump
an element over its equal twin. Then the classic bug: change `<` to `<=` in insertion sort (or `>` to `>=` in
bubble sort) and the output is still sorted, every "is it sorted?" test still passes, and stability is gone.
Stability costs these sorts nothing extra, so all three stay Θ(n²). The bug costs nothing either, which is why
nobody notices it.

**Steps:**

1. Sort ten `(key, tag)` records with three repeated keys through five variants and check `sorted?` and `stable?`.
2. Show the smallest input that exposes each unstable variant.
3. Run all 81 length-4 key sequences over three keys and count how many each variant reorders.

```go
package main

import (
	"cmp"
	"fmt"
	"slices"
	"strings"
)

// A sort is STABLE if elements with equal keys come out in the order they went
// in. With plain ints you cannot tell; with records sorted by part of the
// record you can. Each element here is (key, tag): the sorts compare ONLY the
// key, and the tag remembers the input position.
//
// Bubble and insertion sort are stable because they only move an element past
// a STRICTLY larger neighbour. Change one character -- < to <= -- and they
// still sort, but now step over equal keys too. Selection sort is unstable by
// design: its long-distance swap can jump an element over its equal twin.

type pair struct{ key, tag int }

func bubble(a []pair, orEqual bool) {
	for end := len(a) - 1; end > 0; end-- {
		for j := 0; j < end; j++ {
			if a[j].key > a[j+1].key || orEqual && a[j].key == a[j+1].key { // > vs >=
				a[j], a[j+1] = a[j+1], a[j]
			}
		}
	}
}

func insertion(a []pair, orEqual bool) {
	for i := 1; i < len(a); i++ {
		x, j := a[i], i
		for ; j > 0 && (x.key < a[j-1].key || orEqual && x.key == a[j-1].key); j-- { // < vs <=
			a[j] = a[j-1]
		}
		a[j] = x
	}
}

func selection(a []pair, _ bool) {
	for i := 0; i < len(a)-1; i++ {
		m := i
		for j := i + 1; j < len(a); j++ {
			if a[j].key < a[m].key {
				m = j
			}
		}
		a[i], a[m] = a[m], a[i] // the jump: a[i] may land behind its equal twin
	}
}

func isSorted(a []pair) bool {
	return slices.IsSortedFunc(a, func(x, y pair) int { return cmp.Compare(x.key, y.key) })
}

func isStable(a []pair) bool { // equal keys must keep increasing tags
	for i := 1; i < len(a); i++ {
		if a[i-1].key == a[i].key && a[i-1].tag > a[i].tag {
			return false
		}
	}
	return true
}

// pairs builds (key, tag = position) records.
func pairs(keys ...int) []pair {
	a := make([]pair, len(keys))
	for i, k := range keys {
		a[i] = pair{k, i}
	}
	return a
}

// show prints each record as key + letter, the letter being its order among
// equal keys in the INPUT: 2a went in before 2b.
func show(a []pair, in []pair) string {
	letter := map[int]byte{}
	seen := map[int]int{}
	for _, p := range in {
		letter[p.tag] = 'a' + byte(seen[p.key])
		seen[p.key]++
	}
	var b strings.Builder
	for _, p := range a {
		fmt.Fprintf(&b, "%d%c ", p.key, letter[p.tag])
	}
	return "[" + strings.TrimSpace(b.String()) + "]"
}

type variant struct {
	name    string
	sort    func([]pair, bool)
	orEqual bool
}

func main() {
	variants := []variant{
		{"bubble  (>)", bubble, false}, {"insertion (<)", insertion, false},
		{"selection", selection, false},
		{"bubble  (>=)", bubble, true}, {"insertion (<=)", insertion, true},
	}
	in := pairs(3, 1, 4, 2, 1, 3, 0, 2, 3, 1)
	fmt.Println("input:", show(in, in))
	for _, v := range variants {
		a := slices.Clone(in)
		v.sort(a, v.orEqual)
		fmt.Printf("  %-15s %s  sorted? %-5v stable? %v\n", v.name, show(a, in), isSorted(a), isStable(a))
	}

	fmt.Println()
	fmt.Println("the smallest inputs that show it:")
	for _, c := range []struct {
		v  variant
		in []pair
	}{{variants[2], pairs(2, 2, 1)}, {variants[4], pairs(1, 1)}, {variants[3], pairs(1, 1)}} {
		a := slices.Clone(c.in)
		c.v.sort(a, c.v.orEqual)
		fmt.Printf("  %-15s %s -> %s\n", c.v.name, show(c.in, c.in), show(a, c.in))
	}

	fmt.Println()
	fmt.Println("all 3^4 = 81 key sequences of length 4 (keys 0..2): how many come out reordered?")
	for _, v := range variants {
		unstable, unsorted := 0, 0
		for code := range 81 {
			keys := []int{code % 3, code / 3 % 3, code / 9 % 3, code / 27}
			a := pairs(keys...)
			v.sort(a, v.orEqual)
			if !isStable(a) {
				unstable++
			}
			if !isSorted(a) {
				unsorted++
			}
		}
		fmt.Printf("  %-15s unstable on %2d of 81, unsorted on %d\n", v.name, unstable, unsorted)
	}
	fmt.Println("the >= and <= variants pass every \"is it sorted?\" check and are still wrong.")
	fmt.Println("every length-4 sequence over 3 keys repeats a key, and insertion (<=) puts each")
	fmt.Println("new equal key in front of the earlier ones (see 1c 1b 1a above), so it fails all 81.")
}
```

**Output:**

```
input: [3a 1a 4a 2a 1b 3b 0a 2b 3c 1c]
  bubble  (>)     [0a 1a 1b 1c 2a 2b 3a 3b 3c 4a]  sorted? true  stable? true
  insertion (<)   [0a 1a 1b 1c 2a 2b 3a 3b 3c 4a]  sorted? true  stable? true
  selection       [0a 1a 1b 1c 2b 2a 3a 3c 3b 4a]  sorted? true  stable? false
  bubble  (>=)    [0a 1c 1a 1b 2a 2b 3c 3b 3a 4a]  sorted? true  stable? false
  insertion (<=)  [0a 1c 1b 1a 2b 2a 3c 3b 3a 4a]  sorted? true  stable? false

the smallest inputs that show it:
  selection       [2a 2b 1a] -> [1a 2b 2a]
  insertion (<=)  [1a 1b] -> [1b 1a]
  bubble  (>=)    [1a 1b] -> [1b 1a]

all 3^4 = 81 key sequences of length 4 (keys 0..2): how many come out reordered?
  bubble  (>)     unstable on  0 of 81, unsorted on 0
  insertion (<)   unstable on  0 of 81, unsorted on 0
  selection       unstable on 20 of 81, unsorted on 0
  bubble  (>=)    unstable on 73 of 81, unsorted on 0
  insertion (<=)  unstable on 81 of 81, unsorted on 0
the >= and <= variants pass every "is it sorted?" check and are still wrong.
every length-4 sequence over 3 keys repeats a key, and insertion (<=) puts each
new equal key in front of the earlier ones (see 1c 1b 1a above), so it fails all 81.
```

**Rust:**

```rust
// A sort is STABLE if elements with equal keys come out in the order they went
// in. With plain ints you cannot tell; with records sorted by part of the
// record you can. Each element here is (key, tag): the sorts compare ONLY the
// key, and the tag remembers the input position.
//
// Rust note: a tuple derives Ord (key first, then tag), so `a < b` on the
// whole tuple would break ties by tag and every sort would look stable. The
// sorts below therefore compare `.0` only -- exactly like a `sort_by_key`
// on part of a record.
//
// Bubble and insertion sort are stable because they only move an element past
// a STRICTLY larger neighbour. Change one character -- < to <= -- and they
// still sort, but now step over equal keys too. Selection sort is unstable by
// design: its long-distance swap can jump an element over its equal twin.

type Pair = (i64, usize); // (key, tag)

fn bubble(a: &mut [Pair], or_equal: bool) {
    for end in (1..a.len()).rev() {
        for j in 0..end {
            if a[j].0 > a[j + 1].0 || or_equal && a[j].0 == a[j + 1].0 {
                // > vs >=
                a.swap(j, j + 1);
            }
        }
    }
}

fn insertion(a: &mut [Pair], or_equal: bool) {
    for i in 1..a.len() {
        let (x, mut j) = (a[i], i);
        while j > 0 && (x.0 < a[j - 1].0 || or_equal && x.0 == a[j - 1].0) {
            // < vs <=
            a[j] = a[j - 1];
            j -= 1;
        }
        a[j] = x;
    }
}

fn selection(a: &mut [Pair], _: bool) {
    for i in 0..a.len().saturating_sub(1) {
        let mut m = i;
        for j in i + 1..a.len() {
            if a[j].0 < a[m].0 {
                m = j;
            }
        }
        a.swap(i, m); // the jump: a[i] may land behind its equal twin
    }
}

fn is_sorted(a: &[Pair]) -> bool {
    a.is_sorted_by_key(|p| p.0) // keys only
}

fn is_stable(a: &[Pair]) -> bool {
    // equal keys must keep increasing tags
    a.windows(2).all(|w| w[0].0 != w[1].0 || w[0].1 < w[1].1)
}

/// (key, tag = position) records.
fn pairs(keys: &[i64]) -> Vec<Pair> {
    keys.iter().copied().zip(0..).collect()
}

/// Each record as key + letter, the letter being its order among equal keys
/// in the INPUT: 2a went in before 2b.
fn show(a: &[Pair], input: &[Pair]) -> String {
    let mut letter = vec!['?'; input.len()]; // indexed by tag (= input position)
    for (pos, &(key, tag)) in input.iter().enumerate() {
        let earlier = input[..pos].iter().filter(|p| p.0 == key).count();
        letter[tag] = char::from(b'a' + earlier as u8);
    }
    let parts: Vec<String> = a
        .iter()
        .map(|&(key, tag)| format!("{key}{}", letter[tag]))
        .collect();
    format!("[{}]", parts.join(" "))
}

struct Variant {
    name: &'static str,
    sort: fn(&mut [Pair], bool),
    or_equal: bool,
}

impl Variant {
    fn run(&self, a: &mut [Pair]) {
        (self.sort)(a, self.or_equal);
    }
}

fn main() {
    let v = |name, sort, or_equal| Variant {
        name,
        sort,
        or_equal,
    };
    let variants = [
        v("bubble  (>)", bubble, false),
        v("insertion (<)", insertion, false),
        v("selection", selection, false),
        v("bubble  (>=)", bubble, true),
        v("insertion (<=)", insertion, true),
    ];
    let input = pairs(&[3, 1, 4, 2, 1, 3, 0, 2, 3, 1]);
    println!("input: {}", show(&input, &input));
    for v in &variants {
        let mut a = input.clone();
        v.run(&mut a);
        println!(
            "  {:<15} {}  sorted? {:<5} stable? {}",
            v.name,
            show(&a, &input),
            is_sorted(&a),
            is_stable(&a)
        );
    }

    println!();
    println!("the smallest inputs that show it:");
    for (v, keys) in [
        (&variants[2], &[2, 2, 1][..]),
        (&variants[4], &[1, 1]),
        (&variants[3], &[1, 1]),
    ] {
        let input = pairs(keys);
        let mut a = input.clone();
        v.run(&mut a);
        println!(
            "  {:<15} {} -> {}",
            v.name,
            show(&input, &input),
            show(&a, &input)
        );
    }

    println!();
    println!("all 3^4 = 81 key sequences of length 4 (keys 0..2): how many come out reordered?");
    for v in &variants {
        let (mut unstable, mut unsorted) = (0, 0);
        for code in 0..81 {
            let mut a = pairs(&[code % 3, code / 3 % 3, code / 9 % 3, code / 27]);
            v.run(&mut a);
            unstable += usize::from(!is_stable(&a));
            unsorted += usize::from(!is_sorted(&a));
        }
        println!(
            "  {:<15} unstable on {unstable:>2} of 81, unsorted on {unsorted}",
            v.name
        );
    }
    println!("the >= and <= variants pass every \"is it sorted?\" check and are still wrong.");
    println!("every length-4 sequence over 3 keys repeats a key, and insertion (<=) puts each");
    println!(
        "new equal key in front of the earlier ones (see 1c 1b 1a above), so it fails all 81."
    );
}
```

**Rust output:**

```
input: [3a 1a 4a 2a 1b 3b 0a 2b 3c 1c]
  bubble  (>)     [0a 1a 1b 1c 2a 2b 3a 3b 3c 4a]  sorted? true  stable? true
  insertion (<)   [0a 1a 1b 1c 2a 2b 3a 3b 3c 4a]  sorted? true  stable? true
  selection       [0a 1a 1b 1c 2b 2a 3a 3c 3b 4a]  sorted? true  stable? false
  bubble  (>=)    [0a 1c 1a 1b 2a 2b 3c 3b 3a 4a]  sorted? true  stable? false
  insertion (<=)  [0a 1c 1b 1a 2b 2a 3c 3b 3a 4a]  sorted? true  stable? false

the smallest inputs that show it:
  selection       [2a 2b 1a] -> [1a 2b 2a]
  insertion (<=)  [1a 1b] -> [1b 1a]
  bubble  (>=)    [1a 1b] -> [1b 1a]

all 3^4 = 81 key sequences of length 4 (keys 0..2): how many come out reordered?
  bubble  (>)     unstable on  0 of 81, unsorted on 0
  insertion (<)   unstable on  0 of 81, unsorted on 0
  selection       unstable on 20 of 81, unsorted on 0
  bubble  (>=)    unstable on 73 of 81, unsorted on 0
  insertion (<=)  unstable on 81 of 81, unsorted on 0
the >= and <= variants pass every "is it sorted?" check and are still wrong.
every length-4 sequence over 3 keys repeats a key, and insertion (<=) puts each
new equal key in front of the earlier ones (see 1c 1b 1a above), so it fails all 81.
```

> **Go vs Rust:** Identical output, diffed against the Go run. Tuples derive `Ord` (key, then tag), so comparing whole `(i64, usize)` pairs would break ties by tag and make every variant look stable. The sorts compare `.0` only, the way `sort_by_key` sorts by part of a record. In Rust std the property is in the name: `sort`/`sort_by`/`sort_by_key` are stable and `sort_unstable*` are not. Go's names are `slices.SortStableFunc` against `slices.SortFunc`.

**Complexity:** stability is free in bubble and insertion sort — strict comparison, same Θ(n²) · selection sort is unstable (20 of 81 length-4 sequences reordered) and becomes stable only by shifting instead of swapping, which costs Θ(n²) writes · the `>=` / `<=` bug leaves every output sorted and reorders 73 and 81 of the 81 sequences

---

## 6. Inversions: what insertion sort really pays

`🟢 easy` · *Inversions and the adjacent-swap barrier*

An inversion is a pair of positions i < j with a[i] > a[j], and counting them gives insertion sort its exact
price. Each shift removes one inversion, so shifts = I. Each element costs one more comparison than its shifts
unless it travels all the way to the front, so comparisons lie in [I, I + n − 1] and the running time is
Θ(n + I). Bubble sort's swaps equal I too. The same fact sets a floor: swapping two out-of-order neighbours removes
exactly one inversion, so any sort that only swaps neighbours needs at least I swaps, and a random permutation has
n(n−1)/4 of them on average, so Θ(n²) on average, however cleverly it picks the neighbours.

**Steps:**

1. Count inversions by brute force and list them for a five-element array.
2. Check the three identities on 10,000 seeded random inputs with ties, and measure the mean I of random permutations.
3. Compare what one adjacent swap and one long-distance swap do to I.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// An inversion is a pair of positions i < j with a[i] > a[j]: two elements in
// the wrong order. Sorted input has I = 0, reversed has n(n-1)/2.
//
// Insertion sort removes exactly one inversion per shift, so shifts = I, and
// it makes one comparison per shift plus at most one more per element (the
// comparison that stops the walk), so I <= comparisons <= I + n - 1.
// Bubble sort's swaps = I too. The reason is the same: swapping two ADJACENT
// out-of-order elements fixes that one pair and changes no other pair's order.

// inversions counts by brute force, O(n^2): every pair, once.
func inversions(a []int) int {
	inv := 0
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i] > a[j] {
				inv++
			}
		}
	}
	return inv
}

func insertion(a []int) (comps, shifts int) {
	for i := 1; i < len(a); i++ {
		x, j := a[i], i
		for j > 0 {
			comps++
			if !(x < a[j-1]) {
				break
			}
			a[j] = a[j-1]
			shifts++
			j--
		}
		a[j] = x
	}
	return comps, shifts
}

func bubble(a []int) (swaps int) {
	for end := len(a) - 1; end > 0; end-- {
		for j := 0; j < end; j++ {
			if a[j] > a[j+1] {
				a[j], a[j+1] = a[j+1], a[j]
				swaps++
			}
		}
	}
	return swaps
}

func main() {
	a := []int{3, 1, 2, 5, 4}
	fmt.Println("(a) the inversions of", a)
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i] > a[j] {
				fmt.Printf("    positions (%d,%d): %d > %d\n", i, j, a[i], a[j])
			}
		}
	}
	c, s := insertion(slices.Clone(a))
	fmt.Printf("    I = %d; insertion sort: %d shifts, %d comparisons; bubble sort: %d swaps\n",
		inversions(a), s, c, bubble(slices.Clone(a)))

	r := rand.New(rand.NewPCG(12, 6))
	const trials = 10000
	badShift, badSwap, badComp, sumI := 0, 0, 0, 0
	for range trials {
		x := make([]int, r.IntN(51)) // n in [0, 50]
		for i := range x {
			x[i] = r.IntN(8) // values in [0, 8): many ties
		}
		inv := inversions(x)
		sumI += inv
		comps, shifts := insertion(slices.Clone(x))
		if shifts != inv {
			badShift++
		}
		if bubble(slices.Clone(x)) != inv {
			badSwap++
		}
		if comps < inv || comps > inv+max(len(x)-1, 0) {
			badComp++
		}
	}
	fmt.Println()
	fmt.Printf("(b) %d random inputs (n in [0, 50], values in [0, 8)), %d inversions in total\n", trials, sumI)
	fmt.Printf("    insertion shifts != I:              %d violations\n", badShift)
	fmt.Printf("    bubble swaps != I:                  %d violations\n", badSwap)
	fmt.Printf("    insertion comps outside [I, I+n-1]: %d violations\n", badComp)

	const n, perms = 100, 1000
	total := 0
	for range perms {
		p := make([]int, n)
		for i := range p {
			p[i] = i
		}
		for i := n - 1; i > 0; i-- { // Fisher-Yates, drawn with IntN
			j := r.IntN(i + 1)
			p[i], p[j] = p[j], p[i]
		}
		total += inversions(p)
	}
	mean := float64(total) / perms
	fmt.Println()
	fmt.Printf("(c) %d random permutations of n = %d: mean I = %.1f, n(n-1)/4 = %d (%+.1f%%)\n",
		perms, n, mean, n*(n-1)/4, 100*(mean-n*(n-1)/4)/(n*(n-1)/4))
	fmt.Println("    each pair is inverted with probability 1/2, so the expected I is half of n(n-1)/2.")

	fmt.Println()
	fmt.Println("(d) the adjacent-swap barrier")
	b := []int{4, 1, 3, 5, 2, 6}
	fmt.Printf("    %v has I = %d; swap each adjacent out-of-order pair on its own:\n", b, inversions(b))
	for j := 0; j+1 < len(b); j++ {
		if b[j] > b[j+1] {
			t := slices.Clone(b)
			t[j], t[j+1] = t[j+1], t[j]
			fmt.Printf("      swap positions %d,%d -> %v: I = %d (%+d)\n", j, j+1, t, inversions(t), inversions(t)-inversions(b))
		}
	}
	rev := []int{9, 8, 7, 6, 5, 4, 3, 2, 1, 0}
	far := slices.Clone(rev)
	far[0], far[9] = far[9], far[0]
	fmt.Printf("    reversed n = 10, I = %d; swap the two ENDS -> I = %d (%+d, and 2n-3 = %d)\n",
		inversions(rev), inversions(far), inversions(far)-inversions(rev), 2*10-3)
	fmt.Println("    one adjacent swap removes at most one inversion, so a sort that only swaps")
	fmt.Println("    neighbours needs >= I swaps: n(n-1)/4 on average, Theta(n^2). A long-distance")
	fmt.Println("    swap can remove many at once -- the way out (Shell sort, example 14; lesson 13).")
}
```

**Output:**

```
(a) the inversions of [3 1 2 5 4]
    positions (0,1): 3 > 1
    positions (0,2): 3 > 2
    positions (3,4): 5 > 4
    I = 3; insertion sort: 3 shifts, 6 comparisons; bubble sort: 3 swaps

(b) 10000 random inputs (n in [0, 50], values in [0, 8)), 1753391 inversions in total
    insertion shifts != I:              0 violations
    bubble swaps != I:                  0 violations
    insertion comps outside [I, I+n-1]: 0 violations

(c) 1000 random permutations of n = 100: mean I = 2467.5, n(n-1)/4 = 2475 (-0.3%)
    each pair is inverted with probability 1/2, so the expected I is half of n(n-1)/2.

(d) the adjacent-swap barrier
    [4 1 3 5 2 6] has I = 5; swap each adjacent out-of-order pair on its own:
      swap positions 0,1 -> [1 4 3 5 2 6]: I = 4 (-1)
      swap positions 3,4 -> [4 1 3 2 5 6]: I = 4 (-1)
    reversed n = 10, I = 45; swap the two ENDS -> I = 28 (-17, and 2n-3 = 17)
    one adjacent swap removes at most one inversion, so a sort that only swaps
    neighbours needs >= I swaps: n(n-1)/4 on average, Theta(n^2). A long-distance
    swap can remove many at once -- the way out (Shell sort, example 14; lesson 13).
```

**Rust:**

```rust
// An inversion is a pair of positions i < j with a[i] > a[j]: two elements in
// the wrong order. Sorted input has I = 0, reversed has n(n-1)/2.
//
// Insertion sort removes exactly one inversion per shift, so shifts = I, and
// it makes one comparison per shift plus at most one more per element (the
// comparison that stops the walk), so I <= comparisons <= I + n - 1.
// Bubble sort's swaps = I too. The reason is the same: swapping two ADJACENT
// out-of-order elements fixes that one pair and changes no other pair's order.

/// Brute force, O(n^2): every pair once. The iterator form of the double
/// loop -- for each x, count the smaller elements after it.
fn inversions(a: &[i64]) -> usize {
    a.iter()
        .enumerate()
        .map(|(i, x)| a[i + 1..].iter().filter(|&y| x > y).count())
        .sum()
}

/// (comparisons, shifts)
fn insertion(a: &mut [i64]) -> (usize, usize) {
    let (mut comps, mut shifts) = (0, 0);
    for i in 1..a.len() {
        let (x, mut j) = (a[i], i);
        while j > 0 {
            comps += 1;
            if x >= a[j - 1] {
                break;
            }
            a[j] = a[j - 1];
            shifts += 1;
            j -= 1;
        }
        a[j] = x;
    }
    (comps, shifts)
}

/// swaps
fn bubble(a: &mut [i64]) -> usize {
    let mut swaps = 0;
    for end in (1..a.len()).rev() {
        for j in 0..end {
            if a[j] > a[j + 1] {
                a.swap(j, j + 1);
                swaps += 1;
            }
        }
    }
    swaps
}

/// Go's %v of a slice: [1 2 3]
fn go_list(a: &[i64]) -> String {
    let parts: Vec<String> = a.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN (lesson 01, example 14), so the same seed gives Go's exact inputs.
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
        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

fn main() {
    let a = [3, 1, 2, 5, 4];
    println!("(a) the inversions of {}", go_list(&a));
    for i in 0..a.len() {
        for j in i + 1..a.len() {
            if a[i] > a[j] {
                println!("    positions ({i},{j}): {} > {}", a[i], a[j]);
            }
        }
    }
    let (c, s) = insertion(&mut a.clone());
    println!(
        "    I = {}; insertion sort: {s} shifts, {c} comparisons; bubble sort: {} swaps",
        inversions(&a),
        bubble(&mut a.clone())
    );

    let mut r = Pcg::new(12, 6);
    const TRIALS: usize = 10000;
    let (mut bad_shift, mut bad_swap, mut bad_comp, mut sum_i) = (0, 0, 0, 0);
    for _ in 0..TRIALS {
        let n = r.int_n(51); // n in [0, 50]
        // values in [0, 8): many ties
        let x: Vec<i64> = (0..n).map(|_| r.int_n(8) as i64).collect();
        let inv = inversions(&x);
        sum_i += inv;
        let (comps, shifts) = insertion(&mut x.clone());
        bad_shift += usize::from(shifts != inv);
        bad_swap += usize::from(bubble(&mut x.clone()) != inv);
        // saturating_sub: n - 1 would underflow a usize for the empty input
        bad_comp += usize::from(!(inv..=inv + x.len().saturating_sub(1)).contains(&comps));
    }
    println!();
    println!(
        "(b) {TRIALS} random inputs (n in [0, 50], values in [0, 8)), {sum_i} inversions in total"
    );
    println!("    insertion shifts != I:              {bad_shift} violations");
    println!("    bubble swaps != I:                  {bad_swap} violations");
    println!("    insertion comps outside [I, I+n-1]: {bad_comp} violations");

    const N: usize = 100;
    const PERMS: usize = 1000;
    let mut total = 0;
    for _ in 0..PERMS {
        let mut p: Vec<i64> = (0..N as i64).collect();
        for i in (1..N).rev() {
            // Fisher-Yates, drawn with int_n in Go's order
            let j = r.int_n(i as u64 + 1) as usize;
            p.swap(i, j);
        }
        total += inversions(&p);
    }
    let mean = total as f64 / PERMS as f64;
    let expect = N * (N - 1) / 4;
    println!();
    println!(
        "(c) {PERMS} random permutations of n = {N}: mean I = {mean:.1}, n(n-1)/4 = {expect} ({:+.1}%)",
        100.0 * (mean - expect as f64) / expect as f64
    );
    println!(
        "    each pair is inverted with probability 1/2, so the expected I is half of n(n-1)/2."
    );

    println!();
    println!("(d) the adjacent-swap barrier");
    let b = [4, 1, 3, 5, 2, 6];
    let ib = inversions(&b) as i64;
    println!(
        "    {} has I = {ib}; swap each adjacent out-of-order pair on its own:",
        go_list(&b)
    );
    for j in 0..b.len() - 1 {
        if b[j] > b[j + 1] {
            let mut t = b;
            t.swap(j, j + 1);
            let it = inversions(&t) as i64;
            println!(
                "      swap positions {j},{} -> {}: I = {it} ({:+})",
                j + 1,
                go_list(&t),
                it - ib
            );
        }
    }
    let rev: Vec<i64> = (0..10).rev().collect();
    let mut far = rev.clone();
    far.swap(0, 9);
    let (ir, ifar) = (inversions(&rev) as i64, inversions(&far) as i64);
    println!(
        "    reversed n = 10, I = {ir}; swap the two ENDS -> I = {ifar} ({:+}, and 2n-3 = {})",
        ifar - ir,
        2 * 10 - 3
    );
    println!("    one adjacent swap removes at most one inversion, so a sort that only swaps");
    println!("    neighbours needs >= I swaps: n(n-1)/4 on average, Theta(n^2). A long-distance");
    println!(
        "    swap can remove many at once -- the way out (Shell sort, example 14; lesson 13)."
    );
}
```

**Rust output:**

```
(a) the inversions of [3 1 2 5 4]
    positions (0,1): 3 > 1
    positions (0,2): 3 > 2
    positions (3,4): 5 > 4
    I = 3; insertion sort: 3 shifts, 6 comparisons; bubble sort: 3 swaps

(b) 10000 random inputs (n in [0, 50], values in [0, 8)), 1753391 inversions in total
    insertion shifts != I:              0 violations
    bubble swaps != I:                  0 violations
    insertion comps outside [I, I+n-1]: 0 violations

(c) 1000 random permutations of n = 100: mean I = 2467.5, n(n-1)/4 = 2475 (-0.3%)
    each pair is inverted with probability 1/2, so the expected I is half of n(n-1)/2.

(d) the adjacent-swap barrier
    [4 1 3 5 2 6] has I = 5; swap each adjacent out-of-order pair on its own:
      swap positions 0,1 -> [1 4 3 5 2 6]: I = 4 (-1)
      swap positions 3,4 -> [4 1 3 2 5 6]: I = 4 (-1)
    reversed n = 10, I = 45; swap the two ENDS -> I = 28 (-17, and 2n-3 = 17)
    one adjacent swap removes at most one inversion, so a sort that only swaps
    neighbours needs >= I swaps: n(n-1)/4 on average, Theta(n^2). A long-distance
    swap can remove many at once -- the way out (Shell sort, example 14; lesson 13).
```

> **Go vs Rust:** Identical output, diffed against the Go run. The `Pcg` port and a hand-written Fisher–Yates that draws `int_n(i + 1)` for i = n−1 … 1 give the same 10,000 inputs and 1000 permutations, so 1,753,391 and the mean 2467.5 agree digit for digit. That is also the draw order of `math/rand/v2`'s `Shuffle`/`Perm` (read in the source, not run). The brute-force count becomes an iterator chain: for each element, `filter(..).count()` the smaller ones after it. The bound `I + n − 1` needs `saturating_sub`, because `0usize - 1` panics in a debug build, where Go's signed `int` simply goes to −1.

**Complexity:** insertion sort Θ(n + I): shifts = I and comparisons in [I, I + n − 1], with 0 violations in 10,000 random inputs (bubble swaps = I as well) · I = 0 sorted, n(n−1)/2 reversed, n(n−1)/4 on average (measured mean 2467.5 against 2475 at n = 100) · every adjacent-swap sort needs Ω(I) swaps, Ω(n²) on average, while one long-distance swap removed 17 inversions at once

---

> Next tier: [🟡 medium](2-medium.md) — the three sorts counted head to head, nearly sorted data,
> binary insertion, multi-key sorts, and the comparator contract.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
