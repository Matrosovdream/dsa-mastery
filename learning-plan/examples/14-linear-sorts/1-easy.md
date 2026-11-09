# Step 14 — Linear-Time Sorts · 🟢 Easy

Examples **1–6**: the Ω(n log n) wall and the loophole through it, counting sort for bare keys and for
records, the `k` that can ruin it, and LSD radix sort with the stability that makes it correct.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic — your output should match character for character;
**Sample output:** ones print timings, which will differ on your machine.

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

## 1. The decision-tree bound: log2(n!) comparisons

`🟢 easy` · *Lower bound*

A comparison sort learns about its input one bit at a time, from `a < b?`, and the input could be any of
n! orderings — so in the worst case it must ask at least ⌈log₂ n!⌉ questions, whatever the algorithm.
This program makes that bound concrete with an **adversary**: it never fixes the input, keeps every
ordering still consistent with its answers, and answers each question so that the larger half survives.
Real insertion, binary-insertion and merge sorts are forced to at least ⌈log₂ n!⌉ comparisons for
n = 4…8, which is Ω(n log n). Then counting sort sorts with **zero** comparisons, because it uses a key
as an array index — the loophole the rest of the lesson goes through.

**Steps:**

1. Tabulate log₂ n!, its exact ceiling (bit length of n! − 1 for n ≤ 20) and n·log₂ n.
2. Run three sorts that may only call `less(i, j)` against an adversary that holds all n! candidate orderings.
3. Check every count is ≥ the bound and exactly one ordering is left; then place ids by key with no comparison at all.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
)

// A comparison sort learns about its input ONLY through answers to "is a < b?".
// Each answer is one bit, and the input could be any of n! orderings, so the
// algorithm needs at least ceil(log2 n!) answers in the worst case: a binary
// decision tree with n! leaves has height >= ceil(log2 n!).
//
// The adversary below makes that concrete. It never fixes the input in advance.
// It keeps every ordering still consistent with its earlier answers and answers
// each question so that the LARGER half survives. Every answer at best halves
// the candidates, so any algorithm must ask >= ceil(log2 n!) questions before
// one ordering is left.

// ceilLog2Fact is exact for n <= 20 (n! fits in a uint64): ceil(log2 m) is the
// bit length of m-1. No float rounding can sneak in.
func ceilLog2Fact(n int) int {
	f := uint64(1)
	for i := 2; i <= n; i++ {
		f *= uint64(i)
	}
	return bits.Len64(f - 1)
}

// adversary holds every rank assignment still possible: cand[c][id] = rank of id.
type adversary struct {
	cand  [][]uint8
	asked int
}

func newAdversary(n int) *adversary {
	a := &adversary{}
	perm := make([]uint8, n)
	var gen func(k int, used uint)
	gen = func(k int, used uint) { // every permutation of 0..n-1
		if k == n {
			a.cand = append(a.cand, append([]uint8(nil), perm...))
			return
		}
		for r := 0; r < n; r++ {
			if used&(1<<r) == 0 {
				perm[k] = uint8(r)
				gen(k+1, used|1<<r)
			}
		}
	}
	gen(0, 0)
	return a
}

// less answers "is element i smaller than element j?" -- whichever answer keeps
// more candidates alive (a tie answers true), then forgets the others.
func (a *adversary) less(i, j int) bool {
	a.asked++
	yes := 0
	for _, r := range a.cand {
		if r[i] < r[j] {
			yes++
		}
	}
	ans := 2*yes >= len(a.cand)
	kept := a.cand[:0]
	for _, r := range a.cand {
		if (r[i] < r[j]) == ans {
			kept = append(kept, r)
		}
	}
	a.cand = kept
	return ans
}

// Three real comparison sorts. Each sees ids 0..n-1 and may only call less.
func insertion(n int, less func(i, j int) bool) []int {
	a := ids(n)
	for i := 1; i < n; i++ {
		for j := i; j > 0 && less(a[j], a[j-1]); j-- {
			a[j], a[j-1] = a[j-1], a[j]
		}
	}
	return a
}

func binaryInsertion(n int, less func(i, j int) bool) []int {
	a := ids(n)
	for i := 1; i < n; i++ {
		x, lo, hi := a[i], 0, i
		for lo < hi { // first position whose element is greater than x
			mid := (lo + hi) / 2
			if less(x, a[mid]) {
				hi = mid
			} else {
				lo = mid + 1
			}
		}
		copy(a[lo+1:i+1], a[lo:i])
		a[lo] = x
	}
	return a
}

func mergeSort(a []int, less func(i, j int) bool) []int {
	if len(a) <= 1 {
		return a
	}
	l := mergeSort(append([]int(nil), a[:len(a)/2]...), less)
	r := mergeSort(append([]int(nil), a[len(a)/2:]...), less)
	out := make([]int, 0, len(a))
	for len(l) > 0 && len(r) > 0 {
		if less(r[0], l[0]) { // take right only if strictly smaller: stable
			out, r = append(out, r[0]), r[1:]
		} else {
			out, l = append(out, l[0]), l[1:]
		}
	}
	return append(append(out, l...), r...)
}

func ids(n int) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = i
	}
	return a
}

// run sorts against a fresh adversary and checks that the sort really finished:
// exactly one ordering left, and the output lists the ids in that ordering's rank order.
func run(n int, sort func(n int, less func(i, j int) bool) []int) (int, bool) {
	adv := newAdversary(n)
	out := sort(n, adv.less)
	ok := len(adv.cand) == 1
	for pos, id := range out {
		ok = ok && int(adv.cand[0][id]) == pos
	}
	return adv.asked, ok
}

func main() {
	fmt.Println("1. the bound: any comparison sort needs >= ceil(log2 n!) comparisons (worst case)")
	fmt.Println()
	fmt.Printf("  %9s %14s %10s %14s\n", "n", "log2(n!)", "ceil", "n*log2(n)")
	for _, n := range []int{1, 2, 3, 4, 5, 8, 10, 12, 16, 100, 1000, 1000000} {
		sum := 0.0
		for i := 2; i <= n; i++ {
			sum += math.Log2(float64(i))
		}
		c := int(math.Ceil(sum))
		if n <= 20 {
			c = ceilLog2Fact(n) // exact
		}
		fmt.Printf("  %9d %14.2f %10d %14.2f\n", n, sum, c, float64(n)*math.Log2(float64(n)))
	}

	fmt.Println()
	fmt.Println("2. an adversary forces it out of real algorithms (comparisons asked):")
	fmt.Println()
	fmt.Printf("  %2s %7s %6s %10s %11s %6s\n", "n", "n!", "bound", "insertion", "binary-ins", "merge")
	allOK, insWorst, mergeGap := true, true, 0
	for n := 4; n <= 8; n++ {
		bound := ceilLog2Fact(n)
		ins, ok1 := run(n, insertion)
		bin, ok2 := run(n, binaryInsertion)
		mer, ok3 := run(n, func(n int, less func(i, j int) bool) []int { return mergeSort(ids(n), less) })
		allOK = allOK && ok1 && ok2 && ok3 && ins >= bound && bin >= bound && mer >= bound
		fmt.Printf("  %2d %7d %6d %10d %11d %6d\n", n, len(newAdversary(n).cand), bound, ins, bin, mer)
		insWorst = insWorst && ins == n*(n-1)/2
		mergeGap = max(mergeGap, mer-bound)
	}
	fmt.Println("  every count >= bound, and each run ends with exactly 1 ordering left:", allOK)
	if insWorst {
		fmt.Println("  insertion sort is driven to its worst case, n(n-1)/2, every time")
	}
	fmt.Printf("  merge sort was pushed at most %d comparison(s) past the bound\n", mergeGap)

	fmt.Println()
	fmt.Println("3. the loophole: counting sort reads a key's VALUE, not a comparison")
	keys := []int{3, 0, 6, 1, 7, 4, 2, 5} // 8 ids whose keys are 0..7, in some order
	out := make([]int, len(keys))
	reads := 0
	for id, k := range keys {
		out[k] = id // the key IS the index: an 8-way branch in one step
		reads++
	}
	inOrder := make([]int, len(out))
	for pos, id := range out {
		inOrder[pos] = keys[id]
	}
	fmt.Println("  keys (by id):     ", keys)
	fmt.Println("  sorted ids:       ", out)
	fmt.Println("  their keys:       ", inOrder)
	fmt.Println("  key reads:        ", reads)
	fmt.Println("  key comparisons:   0")

	fmt.Println()
	fmt.Println("the bound covers every algorithm whose only access to keys is `<`; it is a")
	fmt.Println("floor, not always reachable: for n = 12 the true minimum is 30, not 29")
	fmt.Println("(Knuth vol. 3, 5.3.1 -- quoted, not computed here).")
}
```

**Output:**

```
1. the bound: any comparison sort needs >= ceil(log2 n!) comparisons (worst case)

          n       log2(n!)       ceil      n*log2(n)
          1           0.00          0           0.00
          2           1.00          1           2.00
          3           2.58          3           4.75
          4           4.58          5           8.00
          5           6.91          7          11.61
          8          15.30         16          24.00
         10          21.79         22          33.22
         12          28.84         29          43.02
         16          44.25         45          64.00
        100         524.76        525         664.39
       1000        8529.40       8530        9965.78
    1000000    18488884.82   18488885    19931568.57

2. an adversary forces it out of real algorithms (comparisons asked):

   n      n!  bound  insertion  binary-ins  merge
   4      24      5          6           5      5
   5     120      7         10           8      8
   6     720     10         15          11     11
   7    5040     13         21          14     14
   8   40320     16         28          17     17
  every count >= bound, and each run ends with exactly 1 ordering left: true
  insertion sort is driven to its worst case, n(n-1)/2, every time
  merge sort was pushed at most 1 comparison(s) past the bound

3. the loophole: counting sort reads a key's VALUE, not a comparison
  keys (by id):      [3 0 6 1 7 4 2 5]
  sorted ids:        [1 3 6 0 5 7 2 4]
  their keys:        [0 1 2 3 4 5 6 7]
  key reads:         8
  key comparisons:   0

the bound covers every algorithm whose only access to keys is `<`; it is a
floor, not always reachable: for n = 12 the true minimum is 30, not 29
(Knuth vol. 3, 5.3.1 -- quoted, not computed here).
```

**Rust:**

```rust
// A comparison sort learns about its input ONLY through answers to "is a < b?".
// Each answer is one bit, and the input could be any of n! orderings, so the
// algorithm needs at least ceil(log2 n!) answers in the worst case: a binary
// decision tree with n! leaves has height >= ceil(log2 n!).
//
// The adversary below makes that concrete. It never fixes the input in advance.
// It keeps every ordering still consistent with its earlier answers and answers
// each question so that the LARGER half survives. Every answer at best halves
// the candidates, so any algorithm must ask >= ceil(log2 n!) questions before
// one ordering is left.

