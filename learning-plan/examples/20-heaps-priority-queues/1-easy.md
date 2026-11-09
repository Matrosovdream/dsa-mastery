# Step 20 — Heaps & Priority Queues · 🟢 Easy

Examples **1–6**: the heap as an array and the index laws it obeys, the two sifts counted, `container/heap`
and the order the package calls your methods in, heapify against repeated pushes, min, max and generic heaps
with their ties, and what a float does to `Less`.

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

## 1. The heap is an array

`🟢 easy` · *Implicit tree*

A complete binary tree needs no nodes: stored level by level, the index arithmetic *is* the structure, and its shape is forced by `n` alone, so parent, children, level, height and leaf count are all Θ(1) computations on an index. This example checks those laws exhaustively for every size up to 1,000, and then shows how weak the invariant is: of the `n!` orderings of `n` distinct keys, only `n! / ∏ (subtree sizes)` are heaps, counted here by trying every ordering up to `n = 10`.

**Steps:**

1. Draw one ten-element heap as a tree and as an index table, and test it against the invariant.
2. Assert the index laws for every `n` from 1 to 1000, and look at the root's parent (`(j-1)/2` at `j = 0`) and the 1-based variant.
3. Enumerate all `n!` permutations for `n` up to 10 and compare the number of heaps with the formula.

```go
package main

import (
	"fmt"
	"math/bits"
	"strings"
)

// A heap is a complete binary tree stored level by level in a slice. There are
// no node objects: the index arithmetic IS the structure, and the shape is
// forced by n alone.
//
//	parent(i) = (i-1)/2     left(i) = 2i+1     right(i) = 2i+2
func parent(i int) int { return (i - 1) / 2 } // only meaningful for i > 0
func left(i int) int   { return 2*i + 1 }
func right(i int) int  { return 2*i + 2 }

// level is the depth of node i: row 0 holds 1 node, row 1 holds 2, row 2 holds 4...
// so node i sits in the row numbered bits.Len(i+1)-1.
func level(i int) int { return bits.Len(uint(i+1)) - 1 }

// isMinHeap states the invariant: no child is less than its parent.
func isMinHeap(a []int) bool {
	for i := 1; i < len(a); i++ {
		if a[i] < a[parent(i)] {
			return false
		}
	}
	return true
}

func isSorted(a []int) bool {
	for i := 1; i < len(a); i++ {
		if a[i] < a[i-1] {
			return false
		}
	}
	return true
}

// countHeaps tries every ordering of 0..n-1 (recursive swap) and counts the
// ones that satisfy the heap invariant.
func countHeaps(a []int, k int) int {
	if k == len(a) {
		if isMinHeap(a) {
			return 1
		}
		return 0
	}
	total := 0
	for j := k; j < len(a); j++ {
		a[k], a[j] = a[j], a[k]
		total += countHeaps(a, k+1)
		a[k], a[j] = a[j], a[k]
	}
	return total
}

// subtreeProduct is the product of every node's subtree size in the n-node heap
// shape. The number of heaps among the n! orderings is n! divided by it.
func subtreeProduct(n int) int {
	size := make([]int, n)
	prod := 1
	for i := n - 1; i >= 0; i-- {
		size[i] = 1
		if left(i) < n {
			size[i] += size[left(i)]
		}
		if right(i) < n {
			size[i] += size[right(i)]
		}
		prod *= size[i]
	}
	return prod
}

func main() {
	h := []int{1, 3, 2, 6, 5, 4, 9, 8, 7, 10}
	n := len(h)

	fmt.Println("(a) the array", h, "read as a tree, one row per level:")
	for lv, i := 0, 0; i < n; lv++ {
		var row []string
		for ; i < n && level(i) == lv; i++ {
			row = append(row, fmt.Sprint(h[i]))
		}
		fmt.Printf("  level %d: %s\n", lv, strings.Join(row, "  "))
	}
	fmt.Println("\n   i  value  parent  left  right  level")
	for i := range h {
		p, l, r := "-", "-", "-"
		if i > 0 {
			p = fmt.Sprint(parent(i))
		}
		if left(i) < n {
			l = fmt.Sprint(left(i))
		}
		if right(i) < n {
			r = fmt.Sprint(right(i))
		}
		fmt.Printf("  %2d  %5d  %6s  %4s  %5s  %5d\n", i, h[i], p, l, r, level(i))
	}
	sorted := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	fmt.Println("\nisMinHeap(heap array)   =", isMinHeap(h))
	fmt.Println("isMinHeap(sorted array) =", isMinHeap(sorted), "(a sorted array is always a heap)")
	fmt.Println("isSorted(heap array)    =", isSorted(h), "(a heap is only partly ordered)")

	// (b) The laws, for every size from 1 to 1000.
	ok := true
	for n := 1; n <= 1000; n++ {
		leaves := 0
		for i := 0; i < n; i++ {
			if left(i) < n && parent(left(i)) != i {
				ok = false
			}
			if right(i) < n && parent(right(i)) != i {
				ok = false
			}
			steps := 0 // walk to the root; the step count must equal the level
			for j := i; j > 0; j = parent(j) {
				steps++
			}
			if steps != level(i) {
				ok = false
			}
			isLeaf := left(i) >= n
			if isLeaf != (i >= n/2) {
				ok = false
			}
			if isLeaf {
				leaves++
			}
		}
		if leaves != n-n/2 || level(n-1) != bits.Len(uint(n))-1 {
			ok = false
		}
	}
	fmt.Println("\n(b) laws hold for n = 1..1000:", ok)
	fmt.Println("\n      n  height  leaves  internal")
	for _, n := range []int{1, 2, 3, 7, 8, 1000} {
		fmt.Printf("  %5d  %6d  %6d  %8d\n", n, bits.Len(uint(n))-1, n-n/2, n/2)
	}

	// (c) The -1 trap. Go's integer division truncates toward zero.
	j := 0
	fmt.Println("\n(c) parent of the root as (j-1)/2 with j = 0:", (j-1)/2, "-> the root is its own parent")
	fmt.Println("    container/heap's up() therefore stops on `i == j`, not on i < 0")
	// The 1-based variant (slot 0 unused): parent k/2, children 2k and 2k+1.
	same := true
	for i := 1; i < n; i++ {
		same = same && (i+1)/2 == parent(i)+1 && 2*(i+1) == left(i)+1 && 2*(i+1)+1 == right(i)+1
	}
	fmt.Println("    1-based variant: parent k/2, children 2k and 2k+1 -- the same tree one slot over:", same)

	// (d) How rare are heaps? Count them among all n! orderings.
	fmt.Println("\n(d) valid min-heaps among all n! orderings of n distinct keys:")
	fmt.Println("     n      heaps         n!    ∏sizes  n!/∏sizes  agree")
	allAgree, lastProd := true, 0
	fact := 1
	for n := 1; n <= 10; n++ {
		fact *= n
		a := make([]int, n)
		for i := range a {
			a[i] = i
		}
		heaps := countHeaps(a, 0)
		prod := subtreeProduct(n)
		agree := fact/prod == heaps && fact%prod == 0
		allAgree = allAgree && agree
		fmt.Printf("  %4d  %9d  %9d  %8d  %9d  %v\n", n, heaps, fact, prod, fact/prod, agree)
		lastProd = prod
	}
	fmt.Println("all rows agree:", allAgree)
	fmt.Printf("so only 1 ordering in %d of 10 distinct keys is a heap (the fraction is 1/∏sizes)\n", lastProd)
}
```

**Output:**

```
(a) the array [1 3 2 6 5 4 9 8 7 10] read as a tree, one row per level:
  level 0: 1
  level 1: 3  2
  level 2: 6  5  4  9
  level 3: 8  7  10

   i  value  parent  left  right  level
   0      1       -     1      2      0
   1      3       0     3      4      1
   2      2       0     5      6      1
   3      6       1     7      8      2
   4      5       1     9      -      2
   5      4       2     -      -      2
   6      9       2     -      -      2
   7      8       3     -      -      3
   8      7       3     -      -      3
   9     10       4     -      -      3

isMinHeap(heap array)   = true
isMinHeap(sorted array) = true (a sorted array is always a heap)
isSorted(heap array)    = false (a heap is only partly ordered)

(b) laws hold for n = 1..1000: true

      n  height  leaves  internal
      1       0       1         0
      2       1       1         1
      3       1       2         1
      7       2       4         3
      8       3       4         4
   1000       9     500       500

(c) parent of the root as (j-1)/2 with j = 0: 0 -> the root is its own parent
    container/heap's up() therefore stops on `i == j`, not on i < 0
    1-based variant: parent k/2, children 2k and 2k+1 -- the same tree one slot over: true

(d) valid min-heaps among all n! orderings of n distinct keys:
     n      heaps         n!    ∏sizes  n!/∏sizes  agree
     1          1          1         1          1  true
     2          1          2         2          1  true
     3          2          6         3          2  true
     4          3         24         8          3  true
     5          8        120        15          8  true
     6         20        720        36         20  true
     7         80       5040        63         80  true
     8        210      40320       192        210  true
     9        896     362880       405        896  true
    10       3360    3628800      1080       3360  true
all rows agree: true
so only 1 ordering in 1080 of 10 distinct keys is a heap (the fraction is 1/∏sizes)
```

**Rust:**

```rust
// A heap is a complete binary tree stored level by level in a slice. There are
// no node objects: the index arithmetic IS the structure, and the shape is
// forced by n alone.
//
//   parent(i) = (i-1)/2     left(i) = 2i+1     right(i) = 2i+2
//
// Indexes are usize, which has no -1: the root's parent is `None`, not a
// number that merely happens to be wrong.
fn parent(i: usize) -> Option<usize> {
    i.checked_sub(1).map(|j| j / 2)
}
fn left(i: usize) -> usize {
    2 * i + 1
}
fn right(i: usize) -> usize {
    2 * i + 2
}

/// level is the depth of node i: row 0 holds 1 node, row 1 holds 2, row 2 holds 4...
/// so node i sits in the row numbered ilog2(i+1).
fn level(i: usize) -> usize {
    (i + 1).ilog2() as usize
}

/// is_min_heap states the invariant: no child is less than its parent.
fn is_min_heap(a: &[usize]) -> bool {
    (1..a.len()).all(|i| a[i] >= a[parent(i).unwrap()])
}

fn is_sorted(a: &[usize]) -> bool {
    a.windows(2).all(|w| w[0] <= w[1])
}

/// count_heaps tries every ordering of the slice (recursive swap) and counts
/// the ones that satisfy the heap invariant.
fn count_heaps(a: &mut [usize], k: usize) -> usize {
    if k == a.len() {
        return usize::from(is_min_heap(a));
    }
    let mut total = 0;
    for j in k..a.len() {
        a.swap(k, j);
        total += count_heaps(a, k + 1);
        a.swap(k, j);
    }
    total
}

/// subtree_product is the product of every node's subtree size in the n-node
/// heap shape. The number of heaps among the n! orderings is n! divided by it.
fn subtree_product(n: usize) -> usize {
    let mut size = vec![0usize; n];
    let mut prod = 1;
    for i in (0..n).rev() {
        size[i] = 1;
        if left(i) < n {
            size[i] += size[left(i)];
        }
        if right(i) < n {
            size[i] += size[right(i)];
        }
        prod *= size[i];
    }
    prod
}

/// Go prints a slice as [1 3 2]; Rust's {:?} prints [1, 3, 2].
fn go_slice(a: &[usize]) -> String {
    let parts: Vec<String> = a.iter().map(usize::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let h: Vec<usize> = vec![1, 3, 2, 6, 5, 4, 9, 8, 7, 10];
    let n = h.len();

    println!(
        "(a) the array {} read as a tree, one row per level:",
        go_slice(&h)
    );
    let mut i = 0;
    let mut lv = 0;
    while i < n {
        let mut row = Vec::new();
        while i < n && level(i) == lv {
            row.push(h[i].to_string());
            i += 1;
        }
        println!("  level {lv}: {}", row.join("  "));
        lv += 1;
    }
    println!("\n   i  value  parent  left  right  level");
    for (i, v) in h.iter().enumerate() {
        let show = |ix: usize| {
            if ix < n {
                ix.to_string()
            } else {
                "-".to_string()
            }
        };
        let p = parent(i).map_or("-".to_string(), |p| p.to_string());
        println!(
            "  {i:>2}  {v:>5}  {p:>6}  {:>4}  {:>5}  {:>5}",
            show(left(i)),
            show(right(i)),
            level(i)
        );
    }
    let sorted: Vec<usize> = (1..=10).collect();
    println!("\nisMinHeap(heap array)   = {}", is_min_heap(&h));
    println!(
        "isMinHeap(sorted array) = {} (a sorted array is always a heap)",
        is_min_heap(&sorted)
    );
    println!(
        "isSorted(heap array)    = {} (a heap is only partly ordered)",
        is_sorted(&h)
    );

    // (b) The laws, for every size from 1 to 1000.
    let mut ok = true;
    for n in 1..=1000usize {
        let mut leaves = 0;
        for i in 0..n {
            if left(i) < n && parent(left(i)) != Some(i) {
                ok = false;
            }
            if right(i) < n && parent(right(i)) != Some(i) {
                ok = false;
            }
            // walk to the root; the step count must equal the level
            let steps = std::iter::successors(parent(i), |&j| parent(j)).count();
            if steps != level(i) {
                ok = false;
            }
            let is_leaf = left(i) >= n;
            if is_leaf != (i >= n / 2) {
                ok = false;
            }
            if is_leaf {
                leaves += 1;
            }
        }
        if leaves != n - n / 2 || level(n - 1) != n.ilog2() as usize {
            ok = false;
        }
    }
    println!("\n(b) laws hold for n = 1..1000: {ok}");
    println!("\n      n  height  leaves  internal");
    for n in [1usize, 2, 3, 7, 8, 1000] {
        println!("  {n:>5}  {:>6}  {:>6}  {:>8}", n.ilog2(), n - n / 2, n / 2);
    }

    // (c) The -1 trap. usize has no -1: the subtraction is checked, not truncated.
    println!(
        "\n(c) parent of the root as 0usize.checked_sub(1): {:?} -> the root has no parent",
        0usize.checked_sub(1)
    );
    println!(
        "    0usize.wrapping_sub(1) = {} (what a plain `0 - 1` becomes under -O; a debug build panics)",
        0usize.wrapping_sub(1)
    );
    // The 1-based variant (slot 0 unused): parent k/2, children 2k and 2k+1.
    let same = (1..n).all(|i| {
        let k = i + 1; // 1-based slot number
        k / 2 == parent(i).unwrap() + 1 && 2 * k == left(i) + 1 && 2 * k + 1 == right(i) + 1
    });
    println!(
        "    1-based variant: parent k/2, children 2k and 2k+1 -- the same tree one slot over: {same}"
    );

    // (d) How rare are heaps? Count them among all n! orderings.
    println!("\n(d) valid min-heaps among all n! orderings of n distinct keys:");
    println!("     n      heaps         n!    ∏sizes  n!/∏sizes  agree");
    let mut all_agree = true;
    let mut last_prod = 0;
    let mut fact = 1;
    for n in 1..=10usize {
        fact *= n;
        let mut a: Vec<usize> = (0..n).collect();
        let heaps = count_heaps(&mut a, 0);
        let prod = subtree_product(n);
        let agree = fact / prod == heaps && fact % prod == 0;
        all_agree &= agree;
        println!(
            "  {n:>4}  {heaps:>9}  {fact:>9}  {prod:>8}  {:>9}  {agree}",
            fact / prod
        );
        last_prod = prod;
    }
    println!("all rows agree: {all_agree}");
    println!(
        "so only 1 ordering in {last_prod} of 10 distinct keys is a heap (the fraction is 1/∏sizes)"
    );
}
```