/// Go's %v of a slice: [1 2 3]
fn list(xs: &[usize]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Exact for n <= 20 (n! fits in a u64): ceil(log2 m) is the bit length of m-1.
/// No float rounding can sneak in.
fn ceil_log2_fact(n: u64) -> u32 {
    let f: u64 = (2..=n).product();
    u64::BITS - (f - 1).leading_zeros()
}

/// Every rank assignment still possible: cand[c][id] = rank of id.
struct Adversary {
    cand: Vec<Vec<u8>>,
    asked: usize,
}

impl Adversary {
    fn new(n: usize) -> Self {
        fn permute(k: usize, used: u32, perm: &mut Vec<u8>, out: &mut Vec<Vec<u8>>) {
            let n = perm.len();
            if k == n {
                out.push(perm.clone());
                return;
            }
            for r in 0..n {
                if used & (1 << r) == 0 {
                    perm[k] = r as u8;
                    permute(k + 1, used | 1 << r, perm, out);
                }
            }
        }
        let mut cand = Vec::new();
        permute(0, 0, &mut vec![0; n], &mut cand);
        Adversary { cand, asked: 0 }
    }

    /// "Is element i smaller than element j?" -- whichever answer keeps more
    /// candidates alive (a tie answers true), then forget the others.
    fn less(&mut self, i: usize, j: usize) -> bool {
        self.asked += 1;
        let yes = self.cand.iter().filter(|r| r[i] < r[j]).count();
        let ans = 2 * yes >= self.cand.len();
        self.cand.retain(|r| (r[i] < r[j]) == ans); // in-place filter, like Go's kept := a.cand[:0]
        ans
    }
}

// Three real comparison sorts. Each sees ids 0..n-1 and may only call `less`.
// They take `&mut impl FnMut`: the closure mutably borrows the adversary for the
// whole sort, so nothing else can touch the candidate set while it runs.
fn insertion(n: usize, less: &mut impl FnMut(usize, usize) -> bool) -> Vec<usize> {
    let mut a: Vec<usize> = (0..n).collect();
    for i in 1..n {
        let mut j = i;
        while j > 0 && less(a[j], a[j - 1]) {
            a.swap(j, j - 1);
            j -= 1;
        }
    }
    a
}

fn binary_insertion(n: usize, less: &mut impl FnMut(usize, usize) -> bool) -> Vec<usize> {
    let mut a: Vec<usize> = (0..n).collect();
    for i in 1..n {
        let (x, mut lo, mut hi) = (a[i], 0, i);
        while lo < hi {
            // first position whose element is greater than x
            let mid = (lo + hi) / 2;
            if less(x, a[mid]) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }
        a[lo..=i].rotate_right(1); // shift a[lo..i] right by one, x lands at lo
    }
    a
}

fn merge_sort(a: &[usize], less: &mut impl FnMut(usize, usize) -> bool) -> Vec<usize> {
    if a.len() <= 1 {
        return a.to_vec();
    }
    let (l, r) = a.split_at(a.len() / 2);
    let (l, r) = (merge_sort(l, less), merge_sort(r, less));
    let (mut i, mut j) = (0, 0);
    let mut out = Vec::with_capacity(a.len());
    while i < l.len() && j < r.len() {
        if less(r[j], l[i]) {
            // take right only if strictly smaller: stable
            out.push(r[j]);
            j += 1;
        } else {
            out.push(l[i]);
            i += 1;
        }
    }
    out.extend_from_slice(&l[i..]);
    out.extend_from_slice(&r[j..]);
    out
}

/// Sort against a fresh adversary and check the sort really finished: exactly one
/// ordering left, and the output lists the ids in that ordering's rank order.
fn run(n: usize, sort: impl FnOnce(&mut Adversary) -> Vec<usize>) -> (usize, bool) {
    let mut adv = Adversary::new(n);
    let out = sort(&mut adv);
    let ok = adv.cand.len() == 1
        && out
            .iter()
            .enumerate()
            .all(|(pos, &id)| adv.cand[0][id] as usize == pos);
    (adv.asked, ok)
}

fn main() {
    println!("1. the bound: any comparison sort needs >= ceil(log2 n!) comparisons (worst case)");
    println!();
    println!(
        "  {:>9} {:>14} {:>10} {:>14}",
        "n", "log2(n!)", "ceil", "n*log2(n)"
    );
    for n in [1u64, 2, 3, 4, 5, 8, 10, 12, 16, 100, 1000, 1_000_000] {
        let mut sum = 0.0f64;
        for i in 2..=n {
            sum += (i as f64).log2();
        }
        let c = if n <= 20 {
            ceil_log2_fact(n) as u64 // exact
        } else {
            sum.ceil() as u64
        };
        let nf = n as f64;
        println!("  {n:>9} {sum:>14.2} {c:>10} {:>14.2}", nf * nf.log2());
    }

    println!();
    println!("2. an adversary forces it out of real algorithms (comparisons asked):");
    println!();
    println!(
        "  {:>2} {:>7} {:>6} {:>10} {:>11} {:>6}",
        "n", "n!", "bound", "insertion", "binary-ins", "merge"
    );
    let (mut all_ok, mut ins_worst, mut merge_gap) = (true, true, 0);
    for n in 4..=8 {
        let bound = ceil_log2_fact(n as u64) as usize;
        // each `|i, j| adv.less(i, j)` closure holds the &mut Adversary for the sort's duration
        let (ins, ok1) = run(n, |adv| insertion(n, &mut |i, j| adv.less(i, j)));
        let (bin, ok2) = run(n, |adv| binary_insertion(n, &mut |i, j| adv.less(i, j)));
        let ids: Vec<usize> = (0..n).collect();
        let (mer, ok3) = run(n, |adv| merge_sort(&ids, &mut |i, j| adv.less(i, j)));
        all_ok = all_ok && ok1 && ok2 && ok3 && ins >= bound && bin >= bound && mer >= bound;
        let fact = Adversary::new(n).cand.len();
        println!("  {n:>2} {fact:>7} {bound:>6} {ins:>10} {bin:>11} {mer:>6}");
        ins_worst = ins_worst && ins == n * (n - 1) / 2;
        merge_gap = merge_gap.max(mer - bound);
    }
    println!("  every count >= bound, and each run ends with exactly 1 ordering left: {all_ok}");
    if ins_worst {
        println!("  insertion sort is driven to its worst case, n(n-1)/2, every time");
    }
    println!("  merge sort was pushed at most {merge_gap} comparison(s) past the bound");

    println!();
    println!("3. the loophole: counting sort reads a key's VALUE, not a comparison");
    let keys = [3usize, 0, 6, 1, 7, 4, 2, 5]; // 8 ids whose keys are 0..7, in some order
    let mut out = [0usize; 8];
    let mut reads = 0;
    for (id, &k) in keys.iter().enumerate() {
        out[k] = id; // the key IS the index: an 8-way branch in one step
        reads += 1;
    }
    let in_order: Vec<usize> = out.iter().map(|&id| keys[id]).collect();
    println!("  keys (by id):      {}", list(&keys));
    println!("  sorted ids:        {}", list(&out));
    println!("  their keys:        {}", list(&in_order));
    println!("  key reads:         {reads}");
    println!("  key comparisons:   0");

    println!();
    println!("the bound covers every algorithm whose only access to keys is `<`; it is a");
    println!("floor, not always reachable: for n = 12 the true minimum is 30, not 29");
    println!("(Knuth vol. 3, 5.3.1 -- quoted, not computed here).");
}
```

**Rust output:**

```
1. the bound: any comparison sort needs >= ceil(log2 n!) comparisons (worst case)

          n       log2(n!)       ceil      n*log2(n)
          1           0.00          0           0.00
          2           1.00          1           2.00
          3           2.58          3           4.75
          4           4.58          5           8.00
          5           6.91          7          11.61
          8          15.30         16          24.00
         10          21.79         22          33.22
         12          28.84         29          43.02
         16          44.25         45          64.00
        100         524.76        525         664.39
       1000        8529.40       8530        9965.78
    1000000    18488884.82   18488885    19931568.57

2. an adversary forces it out of real algorithms (comparisons asked):

   n      n!  bound  insertion  binary-ins  merge
   4      24      5          6           5      5
   5     120      7         10           8      8
   6     720     10         15          11     11
   7    5040     13         21          14     14
   8   40320     16         28          17     17
  every count >= bound, and each run ends with exactly 1 ordering left: true
  insertion sort is driven to its worst case, n(n-1)/2, every time
  merge sort was pushed at most 1 comparison(s) past the bound

3. the loophole: counting sort reads a key's VALUE, not a comparison
  keys (by id):      [3 0 6 1 7 4 2 5]
  sorted ids:        [1 3 6 0 5 7 2 4]
  their keys:        [0 1 2 3 4 5 6 7]
  key reads:         8
  key comparisons:   0

the bound covers every algorithm whose only access to keys is `<`; it is a
floor, not always reachable: for n = 12 the true minimum is 30, not 29
(Knuth vol. 3, 5.3.1 -- quoted, not computed here).
```

> **Go vs Rust:** Output identical to Go, character for character — the comparison counts are properties of the algorithms and the adversary, not of the language (and the summed `log2` floats agree to the two printed decimals). The sorts take `less: &mut impl FnMut(usize, usize) -> bool`; the closure `|i, j| adv.less(i, j)` mutably borrows the adversary for the whole sort, so the borrow checker guarantees nothing else touches the candidate set mid-sort, which Go's method value `adv.less` leaves to discipline. `Vec::retain` is the in-place filter Go writes by hand as `kept := a.cand[:0]`.

**Complexity:** any comparison sort: Ω(n log n) worst case — at least ⌈log₂ n!⌉ comparisons (22 at n = 10, 18,488,885 at n = 10⁶) · against the adversary insertion sort was forced to n(n−1)/2 and merge sort to at most bound + 1 · counting sort: 0 comparisons · the adversary itself costs O(n!) per answer (a demonstration, not an algorithm).

---

## 2. Counting sort: count, then rewrite

`🟢 easy` · *Counting sort / bare keys*

When the keys are small integers and nothing is attached to them, you never need to move an element at
all: count how many of each value there are, then rewrite the array from the counts in index order. That
is Θ(n + k) for keys in [0, k), with **zero** key comparisons — each key is used as an array index, a
k-way branch in one step. It is the simplest linear sort there is, and it works only because equal bare
keys are indistinguishable: a counter can stand in for "the 4s".

**Steps:**

1. Count the keys of a 12-element array with k = 5 and print the `counts` next to the output.
2. Sort the bytes of a sentence with k = 256 — the same code, a bigger counter array.
3. Count every increment, counter slot and write on 1,000,000 bytes and check the result against `slices.Sort`.

```go
package main

import (
	"fmt"
	"slices"
)

// Counting sort for BARE small keys: every key is a byte in [0, k), k <= 256.
// Nothing ever compares two keys. Pass 1 uses each key as an index and bumps a
// counter; pass 2 walks the k counters in order and rewrites the array. Because
// equal bare keys are indistinguishable, writing "a 4" from the count is as good
// as moving the original 4 -- that is the trick, and also its limit (example 3).

type stats struct {
	increments, slotsScanned, writes int
}

// countingSort sorts a in place and returns the counts it used.
func countingSort(a []byte, k int, st *stats) [256]int {
	var counts [256]int // a fixed-size array: lives on the stack, no allocation
	for _, x := range a {
		counts[x]++ // the key IS the index -- a k-way branch, not a comparison
		st.increments++
	}
	i := 0
	for v := 0; v < k; v++ { // keys come out in index order, i.e. sorted
		st.slotsScanned++
		for c := 0; c < counts[v]; c++ {
			a[i] = byte(v)
			i++
			st.writes++
		}
	}
	return counts
}

func main() {
	fmt.Println("1. small keys, k = 5")
	keys := []byte{4, 1, 3, 4, 2, 1, 0, 4, 3, 0, 2, 4}
	fmt.Println("  input: ", keys)
	var st stats
	counts := countingSort(keys, 5, &st)
	fmt.Println("  counts:", counts[:5], " (how many 0s, 1s, 2s, 3s, 4s)")
	fmt.Println("  output:", keys)

	fmt.Println()
	fmt.Println("2. the bytes of a sentence, k = 256")
	s := []byte("counting sort never compares")
	countingSort(s, 256, &stats{})
	fmt.Printf("  %q\n", s)

	fmt.Println()
	const n = 1_000_000
	fmt.Println("3. n = 1,000,000 bytes from a formula, k = 256, everything counted")
	big := make([]byte, n)
	for i := range big {
		big[i] = byte(i*131 + i>>3) // >> binds like *, so this is i*131 + (i>>3)
	}
	want := slices.Clone(big)
	slices.Sort(want)
	st = stats{}
	countingSort(big, 256, &st)
	fmt.Printf("  key comparisons:       %9d\n", 0)
	fmt.Printf("  count increments:      %9d  (= n)\n", st.increments)
	fmt.Printf("  counter slots scanned: %9d  (= k)\n", st.slotsScanned)
	fmt.Printf("  writes:                %9d  (= n)\n", st.writes)
	fmt.Printf("  total work:            %9d  (= 2n + k)\n", st.increments+st.slotsScanned+st.writes)
	fmt.Println("  matches slices.Sort:  ", slices.Equal(big, want))

	fmt.Println()
	fmt.Println("Theta(n + k) and not one comparison. It only works because the keys are")
	fmt.Println("the whole element: a counter can stand in for 'the 4s'. Attach a name to")
	fmt.Println("each 4 and rewriting from counts would lose it -- example 3.")
}
```

**Output:**

```
1. small keys, k = 5
  input:  [4 1 3 4 2 1 0 4 3 0 2 4]
  counts: [2 2 2 2 4]  (how many 0s, 1s, 2s, 3s, 4s)
  output: [0 0 1 1 2 2 3 3 4 4 4 4]

2. the bytes of a sentence, k = 256
  "   acceeegimnnnoooprrrssttuv"

3. n = 1,000,000 bytes from a formula, k = 256, everything counted
  key comparisons:               0
  count increments:        1000000  (= n)
  counter slots scanned:       256  (= k)
  writes:                  1000000  (= n)
  total work:              2000256  (= 2n + k)
  matches slices.Sort:   true

Theta(n + k) and not one comparison. It only works because the keys are
the whole element: a counter can stand in for 'the 4s'. Attach a name to
each 4 and rewriting from counts would lose it -- example 3.
```

**Rust:**

```rust
// Counting sort for BARE small keys: every key is a byte in [0, k), k <= 256.
// Nothing ever compares two keys. Pass 1 uses each key as an index and bumps a
// counter; pass 2 walks the k counters in order and rewrites the array. Because
// equal bare keys are indistinguishable, writing "a 4" from the count is as good
// as moving the original 4 -- that is the trick, and also its limit (example 3).

#[derive(Default)]
struct Stats {
    increments: usize,
    slots_scanned: usize,
    writes: usize,
}

/// Go's %v of a slice: [1 2 3]
fn list<T: ToString>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Sorts `a` in place and returns the counts it used.
fn counting_sort(a: &mut [u8], k: usize, st: &mut Stats) -> [usize; 256] {
    let mut counts = [0usize; 256]; // a fixed-size array on the stack, no allocation
    for &x in a.iter() {
        // the key IS the index -- a k-way branch, not a comparison. Indexes are
        // usize in Rust, so the "key as index" step is an explicit `as usize`.
        counts[x as usize] += 1;
        st.increments += 1;
    }
    let mut i = 0;
    for (v, &c) in counts.iter().enumerate().take(k) {
        // keys come out in index order, i.e. sorted
        st.slots_scanned += 1;
        a[i..i + c].fill(v as u8); // write c copies of v
        i += c;
        st.writes += c;
    }
    counts
}

fn main() {
    println!("1. small keys, k = 5");
    let mut keys: [u8; 12] = [4, 1, 3, 4, 2, 1, 0, 4, 3, 0, 2, 4];
    println!("  input:  {}", list(&keys));
    let mut st = Stats::default();
    let counts = counting_sort(&mut keys, 5, &mut st);
    println!(
        "  counts: {}  (how many 0s, 1s, 2s, 3s, 4s)",
        list(&counts[..5])
    );
    println!("  output: {}", list(&keys));

    println!();
    println!("2. the bytes of a sentence, k = 256");
    let mut s = b"counting sort never compares".to_vec();
    counting_sort(&mut s, 256, &mut Stats::default());
    // ASCII in, ASCII out, so from_utf8 cannot fail here
    println!("  {:?}", String::from_utf8(s).unwrap());

    println!();
    const N: usize = 1_000_000;
    println!("3. n = 1,000,000 bytes from a formula, k = 256, everything counted");
    // Rust's >> binds LOOSER than + (Go's binds like *), so the parentheses are
    // required to get Go's i*131 + (i>>3). `as u8` truncates, like Go's byte().
    let mut big: Vec<u8> = (0..N).map(|i| (i * 131 + (i >> 3)) as u8).collect();
    let mut want = big.clone();
    want.sort_unstable();
    let mut st = Stats::default();
    counting_sort(&mut big, 256, &mut st);
    println!("  key comparisons:       {:>9}", 0);
    println!("  count increments:      {:>9}  (= n)", st.increments);
    println!("  counter slots scanned: {:>9}  (= k)", st.slots_scanned);
    println!("  writes:                {:>9}  (= n)", st.writes);
    println!(
        "  total work:            {:>9}  (= 2n + k)",
        st.increments + st.slots_scanned + st.writes
    );
    println!("  matches sort_unstable: {}", big == want);

    println!();
    println!("Theta(n + k) and not one comparison. It only works because the keys are");
    println!("the whole element: a counter can stand in for 'the 4s'. Attach a name to");
    println!("each 4 and rewriting from counts would lose it -- example 3.");
}
```

**Rust output:**

```
1. small keys, k = 5
  input:  [4 1 3 4 2 1 0 4 3 0 2 4]
  counts: [2 2 2 2 4]  (how many 0s, 1s, 2s, 3s, 4s)
  output: [0 0 1 1 2 2 3 3 4 4 4 4]

2. the bytes of a sentence, k = 256
  "   acceeegimnnnoooprrrssttuv"

3. n = 1,000,000 bytes from a formula, k = 256, everything counted
  key comparisons:               0
  count increments:        1000000  (= n)
  counter slots scanned:       256  (= k)
  writes:                  1000000  (= n)
  total work:              2000256  (= 2n + k)
  matches sort_unstable: true

Theta(n + k) and not one comparison. It only works because the keys are
the whole element: a counter can stand in for 'the 4s'. Attach a name to
each 4 and rewriting from counts would lose it -- example 3.
```

> **Go vs Rust:** Output identical to Go except the oracle's name (`sort_unstable` for `slices.Sort`); every count — n increments, k slots, n writes — is the algorithm's, not the language's. The `[usize; 256]` counter array sits on the stack exactly like Go's `[256]int`, and the `counts[x as usize]` cast is the "key as index" step made explicit. One trap in porting the formula: Rust's `>>` binds looser than `+` while Go's binds like `*`, so Go's `i*131 + i>>3` must be written `i * 131 + (i >> 3)`.

**Complexity:** time Θ(n + k) — measured 2n + k = 2,000,256 operations for n = 10⁶, k = 256 · space Θ(k) extra (a 256-counter array) · key comparisons: 0 · stability: not applicable, the keys are bare.

---

## 3. Key-indexed counting: prefix sums and stability

`🟢 easy` · *Counting sort / records*

Records cannot be rewritten from counts — the name attached to each key has to move with it. A prefix
sum over the counts turns them into each key's **start position** in the output, and placing the records
while walking the input forwards puts equal keys in input order: stable by construction, Θ(n + k). The
classic bug walks forwards but places at each key's **end** position, decrementing: the output is still
sorted by key, so a sortedness test passes, and every group of ties comes out reversed. Walking
**backwards** with end positions (the CLRS version) is the other correct way round.

**Steps:**

1. Count 12 students by section A–D and prefix-sum the counts into start positions.
2. Place them three ways: forwards with starts, forwards with ends (the bug), backwards with ends (CLRS).
3. Check each result for sortedness *and* stability, and compare with `slices.SortStableFunc`.

```go
package main

import (
	"cmp"
	"fmt"
	"slices"
	"strings"
)

// Records cannot be rewritten from counts: the name attached to each key must
// MOVE. Key-indexed counting does it in three passes:
//   1. count each key
//   2. prefix-sum the counts: starts[k] = how many records have a smaller key,
//      i.e. the first output slot that belongs to key k
//   3. walk the input FORWARDS and drop each record at starts[key], then bump it
// Records with equal keys land in input order: stable, by construction.

type Student struct {
	Name    string
	Section byte // 'A'..'D'
}

const R = 4 // sections A..D -> key 0..3

func key(s Student) int { return int(s.Section - 'A') }

func countsOf(in []Student) [R]int {
	var c [R]int
	for _, s := range in {
		c[key(s)]++
	}
	return c
}

func startsOf(c [R]int) [R]int {
	var st [R]int
	for k := 1; k < R; k++ {
		st[k] = st[k-1] + c[k-1]
	}
	return st
}

// correct: forwards, start positions, increment.
func correct(in []Student) []Student {
	pos := startsOf(countsOf(in))
	out := make([]Student, len(in))
	for _, s := range in {
		out[pos[key(s)]] = s
		pos[key(s)]++
	}
	return out
}

// buggy: forwards, but with END positions, decrementing -- the classic mix-up.
func buggy(in []Student) []Student {
	c, end := countsOf(in), startsOf(countsOf(in))
	for k := range end {
		end[k] += c[k] // one past the last slot of key k
	}
	out := make([]Student, len(in))
	for _, s := range in {
		end[key(s)]--
		out[end[key(s)]] = s
	}
	return out
}

// clrs: BACKWARDS with end positions, decrementing (CLRS 8.2). Also stable.
func clrs(in []Student) []Student {
	c, end := countsOf(in), startsOf(countsOf(in))
	for k := range end {
		end[k] += c[k]
	}
	out := make([]Student, len(in))
	for i := len(in) - 1; i >= 0; i-- {
		s := in[i]
		end[key(s)]--
		out[end[key(s)]] = s
	}
	return out
}

// check reports sorted-by-key, stable (equal keys keep input order), and which
// sections came out with their input order reversed.
func check(in, out []Student) (sorted, stable bool, reversed string) {
	idx := func(s Student) int { return slices.Index(in, s) }
	sorted, stable = true, true
	for i := 1; i < len(out); i++ {
		sorted = sorted && key(out[i-1]) <= key(out[i])
		if key(out[i-1]) == key(out[i]) && idx(out[i-1]) > idx(out[i]) {
			stable = false
			if !strings.ContainsRune(reversed, rune(out[i].Section)) {
				reversed += string(out[i].Section)
			}
		}
	}
	return
}

func main() {
	in := []Student{ // alphabetical, so a stable sort keeps each section alphabetical
		{"Anderson", 'B'}, {"Brown", 'C'}, {"Davis", 'C'}, {"Garcia", 'D'},
		{"Harris", 'A'}, {"Jackson", 'C'}, {"Johnson", 'D'}, {"Jones", 'C'},
		{"Martin", 'A'}, {"Martinez", 'B'}, {"Miller", 'B'}, {"Moore", 'A'},
	}
	c := countsOf(in)
	fmt.Println("counts  A B C D:", c)
	fmt.Println("starts  A B C D:", startsOf(c), " (prefix sums: first slot of each section)")

	good, bad, back := correct(in), buggy(in), clrs(in)
	stdlib := slices.Clone(in)
	slices.SortStableFunc(stdlib, func(a, b Student) int { return cmp.Compare(a.Section, b.Section) })

	fmt.Println()
	fmt.Printf("%-4s %-12s %-14s %s\n", "pos", "input", "forwards", "BUGGY")
	fmt.Printf("%-4s %-12s %-14s %s\n", "", "", "+ starts", "fwd + ends")
	for i := range in {
		f := func(s Student) string { return fmt.Sprintf("%c %s", s.Section, s.Name) }
		fmt.Printf("%-4d %-12s %-14s %s\n", i, f(in[i]), f(good[i]), f(bad[i]))
	}

	fmt.Println()
	for _, v := range []struct {
		name string
		out  []Student
	}{{"forwards + starts", good}, {"BUGGY fwd + ends", bad}, {"CLRS back + ends", back}} {
		sorted, stable, rev := check(in, v.out)
		fmt.Printf("%-18s sorted by key: %-5v  stable: %v", v.name, sorted, stable)
		if rev != "" {
			fmt.Printf("  <- ties out of input order in sections %s", strings.Join(strings.Split(rev, ""), " "))
		}
		fmt.Println()
	}
	fmt.Println()
	exact := true // is each section of the buggy output its input group, reversed?
	for sec := byte('A'); sec < 'A'+R; sec++ {
		of := func(xs []Student) []Student {
			return slices.DeleteFunc(slices.Clone(xs), func(s Student) bool { return s.Section != sec })
		}
		want := of(in)
		slices.Reverse(want)
		exact = exact && slices.Equal(of(bad), want)
	}
	fmt.Println("BUGGY: every section is exactly its input order reversed:", exact)
	fmt.Println()
	fmt.Println("CLRS (backwards) matches forwards + starts:", slices.Equal(back, good))
	fmt.Println("slices.SortStableFunc matches the correct one:", slices.Equal(stdlib, good))
	fmt.Println()
	fmt.Println("the buggy version passes every 'is it sorted?' test. Only a test with")
	fmt.Println("records -- checking the ORDER of equal keys -- can catch it.")
}
```

**Output:**

```
counts  A B C D: [3 3 4 2]
starts  A B C D: [0 3 6 10]  (prefix sums: first slot of each section)

pos  input        forwards       BUGGY
                  + starts       fwd + ends
0    B Anderson   A Harris       A Moore
1    C Brown      A Martin       A Martin
2    C Davis      A Moore        A Harris
3    D Garcia     B Anderson     B Miller
4    A Harris     B Martinez     B Martinez
5    C Jackson    B Miller       B Anderson
6    D Johnson    C Brown        C Jones
7    C Jones      C Davis        C Jackson
8    A Martin     C Jackson      C Davis
9    B Martinez   C Jones        C Brown
10   B Miller     D Garcia       D Johnson
11   A Moore      D Johnson      D Garcia

forwards + starts  sorted by key: true   stable: true
BUGGY fwd + ends   sorted by key: true   stable: false  <- ties out of input order in sections A B C D
CLRS back + ends   sorted by key: true   stable: true

BUGGY: every section is exactly its input order reversed: true

CLRS (backwards) matches forwards + starts: true
slices.SortStableFunc matches the correct one: true

the buggy version passes every 'is it sorted?' test. Only a test with
records -- checking the ORDER of equal keys -- can catch it.
```

**Rust:**

```rust
// Records cannot be rewritten from counts: the name attached to each key must
// MOVE. Key-indexed counting does it in three passes:
//   1. count each key
//   2. prefix-sum the counts: starts[k] = how many records have a smaller key,
//      i.e. the first output slot that belongs to key k
//   3. walk the input FORWARDS and drop each record at starts[key], then bump it
// Records with equal keys land in input order: stable, by construction.

#[derive(Clone, Copy, PartialEq, Debug)]
struct Student {
    name: &'static str,
    section: u8, // b'A'..=b'D'
}

const R: usize = 4; // sections A..D -> key 0..3

fn key(s: &Student) -> usize {
    (s.section - b'A') as usize
}

/// Go's %v of a slice: [1 2 3]
fn list(xs: &[usize]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn counts_of(input: &[Student]) -> [usize; R] {
    let mut c = [0; R];
    for s in input {
        c[key(s)] += 1;
    }
    c
}

fn starts_of(c: &[usize; R]) -> [usize; R] {
    let mut st = [0; R];
    for k in 1..R {
        st[k] = st[k - 1] + c[k - 1];
    }
    st
}

fn ends_of(input: &[Student]) -> [usize; R] {
    let (c, mut end) = (counts_of(input), starts_of(&counts_of(input)));
    for (e, n) in end.iter_mut().zip(c) {
        *e += n; // one past the last slot of key k
    }
    end
}

// Go's make([]Student, n) is n zero-valued Students. Rust has no zero value for a
// struct holding a &str, so the output buffer is n `None`s, and we unwrap at the end:
// a slot left unfilled would be a bug, and `expect` says so instead of hiding it.
fn finish(out: Vec<Option<Student>>) -> Vec<Student> {
    out.into_iter()
        .collect::<Option<Vec<_>>>()
        .expect("every slot is filled")
}

/// correct: forwards, start positions, increment.
fn correct(input: &[Student]) -> Vec<Student> {
    let mut pos = starts_of(&counts_of(input));
    let mut out = vec![None; input.len()];
    for s in input {
        out[pos[key(s)]] = Some(*s);
        pos[key(s)] += 1;
    }
    finish(out)
}

/// buggy: forwards, but with END positions, decrementing -- the classic mix-up.
fn buggy(input: &[Student]) -> Vec<Student> {
    let mut end = ends_of(input);
    let mut out = vec![None; input.len()];
    for s in input {
        end[key(s)] -= 1;
        out[end[key(s)]] = Some(*s);
    }
    finish(out)
}

/// clrs: BACKWARDS with end positions, decrementing (CLRS 8.2). Also stable.
fn clrs(input: &[Student]) -> Vec<Student> {
    let mut end = ends_of(input);
    let mut out = vec![None; input.len()];
    for s in input.iter().rev() {
        end[key(s)] -= 1;
        out[end[key(s)]] = Some(*s);
    }
    finish(out)
}

/// (sorted by key, stable, sections whose ties came out of input order)
fn check(input: &[Student], out: &[Student]) -> (bool, bool, Vec<String>) {
    let idx = |s: &Student| input.iter().position(|t| t == s);
    let sorted = out.windows(2).all(|w| key(&w[0]) <= key(&w[1]));
    let mut reversed: Vec<String> = Vec::new();
    for w in out.windows(2) {
        if key(&w[0]) == key(&w[1]) && idx(&w[0]) > idx(&w[1]) {
            let sec = (w[1].section as char).to_string();
            if !reversed.contains(&sec) {
                reversed.push(sec);
            }
        }
    }
    (sorted, reversed.is_empty(), reversed)
}

fn main() {
    let st = |name, section| Student { name, section };
    let input = [
        // alphabetical, so a stable sort keeps each section alphabetical
        st("Anderson", b'B'),
        st("Brown", b'C'),
        st("Davis", b'C'),
        st("Garcia", b'D'),
        st("Harris", b'A'),
        st("Jackson", b'C'),
        st("Johnson", b'D'),
        st("Jones", b'C'),
        st("Martin", b'A'),
        st("Martinez", b'B'),
        st("Miller", b'B'),
        st("Moore", b'A'),
    ];
    let c = counts_of(&input);
    println!("counts  A B C D: {}", list(&c));
    println!(
        "starts  A B C D: {}  (prefix sums: first slot of each section)",
        list(&starts_of(&c))
    );

    let (good, bad, back) = (correct(&input), buggy(&input), clrs(&input));
    let mut stdlib = input.to_vec();
    stdlib.sort_by_key(|s| s.section); // stable; at len <= 20 it is insertion sort, no allocation

    println!();
    println!("{:<4} {:<12} {:<14} BUGGY", "pos", "input", "forwards");
    println!("{:<4} {:<12} {:<14} fwd + ends", "", "", "+ starts");
    let f = |s: &Student| format!("{} {}", s.section as char, s.name);
    for i in 0..input.len() {
        println!(
            "{i:<4} {:<12} {:<14} {}",
            f(&input[i]),
            f(&good[i]),
            f(&bad[i])
        );
    }

    println!();
    for (name, out) in [
        ("forwards + starts", &good),
        ("BUGGY fwd + ends", &bad),
        ("CLRS back + ends", &back),
    ] {
        let (sorted, stable, rev) = check(&input, out);
        print!("{name:<18} sorted by key: {sorted:<5}  stable: {stable}");
        if !rev.is_empty() {
            print!("  <- ties out of input order in sections {}", rev.join(" "));
        }
        println!();
    }

    println!();
    // is each section of the buggy output its input group, reversed?
    let exact = (b'A'..b'A' + R as u8).all(|sec| {
        let of = |xs: &[Student]| -> Vec<Student> {
            xs.iter().copied().filter(|s| s.section == sec).collect()
        };
        let mut want = of(&input);
        want.reverse();
        of(&bad) == want
    });
    println!("BUGGY: every section is exactly its input order reversed: {exact}");
    println!();
    println!(
        "CLRS (backwards) matches forwards + starts: {}",
        back == good
    );
    println!(
        "sort_by_key (stable) matches the correct one: {}",
        stdlib == good
    );
    println!();
    println!("the buggy version passes every 'is it sorted?' test. Only a test with");
    println!("records -- checking the ORDER of equal keys -- can catch it.");
}
```

**Rust output:**

```
counts  A B C D: [3 3 4 2]
starts  A B C D: [0 3 6 10]  (prefix sums: first slot of each section)

pos  input        forwards       BUGGY
                  + starts       fwd + ends
0    B Anderson   A Harris       A Moore
1    C Brown      A Martin       A Martin
2    C Davis      A Moore        A Harris
3    D Garcia     B Anderson     B Miller
4    A Harris     B Martinez     B Martinez
5    C Jackson    B Miller       B Anderson
6    D Johnson    C Brown        C Jones
7    C Jones      C Davis        C Jackson
8    A Martin     C Jackson      C Davis
9    B Martinez   C Jones        C Brown
10   B Miller     D Garcia       D Johnson
11   A Moore      D Johnson      D Garcia

forwards + starts  sorted by key: true   stable: true
BUGGY fwd + ends   sorted by key: true   stable: false  <- ties out of input order in sections A B C D
CLRS back + ends   sorted by key: true   stable: true

BUGGY: every section is exactly its input order reversed: true

CLRS (backwards) matches forwards + starts: true
sort_by_key (stable) matches the correct one: true

the buggy version passes every 'is it sorted?' test. Only a test with
records -- checking the ORDER of equal keys -- can catch it.
```

> **Go vs Rust:** Output identical to Go except the oracle's name: the three placements, the reversed groups and the checks are the algorithm's. Go's `make([]Student, n)` hands you n zero-valued records; Rust has no zero value for a struct holding a `&str`, so the buffer is `vec![None; n]` and `collect::<Option<Vec<_>>>()` turns any unfilled slot into a loud failure instead of a silent empty record. The oracle `sort_by_key` is Rust's stable sort; at 12 elements it is plain insertion sort with no allocation, and only above 20 does driftsort allocate scratch, whereas Go's `SortStableFunc` never allocates (both read in the source, not measured here).

**Complexity:** time Θ(n + k) · space Θ(n + k) (the output buffer plus k counters) · stable by construction — forwards with starts or backwards with ends; forwards with ends is sorted by key but reversed every tie group here (sections A, B, C and D).

---

## 4. The k in O(n + k)

`🟢 easy` · *Range vs n*

Counting sort is Θ(n + k), and the k is real work: k counters are allocated and every one of them is
visited, empty or not. Fix n = 10,000 and let the range grow, and n + k overtakes the n·⌈log₂ n⌉
yardstick of a comparison sort — past that point counting sort is the slow choice, and at k = 2³² it is
not even possible. The other half of the lesson: k is the **range** max − min + 1, not the largest key,
so shifting by the minimum makes keys near 10⁹, and negative keys, as cheap as keys near 0.

**Steps:**

1. Sort 10,000 keys for k = 10 … 10⁷, counting increments plus counter slots, and compare with n·⌈log₂ n⌉.
2. Do the arithmetic for k = 2³² instead of allocating 32 GiB.
3. Sort keys near 10⁹ and keys in [−500, 500) with the minimum as offset, and show where the naive index breaks.

```go
package main

import (
	"fmt"
	"math/bits"
	"slices"
)

// Counting sort is Theta(n + k), and the k is not decoration. It allocates k
// counters and scans every one of them, even the empty ones. With n fixed and the
// range k growing, n + k overtakes n*log2(n) and keeps going.
//
// And k is the RANGE max - min + 1, not the largest key: shift every key by the
// minimum and keys near 10^9 -- or negative keys -- cost nothing extra.

// countingSort sorts a in place, keys in [lo, lo+k). Returns the work done:
// one counter increment per key plus one visit per counter slot.
func countingSort(a []int, lo, k int) (work int) {
	counts := make([]int, k)
	for _, x := range a {
		counts[x-lo]++ // x - lo is the offset: the key's index in the counter array
		work++
	}
	i := 0
	for v, c := range counts {
		work++ // every slot is visited, empty or not -- that is the k
		for ; c > 0; c-- {
			a[i] = v + lo
			i++
		}
	}
	return work
}

// commas formats 1234567 as 1,234,567.
func commas(x int) string {
	s := fmt.Sprint(x)
	for i := len(s) - 3; i > 0; i -= 3 {
		s = s[:i] + "," + s[i:]
	}
	return s
}

func main() {
	const n = 10_000
	nlogn := n * bits.Len(n-1) // n * ceil(log2 n), exact in integers
	fmt.Printf("1. n = %s keys, (i*7919) %% k; comparison-sort yardstick n*ceil(log2 n) = %s\n\n",
		commas(n), commas(nlogn))
	fmt.Printf("  %14s %16s %13s %10s  %s\n", "k", "counter bytes", "work n+k", "n*lg n", "winner on paper")
	crossed := 0
	for k := 10; k <= 10_000_000; k *= 10 {
		a := make([]int, n)
		for i := range a {
			a[i] = (i * 7919) % k
		}
		want := slices.Clone(a)
		slices.Sort(want)
		work := countingSort(a, 0, k)
		if !slices.Equal(a, want) {
			panic("counting sort disagrees with slices.Sort")
		}
		winner := "counting sort"
		if work >= nlogn {
			winner = "comparison sort"
			if crossed == 0 {
				crossed = k
			}
		}
		fmt.Printf("  %14s %16s %13s %10s  %s\n", commas(k), commas(8*k), commas(work), commas(nlogn), winner)
	}
	k32 := 1 << 32
	winner := "counting sort"
	if n+k32 >= nlogn {
		winner = "comparison sort"
	}
	fmt.Printf("  %14s %16s %13s %10s  %s\n", "2^32", commas(8*k32), commas(n+k32), commas(nlogn), winner)
	fmt.Printf("  (2^32 row: arithmetic only -- %s bytes of counters, not allocated)\n", commas(8*k32))
	fmt.Println("  every sorted row matched slices.Sort")
	fmt.Printf("  n + k passes n*ceil(log2 n) when k > %s; first such row above: k = %s\n",
		commas(nlogn-n), commas(crossed))

	fmt.Println()
	fmt.Println("2. k is the RANGE: shift by the minimum")
	big := make([]int, n)
	for i := range big {
		big[i] = 1_000_000_000 + (i*7919)%1000
	}
	lo, hi := slices.Min(big), slices.Max(big)
	fmt.Printf("  keys in [%s, %s]\n", commas(lo), commas(hi))
	fmt.Printf("  naive k = max + 1       = %s counters (%s bytes) -- not allocated\n", commas(hi+1), commas(8*(hi+1)))
	want := slices.Clone(big)
	slices.Sort(want)
	work := countingSort(big, lo, hi-lo+1)
	fmt.Printf("  offset k = max - min + 1 = %s counters, work %s, matches slices.Sort: %v\n",
		commas(hi-lo+1), commas(work), slices.Equal(big, want))

	fmt.Println()
	neg := make([]int, n)
	for i := range neg {
		neg[i] = (i*7919)%1000 - 500
	}
	lo, hi = slices.Min(neg), slices.Max(neg)
	fmt.Printf("  keys in [%d, %d]\n", lo, hi)
	// The naive version would do counts[x]++ with x = -500: Go panics with
	// "index out of range [-500]". We check instead of crashing.
	bad := 0
	for _, x := range neg {
		if x < 0 {
			bad++
		}
	}
	fmt.Printf("  naive counts[x]: %s of %s keys would index below 0 (counts[%d] -> panic)\n",
		commas(bad), commas(n), lo)
	want = slices.Clone(neg)
	slices.Sort(want)
	work = countingSort(neg, lo, hi-lo+1)
	fmt.Printf("  offset by min = %d: k = %d, work %s, matches slices.Sort: %v\n",
		lo, hi-lo+1, commas(work), slices.Equal(neg, want))
}
```

**Output:**

```
1. n = 10,000 keys, (i*7919) % k; comparison-sort yardstick n*ceil(log2 n) = 140,000

               k    counter bytes      work n+k     n*lg n  winner on paper
              10               80        10,010    140,000  counting sort
             100              800        10,100    140,000  counting sort
           1,000            8,000        11,000    140,000  counting sort
          10,000           80,000        20,000    140,000  counting sort
         100,000          800,000       110,000    140,000  counting sort
       1,000,000        8,000,000     1,010,000    140,000  comparison sort
      10,000,000       80,000,000    10,010,000    140,000  comparison sort
            2^32   34,359,738,368 4,294,977,296    140,000  comparison sort
  (2^32 row: arithmetic only -- 34,359,738,368 bytes of counters, not allocated)
  every sorted row matched slices.Sort
  n + k passes n*ceil(log2 n) when k > 130,000; first such row above: k = 1,000,000

2. k is the RANGE: shift by the minimum
  keys in [1,000,000,000, 1,000,000,999]
  naive k = max + 1       = 1,000,001,000 counters (8,000,008,000 bytes) -- not allocated
  offset k = max - min + 1 = 1,000 counters, work 11,000, matches slices.Sort: true

  keys in [-500, 499]
  naive counts[x]: 5,000 of 10,000 keys would index below 0 (counts[-500] -> panic)
  offset by min = -500: k = 1000, work 11,000, matches slices.Sort: true
```

**Rust:**

```rust
// Counting sort is Theta(n + k), and the k is not decoration. It allocates k
// counters and scans every one of them, even the empty ones. With n fixed and the
// range k growing, n + k overtakes n*log2(n) and keeps going.
//
// And k is the RANGE max - min + 1, not the largest key: shift every key by the
// minimum and keys near 10^9 -- or negative keys -- cost nothing extra.

/// Sorts `a` in place, keys in [lo, lo+k). Returns the work done: one counter
/// increment per key plus one visit per counter slot.
fn counting_sort(a: &mut [i64], lo: i64, k: usize) -> usize {
    let mut counts = vec![0usize; k];
    let mut work = 0;
    for &x in a.iter() {
        counts[(x - lo) as usize] += 1; // x - lo is the offset: the key's index
        work += 1;
    }
    let mut i = 0;
    for (v, &c) in counts.iter().enumerate() {
        work += 1; // every slot is visited, empty or not -- that is the k
        a[i..i + c].fill(v as i64 + lo);
        i += c;
    }
    work
}

/// Formats 1234567 as 1,234,567 (negative numbers too).
fn commas(x: i64) -> String {
    let mut s = x.unsigned_abs().to_string();
    let mut i = s.len() as isize - 3;
    while i > 0 {
        s.insert(i as usize, ',');
        i -= 3;
    }
    if x < 0 { format!("-{s}") } else { s }
}

fn main() {
    const N: usize = 10_000;
    let lg = (usize::BITS - (N - 1).leading_zeros()) as usize; // ceil(log2 n), exact
    let nlogn = N * lg;
    let c = |x: usize| commas(x as i64);
    println!(
        "1. n = {} keys, (i*7919) % k; comparison-sort yardstick n*ceil(log2 n) = {}\n",
        c(N),
        c(nlogn)
    );
    println!(
        "  {:>14} {:>16} {:>13} {:>10}  winner on paper",
        "k", "counter bytes", "work n+k", "n*lg n"
    );
    let winner = |work: usize| {
        if work >= nlogn {
            "comparison sort"
        } else {
            "counting sort"
        }
    };
    let mut crossed = None;
    for k in (1..=7).map(|e| 10usize.pow(e)) {
        let mut a: Vec<i64> = (0..N).map(|i| ((i * 7919) % k) as i64).collect();
        let mut want = a.clone();
        want.sort_unstable();
        let work = counting_sort(&mut a, 0, k);
        assert_eq!(a, want, "counting sort disagrees with sort_unstable");
        if work >= nlogn && crossed.is_none() {
            crossed = Some(k);
        }
        let row = (c(k), c(8 * k), c(work), c(nlogn));
        println!(
            "  {:>14} {:>16} {:>13} {:>10}  {}",
            row.0,
            row.1,
            row.2,
            row.3,
            winner(work)
        );
    }
    let k32: usize = 1 << 32;
    println!(
        "  {:>14} {:>16} {:>13} {:>10}  {}",
        "2^32",
        c(8 * k32),
        c(N + k32),
        c(nlogn),
        winner(N + k32)
    );
    println!(
        "  (2^32 row: arithmetic only -- {} bytes of counters, not allocated)",
        c(8 * k32)
    );
    println!("  every sorted row matched sort_unstable");
    println!(
        "  n + k passes n*ceil(log2 n) when k > {}; first such row above: k = {}",
        c(nlogn - N),
        c(crossed.expect("some row crossed"))
    );

    println!();
    println!("2. k is the RANGE: shift by the minimum");
    let mut big: Vec<i64> = (0..N)
        .map(|i| 1_000_000_000 + ((i * 7919) % 1000) as i64)
        .collect();
    // min/max return Option: None for an empty slice, where Go's slices.Min panics
    let (lo, hi) = (*big.iter().min().unwrap(), *big.iter().max().unwrap());
    println!("  keys in [{}, {}]", commas(lo), commas(hi));
    println!(
        "  naive k = max + 1       = {} counters ({} bytes) -- not allocated",
        commas(hi + 1),
        commas(8 * (hi + 1))
    );
    let mut want = big.clone();
    want.sort_unstable();
    let work = counting_sort(&mut big, lo, (hi - lo + 1) as usize);
    println!(
        "  offset k = max - min + 1 = {} counters, work {}, matches sort_unstable: {}",
        commas(hi - lo + 1),
        c(work),
        big == want
    );

    println!();
    let mut neg: Vec<i64> = (0..N).map(|i| ((i * 7919) % 1000) as i64 - 500).collect();
    let (lo, hi) = (*neg.iter().min().unwrap(), *neg.iter().max().unwrap());
    println!("  keys in [{lo}, {hi}]");
    // The naive version would index counts[x] directly. An index must be a usize,
    // and `x as usize` on -500 silently WRAPS to 18446744073709551116 (then the
    // bounds check panics). The honest conversion is usize::try_from, which says Err.
    let bad = neg.iter().filter(|&&x| usize::try_from(x).is_err()).count();
    println!(
        "  naive counts[x]: {} of {} keys fail usize::try_from(x) ({lo} as usize = {})",
        c(bad),
        c(N),
        lo as usize
    );
    let mut want = neg.clone();
    want.sort_unstable();
    let work = counting_sort(&mut neg, lo, (hi - lo + 1) as usize);
    println!(
        "  offset by min = {lo}: k = {}, work {}, matches sort_unstable: {}",
        hi - lo + 1,
        c(work),
        neg == want
    );
}
```

**Rust output:**

```
1. n = 10,000 keys, (i*7919) % k; comparison-sort yardstick n*ceil(log2 n) = 140,000

               k    counter bytes      work n+k     n*lg n  winner on paper
              10               80        10,010    140,000  counting sort
             100              800        10,100    140,000  counting sort
           1,000            8,000        11,000    140,000  counting sort
          10,000           80,000        20,000    140,000  counting sort
         100,000          800,000       110,000    140,000  counting sort
       1,000,000        8,000,000     1,010,000    140,000  comparison sort
      10,000,000       80,000,000    10,010,000    140,000  comparison sort
            2^32   34,359,738,368 4,294,977,296    140,000  comparison sort
  (2^32 row: arithmetic only -- 34,359,738,368 bytes of counters, not allocated)
  every sorted row matched sort_unstable
  n + k passes n*ceil(log2 n) when k > 130,000; first such row above: k = 1,000,000

2. k is the RANGE: shift by the minimum
  keys in [1,000,000,000, 1,000,000,999]
  naive k = max + 1       = 1,000,001,000 counters (8,000,008,000 bytes) -- not allocated
  offset k = max - min + 1 = 1,000 counters, work 11,000, matches sort_unstable: true

  keys in [-500, 499]
  naive counts[x]: 5,000 of 10,000 keys fail usize::try_from(x) (-500 as usize = 18446744073709551116)
  offset by min = -500: k = 1000, work 11,000, matches sort_unstable: true
```

> **Go vs Rust:** Every count is identical to Go (the work column, the 130,000 crossover, the offset k); only the oracle's name and the negative-key line differ. That line is the real difference: Go's `counts[-500]` is a runtime panic, while in Rust an index must be a `usize`, and `-500i64 as usize` silently wraps to 18446744073709551116 (the bounds check would then panic), so the twin uses `usize::try_from(x)`, which returns `Err` for all 5,000 negative keys. `iter().min()` returns an `Option` where Go's `slices.Min` panics on an empty slice.

**Complexity:** time Θ(n + k) with k = max − min + 1 · space Θ(k) counters (80 MB at k = 10⁷, 32 GiB at k = 2³²) · for n = 10,000 the counted work n + k passes n·⌈log₂ n⌉ = 140,000 once k > 130,000 · use it when k = O(n), after a min/max scan.

---

## 5. LSD radix sort, one digit at a time

`🟢 easy` · *Radix sort / trace*

A wide key is a few narrow digits. LSD radix sort counting-sorts the whole array by the least
significant digit, then by the next one, and so on — each pass is the stable key-indexed counting of
example 3 with R counters, writing into one reused buffer. After the last pass the array is sorted, with
no comparison anywhere, in Θ(d·(n + R)) for d digits of radix R. This example only shows the mechanism,
traced pass by pass; example 6 shows why it is correct.

**Steps:**

1. Sort eight numbers in base 10 and print the array after the ones, tens and hundreds passes, active digit bracketed.
2. Sort eight `uint32` keys in base 256 — four passes, one byte each — printed in hex.
3. Count digit reads, moves and counter slots for n = 8 and n = 1,000,000, and check against `slices.Sort`.

```go
package main

import (
	"fmt"
	"slices"
	"strings"
)

// LSD radix sort: a wide key is a few narrow digits. Counting-sort the whole
// array by the LEAST significant digit, then by the next one, and so on. Each
// pass is the stable key-indexed counting of example 3 with R counters. After
// the last pass the array is sorted -- example 6 shows why.

type stats struct {
	passes, digitReads, moves, counterSlots int
}

// lsd sorts a with `passes` passes of radix R. digit(x, p) extracts digit p
// (p = 0 is the least significant). after is called with the array after each pass.
func lsd(a []uint32, R, passes int, digit func(x uint32, p int) int,
	after func(p int, a []uint32), st *stats) {
	buf := make([]uint32, len(a)) // ONE extra buffer, reused by every pass
	src, dst := a, buf
	for p := 0; p < passes; p++ {
		counts := make([]int, R+1)
		for _, x := range src {
			counts[digit(x, p)+1]++ // count digit d into slot d+1 ...
			st.digitReads++
		}
		for r := 0; r < R; r++ {
			counts[r+1] += counts[r] // ... so the prefix sum gives START positions
			st.counterSlots++
		}
		for _, x := range src { // forwards + starts: stable
			d := digit(x, p)
			dst[counts[d]] = x
			counts[d]++
			st.digitReads++
			st.moves++
		}
		src, dst = dst, src // ping-pong: swap the two slice headers, copy nothing
		st.passes++
		if after != nil {
			after(p, src)
		}
	}
	if passes%2 == 1 {
		copy(a, src) // an odd number of passes ends in buf: copy back once
	}
}

func main() {
	fmt.Println("1. base 10, three passes; [ ] marks the digit that pass sorted by")
	dec := []uint32{170, 45, 75, 90, 802, 24, 2, 66}
	pow10 := []uint32{1, 10, 100}
	digit10 := func(x uint32, p int) int { return int(x / pow10[p] % 10) }
	mark := func(s string, i, w int) string { // bracket chars [i, i+w) of s
		return s[:i] + "[" + s[i:i+w] + "]" + s[i+w:]
	}
	show10 := func(a []uint32, p int) string {
		parts := make([]string, len(a))
		for i, x := range a {
			parts[i] = mark(fmt.Sprintf("%03d", x), 2-p, 1)
		}
		return strings.Join(parts, " ")
	}
	fmt.Printf("  %-9s %s\n", "input", strings.Join(plain(dec, "%03d"), "   "))
	names := []string{"ones", "tens", "hundreds"}
	lsd(dec, 10, 3, digit10, func(p int, a []uint32) {
		fmt.Printf("  %-9s %s\n", names[p], show10(a, p))
	}, &stats{})
	fmt.Printf("  %-9s %s\n", "sorted", strings.Join(plain(dec, "%03d"), "   "))

	fmt.Println()
	fmt.Println("2. base 256 on uint32: four passes, one byte each (low byte first)")
	keys := []uint32{0x3a7f12c4, 0x1b2c3d4e, 0xc0ffee11, 0x3a00ff02,
		0x7e57ab1e, 0x1bad1dea, 0x00c0ffee, 0xdeadbeef}
	byteOf := func(x uint32, p int) int { return int(x >> (8 * p) & 0xFF) }
	show16 := func(a []uint32, p int) string {
		parts := make([]string, len(a))
		for i, x := range a {
			parts[i] = mark(fmt.Sprintf("%08x", x), 6-2*p, 2)
		}
		return strings.Join(parts, " ")
	}
	want := slices.Clone(keys)
	slices.Sort(want)
	fmt.Printf("  %-7s %s\n", "input", strings.Join(plain(keys, "%08x"), "   "))
	var st stats
	lsd(keys, 256, 4, byteOf, func(p int, a []uint32) {
		fmt.Printf("  pass %d  %s\n", p+1, show16(a, p))
	}, &st)
	fmt.Println("  matches slices.Sort:", slices.Equal(keys, want))

	fmt.Println()
	fmt.Println("3. what it cost, counted (d = 4 passes, R = 256)")
	big := make([]uint32, 1_000_000)
	for i := range big {
		big[i] = uint32(i) * 2654435761 // Knuth's multiplicative hash: scrambled, every byte varies
	}
	wantBig := slices.Clone(big)
	slices.Sort(wantBig)
	var stBig stats
	lsd(big, 256, 4, byteOf, nil, &stBig)
	fmt.Printf("  %9s %7s %12s %12s %14s %16s\n", "n", "passes", "digit reads", "moves", "counter slots", "key comparisons")
	for _, r := range []struct {
		n int
		s stats
	}{{len(keys), st}, {len(big), stBig}} {
		fmt.Printf("  %9d %7d %12d %12d %14d %16d\n", r.n, r.s.passes, r.s.digitReads, r.s.moves, r.s.counterSlots, 0)
	}
	fmt.Println("  n = 1,000,000 matches slices.Sort:", slices.Equal(big, wantBig))
	fmt.Println("  moves = d*n, digit reads = 2*d*n, counter slots = d*R: Theta(d*(n + R))")
	fmt.Printf("  at n = 8 the counters are %d of %d operations: R dominates tiny inputs\n",
		st.counterSlots, st.digitReads+st.moves+st.counterSlots)
}

// plain formats every key with format, unmarked.
func plain(a []uint32, format string) []string {
	s := make([]string, len(a))
	for i, x := range a {
		s[i] = fmt.Sprintf(format, x)
	}
	return s
}
```

**Output:**

```
1. base 10, three passes; [ ] marks the digit that pass sorted by
  input     170   045   075   090   802   024   002   066
  ones      17[0] 09[0] 80[2] 00[2] 02[4] 04[5] 07[5] 06[6]
  tens      8[0]2 0[0]2 0[2]4 0[4]5 0[6]6 1[7]0 0[7]5 0[9]0
  hundreds  [0]02 [0]24 [0]45 [0]66 [0]75 [0]90 [1]70 [8]02
  sorted    002   024   045   066   075   090   170   802

2. base 256 on uint32: four passes, one byte each (low byte first)
  input   3a7f12c4   1b2c3d4e   c0ffee11   3a00ff02   7e57ab1e   1bad1dea   00c0ffee   deadbeef
  pass 1  3a00ff[02] c0ffee[11] 7e57ab[1e] 1b2c3d[4e] 3a7f12[c4] 1bad1d[ea] 00c0ff[ee] deadbe[ef]
  pass 2  3a7f[12]c4 1bad[1d]ea 1b2c[3d]4e 7e57[ab]1e dead[be]ef c0ff[ee]11 3a00[ff]02 00c0[ff]ee
  pass 3  3a[00]ff02 1b[2c]3d4e 7e[57]ab1e 3a[7f]12c4 1b[ad]1dea de[ad]beef 00[c0]ffee c0[ff]ee11
  pass 4  [00]c0ffee [1b]2c3d4e [1b]ad1dea [3a]00ff02 [3a]7f12c4 [7e]57ab1e [c0]ffee11 [de]adbeef
  matches slices.Sort: true

3. what it cost, counted (d = 4 passes, R = 256)
          n  passes  digit reads        moves  counter slots  key comparisons
          8       4           64           32           1024                0
    1000000       4      8000000      4000000           1024                0
  n = 1,000,000 matches slices.Sort: true
  moves = d*n, digit reads = 2*d*n, counter slots = d*R: Theta(d*(n + R))
  at n = 8 the counters are 1024 of 1120 operations: R dominates tiny inputs
```

**Rust:**

```rust
// LSD radix sort: a wide key is a few narrow digits. Counting-sort the whole
// array by the LEAST significant digit, then by the next one, and so on. Each
// pass is the stable key-indexed counting of example 3 with R counters. After
// the last pass the array is sorted -- example 6 shows why.

#[derive(Default, Clone, Copy)]
struct Stats {
    passes: usize,
    digit_reads: usize,
    moves: usize,
    counter_slots: usize,
}

/// Sorts `a` with `passes` passes of radix `r`. `digit(x, p)` extracts digit p
/// (p = 0 is the least significant). `after` sees the array after each pass.
/// Takes `&mut Vec`, not `&mut [u32]`, so that the two buffers can trade places.
fn lsd(
    a: &mut Vec<u32>,
    r: usize,
    passes: usize,
    digit: impl Fn(u32, usize) -> usize,
    mut after: impl FnMut(usize, &[u32]),
    st: &mut Stats,
) {
    let mut buf = vec![0u32; a.len()]; // ONE extra buffer, reused by every pass
    for p in 0..passes {
        let mut counts = vec![0usize; r + 1];
        for &x in a.iter() {
            counts[digit(x, p) + 1] += 1; // count digit d into slot d+1 ...
            st.digit_reads += 1;
        }
        for i in 0..r {
            counts[i + 1] += counts[i]; // ... so the prefix sum gives START positions
            st.counter_slots += 1;
        }
        for &x in a.iter() {
            // forwards + starts: stable
            let d = digit(x, p);
            buf[counts[d]] = x;
            counts[d] += 1;
            st.digit_reads += 1;
            st.moves += 1;
        }
        // ping-pong: swap the two Vecs' (pointer, len, capacity) -- 24 bytes, no
        // element copied. Because we own both buffers, `a` holds the result after
        // EVERY pass, so unlike the Go version there is no copy-back for odd passes.
        std::mem::swap(a, &mut buf);
        st.passes += 1;
        after(p, a);
    }
}

/// Every key formatted, unmarked.
fn plain(a: &[u32], hex: bool) -> Vec<String> {
    a.iter()
        .map(|x| {
            if hex {
                format!("{x:08x}")
            } else {
                format!("{x:03}")
            }
        })
        .collect()
}

/// Bracket chars [i, i+w) of s.
fn mark(s: &str, i: usize, w: usize) -> String {
    format!("{}[{}]{}", &s[..i], &s[i..i + w], &s[i + w..])
}

fn main() {
    println!("1. base 10, three passes; [ ] marks the digit that pass sorted by");
    let mut dec: Vec<u32> = vec![170, 45, 75, 90, 802, 24, 2, 66];
    let pow10 = [1, 10, 100];
    let digit10 = |x: u32, p: usize| (x / pow10[p] % 10) as usize;
    println!("  {:<9} {}", "input", plain(&dec, false).join("   "));
    let names = ["ones", "tens", "hundreds"];
    let show10 = |p: usize, a: &[u32]| {
        let parts: Vec<String> = plain(a, false).iter().map(|s| mark(s, 2 - p, 1)).collect();
        println!("  {:<9} {}", names[p], parts.join(" "));
    };
    lsd(&mut dec, 10, 3, digit10, show10, &mut Stats::default());
    println!("  {:<9} {}", "sorted", plain(&dec, false).join("   "));

    println!();
    println!("2. base 256 on uint32: four passes, one byte each (low byte first)");
    let mut keys: Vec<u32> = vec![
        0x3a7f12c4, 0x1b2c3d4e, 0xc0ffee11, 0x3a00ff02, 0x7e57ab1e, 0x1bad1dea, 0x00c0ffee,
        0xdeadbeef,
    ];
    let byte_of = |x: u32, p: usize| ((x >> (8 * p)) & 0xFF) as usize;
    let mut want = keys.clone();
    want.sort_unstable();
    println!("  {:<7} {}", "input", plain(&keys, true).join("   "));
    let mut st = Stats::default();
    let show16 = |p: usize, a: &[u32]| {
        let parts: Vec<String> = plain(a, true)
            .iter()
            .map(|s| mark(s, 6 - 2 * p, 2))
            .collect();
        println!("  pass {}  {}", p + 1, parts.join(" "));
    };
    lsd(&mut keys, 256, 4, byte_of, show16, &mut st);
    println!("  matches sort_unstable: {}", keys == want);

    println!();
    println!("3. what it cost, counted (d = 4 passes, R = 256)");
    // Knuth's multiplicative hash: scrambled, every byte varies. Go's uint32
    // multiply wraps silently; in Rust the wrap is spelled out.
    let mut big: Vec<u32> = (0..1_000_000u32)
        .map(|i| i.wrapping_mul(2654435761))
        .collect();
    let mut want_big = big.clone();
    want_big.sort_unstable();
    let mut st_big = Stats::default();
    lsd(&mut big, 256, 4, byte_of, |_, _| {}, &mut st_big);
    println!(
        "  {:>9} {:>7} {:>12} {:>12} {:>14} {:>16}",
        "n", "passes", "digit reads", "moves", "counter slots", "key comparisons"
    );
    for (n, s) in [(keys.len(), st), (big.len(), st_big)] {
        println!(
            "  {n:>9} {:>7} {:>12} {:>12} {:>14} {:>16}",
            s.passes, s.digit_reads, s.moves, s.counter_slots, 0
        );
    }
    println!("  n = 1,000,000 matches sort_unstable: {}", big == want_big);
    println!("  moves = d*n, digit reads = 2*d*n, counter slots = d*R: Theta(d*(n + R))");
    println!(
        "  at n = 8 the counters are {} of {} operations: R dominates tiny inputs",
        st.counter_slots,
        st.digit_reads + st.moves + st.counter_slots
    );
}
```

**Rust output:**

```
1. base 10, three passes; [ ] marks the digit that pass sorted by
  input     170   045   075   090   802   024   002   066
  ones      17[0] 09[0] 80[2] 00[2] 02[4] 04[5] 07[5] 06[6]
  tens      8[0]2 0[0]2 0[2]4 0[4]5 0[6]6 1[7]0 0[7]5 0[9]0
  hundreds  [0]02 [0]24 [0]45 [0]66 [0]75 [0]90 [1]70 [8]02
  sorted    002   024   045   066   075   090   170   802

2. base 256 on uint32: four passes, one byte each (low byte first)
  input   3a7f12c4   1b2c3d4e   c0ffee11   3a00ff02   7e57ab1e   1bad1dea   00c0ffee   deadbeef
  pass 1  3a00ff[02] c0ffee[11] 7e57ab[1e] 1b2c3d[4e] 3a7f12[c4] 1bad1d[ea] 00c0ff[ee] deadbe[ef]
  pass 2  3a7f[12]c4 1bad[1d]ea 1b2c[3d]4e 7e57[ab]1e dead[be]ef c0ff[ee]11 3a00[ff]02 00c0[ff]ee
  pass 3  3a[00]ff02 1b[2c]3d4e 7e[57]ab1e 3a[7f]12c4 1b[ad]1dea de[ad]beef 00[c0]ffee c0[ff]ee11
  pass 4  [00]c0ffee [1b]2c3d4e [1b]ad1dea [3a]00ff02 [3a]7f12c4 [7e]57ab1e [c0]ffee11 [de]adbeef
  matches sort_unstable: true

3. what it cost, counted (d = 4 passes, R = 256)
          n  passes  digit reads        moves  counter slots  key comparisons
          8       4           64           32           1024                0
    1000000       4      8000000      4000000           1024                0
  n = 1,000,000 matches sort_unstable: true
  moves = d*n, digit reads = 2*d*n, counter slots = d*R: Theta(d*(n + R))
  at n = 8 the counters are 1024 of 1120 operations: R dominates tiny inputs
```

> **Go vs Rust:** Every trace line and count is identical to Go; only the oracle's name differs. The ping-pong differs in shape: Go swaps two slice headers and must `copy` back after an odd number of passes (the base-10 run has three), while the Rust `lsd` takes `&mut Vec<u32>` and `std::mem::swap`s the two owned Vecs, so the caller's Vec holds the result after every pass and no copy-back exists. Go's `uint32(i) * 2654435761` wraps silently; Rust writes `wrapping_mul` (a plain `*` would panic on overflow in a debug build).

**Complexity:** time Θ(d·(n + R)) for d digits of radix R — counted d·n = 4,000,000 moves and d·R = 1,024 counter slots for 10⁶ `uint32` keys · space Θ(n + R) (one buffer, reused by every pass) · key comparisons: 0 · at n = 8 the 1,024 counter slots are most of the work.

---

## 6. Why every LSD pass must be stable

`🟢 easy` · *Stability as correctness*

LSD radix sort is correct because of one invariant: *after pass i, the array is sorted by its low i
digits*. Pass i + 1 orders keys with different digits i + 1 by itself; for keys with the **same** digit
i + 1, only stability keeps the order the earlier passes established — so stability *is* the inductive
step, not a nicety. Example 3 used stability to protect satellite data; here it is the proof of the
algorithm. Break it in one pass and the invariant fails at the first tie; run stable passes in the wrong
order and the last pass simply wins. Same Θ(d·(n + R)) either way — only the correct one sorts.

**Steps:**

1. Run stable LSD on 12 three-digit keys and check "sorted by the low i digits" after every pass.
2. Replace the tens pass with the reversed-ties placement from example 3 and watch the invariant fail there.
3. Run stable passes most-significant first over the whole array, and count the inversions each variant leaves.

```go
package main

import (
	"fmt"
	"strings"
)

// Why is LSD radix sort correct? One invariant:
//
//   after pass i, the array is sorted by its low i digits.
//
// Pass i+1 sorts by digit i+1. Two keys with DIFFERENT digit i+1 are now in the
// right order whatever came before. Two keys with the SAME digit i+1 are tied --
// and only stability keeps them in the order the earlier passes put them in,
// which by induction is the order of their low i digits. Stability is not a
// nicety here; it is the inductive step. Break it once and the proof is gone.

var pow10 = [...]int{1, 10, 100, 1000}

func digit(x, p int) int { return x / pow10[p] % 10 }

// pass counting-sorts a by digit p. stable=false uses the reversed-ties
// placement from example 3: forwards, but at END positions, decrementing.
func pass(a []int, p int, stable bool) []int {
	var counts [10]int
	for _, x := range a {
		counts[digit(x, p)]++
	}
	var pos [10]int // starts (stable) or ends (buggy)
	for d, sum := 0, 0; d < 10; d++ {
		if stable {
			pos[d] = sum
		}
		sum += counts[d]
		if !stable {
			pos[d] = sum
		}
	}
	out := make([]int, len(a))
	for _, x := range a {
		d := digit(x, p)
		if stable {
			out[pos[d]] = x
			pos[d]++
		} else {
			pos[d]--
			out[pos[d]] = x
		}
	}
	return out
}

// sortedByLow reports whether a is sorted by its low i digits (x mod 10^i).
func sortedByLow(a []int, i int) bool {
	for j := 1; j < len(a); j++ {
		if a[j-1]%pow10[i] > a[j]%pow10[i] {
			return false
		}
	}
	return true
}

func inversions(a []int) int {
	n := 0
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i] > a[j] {
				n++
			}
		}
	}
	return n
}

func show(a []int, p int) string {
	parts := make([]string, len(a))
	for i, x := range a {
		s := fmt.Sprintf("%03d", x)
		parts[i] = s[:2-p] + "[" + s[2-p:3-p] + "]" + s[3-p:]
	}
	return strings.Join(parts, " ")
}

type step struct {
	digit  int // 0 = ones, 1 = tens, 2 = hundreds
	stable bool
}

// run applies the passes in order and traces each one.
func run(title string, in []int, steps []step) []int {
	fmt.Println(title)
	a := in
	for k, st := range steps {
		p, stable := st.digit, st.stable
		a = pass(a, p, stable)
		label := fmt.Sprintf("digit %d", p+1)
		if !stable {
			label += " UNSTABLE"
		}
		fmt.Printf("  %-16s %s", label, show(a, p))
		if steps[0].digit == 0 { // LSD order: check the invariant after pass k+1
			fmt.Printf("  low %d sorted: %v", k+1, sortedByLow(a, k+1))
		}
		fmt.Println()
	}
	fmt.Printf("  %-16s sorted: %v, inversions left: %d\n\n", "result", inversions(a) == 0, inversions(a))
	return a
}

func main() {
	in := []int{329, 457, 657, 839, 436, 720, 355, 327, 751, 324, 453, 437}
	fmt.Printf("input (%d keys, %d inversions): %v\n\n", len(in), inversions(in), in)

	run("A. LSD, every pass stable", in, []step{{0, true}, {1, true}, {2, true}})
	run("B. LSD, but the tens pass reverses ties", in, []step{{0, true}, {1, false}, {2, true}})
	c := run("C. every pass stable, but most significant first", in, []step{{2, true}, {1, true}, {0, true}})
	fmt.Println("C is sorted by its last pass alone -- the ones digit:", sortedByLow(c, 1))
	fmt.Println("over the whole array, the LAST pass wins; MSD must recurse into buckets instead.")
}
```

**Output:**

```
input (12 keys, 37 inversions): [329 457 657 839 436 720 355 327 751 324 453 437]