**Rust output:**

```
(a) the array [1 3 2 6 5 4 9 8 7 10] read as a tree, one row per level:
  level 0: 1
  level 1: 3  2
  level 2: 6  5  4  9
  level 3: 8  7  10

   i  value  parent  left  right  level
   0      1       -     1      2      0
   1      3       0     3      4      1
   2      2       0     5      6      1
   3      6       1     7      8      2
   4      5       1     9      -      2
   5      4       2     -      -      2
   6      9       2     -      -      2
   7      8       3     -      -      3
   8      7       3     -      -      3
   9     10       4     -      -      3

isMinHeap(heap array)   = true
isMinHeap(sorted array) = true (a sorted array is always a heap)
isSorted(heap array)    = false (a heap is only partly ordered)

(b) laws hold for n = 1..1000: true

      n  height  leaves  internal
      1       0       1         0
      2       1       1         1
      3       1       2         1
      7       2       4         3
      8       3       4         4
   1000       9     500       500

(c) parent of the root as 0usize.checked_sub(1): None -> the root has no parent
    0usize.wrapping_sub(1) = 18446744073709551615 (what a plain `0 - 1` becomes under -O; a debug build panics)
    1-based variant: parent k/2, children 2k and 2k+1 -- the same tree one slot over: true

(d) valid min-heaps among all n! orderings of n distinct keys:
     n      heaps         n!    ∏sizes  n!/∏sizes  agree
     1          1          1         1          1  true
     2          1          2         2          1  true
     3          2          6         3          2  true
     4          3         24         8          3  true
     5          8        120        15          8  true
     6         20        720        36         20  true
     7         80       5040        63         80  true
     8        210      40320       192        210  true
     9        896     362880       405        896  true
    10       3360    3628800      1080       3360  true
all rows agree: true
so only 1 ordering in 1080 of 10 distinct keys is a heap (the fraction is 1/∏sizes)
```

> **Go vs Rust:** The output is the Go output line for line except the two lines about the root's parent: Go's `(0-1)/2` truncates to `0`, so the root is silently its own parent (hence the `i == j` guard in `container/heap`'s `up`), while `parent` here returns `Option<usize>` and `0usize.checked_sub(1)` is `None`. `usize::ilog2` replaces `bits.Len(i+1)-1`, and the exhaustive count (3,628,800 orderings at n = 10) agrees with the formula in both languages.

**Complexity:** `parent`, `left`, `right`, `level` Θ(1) with zero pointers · height `⌊lg n⌋`, leaves `⌈n/2⌉` (indices `⌊n/2⌋ … n-1`) for every n from 1 to 1000 · of the n! orderings of n distinct keys exactly `n! / ∏ subtree sizes` are heaps: 3,360 of 3,628,800 at n = 10, one in 1,080

---

## 2. Sift up and sift down, counted

`🟢 easy` · *Sift / Θ(log n)*

`Push` is sift-up and `Pop` is sift-down, and their comparison and swap counts are the whole cost model. A push costs at most `⌊lg n⌋` swaps but only Θ(1) on average when keys arrive in random order, and Θ(log n) when they arrive in the wrong direction; a pop spends two comparisons per level it descends, Θ(log n). The heap here is hand-written, with its sifts copied from `container/heap`'s `up` and `down`, so every number printed is a property of the algorithm (lesson 13 example 6 counted this sift inside heapsort; here it is `Push`/`Pop` on three input shapes).

**Steps:**

1. Write `MinHeap` with `less` and `swap` that bump counters, and `Push` (sift up) and `Pop` (sift down) on top of them.
2. Trace seven pushes and three pops, then count n pushes and n pops at n = 2^10, 2^14, 2^18 on random, ascending and descending input.
3. Compare the per-operation counts against the exact formulas, and run 20,000 random operations against a sorted-slice model.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// MinHeap is a hand-written binary min-heap whose two sifts are written exactly
// like container/heap's up and down (strict <, the LEFT child wins ties, one
// swap per level), with a counter on every comparison and every swap.
type MinHeap struct {
	a           []int
	cmps, swaps int
}

func (h *MinHeap) less(i, j int) bool { h.cmps++; return h.a[i] < h.a[j] }
func (h *MinHeap) swap(i, j int)      { h.swaps++; h.a[i], h.a[j] = h.a[j], h.a[i] }

// Push appends at the only slot the shape allows, then sifts UP: swap with the
// parent while smaller. At most floor(lg n) swaps; each costs one comparison.
func (h *MinHeap) Push(x int) {
	h.a = append(h.a, x)
	for j := len(h.a) - 1; ; {
		i := (j - 1) / 2 // parent; at the root (0-1)/2 == 0 == j in Go
		if i == j || !h.less(j, i) {
			break
		}
		h.swap(i, j)
		j = i
	}
}

// Pop removes the root: move the LAST element to the top, shrink, and sift DOWN.
// Each level costs two comparisons (the smaller child, then child vs sinking
// element). Moving the last element to the root is not a sift step: not counted.
func (h *MinHeap) Pop() int {
	n := len(h.a) - 1
	x := h.a[0]
	h.a[0] = h.a[n]
	h.a = h.a[:n]
	for i := 0; ; {
		j := 2*i + 1 // left child
		if j >= n {
			break
		}
		if j+1 < n && h.less(j+1, j) { // right child wins only if strictly smaller
			j++
		}
		if !h.less(j, i) {
			break
		}
		h.swap(i, j)
		i = j
	}
	return x
}

func (h *MinHeap) reset() { h.cmps, h.swaps = 0, 0 }

func isMinHeap(a []int) bool {
	for i := 1; i < len(a); i++ {
		if a[i] < a[(i-1)/2] {
			return false
		}
	}
	return true
}

// permutation is a hand-written Fisher-Yates shuffle of 0..n-1.
func permutation(r *rand.Rand, n int) []int {
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	// (a) A trace: every push and pop with its own comparison and swap count.
	fmt.Println("(a) trace of an empty min-heap:")
	h := &MinHeap{}
	for _, x := range []int{5, 3, 8, 1, 9, 2, 7} {
		h.reset()
		h.Push(x)
		fmt.Printf("  push %d    %-24s cmps %d swaps %d\n", x, fmt.Sprint(h.a), h.cmps, h.swaps)
	}
	for range 3 {
		h.reset()
		x := h.Pop()
		fmt.Printf("  pop -> %d  %-24s cmps %d swaps %d\n", x, fmt.Sprint(h.a), h.cmps, h.swaps)
	}

	// (b) The cost model at scale: n pushes, then n pops, for three input shapes.
	fmt.Println("\n(b) n pushes, then n pops (counts per operation):")
	fmt.Printf("  %7s %-10s  %9s %9s  %9s %9s  %6s %8s\n",
		"n", "shape", "cmp/push", "swp/push", "cmp/pop", "swp/pop", "2 lg n", "pop/2lgn")
	exact := true
	var perLevel []float64 // random pops: comparisons per level descended
	for _, n := range []int{1 << 10, 1 << 14, 1 << 18} {
		asc := make([]int, n)
		desc := make([]int, n)
		for i := range asc {
			asc[i], desc[i] = i, n-1-i
		}
		shapes := []struct {
			name string
			in   []int
		}{{"random", permutation(r, n)}, {"ascending", asc}, {"descending", desc}}
		for _, s := range shapes {
			h := &MinHeap{}
			for _, x := range s.in {
				h.Push(x)
			}
			pc, ps := h.cmps, h.swaps
			h.reset()
			for len(h.a) > 0 {
				h.Pop()
			}
			twoLg := 2 * (bits.Len(uint(n)) - 1)
			fmt.Printf("  %7d %-10s  %9.2f %9.2f  %9.2f %9.2f  %6d %8.2f\n", n, s.name,
				float64(pc)/float64(n), float64(ps)/float64(n),
				float64(h.cmps)/float64(n), float64(h.swaps)/float64(n),
				twoLg, float64(h.cmps)/float64(n)/float64(twoLg))
			switch s.name {
			case "random":
				perLevel = append(perLevel, float64(h.cmps)/float64(h.swaps))
			case "ascending": // the new key never beats its parent: 1 comparison, 0 swaps (the root pays none)
				exact = exact && pc == n-1 && ps == 0
			case "descending": // every new key is a new minimum: it rises through all its levels
				want := 0
				for i := 0; i < n; i++ {
					want += bits.Len(uint(i+1)) - 1
				}
				exact = exact && ps == want
			}
		}
	}
	fmt.Printf("  random pops: comparisons per level descended (cmps/swaps) = %.2f, %.2f, %.2f\n", perLevel[0], perLevel[1], perLevel[2])
	fmt.Println("  ascending = n-1 comparisons and 0 swaps; descending swaps = sum of every node's level:", exact)

	// (c) Oracle: 20,000 random operations against a sorted-slice model.
	h = &MinHeap{}
	var model []int
	pushes, pops, bad, broken := 0, 0, 0, 0
	for range 20000 {
		if r.IntN(10) < 6 {
			x := r.IntN(50)
			h.Push(x)
			i, _ := slices.BinarySearch(model, x)
			model = slices.Insert(model, i, x)
			pushes++
		} else if len(model) > 0 {
			if h.Pop() != model[0] {
				bad++
			}
			model = model[1:]
			pops++
		}
		if len(h.a) != len(model) {
			bad++
		}
		if !isMinHeap(h.a) {
			broken++
		}
	}
	fmt.Printf("\n(c) %d pushes and %d pops against a sorted-slice model, final size %d\n", pushes, pops, len(h.a))
	fmt.Printf("    mismatches: %d  invariant violations: %d\n", bad, broken)
}
```

**Output:**

```
(a) trace of an empty min-heap:
  push 5    [5]                      cmps 0 swaps 0
  push 3    [3 5]                    cmps 1 swaps 1
  push 8    [3 5 8]                  cmps 1 swaps 0
  push 1    [1 3 8 5]                cmps 2 swaps 2
  push 9    [1 3 8 5 9]              cmps 1 swaps 0
  push 2    [1 3 2 5 9 8]            cmps 2 swaps 1
  push 7    [1 3 2 5 9 8 7]          cmps 1 swaps 0
  pop -> 1  [2 3 7 5 9 8]            cmps 3 swaps 1
  pop -> 2  [3 5 7 8 9]              cmps 4 swaps 2
  pop -> 3  [5 8 7 9]                cmps 3 swaps 2

(b) n pushes, then n pops (counts per operation):
        n shape        cmp/push  swp/push    cmp/pop   swp/pop  2 lg n pop/2lgn
     1024 random           2.23      1.24      15.04      7.39      20     0.75
     1024 ascending        1.00      0.00      15.02      7.34      20     0.75
     1024 descending       8.01      8.01      14.52      7.01      20     0.73
    16384 random           2.26      1.26      23.04     11.37      28     0.82
    16384 ascending        1.00      0.00      23.08     11.39      28     0.82
    16384 descending      12.00     12.00      22.50     11.00      28     0.80
   262144 random           2.28      1.28      31.03     15.37      36     0.86
   262144 ascending        1.00      0.00      31.03     15.35      36     0.86
   262144 descending      16.00     16.00      30.50     15.00      36     0.85
  random pops: comparisons per level descended (cmps/swaps) = 2.04, 2.03, 2.02
  ascending = n-1 comparisons and 0 swaps; descending swaps = sum of every node's level: true

(c) 11975 pushes and 8024 pops against a sorted-slice model, final size 3951
    mismatches: 0  invariant violations: 0
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same permutations as Go.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
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
        assert!(n > 0, "invalid argument to int_n");
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

/// MinHeap is a hand-written binary min-heap whose two sifts are written exactly
/// like container/heap's up and down (strict <, the LEFT child wins ties, one
/// swap per level), with a counter on every comparison and every swap.
/// (Std's BinaryHeap is different: a max-heap that moves a hole instead of
/// swapping, and its pop sifts bottom-up -- example 15 counts that.)
#[derive(Default)]
struct MinHeap {
    a: Vec<u64>,
    cmps: u64,
    swaps: u64,
}

impl MinHeap {
    fn less(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        self.a[i] < self.a[j]
    }

    fn swap(&mut self, i: usize, j: usize) {
        self.swaps += 1;
        self.a.swap(i, j);
    }

    /// Push appends at the only slot the shape allows, then sifts UP: swap with
    /// the parent while smaller. At most floor(lg n) swaps; each costs one comparison.
    fn push(&mut self, x: u64) {
        self.a.push(x);
        let mut j = self.a.len() - 1;
        // usize has no -1, so the loop condition is `j > 0` where Go tests `i == j`
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less(j, i) {
                break;
            }
            self.swap(i, j);
            j = i;
        }
    }

    /// Pop removes the root: move the LAST element to the top, shrink, and sift
    /// DOWN. Each level costs two comparisons (the smaller child, then child vs
    /// sinking element). Moving the last element to the root is not a sift
    /// step: not counted. Returns None on an empty heap (Go would panic).
    fn pop(&mut self) -> Option<u64> {
        let last = self.a.pop()?;
        if self.a.is_empty() {
            return Some(last);
        }
        let x = std::mem::replace(&mut self.a[0], last);
        let n = self.a.len();
        let mut i = 0;
        loop {
            let mut j = 2 * i + 1; // left child
            if j >= n {
                break;
            }
            if j + 1 < n && self.less(j + 1, j) {
                // right child wins only if strictly smaller
                j += 1;
            }
            if !self.less(j, i) {
                break;
            }
            self.swap(i, j);
            i = j;
        }
        Some(x)
    }

    fn reset(&mut self) {
        self.cmps = 0;
        self.swaps = 0;
    }
}