A. LSD, every pass stable
  digit 1          72[0] 75[1] 45[3] 32[4] 35[5] 43[6] 45[7] 65[7] 32[7] 43[7] 32[9] 83[9]  low 1 sorted: true
  digit 2          7[2]0 3[2]4 3[2]7 3[2]9 4[3]6 4[3]7 8[3]9 7[5]1 4[5]3 3[5]5 4[5]7 6[5]7  low 2 sorted: true
  digit 3          [3]24 [3]27 [3]29 [3]55 [4]36 [4]37 [4]53 [4]57 [6]57 [7]20 [7]51 [8]39  low 3 sorted: true
  result           sorted: true, inversions left: 0

B. LSD, but the tens pass reverses ties
  digit 1          72[0] 75[1] 45[3] 32[4] 35[5] 43[6] 45[7] 65[7] 32[7] 43[7] 32[9] 83[9]  low 1 sorted: true
  digit 2 UNSTABLE 3[2]9 3[2]7 3[2]4 7[2]0 8[3]9 4[3]7 4[3]6 6[5]7 4[5]7 3[5]5 4[5]3 7[5]1  low 2 sorted: false
  digit 3          [3]29 [3]27 [3]24 [3]55 [4]37 [4]36 [4]57 [4]53 [6]57 [7]20 [7]51 [8]39  low 3 sorted: false
  result           sorted: false, inversions left: 5

C. every pass stable, but most significant first
  digit 3          [3]29 [3]55 [3]27 [3]24 [4]57 [4]36 [4]53 [4]37 [6]57 [7]20 [7]51 [8]39
  digit 2          3[2]9 3[2]7 3[2]4 7[2]0 4[3]6 4[3]7 8[3]9 3[5]5 4[5]7 4[5]3 6[5]7 7[5]1
  digit 1          72[0] 75[1] 45[3] 32[4] 35[5] 43[6] 32[7] 43[7] 45[7] 65[7] 32[9] 83[9]
  result           sorted: false, inversions left: 31

C is sorted by its last pass alone -- the ones digit: true
over the whole array, the LAST pass wins; MSD must recurse into buckets instead.
```

**Rust:**

```rust
// Why is LSD radix sort correct? One invariant:
//
//   after pass i, the array is sorted by its low i digits.
//
// Pass i+1 sorts by digit i+1. Two keys with DIFFERENT digit i+1 are now in the
// right order whatever came before. Two keys with the SAME digit i+1 are tied --
// and only stability keeps them in the order the earlier passes put them in,
// which by induction is the order of their low i digits. Stability is not a
// nicety here; it is the inductive step. Break it once and the proof is gone.

const POW10: [u32; 4] = [1, 10, 100, 1000];