fn is_min_heap(a: &[u64]) -> bool {
    (1..a.len()).all(|i| a[i] >= a[(i - 1) / 2])
}

/// permutation is a hand-written Fisher-Yates shuffle of 0..n-1.
fn permutation(r: &mut Pcg, n: usize) -> Vec<u64> {
    let mut p: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

/// Go prints a slice as [1 3 2]; Rust's {:?} prints [1, 3, 2].
fn go_slice(a: &[u64]) -> String {
    let parts: Vec<String> = a.iter().map(u64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let mut r = Pcg::new(1, 2);

    // (a) A trace: every push and pop with its own comparison and swap count.
    println!("(a) trace of an empty min-heap:");
    let mut h = MinHeap::default();
    for x in [5, 3, 8, 1, 9, 2, 7] {
        h.reset();
        h.push(x);
        println!(
            "  push {x}    {:<24} cmps {} swaps {}",
            go_slice(&h.a),
            h.cmps,
            h.swaps
        );
    }
    for _ in 0..3 {
        h.reset();
        let x = h.pop().unwrap();
        println!(
            "  pop -> {x}  {:<24} cmps {} swaps {}",
            go_slice(&h.a),
            h.cmps,
            h.swaps
        );
    }

    // (b) The cost model at scale: n pushes, then n pops, for three input shapes.
    println!("\n(b) n pushes, then n pops (counts per operation):");
    println!(
        "  {:>7} {:<10}  {:>9} {:>9}  {:>9} {:>9}  {:>6} {:>8}",
        "n", "shape", "cmp/push", "swp/push", "cmp/pop", "swp/pop", "2 lg n", "pop/2lgn"
    );
    let mut exact = true;
    let mut per_level = Vec::new(); // random pops: comparisons per level descended
    for n in [1usize << 10, 1 << 14, 1 << 18] {
        let asc: Vec<u64> = (0..n as u64).collect();
        let desc: Vec<u64> = (0..n as u64).rev().collect();
        let shapes = [
            ("random", permutation(&mut r, n)),
            ("ascending", asc),
            ("descending", desc),
        ];
        for (name, input) in &shapes {
            let mut h = MinHeap::default();
            for &x in input {
                h.push(x);
            }
            let (pc, ps) = (h.cmps, h.swaps);
            h.reset();
            while h.pop().is_some() {}
            let two_lg = 2 * n.ilog2() as u64;
            let nf = n as f64;
            println!(
                "  {n:>7} {name:<10}  {:>9.2} {:>9.2}  {:>9.2} {:>9.2}  {two_lg:>6} {:>8.2}",
                pc as f64 / nf,
                ps as f64 / nf,
                h.cmps as f64 / nf,
                h.swaps as f64 / nf,
                h.cmps as f64 / nf / two_lg as f64
            );
            match *name {
                "random" => per_level.push(h.cmps as f64 / h.swaps as f64),
                // the new key never beats its parent: 1 comparison, 0 swaps (the root pays none)
                "ascending" => exact = exact && pc == n as u64 - 1 && ps == 0,
                // every new key is a new minimum: it rises through all its levels
                _ => exact = exact && ps == (0..n).map(|i| (i + 1).ilog2() as u64).sum::<u64>(),
            }
        }
    }
    println!(
        "  random pops: comparisons per level descended (cmps/swaps) = {:.2}, {:.2}, {:.2}",
        per_level[0], per_level[1], per_level[2]
    );
    println!(
        "  ascending = n-1 comparisons and 0 swaps; descending swaps = sum of every node's level: {exact}"
    );

    // (c) Oracle: 20,000 random operations against a sorted-slice model.
    let mut h = MinHeap::default();
    let mut model: Vec<u64> = Vec::new();
    let (mut pushes, mut pops, mut bad, mut broken) = (0, 0, 0, 0);
    for _ in 0..20000 {
        if r.int_n(10) < 6 {
            let x = r.int_n(50);
            h.push(x);
            let i = model.binary_search(&x).unwrap_or_else(|i| i);
            model.insert(i, x);
            pushes += 1;
        } else if !model.is_empty() {
            if h.pop() != Some(model.remove(0)) {
                bad += 1;
            }
            pops += 1;
        }
        if h.a.len() != model.len() {
            bad += 1;
        }
        if !is_min_heap(&h.a) {
            broken += 1;
        }
    }
    println!(
        "\n(c) {pushes} pushes and {pops} pops against a sorted-slice model, final size {}",
        h.a.len()
    );
    println!("    mismatches: {bad}  invariant violations: {broken}");
}
```

**Rust output:**

```
(a) trace of an empty min-heap:
  push 5    [5]                      cmps 0 swaps 0
  push 3    [3 5]                    cmps 1 swaps 1
  push 8    [3 5 8]                  cmps 1 swaps 0
  push 1    [1 3 8 5]                cmps 2 swaps 2
  push 9    [1 3 8 5 9]              cmps 1 swaps 0
  push 2    [1 3 2 5 9 8]            cmps 2 swaps 1
  push 7    [1 3 2 5 9 8 7]          cmps 1 swaps 0
  pop -> 1  [2 3 7 5 9 8]            cmps 3 swaps 1
  pop -> 2  [3 5 7 8 9]              cmps 4 swaps 2
  pop -> 3  [5 8 7 9]                cmps 3 swaps 2

(b) n pushes, then n pops (counts per operation):
        n shape        cmp/push  swp/push    cmp/pop   swp/pop  2 lg n pop/2lgn
     1024 random           2.23      1.24      15.04      7.39      20     0.75
     1024 ascending        1.00      0.00      15.02      7.34      20     0.75
     1024 descending       8.01      8.01      14.52      7.01      20     0.73
    16384 random           2.26      1.26      23.04     11.37      28     0.82
    16384 ascending        1.00      0.00      23.08     11.39      28     0.82
    16384 descending      12.00     12.00      22.50     11.00      28     0.80
   262144 random           2.28      1.28      31.03     15.37      36     0.86
   262144 ascending        1.00      0.00      31.03     15.35      36     0.86
   262144 descending      16.00     16.00      30.50     15.00      36     0.85
  random pops: comparisons per level descended (cmps/swaps) = 2.04, 2.03, 2.02
  ascending = n-1 comparisons and 0 swaps; descending swaps = sum of every node's level: true

(c) 11975 pushes and 8024 pops against a sorted-slice model, final size 3951
    mismatches: 0  invariant violations: 0
```

> **Go vs Rust:** Identical output, line for line (diffed against a live run of the Go program): `Pcg` is the same port of `math/rand/v2` used in lesson 01, so the same seed `(1, 2)` yields the same permutations, and the hand-written sifts are the same loops, so every count matches; only the root guard changes, `while j > 0` for Go's `i == j`. `pop` returns `Option<u64>` where Go's would panic on an empty heap. This is *not* what `BinaryHeap` does: std moves a hole instead of swapping and its `pop` sifts bottom-up with one comparison per level (read in the source, not run here; example 15 counts it).

**Complexity:** `Push` ≤ `⌊lg n⌋` swaps, measured: ascending input costs 1 comparison and 0 swaps per push, descending input makes every key rise through its whole level (8.01 / 12.00 / 16.00 swaps per push), random input about 2.2–2.3 comparisons and 1.2–1.3 swaps, barely growing from n = 2^10 to 2^18 · `Pop` ≤ `2⌊lg n⌋` comparisons, about 15 / 23 / 31 per pop at 2^10 / 2^14 / 2^18, two per level descended · space Θ(1) extra

---

## 3. `container/heap`: five methods and a call order

`🟢 easy` · *The interface contract*

`heap.Interface` is five methods, but you never call two of them: the `Push` and `Pop` methods on your type are **hooks** that the package calls. `heap.Push` calls your `Push` (an append) and then sifts up; `heap.Pop` swaps the root to the tail, sifts down within the first `n-1` elements, and only then calls your `Pop`, which removes the tail. This example logs every method call to show that order exactly, then shows what calling the hooks directly does, and finally counts the calls over 1,000 pushes and 1,000 pops.

**Steps:**

1. Implement all five methods on a `traced` type that records each call.
2. Run one `heap.Push` and one `heap.Pop` and read the call sequence; then call `h.Push` and `h.Pop` directly on a valid heap and see what breaks (and `heap.Init` repair it).
3. Count the calls to each method over 1,000 `heap.Push` and 1,000 `heap.Pop`.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// heap.Interface is five methods: sort.Interface (Len, Less, Swap) plus Push and
// Pop. You implement all five, but you only ever CALL the package functions:
// heap.Push, heap.Pop, heap.Init, heap.Fix, heap.Remove. The Push and Pop
// METHODS are hooks that the package calls for you.
//
// traced is a min-heap of ints that counts every method call and can log them.
type traced struct {
	a     []int
	log   []string
	on    bool
	calls map[string]int
}

func (t *traced) rec(kind, text string) {
	t.calls[kind]++
	if t.on {
		t.log = append(t.log, text)
	}
}

func (t *traced) Len() int { t.rec("Len", "Len()"); return len(t.a) }
func (t *traced) Less(i, j int) bool {
	r := t.a[i] < t.a[j]
	t.rec("Less", fmt.Sprintf("Less(%d,%d) = %v", i, j, r))
	return r
}
func (t *traced) Swap(i, j int) {
	t.rec("Swap", fmt.Sprintf("Swap(%d,%d)", i, j))
	t.a[i], t.a[j] = t.a[j], t.a[i]
}

// Push is a hook: it only appends. The sift is heap.Push's job.
func (t *traced) Push(x any) { t.rec("Push", fmt.Sprintf("Push(%v)", x)); t.a = append(t.a, x.(int)) }

// Pop is a hook: it removes the LAST element -- not the minimum.
func (t *traced) Pop() any {
	t.rec("Pop", "Pop()")
	n := len(t.a) - 1
	x := t.a[n]
	t.a = t.a[:n]
	return x
}

func isMinHeap(a []int) bool {
	for i := 1; i < len(a); i++ {
		if a[i] < a[(i-1)/2] {
			return false
		}
	}
	return true
}

// siftOnly reports whether every logged call is a Less or a Swap.
func siftOnly(log []string) bool {
	for _, c := range log {
		if !strings.HasPrefix(c, "Less") && !strings.HasPrefix(c, "Swap") {
			return false
		}
	}
	return true
}

// show prints one numbered call per line with an optional hint.
func show(log []string, hint func(i int) string) {
	for i, c := range log {
		fmt.Println(strings.TrimRight(fmt.Sprintf("      %d. %-20s %s", i+1, c, hint(i)), " "))
	}
}

func newTraced(a ...int) *traced { return &traced{a: slices.Clone(a), calls: map[string]int{}} }

func main() {
	// (a) The exact order in which heap.Push and heap.Pop call your methods.
	h := newTraced()
	for _, x := range []int{5, 3, 8, 1} {
		heap.Push(h, x)
	}
	fmt.Println("(a) built with heap.Push of 5, 3, 8, 1:", h.a)

	h.on = true
	heap.Push(h, 2)
	fmt.Println("\n    heap.Push(h, 2) calls, in order:")
	show(h.log, func(i int) string {
		switch i {
		case 0:
			return "<- your hook: append at the tail"
		case 1:
			return "<- the package reads the length to find the tail index"
		case 2:
			return "<- sift up: the tail against its parent"
		}
		return ""
	})
	pushOK := h.log[0] == "Push(2)" && h.log[1] == "Len()" && siftOnly(h.log[2:])
	fmt.Printf("    -> %v   (your Push, then Len, then only Less/Swap: %v)\n", h.a, pushOK)

	h.log = nil
	n := len(h.a) - 1 // tail index before the pop
	got := heap.Pop(h)
	fmt.Println("\n    heap.Pop(h) calls, in order:")
	show(h.log, func(i int) string {
		switch i {
		case 0:
			return "<- n = Len()-1"
		case 1:
			return "<- the root goes to the tail BEFORE any sifting"
		case 2:
			return fmt.Sprintf("<- sift down within the first n = %d elements only", n)
		case len(h.log) - 1:
			return "<- your hook: remove the tail, which is the old root"
		}
		return ""
	})
	last := len(h.log) - 1
	popOK := h.log[0] == "Len()" && h.log[1] == fmt.Sprintf("Swap(0,%d)", n) && siftOnly(h.log[2:last]) && h.log[last] == "Pop()"
	fmt.Printf("    -> returned %v, array %v   (Len, Swap(0,n-1), only Less/Swap, then your Pop: %v)\n", got, h.a, popOK)
	h.on = false

	// (b) What calling the hooks directly does. Start from a valid heap each time.
	valid := []int{1, 3, 2, 6, 5, 4}
	fmt.Println("\n(b) misuse, starting from the valid heap", valid)

	m := newTraced(valid...)
	m.Push(0) // WRONG: appends 0 at the tail, no sift
	fmt.Printf("    h.Push(0) directly:   %v  isMinHeap = %v\n", m.a, isMinHeap(m.a))
	c := newTraced(m.a...)
	x1 := heap.Pop(c)
	x2 := heap.Pop(c)
	fmt.Printf("    heap.Pop on that returns %v, then %v: the minimum 0 does not come out first\n", x1, x2)

	m2 := newTraced(valid...)
	popped := m2.Pop() // WRONG: removes the last element, not the minimum
	fmt.Printf("    h.Pop() directly:     returns %v (the last element, the minimum is %d), leaves %v\n", popped, valid[0], m2.a)

	heap.Init(m) // the package's own repair: Floyd's heapify, in place
	fmt.Printf("    heap.Init(h) repairs: %v  isMinHeap = %v\n", m.a, isMinHeap(m.a))
	fmt.Printf("    and heap.Pop(h) now returns %v\n", heap.Pop(m))

	// (c) Call counts over 1,000 pushes and 1,000 pops of seeded random ints.
	r := rand.New(rand.NewPCG(1, 2))
	k := newTraced()
	for range 1000 {
		heap.Push(k, r.IntN(1000))
	}
	pushPhase := copyCounts(k.calls)
	var drained []int
	for len(k.a) > 0 {
		drained = append(drained, heap.Pop(k).(int))
	}
	fmt.Println("\n(c) 1,000 heap.Push, then 1,000 heap.Pop, of random ints (calls to each method):")
	fmt.Println("    method   1000 Pushes   1000 Pops")
	for _, name := range []string{"Len", "Less", "Swap", "Push", "Pop"} {
		fmt.Printf("    %-6s %11d %11d\n", name, pushPhase[name], k.calls[name]-pushPhase[name])
	}
	fmt.Printf("    your Push once per heap.Push, your Pop once per heap.Pop: %v\n",
		k.calls["Push"] == 1000 && k.calls["Pop"] == 1000)
	fmt.Printf("    Len once per heap.Push and once per heap.Pop: %v\n", pushPhase["Len"] == 1000 && k.calls["Len"] == 2000)
	fmt.Println("    drain is sorted:", slices.IsSorted(drained))
}

func copyCounts(m map[string]int) map[string]int {
	c := map[string]int{}
	for k, v := range m {
		c[k] = v
	}
	return c
}
```

**Output:**

```
(a) built with heap.Push of 5, 3, 8, 1: [1 3 8 5]

    heap.Push(h, 2) calls, in order:
      1. Push(2)              <- your hook: append at the tail
      2. Len()                <- the package reads the length to find the tail index
      3. Less(4,1) = true     <- sift up: the tail against its parent
      4. Swap(1,4)
      5. Less(1,0) = false
    -> [1 2 8 5 3]   (your Push, then Len, then only Less/Swap: true)

    heap.Pop(h) calls, in order:
      1. Len()                <- n = Len()-1
      2. Swap(0,4)            <- the root goes to the tail BEFORE any sifting
      3. Less(2,1) = false    <- sift down within the first n = 4 elements only
      4. Less(1,0) = true
      5. Swap(0,1)
      6. Less(3,1) = false
      7. Pop()                <- your hook: remove the tail, which is the old root
    -> returned 1, array [2 3 8 5]   (Len, Swap(0,n-1), only Less/Swap, then your Pop: true)

(b) misuse, starting from the valid heap [1 3 2 6 5 4]
    h.Push(0) directly:   [1 3 2 6 5 4 0]  isMinHeap = false
    heap.Pop on that returns 1, then 0: the minimum 0 does not come out first
    h.Pop() directly:     returns 4 (the last element, the minimum is 1), leaves [1 3 2 6 5]
    heap.Init(h) repairs: [0 3 1 6 5 4 2]  isMinHeap = true
    and heap.Pop(h) now returns 0

(c) 1,000 heap.Push, then 1,000 heap.Pop, of random ints (calls to each method):
    method   1000 Pushes   1000 Pops
    Len           1000        1000
    Less          2261       15010
    Swap          1268        8360
    Push          1000           0
    Pop              0        1000
    your Push once per heap.Push, your Pop once per heap.Pop: true
    Len once per heap.Push and once per heap.Pop: true
    drain is sorted: true
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same permutations as Go.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
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
        assert!(n > 0, "invalid argument to int_n");
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

// There is no heap.Interface to implement in Rust and no hook to call by
// mistake: BinaryHeap<T: Ord> owns its Vec, and `push` / `pop` are the only
// ways in and out, so safe code cannot skip a sift. The comparison is
// `Ord::cmp` on the element type, so to see what std does we log inside it.
//
// BinaryHeap is a MAX-heap; Reverse<Logged> turns it into a min-heap.
use std::cell::{Cell, RefCell};
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;

thread_local! {
    // No mutable globals in Rust: a thread_local Cell/RefCell is the usual stand-in.
    static CMPS: Cell<u64> = const { Cell::new(0) };
    static LOG_ON: Cell<bool> = const { Cell::new(false) };
    static LOG: RefCell<Vec<String>> = const { RefCell::new(Vec::new()) };
}

/// Logged is an i32 whose `cmp` counts itself and, when asked, writes itself to the log.
#[derive(PartialEq, Eq)]
struct Logged(i32);

impl Ord for Logged {
    fn cmp(&self, other: &Self) -> Ordering {
        let o = self.0.cmp(&other.0);
        CMPS.set(CMPS.get() + 1);
        if LOG_ON.get() {
            LOG.with_borrow_mut(|l| l.push(format!("cmp({}, {}) = {:?}", self.0, other.0, o)));
        }
        o
    }
}

// std's `<`, `<=`, `>=` on Logged go through partial_cmp, which delegates to cmp:
// one logged call per comparison.
impl PartialOrd for Logged {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

type MinHeap = BinaryHeap<Reverse<Logged>>;

fn keys(h: &MinHeap) -> Vec<i32> {
    h.as_slice().iter().map(|r| r.0.0).collect()
}

fn go_slice(a: &[i32]) -> String {
    let parts: Vec<String> = a.iter().map(i32::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn is_min_heap(a: &[i32]) -> bool {
    (1..a.len()).all(|i| a[i] >= a[(i - 1) / 2])
}

fn take_log() -> Vec<String> {
    LOG.with_borrow_mut(std::mem::take)
}

fn print_log(log: &[String]) {
    for (i, c) in log.iter().enumerate() {
        println!("      {}. {c}", i + 1);
    }
}

fn main() {
    // (a) std's comparison sequence for one push and one pop.
    let mut h: MinHeap = BinaryHeap::new();
    for x in [5, 3, 8, 1] {
        h.push(Reverse(Logged(x)));
    }
    println!("(a) built with push of 5, 3, 8, 1: {}", go_slice(&keys(&h)));

    LOG_ON.set(true);
    h.push(Reverse(Logged(2)));
    let log = take_log();
    println!("\n    push(2) compares, in order (no hooks, no Len, no Swap -- a hole moves):");
    print_log(&log);
    println!("    -> {}", go_slice(&keys(&h)));

    let got = h.pop().map(|r| r.0.0);
    let log = take_log();
    println!(
        "\n    pop() compares, in order (bottom-up: one cmp per level, then a short sift up):"
    );
    print_log(&log);
    println!("    -> returned {got:?}, array {}", go_slice(&keys(&h)));
    LOG_ON.set(false);

    // (b) There is nothing to misuse. The nearest thing is to take the buffer
    // out (`into_vec`), write to it as a plain Vec, and put it back.
    let valid = [1, 3, 2, 6, 5, 4];
    println!(
        "\n(b) the nearest misuse, starting from the valid heap {}",
        go_slice(&valid)
    );
    let heap: MinHeap = valid
        .iter()
        .map(|&x| Reverse(Logged(x)))
        .collect::<Vec<_>>()
        .into();
    let mut raw: Vec<Reverse<Logged>> = heap.into_vec(); // the heap no longer exists
    raw.push(Reverse(Logged(0))); // the raw Vec has no sift: the same mistake as h.Push(0)
    let broken: Vec<i32> = raw.iter().map(|r| r.0.0).collect();
    println!(
        "    Vec::push(0) on into_vec(): {}  is_min_heap = {}",
        go_slice(&broken),
        is_min_heap(&broken)
    );
    let mut repaired: MinHeap = BinaryHeap::from(raw); // heap.Init: Floyd's heapify
    println!(
        "    BinaryHeap::from(vec) repairs: {}  is_min_heap = {}",
        go_slice(&keys(&repaired)),
        is_min_heap(&keys(&repaired))
    );
    println!(
        "    and pop() now returns {:?}",
        repaired.pop().map(|r| r.0.0)
    );

    // (c) Comparison counts over 1,000 pushes and 1,000 pops of seeded random ints.
    let mut r = Pcg::new(1, 2);
    let mut k: MinHeap = BinaryHeap::new();
    CMPS.set(0);
    for _ in 0..1000 {
        k.push(Reverse(Logged(r.int_n(1000) as i32)));
    }
    let push_cmps = CMPS.get();
    CMPS.set(0);
    let mut drained = Vec::new();
    while let Some(Reverse(Logged(x))) = k.pop() {
        drained.push(x);
    }
    let pop_cmps = CMPS.get();
    println!("\n(c) 1,000 push, then 1,000 pop, of random ints (std comparisons, Ord::cmp calls):");
    println!("    1000 pushes: {push_cmps}   1000 pops: {pop_cmps}");
    println!(
        "    per push {:.2}, per pop {:.2}",
        push_cmps as f64 / 1000.0,
        pop_cmps as f64 / 1000.0
    );
    println!("    drain is sorted: {}", drained.is_sorted());
}
```

**Rust output:**

```
(a) built with push of 5, 3, 8, 1: [1 3 8 5]

    push(2) compares, in order (no hooks, no Len, no Swap -- a hole moves):
      1. cmp(3, 2) = Greater
      2. cmp(1, 2) = Less
    -> [1 2 8 5 3]

    pop() compares, in order (bottom-up: one cmp per level, then a short sift up):
      1. cmp(8, 2) = Greater
      2. cmp(5, 3) = Greater
      3. cmp(2, 3) = Less
    -> returned Some(1), array [2 3 8 5]

(b) the nearest misuse, starting from the valid heap [1 3 2 6 5 4]
    Vec::push(0) on into_vec(): [1 3 2 6 5 4 0]  is_min_heap = false
    BinaryHeap::from(vec) repairs: [0 3 1 6 5 4 2]  is_min_heap = true
    and pop() now returns Some(0)

(c) 1,000 push, then 1,000 pop, of random ints (std comparisons, Ord::cmp calls):
    1000 pushes: 2261   1000 pops: 8669
    per push 2.26, per pop 8.67
    drain is sorted: true
```

> **Go vs Rust:** There is no interface to implement and no hook to call by mistake, so (b) can only imitate the misuse by leaving the type: `into_vec()` gives up the heap, `Vec::push(0)` is the same mistake as `h.Push(0)`, and `BinaryHeap::from(vec)` is `heap.Init` (same repaired array as Go, `[0 3 1 6 5 4 2]`). The logs differ in shape: std has no `Len` or `Swap` calls, and `Reverse` swaps the operands, so the logged `cmp(3, 2)` is std testing `Reverse(2) <= Reverse(3)`. Push and pop make 2 and 3 comparisons here, the same counts as Go's `Less` log, but over 1,000 random operations pushes tie (2,261 each) while pops do not: 8,669 here against Go's 15,010, because std's `pop` sifts to the bottom first and back up (`sift_down_to_bottom`, read in the source, not run: example 15 counts it).

**Complexity:** `heap.Push` Θ(log n) · `heap.Pop` Θ(log n) · the hooks are Θ(1) amortized (an `append`, a reslice) · measured: one `Len` per `heap.Push` and per `heap.Pop`, one call to your `Push` per `heap.Push` and to your `Pop` per `heap.Pop`, and Less/Swap counts that match example 2 (about 2.3 `Less` per push and 15 per pop at n ≈ 1000)

---

## 4. Heapify is Θ(n), and n Pushes are not

`🟢 easy` · *Floyd's construction*

`heap.Init` and `BinaryHeap::from(vec)` build a heap by sifting down every internal node from the last to the first, in place: Θ(n) total, at most about `2n` comparisons, because half the nodes are leaves and a node at height `h` descends at most `h` levels. Building the same heap by `n` separate `Push`es is a different algorithm: Θ(n log n) in the worst case, though Θ(n) on random input, so heapify's real edge is the **guarantee**. Lesson 13 example 6 used heapify as phase one of heapsort; here it is the construction, measured per height, searched exhaustively for its worst case, and compared against `n` pushes on four input shapes.

**Steps:**

1. Heapify a random perfect heap of 1,023 keys and tally the levels descended by the nodes of each height.
2. Run heapify on every ordering of 1..10 distinct keys to find the fewest and most comparisons, and compare the most with a closed formula.
3. Compare heapify with `n` pushes at n = 2^10 … 2^20 on random, ascending, descending and all-equal input, and check that `container/heap` makes exactly the same comparisons.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// Heap is a hand-written min-heap whose sifts are written exactly like
// container/heap's up and down, counting every comparison.
type Heap struct {
	a    []int
	cmps int
}

func (h *Heap) less(i, j int) bool { h.cmps++; return h.a[i] < h.a[j] }

// down sinks the element at i0 within the first n slots and returns how many
// levels it descended (one swap per level).
func (h *Heap) down(i0, n int) int {
	i := i0
	for {
		j := 2*i + 1
		if j >= n {
			break
		}
		if j+1 < n && h.less(j+1, j) {
			j++
		}
		if !h.less(j, i) {
			break
		}
		h.a[i], h.a[j] = h.a[j], h.a[i]
		i = j
	}
	return levelOf(i) - levelOf(i0)
}

func levelOf(i int) int { return bits.Len(uint(i+1)) - 1 }

// heapify is Floyd's construction: sift down every internal node, last to first.
// The leaves (the second half) are already heaps of one element.
func (h *Heap) heapify() {
	n := len(h.a)
	for i := n/2 - 1; i >= 0; i-- {
		h.down(i, n)
	}
}

// push appends and sifts up (strict <, like container/heap's up).
func (h *Heap) push(x int) {
	h.a = append(h.a, x)
	for j := len(h.a) - 1; j > 0; {
		i := (j - 1) / 2
		if !h.less(j, i) {
			break
		}
		h.a[i], h.a[j] = h.a[j], h.a[i]
		j = i
	}
}

func permutation(r *rand.Rand, n int) []int {
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

// ci is the same heap behind container/heap's interface, with a counting Less.
type ci struct {
	a    []int
	cmps int
}

func (c *ci) Len() int           { return len(c.a) }
func (c *ci) Less(i, j int) bool { c.cmps++; return c.a[i] < c.a[j] }
func (c *ci) Swap(i, j int)      { c.a[i], c.a[j] = c.a[j], c.a[i] }
func (c *ci) Push(x any)         { c.a = append(c.a, x.(int)) }
func (c *ci) Pop() any           { n := len(c.a) - 1; x := c.a[n]; c.a = c.a[:n]; return x }

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	// (a) Where the work of heapify goes, on a perfect heap: n = 2^10 - 1.
	n := 1<<10 - 1
	perfect := &Heap{a: permutation(r, n)}
	byHeight := make([]int, 10) // levels descended, summed over the nodes of each height
	for i := n/2 - 1; i >= 0; i-- {
		byHeight[9-levelOf(i)] += perfect.down(i, n)
	}
	fmt.Printf("(a) heapify of a random permutation, n = %d (a perfect heap, 10 levels):\n", n)
	fmt.Println("    height  nodes  max levels each  max in total  measured")
	boundSum, measured := 0, 0
	for height := 0; height <= 9; height++ {
		nodes := 1 << (9 - height)
		fmt.Printf("    %6d  %5d  %15d  %12d  %8d\n", height, nodes, height, height*nodes, byHeight[height])
		boundSum += height * nodes
		measured += byHeight[height]
	}
	fmt.Printf("    total levels descended: bound %d = n - popcount(n) = %d, measured %d; comparisons %d (2n = %d)\n",
		boundSum, n-bits.OnesCount(uint(n)), measured, perfect.cmps, 2*n)

	// (b) Exhaustive: every ordering of n distinct keys, n = 1..10.
	fmt.Println("\n(b) heapify comparisons over ALL n! orderings, against 2n - 2*popcount(n) - trailingZeros(n):")
	fmt.Println("     n  orderings  fewest  n-1  most  formula  most == formula")
	allMatch := true
	for n := 1; n <= 10; n++ {
		lo, hi, count := 1<<30, 0, 0
		work := &Heap{a: make([]int, n)}
		var visit func(a []int, k int)
		visit = func(a []int, k int) {
			if k == n {
				copy(work.a, a)
				work.cmps = 0
				work.heapify()
				lo, hi, count = min(lo, work.cmps), max(hi, work.cmps), count+1
				return
			}
			for j := k; j < n; j++ {
				a[k], a[j] = a[j], a[k]
				visit(a, k+1)
				a[k], a[j] = a[j], a[k]
			}
		}
		a := make([]int, n)
		for i := range a {
			a[i] = i
		}
		visit(a, 0)
		formula := 2*n - 2*bits.OnesCount(uint(n)) - bits.TrailingZeros(uint(n))
		allMatch = allMatch && hi == formula
		fmt.Printf("  %4d  %9d  %6d  %3d  %4d  %7d  %v\n", n, count, lo, n-1, hi, formula, hi == formula)
	}
	fmt.Println("    max matches formula in every row:", allMatch)

	// (c) heapify against n pushes, four input shapes.
	fmt.Println("\n(c) comparisons per element, heapify against n pushes:")
	fmt.Printf("    %8s %-10s  %9s  %9s\n", "n", "shape", "heapify/n", "pushes/n")
	var keep map[string][]int // the n = 2^14 inputs, reused in (d)
	for _, n := range []int{1 << 10, 1 << 14, 1 << 18, 1 << 20} {
		asc, desc, eq := make([]int, n), make([]int, n), make([]int, n)
		for i := range asc {
			asc[i], desc[i] = i, n-1-i
		}
		shapes := []struct {
			name string
			in   []int
		}{{"random", permutation(r, n)}, {"ascending", asc}, {"descending", desc}, {"all-equal", eq}}
		if n == 1<<14 {
			keep = map[string][]int{}
		}
		for _, s := range shapes {
			hf := &Heap{a: slices.Clone(s.in)}
			hf.heapify()
			hp := &Heap{}
			for _, x := range s.in {
				hp.push(x)
			}
			fmt.Printf("    %8d %-10s  %9.2f  %9.2f\n", n, s.name, float64(hf.cmps)/float64(n), float64(hp.cmps)/float64(n))
			if n == 1<<14 {
				keep[s.name] = s.in
			}
		}
	}

	// (d) container/heap runs the same loops: its counts equal the hand-written ones.
	fmt.Println("\n(d) container/heap through a counting Less, n = 2^14:")
	for _, name := range []string{"random", "ascending", "descending", "all-equal"} {
		in := keep[name]
		own := &Heap{a: slices.Clone(in)}
		own.heapify()
		std := &ci{a: slices.Clone(in)}
		heap.Init(std)
		ownP, stdP := &Heap{}, &ci{}
		for _, x := range in {
			ownP.push(x)
			heap.Push(stdP, x)
		}
		fmt.Printf("    %-10s heap.Init == own heapify: %-5v (%d)   n x heap.Push == own pushes: %-5v (%d)\n",
			name, std.cmps == own.cmps && slices.Equal(std.a, own.a), std.cmps, stdP.cmps == ownP.cmps && slices.Equal(stdP.a, ownP.a), stdP.cmps)
	}
}
```

**Output:**

```
(a) heapify of a random permutation, n = 1023 (a perfect heap, 10 levels):
    height  nodes  max levels each  max in total  measured
         0    512                0             0         0
         1    256                1           256       173
         2    128                2           256       175
         3     64                3           192       139
         4     32                4           128        94
         5     16                5            80        64
         6      8                6            48        45
         7      4                7            28        22
         8      2                8            16        11
         9      1                9             9         8
    total levels descended: bound 1013 = n - popcount(n) = 1013, measured 731; comparisons 1882 (2n = 2046)

(b) heapify comparisons over ALL n! orderings, against 2n - 2*popcount(n) - trailingZeros(n):
     n  orderings  fewest  n-1  most  formula  most == formula
     1          1       0    0     0        0  true
     2          2       1    1     1        1  true
     3          6       2    2     2        2  true
     4         24       3    3     4        4  true
     5        120       4    4     6        6  true
     6        720       5    5     7        7  true
     7       5040       6    6     8        8  true
     8      40320       7    7    11       11  true
     9     362880       8    8    14       14  true
    10    3628800       9    9    15       15  true
    max matches formula in every row: true

(c) comparisons per element, heapify against n pushes:
           n shape       heapify/n   pushes/n
        1024 random           1.87       2.20
        1024 ascending        1.00       1.00
        1024 descending       1.98       8.01
        1024 all-equal        1.00       1.00
       16384 random           1.89       2.30
       16384 ascending        1.00       1.00
       16384 descending       2.00      12.00
       16384 all-equal        1.00       1.00
      262144 random           1.88       2.28
      262144 ascending        1.00       1.00
      262144 descending       2.00      16.00
      262144 all-equal        1.00       1.00
     1048576 random           1.88       2.28
     1048576 ascending        1.00       1.00
     1048576 descending       2.00      18.00
     1048576 all-equal        1.00       1.00

(d) container/heap through a counting Less, n = 2^14:
    random     heap.Init == own heapify: true  (30916)   n x heap.Push == own pushes: true  (37627)
    ascending  heap.Init == own heapify: true  (16383)   n x heap.Push == own pushes: true  (16383)
    descending heap.Init == own heapify: true  (32740)   n x heap.Push == own pushes: true  (196624)
    all-equal  heap.Init == own heapify: true  (16383)   n x heap.Push == own pushes: true  (16383)
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same permutations as Go.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
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
        assert!(n > 0, "invalid argument to int_n");
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

use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;

/// Heap is a hand-written min-heap whose sifts are written exactly like
/// container/heap's up and down, counting every comparison.
#[derive(Default)]
struct Heap {
    a: Vec<usize>,
    cmps: u64,
}

fn level_of(i: usize) -> usize {
    (i + 1).ilog2() as usize
}

impl Heap {
    fn less(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        self.a[i] < self.a[j]
    }

    /// down sinks the element at i0 within the first n slots and returns how
    /// many levels it descended (one swap per level).
    fn down(&mut self, i0: usize, n: usize) -> usize {
        let mut i = i0;
        loop {
            let mut j = 2 * i + 1;
            if j >= n {
                break;
            }
            if j + 1 < n && self.less(j + 1, j) {
                j += 1;
            }
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            i = j;
        }
        level_of(i) - level_of(i0)
    }

    /// heapify is Floyd's construction: sift down every internal node, last to
    /// first. The leaves (the second half) are already heaps of one element.
    fn heapify(&mut self) {
        let n = self.a.len();
        for i in (0..n / 2).rev() {
            self.down(i, n);
        }
    }

    /// push appends and sifts up (strict <, like container/heap's up).
    fn push(&mut self, x: usize) {
        self.a.push(x);
        let mut j = self.a.len() - 1;
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            j = i;
        }
    }
}

fn permutation(r: &mut Pcg, n: usize) -> Vec<usize> {
    let mut p: Vec<usize> = (0..n).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

// std's heap compares through Ord, so to count its comparisons we count inside
// `cmp`. Reverse makes the max-heap a min-heap.
thread_local! {
    static CMPS: Cell<u64> = const { Cell::new(0) };
}

#[derive(PartialEq, Eq)]
struct Counted(usize);

impl Ord for Counted {
    fn cmp(&self, other: &Self) -> Ordering {
        CMPS.set(CMPS.get() + 1);
        self.0.cmp(&other.0)
    }
}

// std's `<`, `<=`, `>=` all call partial_cmp once, which delegates to cmp.
impl PartialOrd for Counted {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

type StdMin = BinaryHeap<Reverse<Counted>>;

fn std_keys(h: &StdMin) -> Vec<usize> {
    h.as_slice().iter().map(|r| r.0.0).collect()
}

/// BinaryHeap::from(vec) is std's heapify. Returns (comparisons, resulting array).
fn std_heapify(input: &[usize]) -> (u64, Vec<usize>) {
    let v: Vec<Reverse<Counted>> = input.iter().map(|&x| Reverse(Counted(x))).collect();
    CMPS.set(0);
    let h = StdMin::from(v);
    (CMPS.get(), std_keys(&h))
}

/// n calls of push into an empty BinaryHeap.
fn std_pushes(input: &[usize]) -> (u64, Vec<usize>) {
    let mut h = StdMin::with_capacity(input.len());
    CMPS.set(0);
    for &x in input {
        h.push(Reverse(Counted(x)));
    }
    (CMPS.get(), std_keys(&h))
}

fn main() {
    let mut r = Pcg::new(1, 2);

    // (a) Where the work of heapify goes, on a perfect heap: n = 2^10 - 1.
    let n = (1usize << 10) - 1;
    let mut perfect = Heap {
        a: permutation(&mut r, n),
        cmps: 0,
    };
    let mut by_height = [0usize; 10]; // levels descended, summed over the nodes of each height
    for i in (0..n / 2).rev() {
        by_height[9 - level_of(i)] += perfect.down(i, n);
    }
    println!("(a) heapify of a random permutation, n = {n} (a perfect heap, 10 levels):");
    println!("    height  nodes  max levels each  max in total  measured");
    let (mut bound_sum, mut measured) = (0, 0);
    for (height, &levels) in by_height.iter().enumerate() {
        let nodes = 1usize << (9 - height);
        println!(
            "    {height:>6}  {nodes:>5}  {height:>15}  {:>12}  {levels:>8}",
            height * nodes
        );
        bound_sum += height * nodes;
        measured += levels;
    }
    println!(
        "    total levels descended: bound {bound_sum} = n - popcount(n) = {}, measured {measured}; comparisons {} (2n = {})",
        n - n.count_ones() as usize,
        perfect.cmps,
        2 * n
    );

    // (b) Exhaustive: every ordering of n distinct keys, n = 1..10.
    println!(
        "\n(b) heapify comparisons over ALL n! orderings, against 2n - 2*popcount(n) - trailingZeros(n):"
    );
    println!("     n  orderings  fewest  n-1  most  formula  most == formula");
    let mut all_match = true;
    for n in 1..=10usize {
        let mut stats = (u64::MAX, 0u64, 0u64); // fewest, most, orderings
        let mut work = Heap {
            a: vec![0; n],
            cmps: 0,
        };
        let mut a: Vec<usize> = (0..n).collect();
        visit(&mut a, 0, &mut work, &mut stats);
        let (lo, hi, count) = stats;
        let formula = (2 * n - 2 * n.count_ones() as usize - n.trailing_zeros() as usize) as u64;
        all_match &= hi == formula;
        println!(
            "  {n:>4}  {count:>9}  {lo:>6}  {:>3}  {hi:>4}  {formula:>7}  {}",
            n - 1,
            hi == formula
        );
    }
    println!("    max matches formula in every row: {all_match}");

    // (c) heapify against n pushes, four input shapes.
    println!("\n(c) comparisons per element, heapify against n pushes:");
    println!(
        "    {:>8} {:<10}  {:>9}  {:>9}",
        "n", "shape", "heapify/n", "pushes/n"
    );
    let mut keep: Vec<(&str, Vec<usize>)> = Vec::new(); // the n = 2^14 inputs, reused in (d)
    for n in [1usize << 10, 1 << 14, 1 << 18, 1 << 20] {
        let shapes = [
            ("random", permutation(&mut r, n)),
            ("ascending", (0..n).collect()),
            ("descending", (0..n).rev().collect()),
            ("all-equal", vec![0; n]),
        ];
        for (name, input) in shapes {
            let mut hf = Heap {
                a: input.clone(),
                cmps: 0,
            };
            hf.heapify();
            let mut hp = Heap::default();
            for &x in &input {
                hp.push(x);
            }
            println!(
                "    {n:>8} {name:<10}  {:>9.2}  {:>9.2}",
                hf.cmps as f64 / n as f64,
                hp.cmps as f64 / n as f64
            );
            if n == 1 << 14 {
                keep.push((name, input));
            }
        }
    }

    // (d) std's BinaryHeap, through a counting Ord: is it the same algorithm?
    println!(
        "\n(d) std BinaryHeap<Reverse<_>> through a counting Ord, n = 2^14, comparisons as (std vs own):"
    );
    for (name, input) in &keep {
        let mut own = Heap {
            a: input.clone(),
            cmps: 0,
        };
        own.heapify();
        let (sc, sa) = std_heapify(input);
        let mut own_p = Heap::default();
        for &x in input {
            own_p.push(x);
        }
        let (pc, pa) = std_pushes(input);
        println!(
            "    {name:<10} from(vec) == own heapify: {:<5} ({sc} vs {})   n x push == own pushes: {:<5} ({pc} vs {})",
            sc == own.cmps && sa == own.a,
            own.cmps,
            pc == own_p.cmps && pa == own_p.a,
            own_p.cmps
        );
    }
}

/// visit tries every ordering of a (recursive swap) and records the fewest and
/// most heapify comparisons and the number of orderings.
fn visit(a: &mut [usize], k: usize, work: &mut Heap, stats: &mut (u64, u64, u64)) {
    if k == a.len() {
        work.a.copy_from_slice(a);
        work.cmps = 0;
        work.heapify();
        *stats = (stats.0.min(work.cmps), stats.1.max(work.cmps), stats.2 + 1);
        return;
    }
    for j in k..a.len() {
        a.swap(k, j);
        visit(a, k + 1, work, stats);
        a.swap(k, j);
    }
}
```

**Rust output:**

```
(a) heapify of a random permutation, n = 1023 (a perfect heap, 10 levels):
    height  nodes  max levels each  max in total  measured
         0    512                0             0         0
         1    256                1           256       173
         2    128                2           256       175
         3     64                3           192       139
         4     32                4           128        94
         5     16                5            80        64
         6      8                6            48        45
         7      4                7            28        22
         8      2                8            16        11
         9      1                9             9         8
    total levels descended: bound 1013 = n - popcount(n) = 1013, measured 731; comparisons 1882 (2n = 2046)

(b) heapify comparisons over ALL n! orderings, against 2n - 2*popcount(n) - trailingZeros(n):
     n  orderings  fewest  n-1  most  formula  most == formula
     1          1       0    0     0        0  true
     2          2       1    1     1        1  true
     3          6       2    2     2        2  true
     4         24       3    3     4        4  true
     5        120       4    4     6        6  true
     6        720       5    5     7        7  true
     7       5040       6    6     8        8  true
     8      40320       7    7    11       11  true
     9     362880       8    8    14       14  true
    10    3628800       9    9    15       15  true
    max matches formula in every row: true

(c) comparisons per element, heapify against n pushes:
           n shape       heapify/n   pushes/n
        1024 random           1.87       2.20
        1024 ascending        1.00       1.00
        1024 descending       1.98       8.01
        1024 all-equal        1.00       1.00
       16384 random           1.89       2.30
       16384 ascending        1.00       1.00
       16384 descending       2.00      12.00
       16384 all-equal        1.00       1.00
      262144 random           1.88       2.28
      262144 ascending        1.00       1.00
      262144 descending       2.00      16.00
      262144 all-equal        1.00       1.00
     1048576 random           1.88       2.28
     1048576 ascending        1.00       1.00
     1048576 descending       2.00      18.00
     1048576 all-equal        1.00       1.00

(d) std BinaryHeap<Reverse<_>> through a counting Ord, n = 2^14, comparisons as (std vs own):
    random     from(vec) == own heapify: true  (30916 vs 30916)   n x push == own pushes: true  (37627 vs 37627)
    ascending  from(vec) == own heapify: true  (16383 vs 16383)   n x push == own pushes: true  (16383 vs 16383)
    descending from(vec) == own heapify: true  (32740 vs 32740)   n x push == own pushes: true  (196624 vs 196624)
    all-equal  from(vec) == own heapify: true  (16383 vs 16383)   n x push == own pushes: true  (16383 vs 16383)
```

> **Go vs Rust:** Parts (a) to (c) are identical to Go line for line (same PCG stream, same loops); only (d) is rewritten for std. `BinaryHeap::from(vec)` is `rebuild()` (read in the source, not run), and through a counting `Ord` under `Reverse` both it and `n` calls of `push` make exactly the own comparison counts and build the same array, in all four shapes (30,916 and 37,627 on random input) — the very numbers Go's `heap.Init` and `heap.Push` made, so for heapify and push the two standard libraries do the same work even though std moves a hole and Go swaps. Rust's `pop` is where they diverge (example 3, example 15).

**Complexity:** heapify Θ(n): the levels descended total at most `n - popcount(n)` (731 of 1,013 on the random n = 1,023 heap), and the comparisons range from `n - 1` (fewest, every ordering up to n = 10) to `2n - 2*popcount(n) - trailingZeros(n)` (most, equal to the formula for every n up to 10); about 1.9n on random input, 1.0n on sorted or all-equal input, 2.0n on descending · `n` pushes Θ(n log n) worst: 18 comparisons per element on descending input at n = 2^20, but only 2.2–2.3 on random input and 1.0 on ascending · both in place

---

## 5. Min, max, generic, and ties

`🟢 easy` · *Less direction, generics, stability*

Min versus max is one character in the comparison, and a generic `PQ[T]` over `heap.Interface` takes the ordering as a function, so the five methods are written once for every element type. What neither language's heap gives you is **stability**: equal priorities come out in whatever order the sifts left them. Adding a sequence number to the comparison, `(priority, seq)`, makes the order a strict total order, so ties cannot exist and first-in-first-out falls out; the cost is one integer per element, and every operation stays Θ(log n).

**Steps:**

1. Write `PQ[T]` with a `less` function, and drain the same five numbers as a min-heap and as a max-heap.
2. Push 12 tasks with only three distinct priorities and count the **FIFO violations** (an equal-priority task popping before an earlier arrival), by priority alone and by `(priority, seq)`.
3. Repeat on 1,000 batches of 40 tasks and check the `(priority, seq)` heap against sorting by the same key.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// PQ[T] is one heap for every element type. It implements heap.Interface with
// POINTER receivers (Push and Pop change the slice's length), and the whole
// ordering is the function you pass in: `<` makes a min-heap, `>` a max-heap.
type PQ[T any] struct {
	items []T
	less  func(a, b T) bool
}

func NewPQ[T any](less func(a, b T) bool) *PQ[T] { return &PQ[T]{less: less} }

func (q *PQ[T]) Len() int           { return len(q.items) }
func (q *PQ[T]) Less(i, j int) bool { return q.less(q.items[i], q.items[j]) }
func (q *PQ[T]) Swap(i, j int)      { q.items[i], q.items[j] = q.items[j], q.items[i] }
func (q *PQ[T]) Push(x any)         { q.items = append(q.items, x.(T)) } // hook: only appends
func (q *PQ[T]) Pop() any { // hook: removes the LAST element
	n := len(q.items) - 1
	x := q.items[n]
	var zero T
	q.items[n] = zero // let the GC reclaim a pointer element
	q.items = q.items[:n]
	return x
}

// Add and Next hide the package calls and the `any` conversions.
func (q *PQ[T]) Add(x T)     { heap.Push(q, x) }
func (q *PQ[T]) Next() T     { return heap.Pop(q).(T) }
func (q *PQ[T]) Empty() bool { return len(q.items) == 0 }

// task is a unit of work: lower pri comes out first, seq is its arrival number.
type task struct{ pri, seq int }

func (t task) String() string { return fmt.Sprintf("%d:%d", t.pri, t.seq) }

// drain pops everything.
func drain[T any](q *PQ[T]) []T {
	var out []T
	for !q.Empty() {
		out = append(out, q.Next())
	}
	return out
}

// fifoViolations counts consecutive pops with equal priority whose seq goes DOWN:
// a later arrival came out before an earlier one of the same priority.
func fifoViolations(order []task) int {
	v := 0
	for i := 1; i < len(order); i++ {
		if order[i].pri == order[i-1].pri && order[i].seq < order[i-1].seq {
			v++
		}
	}
	return v
}

func render(ts []task) string {
	parts := make([]string, len(ts))
	for i, t := range ts {
		parts[i] = t.String()
	}
	return strings.Join(parts, " ")
}

func main() {
	// (a) Min vs max is one character in the comparison.
	minPQ := NewPQ(func(a, b int) bool { return a < b })
	maxPQ := NewPQ(func(a, b int) bool { return a > b })
	for _, x := range []int{5, 1, 4, 2, 3} {
		minPQ.Add(x)
		maxPQ.Add(x)
	}
	fmt.Println("(a) pushed 5 1 4 2 3")
	fmt.Println("    min-heap (a < b):", drain(minPQ))
	fmt.Println("    max-heap (a > b):", drain(maxPQ))

	// (b) Ties. Compare priorities only, then (priority, sequence number).
	byPri := func(a, b task) bool { return a.pri < b.pri }
	byPriSeq := func(a, b task) bool { return a.pri < b.pri || a.pri == b.pri && a.seq < b.seq }

	r := rand.New(rand.NewPCG(1, 2))
	tasks := make([]task, 12)
	for i := range tasks {
		tasks[i] = task{r.IntN(3), i}
	}
	fmt.Println("\n(b) 12 tasks (pri:seq), arrival order:", render(tasks))
	for _, c := range []struct {
		name string
		less func(a, b task) bool
	}{{"priority only", byPri}, {"(priority, seq)", byPriSeq}} {
		q := NewPQ(c.less)
		for _, t := range tasks {
			q.Add(t)
		}
		out := drain(q)
		fmt.Printf("    %-16s pops: %s   FIFO violations: %d\n", c.name, render(out), fifoViolations(out))
	}

	const batches, size = 1000, 40
	before, after, same := 0, 0, true
	for range batches {
		ts := make([]task, size)
		for i := range ts {
			ts[i] = task{r.IntN(3), i}
		}
		qb, qa := NewPQ(byPri), NewPQ(byPriSeq)
		for _, t := range ts {
			qb.Add(t)
			qa.Add(t)
		}
		before += fifoViolations(drain(qb))
		outA := drain(qa)
		after += fifoViolations(outA)
		slices.SortFunc(ts, func(a, b task) int { // the one correct answer: sort by (pri, seq)
			if a.pri != b.pri {
				return a.pri - b.pri
			}
			return a.seq - b.seq
		})
		same = same && slices.Equal(outA, ts)
	}
	fmt.Printf("\n    %d batches of %d tasks, priorities 0..2:\n", batches, size)
	fmt.Printf("    FIFO violations, priority only:   %d (%.1f per batch)\n", before, float64(before)/batches)
	fmt.Printf("    FIFO violations, (priority, seq): %d\n", after)
	fmt.Println("    (priority, seq) pops exactly the sorted order in every batch:", same)
}
```

**Output:**

```
(a) pushed 5 1 4 2 3
    min-heap (a < b): [1 2 3 4 5]
    max-heap (a > b): [5 4 3 2 1]

(b) 12 tasks (pri:seq), arrival order: 2:0 1:1 2:2 2:3 0:4 0:5 1:6 1:7 0:8 0:9 1:10 1:11
    priority only    pops: 0:4 0:8 0:9 0:5 1:1 1:7 1:6 1:10 1:11 2:2 2:3 2:0   FIFO violations: 3
    (priority, seq)  pops: 0:4 0:5 0:8 0:9 1:1 1:6 1:7 1:10 1:11 2:0 2:2 2:3   FIFO violations: 0

    1000 batches of 40 tasks, priorities 0..2:
    FIFO violations, priority only:   14555 (14.6 per batch)
    FIFO violations, (priority, seq): 0
    (priority, seq) pops exactly the sorted order in every batch: true
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same permutations as Go.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
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
        assert!(n > 0, "invalid argument to int_n");
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

use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;
use std::fmt::Display;

// Rust's BinaryHeap<T: Ord> is a MAX-heap, and the ordering lives in the
// ELEMENT TYPE (Ord), not in the container: there is no `less` function to
// pass. A min-heap is Reverse(x), or an Ord that is written backwards. To heap
// by one field, wrap the item in a type with its own Ord.

/// task is a unit of work: lower pri comes out first, seq is its arrival number.
#[derive(Clone, Copy, PartialEq, Eq)]
struct Task {
    pri: u64,
    seq: u64,
}

impl Display for Task {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}:{}", self.pri, self.seq)
    }
}

/// ByPri orders by priority ONLY: ties compare Equal (and so must `==`, which
/// is why PartialEq is written by hand). Reversed, so the smallest pri is the "max".
struct ByPri(Task);

impl PartialEq for ByPri {
    fn eq(&self, other: &Self) -> bool {
        self.0.pri == other.0.pri
    }
}
impl Eq for ByPri {}
impl Ord for ByPri {
    fn cmp(&self, other: &Self) -> Ordering {
        other.0.pri.cmp(&self.0.pri)
    }
}
impl PartialOrd for ByPri {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// ByPriSeq orders by (pri, seq). Under a max-heap, first-in-first-out means the
/// SMALLER seq must compare greater, so both fields are compared reversed.
#[derive(PartialEq, Eq)]
struct ByPriSeq(Task);

impl Ord for ByPriSeq {
    fn cmp(&self, other: &Self) -> Ordering {
        other
            .0
            .pri
            .cmp(&self.0.pri)
            .then(other.0.seq.cmp(&self.0.seq))
    }
}
impl PartialOrd for ByPriSeq {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// drain pops everything.
fn drain<T: Ord>(mut h: BinaryHeap<T>) -> Vec<T> {
    let mut out = Vec::with_capacity(h.len());
    while let Some(x) = h.pop() {
        out.push(x);
    }
    out
}

/// fifo_violations counts consecutive pops with equal priority whose seq goes DOWN:
/// a later arrival came out before an earlier one of the same priority.
fn fifo_violations(order: &[Task]) -> usize {
    order
        .windows(2)
        .filter(|w| w[1].pri == w[0].pri && w[1].seq < w[0].seq)
        .count()
}

fn render<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(T::to_string).collect();
    parts.join(" ")
}

fn main() {
    // (a) Min vs max is Reverse or not.
    let items = [5, 1, 4, 2, 3];
    let min_heap: BinaryHeap<Reverse<i32>> = items.iter().map(|&x| Reverse(x)).collect();
    let max_heap: BinaryHeap<i32> = items.iter().copied().collect();
    println!("(a) pushed 5 1 4 2 3");
    let mins: Vec<i32> = drain(min_heap).into_iter().map(|Reverse(x)| x).collect();
    println!("    min-heap (Reverse): [{}]", render(&mins));
    println!("    max-heap (plain):   [{}]", render(&drain(max_heap)));

    // (b) Ties. Compare priorities only, then (priority, sequence number).
    let mut r = Pcg::new(1, 2);
    let tasks: Vec<Task> = (0..12)
        .map(|seq| Task {
            pri: r.int_n(3),
            seq,
        })
        .collect();
    println!(
        "\n(b) 12 tasks (pri:seq), arrival order: {}",
        render(&tasks)
    );
    let by_pri: BinaryHeap<ByPri> = tasks.iter().map(|&t| ByPri(t)).collect();
    let out: Vec<Task> = drain(by_pri).into_iter().map(|b| b.0).collect();
    println!(
        "    {:<16} pops: {}   FIFO violations: {}",
        "priority only",
        render(&out),
        fifo_violations(&out)
    );
    let by_pri_seq: BinaryHeap<ByPriSeq> = tasks.iter().map(|&t| ByPriSeq(t)).collect();
    let out: Vec<Task> = drain(by_pri_seq).into_iter().map(|b| b.0).collect();
    println!(
        "    {:<16} pops: {}   FIFO violations: {}",
        "(priority, seq)",
        render(&out),
        fifo_violations(&out)
    );

    const BATCHES: usize = 1000;
    const SIZE: u64 = 40;
    let (mut before, mut after, mut same) = (0, 0, true);
    for _ in 0..BATCHES {
        let mut ts: Vec<Task> = (0..SIZE)
            .map(|seq| Task {
                pri: r.int_n(3),
                seq,
            })
            .collect();
        // push one at a time, like the Go program's Add (from(vec) would heapify instead)
        let mut qb = BinaryHeap::new();
        let mut qa = BinaryHeap::new();
        for &t in &ts {
            qb.push(ByPri(t));
            qa.push(ByPriSeq(t));
        }
        let out_b: Vec<Task> = drain(qb).into_iter().map(|b| b.0).collect();
        before += fifo_violations(&out_b);
        let out_a: Vec<Task> = drain(qa).into_iter().map(|b| b.0).collect();
        after += fifo_violations(&out_a);
        ts.sort_by_key(|t| (t.pri, t.seq)); // the one correct answer: sort by (pri, seq)
        same &= out_a == ts;
    }
    println!("\n    {BATCHES} batches of {SIZE} tasks, priorities 0..2:");
    println!(
        "    FIFO violations, priority only:   {before} ({:.1} per batch)",
        before as f64 / BATCHES as f64
    );
    println!("    FIFO violations, (priority, seq): {after}");
    println!("    (priority, seq) pops exactly the sorted order in every batch: {same}");
}
```

**Rust output:**

```
(a) pushed 5 1 4 2 3
    min-heap (Reverse): [1 2 3 4 5]
    max-heap (plain):   [5 4 3 2 1]

(b) 12 tasks (pri:seq), arrival order: 2:0 1:1 2:2 2:3 0:4 0:5 1:6 1:7 0:8 0:9 1:10 1:11
    priority only    pops: 0:5 0:4 0:9 0:8 1:6 1:11 1:10 1:1 1:7 2:0 2:3 2:2   FIFO violations: 5
    (priority, seq)  pops: 0:4 0:5 0:8 0:9 1:1 1:6 1:7 1:10 1:11 2:0 2:2 2:3   FIFO violations: 0

    1000 batches of 40 tasks, priorities 0..2:
    FIFO violations, priority only:   16531 (16.5 per batch)
    FIFO violations, (priority, seq): 0
    (priority, seq) pops exactly the sorted order in every batch: true
```

> **Go vs Rust:** The `(priority, seq)` result is identical to Go's (same tasks, 0 violations, exactly the sorted order in all 1,000 batches), because with no ties every correct heap pops the same sequence; the priority-only result is not: 5 violations on the first 12 tasks (Go: 3) and 16,531 over the batches (Go: 14,555). Both heaps are unstable, and the exact count depends on how each one's sifts break ties (Rust's sift-up stops on `<=`, its sift-down takes the right child on a tie and `pop` sifts bottom-up; Go's `up`/`down` use a strict `<` and take the left child; read in the source, not run). The ordering lives in the element type: `Reverse` or a reversed `Ord` for a min-heap, a hand-written `PartialEq` for `ByPri` so that `==` agrees with `cmp`, and `other.seq.cmp(&self.seq)` for FIFO under a max-heap.

**Complexity:** unchanged Θ(log n) per operation · ordering by priority alone is **not** FIFO: 14,555 violations over 1,000 batches of 40 tasks (about 14.6 per batch), while `(priority, seq)` gave 0 and popped exactly the sorted order every time · the sequence number costs one int per element and makes the order a strict total order, so every correct heap pops the same sequence

---

## 6. Floats, `NaN` and the `Less` contract

`🟢 easy` · *Strict weak order*

A heap trusts `Less`: it never checks that the comparison is a consistent order. On floats `<` is not one once a `NaN` is present (every comparison with `NaN` is false, so it is "equal" to everything while `1 < 2` still holds), so the heap does not crash, it misorders silently. Go's fix is `cmp.Less`, which defines a total order (every `NaN` first, `-0 == +0`); the other fix is to map each float to an unsigned integer that sorts the same way (lesson 14 example 8) and heap the integers, which costs one bit trick per key and no `NaN` test at all. This example measures how often `<` fails and what each fix does to the order; cost per operation stays Θ(log n).

**Steps:**

1. Write a heap whose `Less` is a function, and drain one array containing two `NaN`s with `x < y` and with `cmp.Less`.
2. Draw 20,000 random 10-element arrays with one `NaN` each and count the drains that are not sorted under each rule, plus a control with no `NaN`; then look at `-0` and `+0`.
3. Drain through an `IntHeap` of ordered `uint64` keys and compare where the `NaN`s and the zeros land.

```go
package main

import (
	"cmp"
	"container/heap"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
)

// A heap trusts its Less: it never checks that Less is a consistent order. For
// floats, `<` is not a strict weak order once a NaN is present -- every
// comparison with NaN is false, so NaN is "equal" to everything while 1 < 2 --
// so the heap does not crash, it quietly misorders.
type fheap struct {
	a    []float64
	less func(x, y float64) bool
}

func (h *fheap) Len() int           { return len(h.a) }
func (h *fheap) Less(i, j int) bool { return h.less(h.a[i], h.a[j]) }
func (h *fheap) Swap(i, j int)      { h.a[i], h.a[j] = h.a[j], h.a[i] }
func (h *fheap) Push(x any)         { h.a = append(h.a, x.(float64)) }
func (h *fheap) Pop() any {
	n := len(h.a) - 1
	x := h.a[n]
	h.a = h.a[:n]
	return x
}

// drain pushes the input in order, then pops everything.
func drain(less func(x, y float64) bool, in []float64) []float64 {
	h := &fheap{less: less}
	for _, x := range in {
		heap.Push(h, x)
	}
	out := make([]float64, 0, len(in))
	for h.Len() > 0 {
		out = append(out, heap.Pop(h).(float64))
	}
	return out
}

func lessLT(x, y float64) bool { return x < y }

// key maps a float64 to a uint64 with the same order as the IEEE total order
// (lesson 14 example 8): flip every bit of a negative, only the sign bit of a
// non-negative. Integers compare correctly, so no NaN test is ever needed.
func key(x float64) uint64 {
	b := math.Float64bits(x)
	if b>>63 == 1 {
		return ^b
	}
	return b | 1<<63
}

func unkey(k uint64) float64 {
	if k>>63 == 1 {
		return math.Float64frombits(k &^ (1 << 63))
	}
	return math.Float64frombits(^k)
}

type keyHeap []uint64

func (h keyHeap) Len() int           { return len(h) }
func (h keyHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h keyHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *keyHeap) Push(x any)        { *h = append(*h, x.(uint64)) }
func (h *keyHeap) Pop() any {
	old := *h
	n := len(old) - 1
	x := old[n]
	*h = old[:n]
	return x
}

func drainKeys(in []float64) []float64 {
	h := &keyHeap{}
	for _, x := range in {
		heap.Push(h, key(x))
	}
	var out []float64
	for h.Len() > 0 {
		out = append(out, unkey(heap.Pop(h).(uint64)))
	}
	return out
}

// show prints a float so that -0, the two NaNs and the infinities are visible.
func show(x float64) string {
	switch {
	case math.IsNaN(x) && math.Signbit(x):
		return "-NaN"
	case math.IsNaN(x):
		return "NaN"
	case math.IsInf(x, 1):
		return "+Inf"
	case math.IsInf(x, -1):
		return "-Inf"
	}
	return fmt.Sprint(x)
}

func showAll(xs []float64) string {
	parts := make([]string, len(xs))
	for i, x := range xs {
		parts[i] = show(x)
	}
	return "[" + strings.Join(parts, " ") + "]"
}

func main() {
	nan := math.NaN()
	negNaN := math.Float64frombits(math.Float64bits(nan) | 1<<63)
	negZero := math.Copysign(0, -1)
	sorted := func(xs []float64) bool { return slices.IsSortedFunc(xs, cmp.Compare[float64]) }

	// (a) The same seven numbers, through a min-heap with two different Less.
	a := []float64{3, nan, 1, 2, nan, 0, -1}
	fmt.Println("(a) drain of", showAll(a), "through a min-heap:")
	fmt.Println("    Less is x < y:     ", showAll(drain(lessLT, a)))
	fmt.Println("    Less is cmp.Less:  ", showAll(drain(cmp.Less[float64], a)))

	// (b) How often? 20,000 random arrays of 10 integer-valued floats, one slot a NaN.
	r := rand.New(rand.NewPCG(1, 2))
	var badLT, badLTrest, badCmp, badClean int
	const cases = 20000
	for range cases {
		clean := make([]float64, 10)
		for i := range clean {
			clean[i] = float64(r.IntN(100))
		}
		dirty := slices.Clone(clean)
		dirty[r.IntN(10)] = nan
		out := drain(lessLT, dirty)
		if !sorted(out) {
			badLT++
			rest := slices.DeleteFunc(out, math.IsNaN) // do the numbers themselves still come out in order?
			if !sorted(rest) {
				badLTrest++
			}
		}
		if !sorted(drain(cmp.Less[float64], dirty)) {
			badCmp++
		}
		if !sorted(drain(lessLT, clean)) {
			badClean++
		}
	}
	fmt.Printf("\n(b) %d arrays of 10 integer-valued floats, one slot replaced by NaN; drains not sorted (cmp.Compare):\n", cases)
	fmt.Printf("    Less is x < y, with a NaN:    %5d  (%.1f%%; in %d of them the plain numbers are themselves out of order)\n",
		badLT, 100*float64(badLT)/cases, badLTrest)
	fmt.Printf("    Less is cmp.Less, with a NaN: %5d\n", badCmp)
	fmt.Printf("    Less is x < y, no NaN at all: %5d\n", badClean)

	// (c) Signed zeros.
	z := []float64{0, negZero, 0, negZero}
	fmt.Println("\n(c) drain of", showAll(z), ":")
	fmt.Println("    Less is x < y:      ", showAll(drain(lessLT, z)), "(-0 < 0 is false: all four are 'equal')")
	fmt.Println("    Less is cmp.Less:   ", showAll(drain(cmp.Less[float64], z)), "(cmp.Compare says -0 == +0)")
	fmt.Println("    ordered uint64 key: ", showAll(drainKeys(z)))

	// (d) The ordered-integer key: no NaN test, and a different NaN placement.
	fmt.Println("\n(d) the same drains through an IntHeap of ordered uint64 keys:")
	fmt.Println("    ", showAll(a), "->", showAll(drainKeys(a)))
	b := []float64{3, negNaN, 0, negZero, math.Inf(1), nan, -1, math.Inf(-1)}
	fmt.Println("    ", showAll(b), "->", showAll(drainKeys(b)))
	fmt.Println("     cmp.Less on the second array:", showAll(drain(cmp.Less[float64], b)))
	fmt.Println("     cmp.Compare puts EVERY NaN first and -0 == +0; the bit-flip order puts -NaN first,")
	fmt.Println("     +NaN last, and -0 before +0: it is the IEEE total order, a different (finer) rule.")
}
```

**Output:**

```
(a) drain of [3 NaN 1 2 NaN 0 -1] through a min-heap:
    Less is x < y:      [-1 1 3 NaN 2 0 NaN]
    Less is cmp.Less:   [NaN NaN -1 0 1 2 3]

(b) 20000 arrays of 10 integer-valued floats, one slot replaced by NaN; drains not sorted (cmp.Compare):
    Less is x < y, with a NaN:    18015  (90.1%; in 5652 of them the plain numbers are themselves out of order)
    Less is cmp.Less, with a NaN:     0
    Less is x < y, no NaN at all:     0

(c) drain of [0 -0 0 -0] :
    Less is x < y:       [0 -0 0 -0] (-0 < 0 is false: all four are 'equal')
    Less is cmp.Less:    [0 -0 0 -0] (cmp.Compare says -0 == +0)
    ordered uint64 key:  [-0 -0 0 0]

(d) the same drains through an IntHeap of ordered uint64 keys:
     [3 NaN 1 2 NaN 0 -1] -> [-1 0 1 2 3 NaN NaN]
     [3 -NaN 0 -0 +Inf NaN -1 -Inf] -> [-NaN -Inf -1 -0 0 3 +Inf NaN]
     cmp.Less on the second array: [-NaN NaN -Inf -1 -0 0 3 +Inf]
     cmp.Compare puts EVERY NaN first and -0 == +0; the bit-flip order puts -NaN first,
     +NaN last, and -0 before +0: it is the IEEE total order, a different (finer) rule.
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same permutations as Go.
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
        // state = state * MUL + INC, with explicit 128-bit wraparound
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
        assert!(n > 0, "invalid argument to int_n");
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

use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;

// `BinaryHeap<f64>` does not compile: `push` needs `T: Ord`, and f64 is only
// PartialOrd. The compiler error is E0277, "the trait bound `f64: Ord` is not
// satisfied" (a one-line program, compiled separately: it cannot be in a
// program that runs). So Rust stops the ACCIDENT; the options are a wrapper
// with `total_cmp`, or an ordered integer key. And a wrapper whose `Ord` lies
// is still allowed -- `Lie` below -- which shows what Go's `x < y` is.

/// A float ordered by `f64::total_cmp`: a real total order (-NaN < -Inf < ... < -0 < +0 < ... < +Inf < +NaN).
struct F(f64);

impl PartialEq for F {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == Ordering::Equal
    }
}
impl Eq for F {}
impl Ord for F {
    fn cmp(&self, other: &Self) -> Ordering {
        self.0.total_cmp(&other.0)
    }
}
impl PartialOrd for F {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// Lie is the order Go's `Less(x, y) = x < y` defines, written as an Ord: Less
/// if x < y, Greater if x > y, otherwise "Equal" -- which includes every
/// comparison with NaN, so NaN is Equal to everything and Equal is not
/// transitive. The compiler accepts it. The BinaryHeap docs call a broken Ord
/// a logic error: the result is unspecified but safe (no UB, no panic).
struct Lie(f64);

impl PartialEq for Lie {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == Ordering::Equal
    }
}
impl Eq for Lie {}
impl Ord for Lie {
    fn cmp(&self, other: &Self) -> Ordering {
        if self.0 < other.0 {
            Ordering::Less
        } else if self.0 > other.0 {
            Ordering::Greater
        } else {
            Ordering::Equal
        }
    }
}
impl PartialOrd for Lie {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// Wrapper lets one generic drain run over both orders.
trait Wrapper: Ord {
    fn new(x: f64) -> Self;
    fn get(&self) -> f64;
}
impl Wrapper for F {
    fn new(x: f64) -> Self {
        F(x)
    }
    fn get(&self) -> f64 {
        self.0
    }
}
impl Wrapper for Lie {
    fn new(x: f64) -> Self {
        Lie(x)
    }
    fn get(&self) -> f64 {
        self.0
    }
}

/// drain pushes the input in order into a MIN-heap (Reverse), then pops everything.
fn drain<W: Wrapper>(input: &[f64]) -> Vec<f64> {
    let mut h = BinaryHeap::new();
    for &x in input {
        h.push(Reverse(W::new(x)));
    }
    let mut out = Vec::with_capacity(input.len());
    while let Some(Reverse(w)) = h.pop() {
        out.push(w.get());
    }
    out
}

/// key maps an f64 to a u64 with the same order as the IEEE total order
/// (lesson 14 example 8): flip every bit of a negative, only the sign bit of a
/// non-negative. u64 is Ord, so BinaryHeap<Reverse<u64>> needs no wrapper at all.
fn key(x: f64) -> u64 {
    let b = x.to_bits();
    if b >> 63 == 1 { !b } else { b | 1 << 63 }
}

fn unkey(k: u64) -> f64 {
    f64::from_bits(if k >> 63 == 1 { k & !(1 << 63) } else { !k })
}

fn drain_keys(input: &[f64]) -> Vec<f64> {
    let mut h: BinaryHeap<Reverse<u64>> = input.iter().map(|&x| Reverse(key(x))).collect();
    let mut out = Vec::with_capacity(input.len());
    while let Some(Reverse(k)) = h.pop() {
        out.push(unkey(k));
    }
    out
}

/// show prints a float so that -0, the two NaNs and the infinities are visible.
fn show(x: f64) -> String {
    if x.is_nan() {
        (if x.is_sign_negative() { "-NaN" } else { "NaN" }).to_string()
    } else if x.is_infinite() {
        (if x > 0.0 { "+Inf" } else { "-Inf" }).to_string()
    } else {
        x.to_string()
    }
}

fn show_all(xs: &[f64]) -> String {
    let parts: Vec<String> = xs.iter().map(|&x| show(x)).collect();
    format!("[{}]", parts.join(" "))
}

/// is_sorted_total: sorted under total_cmp, the order Rust gives f64 on request.
fn is_sorted_total(xs: &[f64]) -> bool {
    xs.is_sorted_by(|a, b| a.total_cmp(b) != Ordering::Greater)
}

fn main() {
    let nan = f64::NAN; // positive
    let neg_nan = -f64::NAN;

    // (a) The same seven numbers, through a min-heap with two different orders.
    let a = [3.0, nan, 1.0, 2.0, nan, 0.0, -1.0];
    println!("(a) drain of {} through a min-heap:", show_all(&a));
    println!("    Ord is x < y by hand: {}", show_all(&drain::<Lie>(&a)));
    println!("    Ord is total_cmp:     {}", show_all(&drain::<F>(&a)));

    // (b) How often? 20,000 random arrays of 10 integer-valued floats, one slot a NaN.
    let mut r = Pcg::new(1, 2);
    let (mut bad_lie, mut bad_lie_rest, mut bad_total, mut bad_clean) = (0, 0, 0, 0);
    const CASES: usize = 20000;
    for _ in 0..CASES {
        let clean: Vec<f64> = (0..10).map(|_| r.int_n(100) as f64).collect();
        let mut dirty = clean.clone();
        dirty[r.int_n(10) as usize] = nan;
        let out = drain::<Lie>(&dirty);
        if !is_sorted_total(&out) {
            bad_lie += 1;
            // do the numbers themselves still come out in order?
            let rest: Vec<f64> = out.into_iter().filter(|x| !x.is_nan()).collect();
            if !is_sorted_total(&rest) {
                bad_lie_rest += 1;
            }
        }
        if !is_sorted_total(&drain::<F>(&dirty)) {
            bad_total += 1;
        }
        if !is_sorted_total(&drain::<Lie>(&clean)) {
            bad_clean += 1;
        }
    }
    println!(
        "\n(b) {CASES} arrays of 10 integer-valued floats, one slot replaced by NaN; drains not sorted (total_cmp):"
    );
    println!(
        "    Ord is x < y by hand, with a NaN: {bad_lie:>5}  ({:.1}%; in {bad_lie_rest} of them the plain numbers are themselves out of order)",
        100.0 * bad_lie as f64 / CASES as f64
    );
    println!("    Ord is total_cmp, with a NaN:     {bad_total:>5}");
    println!("    Ord is x < y by hand, no NaN:     {bad_clean:>5}");

    // (c) Signed zeros.
    let z = [0.0, -0.0, 0.0, -0.0];
    println!("\n(c) drain of {} :", show_all(&z));
    println!(
        "    Ord is x < y by hand: {} (-0 < 0 is false: all four are 'equal')",
        show_all(&drain::<Lie>(&z))
    );
    println!(
        "    Ord is total_cmp:     {} (total_cmp puts -0 before +0)",
        show_all(&drain::<F>(&z))
    );
    println!("    ordered u64 key:      {}", show_all(&drain_keys(&z)));

    // (d) The ordered-integer key: no NaN test, and the same NaN placement as total_cmp.
    println!("\n(d) the same drains through a BinaryHeap<Reverse<u64>> of ordered keys:");
    println!("     {} -> {}", show_all(&a), show_all(&drain_keys(&a)));
    let b = [
        3.0,
        neg_nan,
        0.0,
        -0.0,
        f64::INFINITY,
        nan,
        -1.0,
        f64::NEG_INFINITY,
    ];
    let by_key = drain_keys(&b);
    println!("     {} -> {}", show_all(&b), show_all(&by_key));
    let by_total = drain::<F>(&b);
    println!(
        "     total_cmp on the second array: {}  same as the key drain: {}",
        show_all(&by_total),
        by_total
            .iter()
            .map(|x| x.to_bits())
            .eq(by_key.iter().map(|x| x.to_bits()))
    );
    println!(
        "     total_cmp and the bit-flip are the same rule: -NaN first, +NaN last, -0 before +0."
    );
    println!("     (Go's cmp.Compare is a different one: every NaN first, -0 == +0.)");
}
```

**Rust output:**

```
(a) drain of [3 NaN 1 2 NaN 0 -1] through a min-heap:
    Ord is x < y by hand: [-1 0 1 3 NaN NaN 2]
    Ord is total_cmp:     [-1 0 1 2 3 NaN NaN]

(b) 20000 arrays of 10 integer-valued floats, one slot replaced by NaN; drains not sorted (total_cmp):
    Ord is x < y by hand, with a NaN: 19420  (97.1%; in 6521 of them the plain numbers are themselves out of order)
    Ord is total_cmp, with a NaN:         0
    Ord is x < y by hand, no NaN:         0

(c) drain of [0 -0 0 -0] :
    Ord is x < y by hand: [0 0 -0 -0] (-0 < 0 is false: all four are 'equal')
    Ord is total_cmp:     [-0 -0 0 0] (total_cmp puts -0 before +0)
    ordered u64 key:      [-0 -0 0 0]

(d) the same drains through a BinaryHeap<Reverse<u64>> of ordered keys:
     [3 NaN 1 2 NaN 0 -1] -> [-1 0 1 2 3 NaN NaN]
     [3 -NaN 0 -0 +Inf NaN -1 -Inf] -> [-NaN -Inf -1 -0 0 3 +Inf NaN]
     total_cmp on the second array: [-NaN -Inf -1 -0 0 3 +Inf NaN]  same as the key drain: true
     total_cmp and the bit-flip are the same rule: -NaN first, +NaN last, -0 before +0.
     (Go's cmp.Compare is a different one: every NaN first, -0 == +0.)
```

> **Go vs Rust:** `BinaryHeap<f64>` does not compile: `push` fails with E0277, "the trait bound `f64: Ord` is not satisfied" (checked with a one-line program, which cannot be in this one), so the accident Go makes silently is a compile error. The two fixes shown, a `total_cmp` wrapper and an ordered `u64` key, give the same order (-NaN first, +NaN last, -0 before +0), unlike Go's `cmp.Less` (every NaN first, `-0 == +0`). The compiler still lets you write a lying `Ord`: `Lie` is Go's `x < y` as an `Ord`, and std then misorders just as silently (no panic, no undefined behaviour): 19,420 of 20,000 drains are unsorted here, judged by `total_cmp` with NaN last, while Go's 18,015 was judged by `cmp.Compare` with NaN first, so compare the judge-independent count instead: drains whose plain numbers are out of order, 6,521 here against Go's 5,652, different because the sifts differ.

**Complexity:** unchanged Θ(log n) · measured: with `x < y` and one `NaN`, 18,015 of 20,000 drains (90.1%) came out unsorted, and in 5,652 of those even the plain numbers were out of order; `cmp.Less` gave 0 and `x < y` with no `NaN` gave 0 · `cmp.Less` and the ordered-key trick are both correct but disagree on where the `NaN`s go (all first, against -NaN first and +NaN last) and on `-0` against `+0` · the cost of correctness is one `isNaN` test per comparison, or one bit trick per key

---

> Next tier: [🟡 medium](2-medium.md) — top-K, merging k sorted lists, the running median, scheduling,
> and cancelling an entry without a decrease-key.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