fn digit(x: u32, p: usize) -> usize {
    (x / POW10[p] % 10) as usize
}

/// Counting-sorts `a` by digit p. `stable == false` uses the reversed-ties
/// placement from example 3: forwards, but at END positions, decrementing.
fn pass(a: &[u32], p: usize, stable: bool) -> Vec<u32> {
    let mut counts = [0usize; 10];
    for &x in a {
        counts[digit(x, p)] += 1;
    }
    let mut pos = [0usize; 10]; // starts (stable) or ends (buggy)
    let mut sum = 0;
    for d in 0..10 {
        if stable {
            pos[d] = sum;
        }
        sum += counts[d];
        if !stable {
            pos[d] = sum;
        }
    }
    let mut out = vec![0u32; a.len()];
    for &x in a {
        let d = digit(x, p);
        if stable {
            out[pos[d]] = x;
            pos[d] += 1;
        } else {
            pos[d] -= 1;
            out[pos[d]] = x;
        }
    }
    out
}

/// Is `a` sorted by its low i digits (x mod 10^i)? The Go loop, as one iterator.
fn sorted_by_low(a: &[u32], i: usize) -> bool {
    a.windows(2).all(|w| w[0] % POW10[i] <= w[1] % POW10[i])
}

fn inversions(a: &[u32]) -> usize {
    (0..a.len())
        .map(|i| a[i + 1..].iter().filter(|&&y| a[i] > y).count())
        .sum()
}

fn show(a: &[u32], p: usize) -> String {
    let parts: Vec<String> = a
        .iter()
        .map(|x| {
            let s = format!("{x:03}");
            format!("{}[{}]{}", &s[..2 - p], &s[2 - p..3 - p], &s[3 - p..])
        })
        .collect();
    parts.join(" ")
}

/// Go's %v of a slice: [1 2 3]
fn list(xs: &[u32]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Applies the passes in order -- each one is (digit, stable), digit 0 = ones --
/// and traces them.
fn run(title: &str, input: &[u32], steps: &[(usize, bool)]) -> Vec<u32> {
    println!("{title}");
    let lsd_order = steps[0].0 == 0;
    let mut a = input.to_vec();
    for (k, &(p, stable)) in steps.iter().enumerate() {
        a = pass(&a, p, stable); // the old Vec is dropped here, freed at once
        let label = format!("digit {}{}", p + 1, if stable { "" } else { " UNSTABLE" });
        print!("  {label:<16} {}", show(&a, p));
        if lsd_order {
            // LSD order: check the invariant after pass k+1
            print!("  low {} sorted: {}", k + 1, sorted_by_low(&a, k + 1));
        }
        println!();
    }
    let inv = inversions(&a);
    println!(
        "  {:<16} sorted: {}, inversions left: {inv}\n",
        "result",
        inv == 0
    );
    a
}

fn main() {
    let input = [329, 457, 657, 839, 436, 720, 355, 327, 751, 324, 453, 437];
    println!(
        "input ({} keys, {} inversions): {}\n",
        input.len(),
        inversions(&input),
        list(&input)
    );

    run(
        "A. LSD, every pass stable",
        &input,
        &[(0, true), (1, true), (2, true)],
    );
    run(
        "B. LSD, but the tens pass reverses ties",
        &input,
        &[(0, true), (1, false), (2, true)],
    );
    let c = run(
        "C. every pass stable, but most significant first",
        &input,
        &[(2, true), (1, true), (0, true)],
    );
    println!(
        "C is sorted by its last pass alone -- the ones digit: {}",
        sorted_by_low(&c, 1)
    );
    println!("over the whole array, the LAST pass wins; MSD must recurse into buckets instead.");
}
```

**Rust output:**

```
input (12 keys, 37 inversions): [329 457 657 839 436 720 355 327 751 324 453 437]

A. LSD, every pass stable
  digit 1          72[0] 75[1] 45[3] 32[4] 35[5] 43[6] 45[7] 65[7] 32[7] 43[7] 32[9] 83[9]  low 1 sorted: true
  digit 2          7[2]0 3[2]4 3[2]7 3[2]9 4[3]6 4[3]7 8[3]9 7[5]1 4[5]3 3[5]5 4[5]7 6[5]7  low 2 sorted: true
  digit 3          [3]24 [3]27 [3]29 [3]55 [4]36 [4]37 [4]53 [4]57 [6]57 [7]20 [7]51 [8]39  low 3 sorted: true
  result           sorted: true, inversions left: 0

B. LSD, but the tens pass reverses ties
  digit 1          72[0] 75[1] 45[3] 32[4] 35[5] 43[6] 45[7] 65[7] 32[7] 43[7] 32[9] 83[9]  low 1 sorted: true
  digit 2 UNSTABLE 3[2]9 3[2]7 3[2]4 7[2]0 8[3]9 4[3]7 4[3]6 6[5]7 4[5]7 3[5]5 4[5]3 7[5]1  low 2 sorted: false
  digit 3          [3]29 [3]27 [3]24 [3]55 [4]37 [4]36 [4]57 [4]53 [6]57 [7]20 [7]51 [8]39  low 3 sorted: false
  result           sorted: false, inversions left: 5

C. every pass stable, but most significant first
  digit 3          [3]29 [3]55 [3]27 [3]24 [4]57 [4]36 [4]53 [4]37 [6]57 [7]20 [7]51 [8]39
  digit 2          3[2]9 3[2]7 3[2]4 7[2]0 4[3]6 4[3]7 8[3]9 3[5]5 4[5]7 4[5]3 6[5]7 7[5]1
  digit 1          72[0] 75[1] 45[3] 32[4] 35[5] 43[6] 32[7] 43[7] 45[7] 65[7] 32[9] 83[9]
  result           sorted: false, inversions left: 31

C is sorted by its last pass alone -- the ones digit: true
over the whole array, the LAST pass wins; MSD must recurse into buckets instead.
```

> **Go vs Rust:** Output identical to Go, character for character: the traces, the invariant failing at the tens pass, and the 5 and 31 inversions are properties of the algorithm. The invariant check is `windows(2).all(..)` where Go writes a loop, and a pass is a `(digit, stable)` tuple where Go declares a small struct; nothing semantic differs. Each `a = pass(&a, ..)` drops the previous Vec on the spot, where Go leaves the old slice to the GC.

**Complexity:** Θ(d·(n + R)) for all three variants — but only stable passes, least significant first, sort: one tie-reversing pass left 5 of the input's 37 inversions, and most-significant-first left 31, sorted by the ones digit alone.

---

> Next tier: [🟡 medium](2-medium.md) — choosing the radix, signed and float keys, strings,
> bucket sort, and the problems linear sorts own.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
