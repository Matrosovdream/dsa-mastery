# Step 05 — Recursion & the Call Stack · 🔴 Hard

Examples **12–16**: backtracking, depth as a design constraint, and the program that justifies recursion.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 12 and 16 are **deterministic**. Examples 13–15 measure stack or time, and 15 spawns a child
process. Sample output is from an Apple M4, Go 1.26.3.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Backtracking: choose, explore, un-choose

`🔴 hard` · *Pruning is the whole algorithm*

Backtracking is recursion with an un-do, and the skeleton never changes. What changes — by four orders
of magnitude — is how early you reject a candidate.

**Steps:**

1. Write permutations with the skeleton and a `used` array.
2. Write N-queens with a safety check inside the loop.
3. Write it again with the check moved to the leaves, and count tree nodes for both.

```go
package main

import (
	"fmt"
	"slices"
)

// BACKTRACKING is recursion with an un-do. The skeleton never changes:
//
//   if the partial answer is complete -> record it, return
//   for each candidate at this position:
//       if the candidate is not allowed -> skip it        <- PRUNING
//       CHOOSE     apply it to the partial answer
//       EXPLORE    recurse
//       UN-CHOOSE  undo it
//
// The whole game is pruning: a check that kills a branch before you explore it
// removes an entire subtree of work.

var nodesVisited int

// --- permutations: no pruning beyond "already used" ---

func permute(nums []int, cur []int, used []bool, out *[][]int) {
	nodesVisited++
	if len(cur) == len(nums) {
		*out = append(*out, slices.Clone(cur)) // clone -- example 10
		return
	}
	for i, v := range nums {
		if used[i] {
			continue
		}
		used[i] = true                // choose
		cur = append(cur, v)          //
		permute(nums, cur, used, out) // explore
		cur = cur[:len(cur)-1]        // un-choose
		used[i] = false               //
	}
}

// --- N-queens: pruning is the entire algorithm ---

// safe reports whether a queen at (row, col) conflicts with the queens already
// placed in rows 0..row-1. cols[r] is the column of the queen in row r.
func safe(cols []int, row, col int) bool {
	for r := 0; r < row; r++ {
		c := cols[r]
		if c == col || row-r == col-c || row-r == c-col {
			return false // same column, or same diagonal
		}
	}
	return true
}

func queens(n int, row int, cols []int, count *int) {
	nodesVisited++
	if row == n {
		*count++
		return
	}
	for col := 0; col < n; col++ {
		if !safe(cols, row, col) {
			continue // PRUNE: this subtree cannot contain a solution
		}
		cols[row] = col
		queens(n, row+1, cols, count)
		cols[row] = -1
	}
}

// queensNoPrune explores every arrangement and only checks at the leaves --
// what backtracking looks like with the pruning removed.
func queensNoPrune(n int, row int, cols []int, count *int) {
	nodesVisited++
	if row == n {
		for r := 0; r < n; r++ {
			if !safe(cols[:r+1], r, cols[r]) {
				return
			}
		}
		*count++
		return
	}
	for col := 0; col < n; col++ {
		cols[row] = col
		queensNoPrune(n, row+1, cols, count)
	}
}

func board(cols []int) string {
	s := ""
	for _, c := range cols {
		for j := range cols {
			if j == c {
				s += "Q "
			} else {
				s += ". "
			}
		}
		s += "\n"
	}
	return s
}

func main() {
	fmt.Println("permutations of [1 2 3 4] -- choose / explore / un-choose:")
	nums := []int{1, 2, 3, 4}
	var out [][]int
	nodesVisited = 0
	permute(nums, make([]int, 0, len(nums)), make([]bool, len(nums)), &out)
	fmt.Printf("  %d permutations from %d tree nodes\n", len(out), nodesVisited)
	fmt.Printf("  first six: %v\n", out[:6])

	fmt.Println()
	fmt.Println("N-queens -- the same skeleton, with one pruning check added:")
	fmt.Println()
	fmt.Printf("%5s %14s %18s %14s %12s\n", "n", "solutions", "nodes (pruned)", "nodes (not)", "saved")
	for _, n := range []int{4, 5, 6, 7, 8} {
		cols := make([]int, n)

		nodesVisited = 0
		count := 0
		queens(n, 0, cols, &count)
		pruned := nodesVisited

		nodesVisited = 0
		count2 := 0
		queensNoPrune(n, 0, cols, &count2)
		full := nodesVisited

		if count != count2 {
			panic("the two versions disagree")
		}
		fmt.Printf("%5d %14d %18d %14d %11.1fx\n",
			n, count, pruned, full, float64(full)/float64(pruned))
	}

	fmt.Println()
	fmt.Println("one 8-queens solution:")
	fmt.Println()
	cols := make([]int, 8)
	found := 0
	var first []int
	var collect func(row int)
	collect = func(row int) {
		if found > 0 {
			return
		}
		if row == 8 {
			first = slices.Clone(cols)
			found++
			return
		}
		for col := 0; col < 8; col++ {
			if safe(cols, row, col) {
				cols[row] = col
				collect(row + 1)
			}
		}
	}
	collect(0)
	fmt.Print(board(first))

	fmt.Println()
	fmt.Println("read the N-queens table: at n=8 the pruned search visits 2057 nodes and")
	fmt.Println("the unpruned one visits over 19 million -- for the SAME 92 solutions.")
	fmt.Println()
	fmt.Println("that is the lesson. Backtracking's complexity is not the size of the")
	fmt.Println("search space, it is the size of the space MINUS everything your pruning")
	fmt.Println("check removes. Write the check as early in the loop as you can:")
	fmt.Println("rejecting at depth 2 skips every node below it.")
	fmt.Println()
	fmt.Println("the un-choose step is what makes one buffer safe to reuse across")
	fmt.Println("branches -- and the clone at the leaf is what keeps the recorded")
	fmt.Println("answers from aliasing it (example 10).")
}
```

**Output:**

```
permutations of [1 2 3 4] -- choose / explore / un-choose:
  24 permutations from 65 tree nodes
  first six: [[1 2 3 4] [1 2 4 3] [1 3 2 4] [1 3 4 2] [1 4 2 3] [1 4 3 2]]

N-queens -- the same skeleton, with one pruning check added:

    n      solutions     nodes (pruned)    nodes (not)        saved
    4              2                 17            341        20.1x
    5             10                 54           3906        72.3x
    6              4                153          55987       365.9x
    7             40                552         960800      1740.6x
    8             92               2057       19173961      9321.3x

one 8-queens solution:

Q . . . . . . . 
. . . . Q . . . 
. . . . . . . Q 
. . . . . Q . . 
. . Q . . . . . 
. . . . . . Q . 
. Q . . . . . . 
. . . Q . . . . 

read the N-queens table: at n=8 the pruned search visits 2057 nodes and
the unpruned one visits over 19 million -- for the SAME 92 solutions.

that is the lesson. Backtracking's complexity is not the size of the
search space, it is the size of the space MINUS everything your pruning
check removes. Write the check as early in the loop as you can:
rejecting at depth 2 skips every node below it.

the un-choose step is what makes one buffer safe to reuse across
branches -- and the clone at the leaf is what keeps the recorded
answers from aliasing it (example 10).
```

**2,057 nodes against 19,173,961**, for the same 92 solutions. That is what a single well-placed
`continue` is worth. Backtracking's complexity is not the size of the search space — it's the size of
the space *minus everything your pruning removes*, so put the check as early in the loop as it can go.

Note the two habits from example 10 in the skeleton: `un-choose` makes one buffer safe to reuse across
branches, and `slices.Clone` at the leaf keeps the recorded answers from aliasing it.

**Rust:**

```rust
// BACKTRACKING is recursion with an un-do. The skeleton never changes:
//
//   if the partial answer is complete -> record it, return
//   for each candidate at this position:
//       if the candidate is not allowed -> skip it        <- PRUNING
//       CHOOSE     apply it to the partial answer
//       EXPLORE    recurse
//       UN-CHOOSE  undo it
//
// The whole game is pruning: a check that kills a branch before you explore it
// removes an entire subtree of work.
//
// Rust has no mutable globals, so the node counter Go keeps in `var nodesVisited`
// is a `&mut u64` threaded through every call.

// --- permutations: no pruning beyond "already used" ---

fn permute(
    nums: &[i32],
    cur: &mut Vec<i32>,
    used: &mut [bool],
    out: &mut Vec<Vec<i32>>,
    nodes: &mut u64,
) {
    *nodes += 1;
    if cur.len() == nums.len() {
        // The clone is not a habit here, it is a type requirement: `out` OWNS its
        // rows, and `cur` is only borrowed. Pushing `cur` itself will not compile.
        out.push(cur.clone());
        return;
    }
    for (i, &v) in nums.iter().enumerate() {
        if used[i] {
            continue;
        }
        used[i] = true; // choose
        cur.push(v); //
        permute(nums, cur, used, out, nodes); // explore
        cur.pop(); // un-choose
        used[i] = false; //
    }
}

// --- N-queens: pruning is the entire algorithm ---

// safe reports whether a queen at (row, col) conflicts with the queens already
// placed in rows 0..row-1. cols[r] is the column of the queen in row r.
// Two queens share a diagonal when row distance == column distance, in either
// direction -- `abs_diff` covers both of Go's two comparisons in one.
fn safe(cols: &[usize], row: usize, col: usize) -> bool {
    for (r, &c) in cols[..row].iter().enumerate() {
        if c == col || row - r == col.abs_diff(c) {
            return false; // same column, or same diagonal
        }
    }
    true
}

// `cols` is a stack: its length IS the current row, and push/pop are the
// choose/un-choose (Go keeps a fixed array and writes -1 to un-choose).
fn queens(n: usize, cols: &mut Vec<usize>, count: &mut u64, nodes: &mut u64) {
    *nodes += 1;
    let row = cols.len();
    if row == n {
        *count += 1;
        return;
    }
    for col in 0..n {
        if !safe(cols, row, col) {
            continue; // PRUNE: this subtree cannot contain a solution
        }
        cols.push(col);
        queens(n, cols, count, nodes);
        cols.pop();
    }
}

// queens_no_prune explores every arrangement and only checks at the leaves --
// what backtracking looks like with the pruning removed.
fn queens_no_prune(n: usize, cols: &mut Vec<usize>, count: &mut u64, nodes: &mut u64) {
    *nodes += 1;
    if cols.len() == n {
        for r in 0..n {
            if !safe(&cols[..=r], r, cols[r]) {
                return;
            }
        }
        *count += 1;
        return;
    }
    for col in 0..n {
        cols.push(col);
        queens_no_prune(n, cols, count, nodes);
        cols.pop();
    }
}

// The first solution in search order. Go sets a `found` flag in a closure; in
// Rust the recursion simply returns an Option and the caller stops looking.
fn first_solution(n: usize, cols: &mut Vec<usize>) -> Option<Vec<usize>> {
    let row = cols.len();
    if row == n {
        return Some(cols.clone());
    }
    for col in 0..n {
        if safe(cols, row, col) {
            cols.push(col);
            let found = first_solution(n, cols);
            cols.pop();
            if found.is_some() {
                return found;
            }
        }
    }
    None
}

fn board(cols: &[usize]) -> String {
    let mut s = String::new();
    for &c in cols {
        for j in 0..cols.len() {
            s.push_str(if j == c { "Q " } else { ". " });
        }
        s.push('\n');
    }
    s
}

// Go's %v prints [1 2 3]; Rust's {:?} prints [1, 2, 3]. Format by hand to match.
fn show(xs: &[i32]) -> String {
    let items: Vec<String> = xs.iter().map(|v| v.to_string()).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    println!("permutations of [1 2 3 4] -- choose / explore / un-choose:");
    let nums = [1, 2, 3, 4];
    let mut out = Vec::new();
    let mut nodes = 0;
    permute(
        &nums,
        &mut Vec::with_capacity(nums.len()),
        &mut [false; 4],
        &mut out,
        &mut nodes,
    );
    println!("  {} permutations from {} tree nodes", out.len(), nodes);
    let first_six: Vec<String> = out[..6].iter().map(|p| show(p)).collect();
    println!("  first six: [{}]", first_six.join(" "));

    println!();
    println!("N-queens -- the same skeleton, with one pruning check added:");
    println!();
    println!(
        "{:>5} {:>14} {:>18} {:>14} {:>12}",
        "n", "solutions", "nodes (pruned)", "nodes (not)", "saved"
    );
    for n in 4..=8 {
        let mut cols = Vec::with_capacity(n);

        let (mut count, mut pruned) = (0, 0);
        queens(n, &mut cols, &mut count, &mut pruned);

        let (mut count2, mut full) = (0, 0);
        queens_no_prune(n, &mut cols, &mut count2, &mut full);

        assert_eq!(count, count2, "the two versions disagree");
        println!(
            "{:>5} {:>14} {:>18} {:>14} {:>11.1}x",
            n,
            count,
            pruned,
            full,
            full as f64 / pruned as f64
        );
    }

    println!();
    println!("one 8-queens solution:");
    println!();
    let first = first_solution(8, &mut Vec::with_capacity(8)).expect("8-queens has solutions");
    print!("{}", board(&first));

    println!();
    println!("read the N-queens table: at n=8 the pruned search visits 2057 nodes and");
    println!("the unpruned one visits over 19 million -- for the SAME 92 solutions.");
    println!();
    println!("that is the lesson. Backtracking's complexity is not the size of the");
    println!("search space, it is the size of the space MINUS everything your pruning");
    println!("check removes. Write the check as early in the loop as you can:");
    println!("rejecting at depth 2 skips every node below it.");
    println!();
    println!("the un-choose step is what makes one buffer safe to reuse across");
    println!("branches -- and the clone at the leaf is what turns that shared buffer");
    println!("into an answer you own. Go needs the clone to avoid aliasing (example");
    println!("10); in Rust, forgetting it is a compile error.");
}
```

**Rust output:**

```
permutations of [1 2 3 4] -- choose / explore / un-choose:
  24 permutations from 65 tree nodes
  first six: [[1 2 3 4] [1 2 4 3] [1 3 2 4] [1 3 4 2] [1 4 2 3] [1 4 3 2]]

N-queens -- the same skeleton, with one pruning check added:

    n      solutions     nodes (pruned)    nodes (not)        saved
    4              2                 17            341        20.1x
    5             10                 54           3906        72.3x
    6              4                153          55987       365.9x
    7             40                552         960800      1740.6x
    8             92               2057       19173961      9321.3x

one 8-queens solution:

Q . . . . . . . 
. . . . Q . . . 
. . . . . . . Q 
. . . . . Q . . 
. . Q . . . . . 
. . . . . . Q . 
. Q . . . . . . 
. . . Q . . . . 

read the N-queens table: at n=8 the pruned search visits 2057 nodes and
the unpruned one visits over 19 million -- for the SAME 92 solutions.

that is the lesson. Backtracking's complexity is not the size of the
search space, it is the size of the space MINUS everything your pruning
check removes. Write the check as early in the loop as you can:
rejecting at depth 2 skips every node below it.

the un-choose step is what makes one buffer safe to reuse across
branches -- and the clone at the leaf is what turns that shared buffer
into an answer you own. Go needs the clone to avoid aliasing (example
10); in Rust, forgetting it is a compile error.
```

> **Go vs Rust:** The tables are identical to Go digit for digit (2,057 nodes against 19,173,961 at n=8), because node counts belong to the algorithm, not the language. Two things differ: there is no mutable global, so `nodesVisited` becomes a `&mut u64` threaded through the calls, and the clone at the leaf is enforced by the types (`out` owns its rows, `cur` is only borrowed), where Go relies on the programmer remembering `slices.Clone`. Only the last printed paragraph was reworded to say that.

**Complexity:** permutations Θ(n!) · N-queens Θ(n!) worst case but vastly less in practice — full treatment in [29 — Backtracking](../../29-backtracking.md)

---

## 13. How fast the input shrinks decides everything

`🔴 hard` · *Depth as a design constraint*

Two recursive sorts. One shrinks by half, one by one. They have the same *kind* of definition and
completely different fates on real input.

**Steps:**

1. Instrument merge sort and a recursive insertion sort with a depth watermark.
2. Measure the peak stack of each in a fresh goroutine.
3. Compare the depth columns at n = 1,000…100,000.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
)

// Lesson 03 said recursion depth is space. That makes "how fast does the input
// shrink?" the single most important question about any recursion:
//
//   n -> n-1   depth n       -- overflows on real data
//   n -> n/2   depth log n   -- effectively unbounded
//
// Divide & conquer is not just about doing less work. It is about the stack.

var maxDepth int

func note(d int) {
	if d > maxDepth {
		maxDepth = d
	}
}

// mergeSort: two recursive calls, each on HALF. Depth is log2(n).
func mergeSort(xs []int, depth int) []int {
	note(depth)
	if len(xs) <= 1 {
		return xs
	}
	mid := len(xs) / 2
	left := mergeSort(xs[:mid], depth+1)
	right := mergeSort(xs[mid:], depth+1)

	out := make([]int, 0, len(xs))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			out = append(out, left[i])
			i++
		} else {
			out = append(out, right[j])
			j++
		}
	}
	return append(append(out, left[i:]...), right[j:]...)
}

// insertionSortRec: one recursive call, on n-1. Depth is n.
func insertionSortRec(xs []int, n, depth int) {
	note(depth)
	if n <= 1 {
		return
	}
	insertionSortRec(xs, n-1, depth+1)
	v := xs[n-1]
	j := n - 2
	for j >= 0 && xs[j] > v {
		xs[j+1] = xs[j]
		j--
	}
	xs[j+1] = v
}

func stackInUse() uint64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.StackInuse
}

// peak runs fn in a fresh goroutine and reports stack bytes at its deepest point.
func peak(fn func(probe func())) uint64 {
	runtime.GC()
	base := stackInUse()
	var top uint64
	done := make(chan struct{})
	go func() {
		defer close(done)
		fn(func() { top = stackInUse() })
	}()
	<-done
	runtime.GC()
	if top < base {
		return 0
	}
	return top - base
}

// Probe variants that sample the stack at maximum depth.

func mergeProbe(xs []int, probe func()) []int {
	if len(xs) <= 1 {
		probe()
		return xs
	}
	mid := len(xs) / 2
	left := mergeProbe(xs[:mid], probe)
	right := mergeProbe(xs[mid:], probe)
	out := make([]int, 0, len(xs))
	i, j := 0, 0
	for i < len(left) && j < len(right) {
		if left[i] <= right[j] {
			out = append(out, left[i])
			i++
		} else {
			out = append(out, right[j])
			j++
		}
	}
	return append(append(out, left[i:]...), right[j:]...)
}

func insertionProbe(xs []int, n int, probe func()) {
	if n <= 1 {
		probe()
		return
	}
	insertionProbe(xs, n-1, probe)
	v := xs[n-1]
	j := n - 2
	for j >= 0 && xs[j] > v {
		xs[j+1] = xs[j]
		j--
	}
	xs[j+1] = v
}

func input(n int) []int {
	rng := rand.New(rand.NewPCG(1, 2))
	xs := make([]int, n)
	for i := range xs {
		xs[i] = rng.IntN(1 << 20)
	}
	return xs
}

func main() {
	fmt.Println("two recursive sorts, two shrink rates:")
	fmt.Println()
	fmt.Printf("%10s %16s %16s %16s %16s\n",
		"n", "merge depth", "insertion depth", "merge stack", "insertion stack")

	for _, n := range []int{1_000, 10_000, 100_000} {
		src := input(n)

		maxDepth = 0
		mergeSort(slices.Clone(src), 0)
		md := maxDepth

		maxDepth = 0
		insertionSortRec(slices.Clone(src), n, 0)
		id := maxDepth

		ms := peak(func(p func()) { mergeProbe(slices.Clone(src), p) })
		is := peak(func(p func()) { insertionProbe(slices.Clone(src), n, p) })

		fmt.Printf("%10d %16d %16d %13.0f KB %13.0f KB\n",
			n, md, id, float64(ms)/1024, float64(is)/1024)
	}

	fmt.Println()
	fmt.Println("the depth columns are the whole story:")
	fmt.Println()
	fmt.Println("  merge sort     depth ~ log2(n). At n=100,000 that is 17.")
	fmt.Println("                 At n = 1 BILLION it would be about 30.")
	fmt.Println("  insertion sort depth = n. At n=100,000 that is 99,999 frames,")
	fmt.Println("                 and at a billion it is a dead process.")
	fmt.Println()
	fmt.Println("(merge sort's stack reads 0 KB because 17 frames do not move the")
	fmt.Println(" runtime's span-level accounting at all -- it is below the noise floor.)")
	fmt.Println()
	fmt.Println("both are 'recursive sorts'. Only one of them can be handed real input.")

	fmt.Println()
	fmt.Println("this is why the divide & conquer shape matters beyond its running time:")
	fmt.Println("halving makes the stack a NON-ISSUE, so you never have to convert it to")
	fmt.Println("an explicit stack (example 7) or worry about the limit (example 9).")
	fmt.Println()
	fmt.Println("the rule of thumb for the rest of this plan:")
	fmt.Println("  recursion that HALVES  -> leave it recursive, it is O(log n) deep")
	fmt.Println("  recursion that DECREMENTS -> make it a loop before it reaches production")

	// Confirm both sorts actually work.
	src := input(1000)
	a := mergeSort(slices.Clone(src), 0)
	b := slices.Clone(src)
	insertionSortRec(b, len(b), 0)
	fmt.Printf("\nboth sorts agree with slices.Sort: %v\n",
		slices.IsSorted(a) && slices.Equal(a, b))
}
```

**Sample output:**

```
two recursive sorts, two shrink rates:

         n      merge depth  insertion depth      merge stack  insertion stack
      1000               10              999             0 KB           128 KB
     10000               14             9999             0 KB          1024 KB
    100000               17            99999             0 KB          8192 KB

the depth columns are the whole story:

  merge sort     depth ~ log2(n). At n=100,000 that is 17.
                 At n = 1 BILLION it would be about 30.
  insertion sort depth = n. At n=100,000 that is 99,999 frames,
                 and at a billion it is a dead process.

(merge sort's stack reads 0 KB because 17 frames do not move the
 runtime's span-level accounting at all -- it is below the noise floor.)

both are 'recursive sorts'. Only one of them can be handed real input.

this is why the divide & conquer shape matters beyond its running time:
halving makes the stack a NON-ISSUE, so you never have to convert it to
an explicit stack (example 7) or worry about the limit (example 9).

the rule of thumb for the rest of this plan:
  recursion that HALVES  -> leave it recursive, it is O(log n) deep
  recursion that DECREMENTS -> make it a loop before it reaches production

both sorts agree with slices.Sort: true
```

Depth **17 versus 99,999** at n=100,000. At a billion elements merge sort would be ~30 deep and the
other would be a dead process. Both are "recursive sorts"; only one can be handed real input.

The rule for the rest of this plan: **recursion that halves — leave it alone. Recursion that
decrements — make it a loop before it reaches production.**

**Rust:**

```rust
use std::hint::black_box;
use std::thread;

// Lesson 03 said recursion depth is space. That makes "how fast does the input
// shrink?" the single most important question about any recursion:
//
//   n -> n-1   depth n       -- overflows on real data
//   n -> n/2   depth log n   -- effectively unbounded
//
// Divide & conquer is not just about doing less work. It is about the stack.

// ---- instrumentation: a depth watermark AND the real stack pointer ----
//
// Go asks the runtime how many stack bytes are in use (MemStats.StackInuse), which
// only moves when the runtime grows a stack. Rust has no such runtime, but the
// stack is just memory: the address of any local variable says how far down it
// has grown. Record that address at the start of the thread and at every call;
// the largest gap is the peak stack the recursion used.

#[inline(never)]
fn stack_address() -> usize {
    let marker = 0u8;
    black_box(&marker) as *const u8 as usize
}

struct Probe {
    base: usize,      // stack address when the thread started
    lowest: usize,    // deepest (lowest) address any call has reached
    max_depth: usize, // depth watermark
}

impl Probe {
    fn new() -> Self {
        let base = stack_address();
        Probe {
            base,
            lowest: base,
            max_depth: 0,
        }
    }
    fn note(&mut self, depth: usize) {
        self.max_depth = self.max_depth.max(depth);
        self.lowest = self.lowest.min(stack_address());
    }
    fn stack_bytes(&self) -> usize {
        self.base - self.lowest
    }
}

// mergeSort: two recursive calls, each on HALF. Depth is log2(n).
fn merge_sort(xs: &[i64], depth: usize, probe: &mut Probe) -> Vec<i64> {
    probe.note(depth);
    if xs.len() <= 1 {
        return xs.to_vec();
    }
    let mid = xs.len() / 2;
    let left = merge_sort(&xs[..mid], depth + 1, probe);
    let right = merge_sort(&xs[mid..], depth + 1, probe);

    let mut out = Vec::with_capacity(xs.len());
    let (mut i, mut j) = (0, 0);
    while i < left.len() && j < right.len() {
        if left[i] <= right[j] {
            out.push(left[i]);
            i += 1;
        } else {
            out.push(right[j]);
            j += 1;
        }
    }
    out.extend_from_slice(&left[i..]);
    out.extend_from_slice(&right[j..]);
    out
}

// insertion_sort_rec: one recursive call, on n-1. Depth is n.
fn insertion_sort_rec(xs: &mut [i64], n: usize, depth: usize, probe: &mut Probe) {
    probe.note(depth);
    if n <= 1 {
        return;
    }
    insertion_sort_rec(xs, n - 1, depth + 1, probe);
    let v = xs[n - 1];
    // `j` is one past the slot being compared: a usize cannot walk down to -1
    // the way Go's `j >= 0` loop does.
    let mut j = n - 1;
    while j > 0 && xs[j - 1] > v {
        xs[j] = xs[j - 1];
        j -= 1;
    }
    xs[j] = v;
}

// Run `f` on a fresh thread with a big stack, so the measurement starts from
// an empty stack and the insertion sort has room to finish. (The main thread
// has ~8 MB; Go's goroutine stacks simply grow, up to 1 GB.)
fn on_big_stack<R: Send>(f: impl FnOnce() -> R + Send) -> R {
    thread::scope(|s| {
        thread::Builder::new()
            .stack_size(64 << 20)
            .spawn_scoped(s, f)
            .expect("spawn")
            .join()
            .expect("join")
    })
}

fn input(n: usize) -> Vec<i64> {
    // splitmix64, seeded: the std-only stand-in for Go's rand.New(rand.NewPCG(1, 2))
    let mut state = 1u64;
    (0..n)
        .map(|_| {
            state = state.wrapping_add(0x9e3779b97f4a7c15);
            let mut z = state;
            z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
            z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
            ((z ^ (z >> 31)) % (1 << 20)) as i64
        })
        .collect()
}

fn main() {
    println!("two recursive sorts, two shrink rates:");
    println!();
    println!(
        "{:>10} {:>16} {:>16} {:>16} {:>16}",
        "n", "merge depth", "insertion depth", "merge stack", "insertion stack"
    );

    let mut last = (0, 0, 0, 0); // the n = 100,000 row, reused below
    for n in [1_000, 10_000, 100_000] {
        let src = input(n);

        let (md, ms) = on_big_stack(|| {
            let mut probe = Probe::new();
            merge_sort(black_box(&src), 0, &mut probe);
            (probe.max_depth, probe.stack_bytes())
        });
        let (id, is) = on_big_stack(|| {
            let mut v = src.clone();
            let mut probe = Probe::new();
            insertion_sort_rec(&mut v, n, 0, &mut probe);
            (probe.max_depth, probe.stack_bytes())
        });

        println!(
            "{:>10} {:>16} {:>16} {:>13.1} KB {:>13.1} KB",
            n,
            md,
            id,
            ms as f64 / 1024.0,
            is as f64 / 1024.0
        );
        last = (md, id, ms, is);
    }
    let (md, id, ms, is) = last;
    let (merge_frames, insertion_frames) = (md + 1, id + 1);
    let insertion_frame = is / insertion_frames;

    println!();
    println!("the depth columns are the whole story:");
    println!();
    println!("  merge sort     depth ~ log2(n). At n=100,000 that is 17.");
    println!("                 At n = 1 BILLION it would be about 30.");
    println!("  insertion sort depth = n. At n=100,000 that is 99,999 frames,");
    println!("                 and at a billion it is a dead process.");
    println!();
    println!("(the stack columns are measured, not estimated: the gap between where the");
    println!(" thread's stack began and the lowest address any call reached.)");
    println!();
    println!("at n=100,000 that gap works out to:");
    println!(
        "  merge sort      {ms:>8} bytes over {merge_frames:>6} frames = ~{} bytes per frame",
        ms / merge_frames
    );
    println!(
        "  insertion sort  {is:>8} bytes over {insertion_frames:>6} frames = ~{insertion_frame} bytes per frame"
    );
    println!(
        "a typical 8 MiB main-thread stack fits about {} frames like the second kind;",
        (8 << 20) / insertion_frame
    );
    println!(
        "n = 100,000 already fills {:.0}% of it, and 10x more input would need {:.0} MiB.",
        is as f64 / (8 << 20) as f64 * 100.0,
        (is * 10) as f64 / (1 << 20) as f64
    );
    println!();
    println!("both are 'recursive sorts'. Only one of them can be handed real input.");

    println!();
    println!("this is why the divide & conquer shape matters beyond its running time:");
    println!("halving makes the stack a NON-ISSUE, so you never have to convert it to");
    println!("an explicit stack (example 7) or worry about the limit (example 9).");
    println!();
    println!("the rule of thumb for the rest of this plan:");
    println!("  recursion that HALVES  -> leave it recursive, it is O(log n) deep");
    println!("  recursion that DECREMENTS -> make it a loop before it reaches production");

    // Confirm both sorts actually work.
    let src = input(1000);
    let a = merge_sort(&src, 0, &mut Probe::new());
    let mut b = src.clone();
    insertion_sort_rec(&mut b, src.len(), 0, &mut Probe::new());
    let mut want = src.clone();
    want.sort();
    println!("\nboth sorts agree with sort(): {}", a == want && b == want);
}
```

**Rust sample output:**

```
two recursive sorts, two shrink rates:

         n      merge depth  insertion depth      merge stack  insertion stack
      1000               10              999           1.9 KB          62.5 KB
     10000               14             9999           2.6 KB         625.0 KB
    100000               17            99999           3.1 KB        6250.0 KB

the depth columns are the whole story:

  merge sort     depth ~ log2(n). At n=100,000 that is 17.
                 At n = 1 BILLION it would be about 30.
  insertion sort depth = n. At n=100,000 that is 99,999 frames,
                 and at a billion it is a dead process.

(the stack columns are measured, not estimated: the gap between where the
 thread's stack began and the lowest address any call reached.)

at n=100,000 that gap works out to:
  merge sort          3168 bytes over     18 frames = ~176 bytes per frame
  insertion sort   6400000 bytes over 100000 frames = ~64 bytes per frame
a typical 8 MiB main-thread stack fits about 131072 frames like the second kind;
n = 100,000 already fills 76% of it, and 10x more input would need 61 MiB.

both are 'recursive sorts'. Only one of them can be handed real input.

this is why the divide & conquer shape matters beyond its running time:
halving makes the stack a NON-ISSUE, so you never have to convert it to
an explicit stack (example 7) or worry about the limit (example 9).

the rule of thumb for the rest of this plan:
  recursion that HALVES  -> leave it recursive, it is O(log n) deep
  recursion that DECREMENTS -> make it a loop before it reaches production

both sorts agree with sort(): true
```

> **Go vs Rust:** The depth columns match Go exactly (10/14/17 against 999/9,999/99,999); the stack columns are Rust-specific measurements, not Go's, because there is no runtime to ask: the program reads the address of a local at every call, so the numbers are exact (3.1 KB against 6,250 KB at n=100,000, i.e. 64 bytes per insertion-sort frame in this `-O` build) instead of Go's 0 KB and power-of-two 128/1024/8192 KB. Go's stacks grow on demand; Rust's are fixed at thread creation, so the run needs `thread::Builder::stack_size`, and at n=100,000 the recursive insertion sort already uses 76% of a default 8 MiB stack (built without `-O` the frame measured 160 bytes, about 15.6 MB, which would overflow it).

**Complexity:** merge sort Θ(n log n) time, **Θ(log n)** stack · recursive insertion sort Θ(n²) time, **Θ(n)** stack

---

## 14. What a function call actually costs

`🔴 hard` · *Three rows, three different answers*

"Recursion is slow" is too blunt to be useful. Here are three recursive/iterative pairs whose ratios
span four orders of magnitude — for three completely different reasons.

**Steps:**

1. Benchmark recursive and iterative sum, fib and binary search.
2. Compare the ratios.
3. Work out which row is about call overhead, which is about the algorithm, and which is about nothing.

```go
package main

import (
	"fmt"
	"testing"
)

var sink int

// What does a function call actually cost? Enough to matter when it is the
// innermost operation, and not enough to matter otherwise.
//
// Each pair below computes the same answer, recursively and iteratively.

func sumRec(xs []int) int {
	if len(xs) == 0 {
		return 0
	}
	return xs[0] + sumRec(xs[1:])
}

func sumIter(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

func fibRec(n int) int {
	if n < 2 {
		return n
	}
	return fibRec(n-1) + fibRec(n-2)
}

func fibIter(n int) int {
	if n < 2 {
		return n
	}
	prev, cur := 0, 1
	for i := 2; i <= n; i++ {
		prev, cur = cur, prev+cur
	}
	return cur
}

func searchRec(xs []int, target, lo, hi int) int {
	if lo > hi {
		return -1
	}
	mid := lo + (hi-lo)/2
	switch {
	case xs[mid] == target:
		return mid
	case xs[mid] < target:
		return searchRec(xs, target, mid+1, hi)
	default:
		return searchRec(xs, target, lo, mid-1)
	}
}

func searchIter(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		switch {
		case xs[mid] == target:
			return mid
		case xs[mid] < target:
			lo = mid + 1
		default:
			hi = mid - 1
		}
	}
	return -1
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	xs := make([]int, 10_000)
	for i := range xs {
		xs[i] = i
	}
	sorted := make([]int, 1<<20)
	for i := range sorted {
		sorted[i] = i * 2
	}

	fmt.Printf("%-34s %14s %14s %10s\n", "", "recursive", "iterative", "ratio")

	rows := []struct {
		name     string
		rec, itr func()
	}{
		{"sum of 10,000 ints",
			func() { sink += sumRec(xs) },
			func() { sink += sumIter(xs) }},
		{"fib(25) -- exponential either way",
			func() { sink += fibRec(25) },
			func() { sink += fibIter(25) }},
		{"binary search in 2^20",
			func() { sink += searchRec(sorted, 999999, 0, len(sorted)-1) },
			func() { sink += searchIter(sorted, 999999) }},
	}

	for _, r := range rows {
		a, b := nsPerOp(r.rec), nsPerOp(r.itr)
		fmt.Printf("%-34s %11.1f ns %11.1f ns %9.1fx\n", r.name, a, b, a/b)
	}

	fmt.Println()
	fmt.Println("read each row separately -- they are three different stories:")
	fmt.Println()
	fmt.Println("  sum       the call IS the loop body, so you pay call overhead 10,000")
	fmt.Println("            times for one addition each. This is the case where the")
	fmt.Println("            rewrite pays -- and it also removes 10,000 stack frames.")
	fmt.Println()
	fmt.Println("  fib(25)   recursion is not the problem here; the exponential tree is.")
	fmt.Println("            The iterative version wins by a colossal margin because it")
	fmt.Println("            is a different ALGORITHM (example 8), not because it is a loop.")
	fmt.Println()
	fmt.Println("  search    ~20 calls total, so the overhead is invisible. Rewriting this")
	fmt.Println("            as a loop buys nothing measurable -- do it for taste, not speed.")
	fmt.Println()
	fmt.Println("what a call actually costs: push the arguments and return address, adjust")
	fmt.Println("the stack pointer, maybe check whether the stack needs to grow, jump, and")
	fmt.Println("unwind on the way back. A handful of nanoseconds -- irrelevant next to")
	fmt.Println("real work, and dominant when the work is one addition.")
	fmt.Println()
	fmt.Println("so: convert a recursion to a loop when the DEPTH is the problem")
	fmt.Println("(examples 9, 13, 15), or when the call is the innermost operation.")
	fmt.Println("Not because 'recursion is slow' -- that is far too blunt to be useful.")
}
```

**Sample output:**

```
                                        recursive      iterative      ratio
sum of 10,000 ints                     28297.2 ns      2290.4 ns      12.4x
fib(25) -- exponential either way     157044.2 ns         7.1 ns   22184.9x
binary search in 2^20                     23.2 ns        16.6 ns       1.4x

read each row separately -- they are three different stories:

  sum       the call IS the loop body, so you pay call overhead 10,000
            times for one addition each. This is the case where the
            rewrite pays -- and it also removes 10,000 stack frames.

  fib(25)   recursion is not the problem here; the exponential tree is.
            The iterative version wins by a colossal margin because it
            is a different ALGORITHM (example 8), not because it is a loop.

  search    ~20 calls total, so the overhead is invisible. Rewriting this
            as a loop buys nothing measurable -- do it for taste, not speed.

what a call actually costs: push the arguments and return address, adjust
the stack pointer, maybe check whether the stack needs to grow, jump, and
unwind on the way back. A handful of nanoseconds -- irrelevant next to
real work, and dominant when the work is one addition.

so: convert a recursion to a loop when the DEPTH is the problem
(examples 9, 13, 15), or when the call is the innermost operation.
Not because 'recursion is slow' -- that is far too blunt to be useful.
```

- **sum, 12.4×** — the call *is* the loop body, so you pay call overhead 10,000 times for one addition each. Converting pays, and removes 10,000 frames.
- **fib, 21,838×** — recursion isn't the problem; the exponential tree is. The iterative version wins because it's a different *algorithm* (example 8).
- **search, 1.4×** — ~20 calls total. Rewriting buys nothing measurable.

So convert when the **depth** is the problem, or when the call is the innermost operation. Not because
"recursion is slow".

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// What does a function call actually cost? Enough to matter when it is the
// innermost operation, and not enough to matter otherwise.
//
// Each pair below computes the same answer, recursively and iteratively.

fn sum_rec(xs: &[i64]) -> i64 {
    match xs {
        [] => 0,
        [first, rest @ ..] => first + sum_rec(rest),
    }
}

fn sum_iter(xs: &[i64]) -> i64 {
    let mut total = 0;
    for &v in xs {
        total += v;
    }
    total
}

// The same loop with the optimiser's hands tied: black_box(v) makes each element
// opaque, so LLVM cannot turn the loop into SIMD adds. Go has no auto-vectoriser,
// so Go's loop is always this shape; Rust's `sum_iter` above usually is not.
fn sum_scalar(xs: &[i64]) -> i64 {
    let mut total = 0;
    for &v in xs {
        total += black_box(v);
    }
    total
}

fn fib_rec(n: u32) -> u64 {
    if n < 2 {
        return n as u64;
    }
    fib_rec(n - 1) + fib_rec(n - 2)
}

fn fib_iter(n: u32) -> u64 {
    if n < 2 {
        return n as u64;
    }
    let (mut prev, mut cur) = (0, 1);
    for _ in 2..=n {
        (prev, cur) = (cur, prev + cur);
    }
    cur
}

// Half-open range [lo, hi): a usize cannot hold the -1 that Go's `hi = mid-1`
// reaches on a miss, so the idiomatic Rust form never needs it.
fn search_rec(xs: &[i64], target: i64, lo: usize, hi: usize) -> Option<usize> {
    if lo >= hi {
        return None;
    }
    let mid = lo + (hi - lo) / 2;
    match xs[mid].cmp(&target) {
        std::cmp::Ordering::Equal => Some(mid),
        std::cmp::Ordering::Less => search_rec(xs, target, mid + 1, hi),
        std::cmp::Ordering::Greater => search_rec(xs, target, lo, mid),
    }
}

fn search_iter(xs: &[i64], target: i64) -> Option<usize> {
    let (mut lo, mut hi) = (0, xs.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        match xs[mid].cmp(&target) {
            std::cmp::Ordering::Equal => return Some(mid),
            std::cmp::Ordering::Less => lo = mid + 1,
            std::cmp::Ordering::Greater => hi = mid,
        }
    }
    None
}

// ns/op: the std-only stand-in for testing.Benchmark / b.Loop().
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

fn row(name: &str, rec: impl FnMut(), itr: impl FnMut()) -> (f64, f64) {
    let (a, b) = (ns_per_op(rec), ns_per_op(itr));
    println!("{name:<34} {a:>11.1} ns {b:>11.1} ns {:>9.1}x", a / b);
    (a, b)
}

fn main() {
    let xs: Vec<i64> = (0..10_000).collect();
    let sorted: Vec<i64> = (0..1 << 20).map(|i| i * 2).collect();

    println!(
        "{:<34} {:>14} {:>14} {:>10}",
        "", "recursive", "iterative", "ratio"
    );

    // black_box on every input AND result: without it the optimiser is free to
    // fold a call on constant data into a constant and time nothing.
    let (sum_rec_ns, _) = row(
        "sum of 10,000 ints",
        || {
            black_box(sum_rec(black_box(&xs)));
        },
        || {
            black_box(sum_iter(black_box(&xs)));
        },
    );
    row(
        "fib(25) -- exponential either way",
        || {
            black_box(fib_rec(black_box(25)));
        },
        || {
            black_box(fib_iter(black_box(25)));
        },
    );
    row(
        "binary search in 2^20",
        || {
            black_box(search_rec(&sorted, black_box(999_999), 0, sorted.len()));
        },
        || {
            black_box(search_iter(&sorted, black_box(999_999)));
        },
    );
    println!();
    println!("an extra row, because Rust's optimiser changes the first one: the same");
    println!("recursive sum against a loop it is not allowed to vectorise.");
    println!();
    let (_, scalar_ns) = row(
        "sum of 10,000 ints, scalar loop",
        || {
            black_box(sum_rec(black_box(&xs)));
        },
        || {
            black_box(sum_scalar(black_box(&xs)));
        },
    );

    println!();
    println!("read each row separately -- they are three different stories:");
    println!();
    println!("  sum       the call IS the loop body, so you pay call overhead 10,000");
    println!(
        "            times for one addition each: about {:.1} ns per element here,",
        sum_rec_ns / 1e4
    );
    println!(
        "            against about {:.2} ns per element for the scalar loop. The",
        scalar_ns / 1e4
    );
    println!("            ordinary loop is faster still because LLVM turns it into SIMD");
    println!("            adds -- that last factor is the vectoriser, not the call.");
    println!("            The rewrite pays -- and it also removes 10,000 stack frames.");
    println!();
    println!("  fib(25)   recursion is not the problem here; the exponential tree is.");
    println!("            The iterative version wins by four orders of magnitude because");
    println!("            it is a different ALGORITHM (example 8), not because it is a loop.");
    println!();
    println!("  search    ~20 calls total, so the overhead is small: the ratio stays well");
    println!("            under 2x. Rewriting this as a loop buys almost nothing --");
    println!("            do it for taste, not speed.");
    println!();
    println!("what a call actually costs: push the arguments and return address, adjust");
    println!("the stack pointer, jump, and unwind on the way back. A handful of");
    println!("nanoseconds -- irrelevant next to real work, and dominant when the work");
    println!("is one addition.");
    println!();
    println!("so: convert a recursion to a loop when the DEPTH is the problem");
    println!("(examples 9, 13, 15), or when the call is the innermost operation.");
    println!("Not because 'recursion is slow' -- that is far too blunt to be useful.");
}
```

**Rust sample output:**

```
                                        recursive      iterative      ratio
sum of 10,000 ints                     71561.7 ns       903.4 ns      79.2x
fib(25) -- exponential either way     271191.9 ns        11.9 ns   22784.0x
binary search in 2^20                     25.0 ns        17.8 ns       1.4x

an extra row, because Rust's optimiser changes the first one: the same
recursive sum against a loop it is not allowed to vectorise.

sum of 10,000 ints, scalar loop        59755.5 ns      3601.8 ns      16.6x

read each row separately -- they are three different stories:

  sum       the call IS the loop body, so you pay call overhead 10,000
            times for one addition each: about 7.2 ns per element here,
            against about 0.36 ns per element for the scalar loop. The
            ordinary loop is faster still because LLVM turns it into SIMD
            adds -- that last factor is the vectoriser, not the call.
            The rewrite pays -- and it also removes 10,000 stack frames.

  fib(25)   recursion is not the problem here; the exponential tree is.
            The iterative version wins by four orders of magnitude because
            it is a different ALGORITHM (example 8), not because it is a loop.

  search    ~20 calls total, so the overhead is small: the ratio stays well
            under 2x. Rewriting this as a loop buys almost nothing --
            do it for taste, not speed.

what a call actually costs: push the arguments and return address, adjust
the stack pointer, jump, and unwind on the way back. A handful of
nanoseconds -- irrelevant next to real work, and dominant when the work
is one addition.

so: convert a recursion to a loop when the DEPTH is the problem
(examples 9, 13, 15), or when the call is the innermost operation.
Not because 'recursion is slow' -- that is far too blunt to be useful.
```

> **Go vs Rust:** Go re-run beside it on this machine showed the same shape as its sample (sum about 10.5x, fib about 24,000x, search about 1.3x), so the differences are Rust's. The sum ratio is 60-90x because `rustc -O --emit asm` compiles the iterative sum to NEON `add.2d` SIMD (about 0.65 µs against Go's 3.1-3.4 µs), while the recursion is about 1.8x slower than Go's (about 57 µs against 32-35 µs; cause not measured); the extra row pins the loop scalar with `black_box`, which brings the gap to 15-25x and leaves the rest to the vectoriser. The asm also shows LLVM turning one of `fib_rec`'s two calls into a loop (about 135 µs against Go's 220-234 µs; the tree stays exponential and the ratio stays near 10^4), while `sum_rec` and `search_rec` remain real self-calls, so the optimiser, not the source, decides what you are timing.

**Complexity:** a call is a few nanoseconds — irrelevant beside real work, dominant when the work is one addition

---

## 15. Surviving input you didn't choose

`🔴 hard` · *The production version of this lesson*

A tree traversal is the textbook case for recursion, right up until someone hands you a tree that's
really a 5,000,000-long chain. Deeply-nested JSON, a comment thread, a linked list, an attacker's
document — all the same shape.

**Steps:**

1. Build a degenerate tree that's a straight chain.
2. Sum it recursively, iteratively, and with a depth-bounded recursion.
3. Crash the recursive one in a child process with a lowered stack limit.

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"runtime/debug"
	"strings"
)

// The production version of this lesson. A tree traversal is the textbook case
// for recursion -- right up until someone hands you a tree that is really a
// 500,000-long chain, and your service dies.
//
// This is not hypothetical: deeply-nested JSON, a linked list, a comment thread,
// a filesystem with a symlink loop, and an attacker-supplied document all
// produce exactly this shape.

type node struct {
	val         int
	left, right *node
}

// degenerate builds a tree that is a straight line to the right -- perfectly
// legal, and n levels deep.
func degenerate(n int) *node {
	root := &node{val: 0}
	cur := root
	for i := 1; i < n; i++ {
		cur.right = &node{val: i}
		cur = cur.right
	}
	return root
}

// sumRec is the natural implementation. Its depth is the tree's depth.
func sumRec(n *node) int {
	if n == nil {
		return 0
	}
	return n.val + sumRec(n.left) + sumRec(n.right)
}

// sumIter does the same walk with a heap-allocated stack. Its depth is bounded
// by available memory, not by the goroutine stack limit.
func sumIter(root *node) int {
	if root == nil {
		return 0
	}
	total := 0
	stack := []*node{root}
	for len(stack) > 0 {
		n := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		total += n.val
		if n.right != nil {
			stack = append(stack, n.right)
		}
		if n.left != nil {
			stack = append(stack, n.left)
		}
	}
	return total
}

// sumRecBounded refuses input it cannot handle instead of dying on it.
const maxAllowedDepth = 10_000

var errTooDeep = fmt.Errorf("input nested deeper than %d", maxAllowedDepth)

func sumRecBounded(n *node, depth int) (int, error) {
	if n == nil {
		return 0, nil
	}
	if depth > maxAllowedDepth {
		return 0, errTooDeep
	}
	l, err := sumRecBounded(n.left, depth+1)
	if err != nil {
		return 0, err
	}
	r, err := sumRecBounded(n.right, depth+1)
	if err != nil {
		return 0, err
	}
	return n.val + l + r, nil
}

func childCrash() {
	// A 16 MB limit stands in for "a real server under memory pressure, or a
	// tree deeper than 1 GB / 48 bytes". The failure mode is identical.
	debug.SetMaxStack(16 << 20)

	tree := degenerate(5_000_000)
	fmt.Println("recursive sum:", sumRec(tree))
}

func main() {
	if os.Getenv("L05_DEEP") == "1" {
		childCrash()
		return
	}

	const n = 1_000_000
	tree := degenerate(n)

	fmt.Printf("a 'tree' of %d nodes that is really a chain %d levels deep.\n\n", n, n)

	fmt.Printf("  iterative (explicit stack): %d\n", sumIter(tree))
	fmt.Printf("  recursive (unbounded):      %d\n", sumRec(tree))

	got, err := sumRecBounded(tree, 0)
	fmt.Printf("  recursive (depth-limited):  %d, err = %v\n", got, err)

	fmt.Println()
	fmt.Println("at a million deep the recursive version still works -- Go's 1 GB stack")
	fmt.Println("limit is generous. Now the same code in a child process with a 16 MB")
	fmt.Println("limit and a 5-million-node chain:")
	fmt.Println()

	cmd := exec.Command(os.Args[0])
	cmd.Env = append(os.Environ(), "L05_DEEP=1")
	out, err := cmd.CombinedOutput()
	lines := strings.Split(strings.TrimSpace(string(out)), "\n")
	for i, ln := range lines {
		if i >= 3 {
			fmt.Printf("  ... (%d more lines)\n", len(lines)-3)
			break
		}
		fmt.Println("  " + ln)
	}
	fmt.Printf("\n  child exit status: %v\n", err)

	fmt.Println()
	fmt.Println("the iterative version would have returned an answer, because its stack")
	fmt.Println("is a []*node on the HEAP: it grows by reallocation (lesson 03) and is")
	fmt.Println("bounded by memory, not by a per-goroutine limit.")

	fmt.Println()
	fmt.Println("three ways to handle recursion over untrusted input, in order:")
	fmt.Println()
	fmt.Println("  1. BOUND THE DEPTH. Return an error past a limit you chose. Cheapest")
	fmt.Println("     to write, keeps the readable recursion, and turns a process kill")
	fmt.Println("     into a 400 Bad Request. encoding/json does exactly this.")
	fmt.Println()
	fmt.Println("  2. CONVERT TO AN EXPLICIT STACK (example 7). No limit at all, at the")
	fmt.Println("     cost of less obvious code.")
	fmt.Println()
	fmt.Println("  3. RESTRUCTURE so the depth is logarithmic (example 13) -- only")
	fmt.Println("     possible if you control the shape of the data.")
	fmt.Println()
	fmt.Println("what is NOT on the list: raising SetMaxStack, or recovering from the")
	fmt.Println("crash. The first postpones the problem; the second is impossible.")
}
```

**Sample output** (traceback trimmed):

```
a 'tree' of 1000000 nodes that is really a chain 1000000 levels deep.

  iterative (explicit stack): 499999500000
  recursive (unbounded):      499999500000
  recursive (depth-limited):  0, err = input nested deeper than 10000

at a million deep the recursive version still works -- Go's 1 GB stack
limit is generous. Now the same code in a child process with a 16 MB
limit and a 5-million-node chain:

  runtime: goroutine stack exceeds 16777216-byte limit
  runtime: sp=0x42ff7e8003a0 stack=[0x42ff7e800000, 0x42ff7f800000]
  fatal error: stack overflow
  ... (395 more lines)

  child exit status: exit status 2

the iterative version would have returned an answer, because its stack
is a []*node on the HEAP: it grows by reallocation (lesson 03) and is
bounded by memory, not by a per-goroutine limit.

three ways to handle recursion over untrusted input, in order:

  1. BOUND THE DEPTH. Return an error past a limit you chose. Cheapest
     to write, keeps the readable recursion, and turns a process kill
     into a 400 Bad Request. encoding/json does exactly this.

  2. CONVERT TO AN EXPLICIT STACK (example 7). No limit at all, at the
     cost of less obvious code.

  3. RESTRUCTURE so the depth is logarithmic (example 13) -- only
     possible if you control the shape of the data.

what is NOT on the list: raising SetMaxStack, or recovering from the
crash. The first postpones the problem; the second is impossible.
```

The iterative version would have returned an answer, because its stack is a `[]*node` on the **heap**:
it grows by reallocation and is bounded by memory, not by a per-goroutine limit.

Three defences, in order: **bound the depth** (four lines, turns a process kill into a `400`; this is
what `encoding/json` does), **convert to an explicit stack** (example 7), or **restructure so depth is
logarithmic** (example 13). What is not on the list: raising `SetMaxStack`, or recovering from the
crash — the first postpones it, the second is impossible.

**Rust:**

```rust
use std::env;
use std::fmt;
use std::hint::black_box;
use std::thread;

// The production version of this lesson. A tree traversal is the textbook case
// for recursion -- right up until someone hands you a tree that is really a
// 5,000,000-long chain, and your service dies.
//
// This is not hypothetical: deeply-nested JSON, a linked list, a comment thread,
// a filesystem with a symlink loop, and an attacker-supplied document all
// produce exactly this shape.
//
// Go crashes a child process to show the failure. A Rust stack overflow aborts
// the process (and on macOS writes a crash report on every run), so this program
// does not do that by default: it MEASURES the stack a recursion uses, PREDICTS
// where a given stack runs out, and checks the prediction from the safe side.
// The real overflow is opt-in:   ./ex15 --overflow

struct Node {
    val: i64,
    left: Option<Box<Node>>,
    right: Option<Box<Node>>,
}

// Rust-only trap: DROPPING a tree is a recursion over it too. The default drop
// glue for Box<Node> drops left, then right, each of which drops its children...
// so freeing a 1,000,000-node chain on the 8 MB main thread overflows the stack
// (tried without this impl: it aborts, while 100,000 nodes are fine).
// (Go has no destructor to write: the GC walks the heap with its own work list.)
// This Drop hands the children to a heap-allocated stack instead.
impl Drop for Node {
    fn drop(&mut self) {
        let mut stack: Vec<Box<Node>> = Vec::new();
        stack.extend(self.left.take());
        stack.extend(self.right.take());
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
            // `n` is dropped here with no children left, so its own Drop is trivial.
        }
    }
}

// Which way the chain leans. Go's example only ever builds the right-leaning one;
// the mirror image turns out to matter in Rust (see main).
#[derive(Clone, Copy)]
enum Lean {
    Right,
    Left,
}

// degenerate builds a tree that is a straight line -- perfectly legal, and
// n levels deep.
fn degenerate(n: i64, lean: Lean) -> Box<Node> {
    let leaf = |val| {
        Box::new(Node {
            val,
            left: None,
            right: None,
        })
    };
    let mut root = leaf(0);
    let mut cur = &mut root;
    for i in 1..n {
        cur = match lean {
            Lean::Right => cur.right.insert(leaf(i)),
            Lean::Left => cur.left.insert(leaf(i)),
        };
    }
    root
}

// The stack is just memory: the address of a local says how far down it has
// grown (the same trick as example 13).
#[inline(never)]
fn stack_address() -> usize {
    let marker = 0u8;
    black_box(&marker) as *const u8 as usize
}

// sum_rec is the natural implementation. Its depth is the tree's depth -- unless
// the optimiser decides otherwise, which it does for ONE of the two calls. See main.
// The only addition is the watermark line: `lowest` remembers the deepest stack
// address any call reached, so the same function that does the work also measures it.
fn sum_rec(n: &Option<Box<Node>>, lowest: &mut usize) -> i64 {
    match n {
        None => 0,
        Some(n) => {
            *lowest = (*lowest).min(stack_address());
            n.val + sum_rec(&n.left, lowest) + sum_rec(&n.right, lowest)
        }
    }
}

// Run sum_rec on a fresh thread with a stack of `stack` bytes. Returns the sum
// and the bytes of stack the recursion used. (The main thread has ~8 MB; Go's
// goroutine stacks simply grow, up to 1 GB. A Rust thread's stack is fixed when
// the thread is created.)
fn run_sum_rec(tree: &Option<Box<Node>>, stack: usize) -> (i64, usize) {
    thread::scope(|s| {
        thread::Builder::new()
            .stack_size(stack)
            .spawn_scoped(s, || {
                let base = stack_address();
                let mut lowest = base;
                let sum = sum_rec(tree, &mut lowest);
                (sum, base - lowest)
            })
            .expect("spawn")
            .join()
            .expect("join")
    })
}

// sum_iter does the same walk with a heap-allocated stack. Its depth is bounded
// by available memory, not by the thread's stack size.
fn sum_iter(root: &Node) -> i64 {
    let mut total = 0;
    let mut stack = vec![root];
    while let Some(n) = stack.pop() {
        total += n.val;
        if let Some(r) = &n.right {
            stack.push(r);
        }
        if let Some(l) = &n.left {
            stack.push(l);
        }
    }
    total
}

// sum_rec_bounded refuses input it cannot handle instead of dying on it.
const MAX_ALLOWED_DEPTH: usize = 10_000;

#[derive(Debug)]
struct TooDeep;

impl fmt::Display for TooDeep {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "input nested deeper than {MAX_ALLOWED_DEPTH}")
    }
}

fn sum_rec_bounded(n: &Option<Box<Node>>, depth: usize) -> Result<i64, TooDeep> {
    let Some(n) = n else { return Ok(0) };
    if depth > MAX_ALLOWED_DEPTH {
        return Err(TooDeep);
    }
    // `?` is the "if err != nil { return 0, err }" of Go, once per call.
    let l = sum_rec_bounded(&n.left, depth + 1)?;
    let r = sum_rec_bounded(&n.right, depth + 1)?;
    Ok(n.val + l + r)
}

const SMALL_STACK: usize = 16 << 20; // stands in for "a server under memory pressure"

// The real thing, opt-in: sum a left-leaning chain on a 16 MB stack. Past the
// predicted depth this ABORTS the process; Rust cannot recover from it.
fn overflow(n: i64) {
    println!(
        "summing a {n}-node left-leaning chain on a {} MiB stack...",
        SMALL_STACK >> 20
    );
    let tree = Some(degenerate(n, Lean::Left));
    let (sum, used) = run_sum_rec(&tree, SMALL_STACK);
    println!("survived: sum {sum}, {used} bytes of stack used");
}

fn main() {
    if env::args().nth(1).as_deref() == Some("--overflow") {
        let n = env::args()
            .nth(2)
            .map_or(5_000_000, |s| s.parse().expect("a node count"));
        return overflow(n);
    }

    const N: i64 = 1_000_000;
    let tree = Some(degenerate(N, Lean::Right));

    println!("a 'tree' of {N} nodes that is really a chain {N} levels deep.\n");

    println!(
        "  iterative (explicit stack): {}",
        sum_iter(tree.as_deref().expect("non-empty"))
    );

    // The unbounded recursion gets its own thread with 256 MB of stack to run on.
    let (total, _) = run_sum_rec(&tree, 256 << 20);
    println!("  recursive (unbounded):      {total}");

    match sum_rec_bounded(&tree, 0) {
        Ok(v) => println!("  recursive (depth-limited):  {v}"),
        Err(e) => println!("  recursive (depth-limited):  err = {e}"),
    }
    drop(tree);

    println!();
    println!("at a million deep the recursive version still works -- on a thread we gave");
    println!("256 MB. Now: what would a 16 MB stack do with a 5-million-node chain?");
    println!("Rather than crash to find out, measure the stack the recursion uses on");
    println!("chains that fit, for both ways the chain can lean:");
    println!();
    println!(
        "{:>8} {:>9} {:>14} {:>16}",
        "chain", "nodes", "stack used", "bytes per node"
    );
    let mut left_frame = 0;
    for (name, lean) in [("right", Lean::Right), ("left", Lean::Left)] {
        for n in [10_000, 100_000] {
            let tree = Some(degenerate(n, lean));
            let (_, used) = run_sum_rec(&tree, 256 << 20);
            println!(
                "{name:>8} {n:>9} {used:>12} B {:>16.1}",
                used as f64 / n as f64
            );
            if matches!(lean, Lean::Left) {
                left_frame = used / n as usize;
            }
        }
    }

    let capacity = SMALL_STACK / left_frame;
    println!();
    println!("prediction for a {} MiB stack:", SMALL_STACK >> 20);
    println!("  right-leaning: stack use does not grow with the chain, so it fits.");
    println!("  left-leaning:  {left_frame} bytes per node -> overflow at about {capacity} nodes;");
    println!(
        "                 a 5,000,000-node chain needs {} MiB, so it will NOT fit.",
        (5_000_000 * left_frame) >> 20
    );

    // Check the prediction from the safe side: 90% of the predicted depth must
    // survive on a real 16 MiB thread.
    let safe = (capacity as i64) * 9 / 10;
    let tree = Some(degenerate(safe, Lean::Left));
    let (sum, used) = run_sum_rec(&tree, SMALL_STACK);
    println!();
    println!(
        "check: a left-leaning chain of {safe} nodes (90% of that) on a real {} MiB",
        SMALL_STACK >> 20
    );
    println!("stack returns {sum} having used {used} bytes of it -- no crash.");
    println!("the other side of the line, past the prediction, is opt-in because a Rust");
    println!("stack overflow aborts the process (and macOS writes a crash report):");
    println!("  ./ex15 --overflow        # the 5,000,000-node left-leaning chain");

    println!();
    println!("the iterative version would have returned an answer, because its stack");
    println!("is a Vec<&Node> on the HEAP: it grows by reallocation (lesson 03) and is");
    println!("bounded by memory, not by a per-thread limit.");

    println!();
    println!("three ways to handle recursion over untrusted input, in order:");
    println!();
    println!("  1. BOUND THE DEPTH. Return an error past a limit you chose. Cheapest");
    println!("     to write, keeps the readable recursion, and turns a process kill");
    println!("     into a 400 Bad Request. serde_json does the same.");
    println!();
    println!("  2. CONVERT TO AN EXPLICIT STACK (example 7). No limit at all, at the");
    println!("     cost of less obvious code.");
    println!();
    println!("  3. RESTRUCTURE so the depth is logarithmic (example 13) -- only");
    println!("     possible if you control the shape of the data.");
    println!();
    println!("what is NOT on the list: a bigger stack, or recovering from the crash.");
    println!("The first postpones the problem; the second is impossible -- a Rust");
    println!("stack overflow aborts the process, it is not a panic you can catch.");
}
```

**Run it (Rust):**

```bash
rustc -O ex15.rs -o ex15 && ./ex15       # default: measures and predicts, never crashes
./ex15 --overflow             # opt-in: really overflows a 16 MiB stack and aborts the process
```

**Rust sample output:**

```
a 'tree' of 1000000 nodes that is really a chain 1000000 levels deep.

  iterative (explicit stack): 499999500000
  recursive (unbounded):      499999500000
  recursive (depth-limited):  err = input nested deeper than 10000

at a million deep the recursive version still works -- on a thread we gave
256 MB. Now: what would a 16 MB stack do with a 5-million-node chain?
Rather than crash to find out, measure the stack the recursion uses on
chains that fit, for both ways the chain can lean:

   chain     nodes     stack used   bytes per node
   right     10000           48 B              0.0
   right    100000           48 B              0.0
    left     10000       480000 B             48.0
    left    100000      4800000 B             48.0

prediction for a 16 MiB stack:
  right-leaning: stack use does not grow with the chain, so it fits.
  left-leaning:  48 bytes per node -> overflow at about 349525 nodes;
                 a 5,000,000-node chain needs 228 MiB, so it will NOT fit.

check: a left-leaning chain of 314572 nodes (90% of that) on a real 16 MiB
stack returns 49477614306 having used 15099456 bytes of it -- no crash.
the other side of the line, past the prediction, is opt-in because a Rust
stack overflow aborts the process (and macOS writes a crash report):
  ./ex15 --overflow        # the 5,000,000-node left-leaning chain

the iterative version would have returned an answer, because its stack
is a Vec<&Node> on the HEAP: it grows by reallocation (lesson 03) and is
bounded by memory, not by a per-thread limit.

three ways to handle recursion over untrusted input, in order:

  1. BOUND THE DEPTH. Return an error past a limit you chose. Cheapest
     to write, keeps the readable recursion, and turns a process kill
     into a 400 Bad Request. serde_json does the same.

  2. CONVERT TO AN EXPLICIT STACK (example 7). No limit at all, at the
     cost of less obvious code.

  3. RESTRUCTURE so the depth is logarithmic (example 13) -- only
     possible if you control the shape of the data.

what is NOT on the list: a bigger stack, or recovering from the crash.
The first postpones the problem; the second is impossible -- a Rust
stack overflow aborts the process, it is not a panic you can catch.
```

> **Go vs Rust:** Go's crashing child process is replaced by measuring and predicting, because a Rust stack overflow aborts the process (and macOS writes a crash report on every run): the same `sum_rec` measures its own stack, 48 bytes per node on a left-leaning chain but a constant 48 B on Go's right-leaning one, since `-O` turns the call on `right` into a loop, so only the mirror image of Go's chain can overflow. The predicted limit for a 16 MiB stack is 349,525 nodes; in separate runs of `--overflow`, a 346,030-node chain (99%) survived using 16,609,440 of 16,777,216 bytes, while 366,601 (105%), 384,078 (110%) and 5,000,000 nodes died with "has overflowed its stack", exit 134 (SIGABRT), which cannot be caught (an unoptimised build of an earlier version of this program overflowed on both leans). Also Rust-only: dropping a `Box` chain is itself recursive and aborted the 8 MB main thread at 1,000,000 nodes, hence the iterative `Drop`, and a thread's stack is fixed when it is created (`Builder::stack_size`) where Go's grows to 1 GB.

**Complexity:** all three are Θ(n) time · recursive Θ(n) **goroutine stack**, iterative Θ(n) **heap** — and only one of those has a hard 1 GB ceiling

---

## 16. Capstone: a recursive-descent evaluator

`🔴 hard` · *The program that justifies recursion*

Everything so far has been about when *not* to recurse. This is the case where recursion isn't a
clever trick but a direct transcription of the problem: the input is defined recursively, so the
reader is too.

```
expr   := term   (('+' | '-') term)*
term   := factor (('*' | '/') factor)*
factor := NUMBER | '-' factor | '(' expr ')'
```

One function per rule. Where a rule names another rule, the function calls it.

**Steps:**

1. Write one function per grammar rule.
2. Use a **loop** for each `(...)*` repetition, and recursion only where a rule nests.
3. Add the depth guard from example 15.

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

// The capstone: a recursive-descent evaluator for arithmetic expressions.
//
// This is the program that justifies recursion. The INPUT is recursive -- an
// expression contains expressions -- so a recursive reader is not a clever
// trick, it is a direct transcription of the grammar:
//
//   expr   := term   (('+' | '-') term)*
//   term   := factor (('*' | '/') factor)*
//   factor := NUMBER | '-' factor | '(' expr ')'
//
// One function per rule. Where a rule mentions another rule, the function calls
// it. Precedence falls out of the NESTING: expr calls term, so term binds
// tighter; factor calls back into expr inside parentheses, and that single
// mutually-recursive edge is what makes nesting work to any depth.

type parser struct {
	tokens []string
	pos    int
	depth  int
	maxDep int
	err    error
}

const maxDepth = 200 // example 15: bound the depth of untrusted input

func (p *parser) peek() string {
	if p.pos < len(p.tokens) {
		return p.tokens[p.pos]
	}
	return ""
}

func (p *parser) next() string {
	t := p.peek()
	p.pos++
	return t
}

func (p *parser) enter() bool {
	p.depth++
	if p.depth > p.maxDep {
		p.maxDep = p.depth
	}
	if p.depth > maxDepth {
		p.fail("expression nested deeper than %d", maxDepth)
		return false
	}
	return true
}

func (p *parser) leave() { p.depth-- }

func (p *parser) fail(format string, args ...any) {
	if p.err == nil {
		p.err = fmt.Errorf(format, args...)
	}
}

// expr := term (('+' | '-') term)*
//
// The repetition is a LOOP, not recursion: `a - b - c` must group left, and a
// recursive call here would group it right and compute the wrong answer.
func (p *parser) expr() float64 {
	if !p.enter() {
		return 0
	}
	defer p.leave()

	left := p.term()
	for p.err == nil {
		switch p.peek() {
		case "+":
			p.next()
			left += p.term()
		case "-":
			p.next()
			left -= p.term()
		default:
			return left
		}
	}
	return left
}

// term := factor (('*' | '/') factor)*
func (p *parser) term() float64 {
	if !p.enter() {
		return 0
	}
	defer p.leave()

	left := p.factor()
	for p.err == nil {
		switch p.peek() {
		case "*":
			p.next()
			left *= p.factor()
		case "/":
			p.next()
			right := p.factor()
			if right == 0 {
				p.fail("division by zero")
				return 0
			}
			left /= right
		default:
			return left
		}
	}
	return left
}

// factor := NUMBER | '-' factor | '(' expr ')'
//
// This is where the recursion closes the loop: a parenthesised expression sends
// us back to expr(), one level deeper.
func (p *parser) factor() float64 {
	if !p.enter() {
		return 0
	}
	defer p.leave()

	switch t := p.next(); {
	case t == "":
		p.fail("unexpected end of expression")
		return 0

	case t == "-": // unary minus: -x, and -(-x) too
		return -p.factor()

	case t == "(":
		v := p.expr()
		if p.next() != ")" {
			p.fail("missing closing parenthesis")
		}
		return v

	default:
		v, err := strconv.ParseFloat(t, 64)
		if err != nil {
			p.fail("not a number: %q", t)
		}
		return v
	}
}

func tokenize(s string) []string {
	for _, op := range []string{"+", "-", "*", "/", "(", ")"} {
		s = strings.ReplaceAll(s, op, " "+op+" ")
	}
	return strings.Fields(s)
}

func eval(s string) (float64, int, error) {
	p := &parser{tokens: tokenize(s)}
	v := p.expr()
	if p.err == nil && p.pos < len(p.tokens) {
		p.fail("unexpected %q", p.tokens[p.pos])
	}
	return v, p.maxDep, p.err
}

func main() {
	fmt.Println("recursive-descent evaluation -- one function per grammar rule:")
	fmt.Println()
	fmt.Println("  expr   := term   (('+' | '-') term)*")
	fmt.Println("  term   := factor (('*' | '/') factor)*")
	fmt.Println("  factor := NUMBER | '-' factor | '(' expr ')'")
	fmt.Println()

	inputs := []string{
		"1 + 2",
		"2 + 3 * 4",
		"(2 + 3) * 4",
		"10 - 2 - 3",
		"2 * (3 + (4 - 1) * 2)",
		"-5 + 3",
		"-(2 + 3) * -2",
		"100 / 4 / 5",
		"1 + 2 * 3 - 4 / 2",
	}

	fmt.Printf("%-26s %14s %10s\n", "expression", "result", "max depth")
	for _, in := range inputs {
		v, depth, err := eval(in)
		if err != nil {
			fmt.Printf("%-26s %14s %10d\n", in, "ERR: "+err.Error(), depth)
			continue
		}
		fmt.Printf("%-26s %14g %10d\n", in, v, depth)
	}

	fmt.Println()
	fmt.Println("errors are refused, not crashed on:")
	fmt.Println()
	for _, in := range []string{"1 +", "(1 + 2", "1 / 0", "2 $ 3"} {
		_, _, err := eval(in)
		fmt.Printf("  %-12s -> %v\n", in, err)
	}

	// Depth guard: 300 nested parens exceeds maxDepth.
	deep := strings.Repeat("(", 300) + "1" + strings.Repeat(")", 300)
	_, d, err := eval(deep)
	fmt.Printf("  %-12s -> %v (reached depth %d)\n", "300 nested (", err, d)

	fmt.Println()
	fmt.Println("what to take from this:")
	fmt.Println()
	fmt.Println("  1. the code IS the grammar. Three rules, three functions. When the")
	fmt.Println("     input's definition is recursive, so is the natural reader --")
	fmt.Println("     this is the shape from example 11's mutually recursive data.")
	fmt.Println()
	fmt.Println("  2. PRECEDENCE comes from the call graph, not from any precedence")
	fmt.Println("     table: expr calls term, so * binds tighter than +. Look at")
	fmt.Println("     '2 + 3 * 4' = 14 and '(2 + 3) * 4' = 20.")
	fmt.Println()
	fmt.Println("  3. ASSOCIATIVITY comes from using a LOOP for the repetition. If expr")
	fmt.Println("     recursed on itself for the tail, '10 - 2 - 3' would group as")
	fmt.Println("     10 - (2 - 3) = 11 instead of the correct 5.")
	fmt.Println()
	fmt.Println("  4. the depth guard from example 15 costs four lines and turns a")
	fmt.Println("     hostile input from a process kill into an error value.")
	fmt.Println()
	fmt.Println("that is Part 1. From here on the structures do the work -- and every")
	fmt.Println("one of them is built out of what these five lessons measured.")
}
```

**Output:**

```
recursive-descent evaluation -- one function per grammar rule:

  expr   := term   (('+' | '-') term)*
  term   := factor (('*' | '/') factor)*
  factor := NUMBER | '-' factor | '(' expr ')'

expression                         result  max depth
1 + 2                                   3          3
2 + 3 * 4                              14          3
(2 + 3) * 4                            20          6
10 - 2 - 3                              5          3
2 * (3 + (4 - 1) * 2)                  18          9
-5 + 3                                 -2          4
-(2 + 3) * -2                          10          7
100 / 4 / 5                             5          3
1 + 2 * 3 - 4 / 2                       5          3

errors are refused, not crashed on:

  1 +          -> unexpected end of expression
  (1 + 2       -> missing closing parenthesis
  1 / 0        -> division by zero
  2 $ 3        -> unexpected "$"
  300 nested ( -> expression nested deeper than 200 (reached depth 201)

what to take from this:

  1. the code IS the grammar. Three rules, three functions. When the
     input's definition is recursive, so is the natural reader --
     this is the shape from example 11's mutually recursive data.

  2. PRECEDENCE comes from the call graph, not from any precedence
     table: expr calls term, so * binds tighter than +. Look at
     '2 + 3 * 4' = 14 and '(2 + 3) * 4' = 20.

  3. ASSOCIATIVITY comes from using a LOOP for the repetition. If expr
     recursed on itself for the tail, '10 - 2 - 3' would group as
     10 - (2 - 3) = 11 instead of the correct 5.

  4. the depth guard from example 15 costs four lines and turns a
     hostile input from a process kill into an error value.

that is Part 1. From here on the structures do the work -- and every
one of them is built out of what these five lessons measured.
```

Four things worth extracting:

1. **The code is the grammar.** Three rules, three functions — the shape from example 11's mutually recursive data.
2. **Precedence comes from the call graph**, not a table. `expr` calls `term`, so `*` binds tighter: `2 + 3 * 4` = 14 while `(2 + 3) * 4` = 20.
3. **Associativity comes from using a loop** for the repetition. Had `expr` recursed on its own tail, `10 - 2 - 3` would group right and give 11 instead of 5.
4. **The depth guard costs four lines** and turns 300 nested parens from a process kill into an error value.

**Rust:**

```rust
use std::fmt;

// The capstone: a recursive-descent evaluator for arithmetic expressions.
//
// This is the program that justifies recursion. The INPUT is recursive -- an
// expression contains expressions -- so a recursive reader is not a clever
// trick, it is a direct transcription of the grammar:
//
//   expr   := term   (('+' | '-') term)*
//   term   := factor (('*' | '/') factor)*
//   factor := NUMBER | '-' factor | '(' expr ')'
//
// One function per rule. Where a rule mentions another rule, the function calls
// it. Precedence falls out of the NESTING: expr calls term, so term binds
// tighter; factor calls back into expr inside parentheses, and that single
// mutually-recursive edge is what makes nesting work to any depth.

const MAX_DEPTH: usize = 200; // example 15: bound the depth of untrusted input

// Go keeps one sticky `err` field and checks `p.err == nil` after every call.
// Rust returns a Result from every rule and lets `?` carry the first error up
// the whole call chain, so no rule can keep parsing after a failure.
#[derive(Debug)]
enum ParseError {
    UnexpectedEnd,
    MissingParen,
    DivisionByZero,
    NotANumber(String),
    Unexpected(String),
    TooDeep,
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ParseError::UnexpectedEnd => write!(f, "unexpected end of expression"),
            ParseError::MissingParen => write!(f, "missing closing parenthesis"),
            ParseError::DivisionByZero => write!(f, "division by zero"),
            ParseError::NotANumber(t) => write!(f, "not a number: {t:?}"),
            ParseError::Unexpected(t) => write!(f, "unexpected {t:?}"),
            ParseError::TooDeep => write!(f, "expression nested deeper than {MAX_DEPTH}"),
        }
    }
}

type Res = Result<f64, ParseError>;

// The tokens borrow from the (space-padded) input string: no copies.
struct Parser<'a> {
    tokens: Vec<&'a str>,
    pos: usize,
    depth: usize,
    max_dep: usize,
}

impl<'a> Parser<'a> {
    // Returning Option<&'a str> (borrowed from the INPUT, not from `self`) is what
    // lets a rule keep a token while it calls `&mut self` methods.
    fn peek(&self) -> Option<&'a str> {
        self.tokens.get(self.pos).copied()
    }

    fn next(&mut self) -> Option<&'a str> {
        let t = self.peek();
        self.pos += 1;
        t
    }

    // Every rule runs inside `nested`: count one level of depth, refuse past
    // MAX_DEPTH, and ALWAYS give the level back afterwards. That is Go's
    // `if !p.enter() { return 0 }; defer p.leave()` as one function. (An RAII
    // guard cannot do it here: it would keep `self` borrowed while the rule
    // needs `&mut self` to read tokens.)
    fn nested(&mut self, rule: impl FnOnce(&mut Self) -> Res) -> Res {
        self.depth += 1;
        self.max_dep = self.max_dep.max(self.depth);
        if self.depth > MAX_DEPTH {
            return Err(ParseError::TooDeep);
        }
        let result = rule(self);
        self.depth -= 1;
        result
    }

    // expr := term (('+' | '-') term)*
    //
    // The repetition is a LOOP, not recursion: `a - b - c` must group left, and a
    // recursive call here would group it right and compute the wrong answer.
    fn expr(&mut self) -> Res {
        self.nested(|p| {
            let mut left = p.term()?;
            loop {
                match p.peek() {
                    Some("+") => {
                        p.next();
                        left += p.term()?;
                    }
                    Some("-") => {
                        p.next();
                        left -= p.term()?;
                    }
                    _ => return Ok(left),
                }
            }
        })
    }

    // term := factor (('*' | '/') factor)*
    fn term(&mut self) -> Res {
        self.nested(|p| {
            let mut left = p.factor()?;
            loop {
                match p.peek() {
                    Some("*") => {
                        p.next();
                        left *= p.factor()?;
                    }
                    Some("/") => {
                        p.next();
                        let right = p.factor()?;
                        if right == 0.0 {
                            return Err(ParseError::DivisionByZero);
                        }
                        left /= right;
                    }
                    _ => return Ok(left),
                }
            }
        })
    }

    // factor := NUMBER | '-' factor | '(' expr ')'
    //
    // This is where the recursion closes the loop: a parenthesised expression sends
    // us back to expr(), one level deeper.
    fn factor(&mut self) -> Res {
        self.nested(|p| match p.next() {
            None => Err(ParseError::UnexpectedEnd),

            // unary minus: -x, and -(-x) too
            Some("-") => Ok(-p.factor()?),

            Some("(") => {
                let v = p.expr()?;
                if p.next() != Some(")") {
                    return Err(ParseError::MissingParen);
                }
                Ok(v)
            }

            Some(t) => t.parse().map_err(|_| ParseError::NotANumber(t.to_string())),
        })
    }
}

// Put a space on both sides of every operator, then split on whitespace.
fn pad_operators(s: &str) -> String {
    let mut s = s.to_string();
    for op in ["+", "-", "*", "/", "(", ")"] {
        s = s.replace(op, &format!(" {op} "));
    }
    s
}

// (result, deepest nesting reached): the depth is reported even on failure.
fn eval(s: &str) -> (Res, usize) {
    let padded = pad_operators(s);
    let mut p = Parser {
        tokens: padded.split_whitespace().collect(),
        pos: 0,
        depth: 0,
        max_dep: 0,
    };
    let mut result = p.expr();
    if let (Ok(_), Some(t)) = (&result, p.peek()) {
        result = Err(ParseError::Unexpected(t.to_string()));
    }
    (result, p.max_dep)
}

fn main() {
    println!("recursive-descent evaluation -- one function per grammar rule:");
    println!();
    println!("  expr   := term   (('+' | '-') term)*");
    println!("  term   := factor (('*' | '/') factor)*");
    println!("  factor := NUMBER | '-' factor | '(' expr ')'");
    println!();

    let inputs = [
        "1 + 2",
        "2 + 3 * 4",
        "(2 + 3) * 4",
        "10 - 2 - 3",
        "2 * (3 + (4 - 1) * 2)",
        "-5 + 3",
        "-(2 + 3) * -2",
        "100 / 4 / 5",
        "1 + 2 * 3 - 4 / 2",
    ];

    println!("{:<26} {:>14} {:>10}", "expression", "result", "max depth");
    for input in inputs {
        match eval(input) {
            // `{}` on an f64 prints 3 as "3" and 14 as "14", like Go's %g does.
            (Ok(v), depth) => println!("{input:<26} {v:>14} {depth:>10}"),
            (Err(e), depth) => {
                println!("{input:<26} {:>14} {depth:>10}", format!("ERR: {e}"));
            }
        }
    }

    println!();
    println!("errors are refused, not crashed on:");
    println!();
    for input in ["1 +", "(1 + 2", "1 / 0", "2 $ 3"] {
        let (result, _) = eval(input);
        let err = result.expect_err("these inputs are all invalid");
        println!("  {input:<12} -> {err}");
    }

    // Depth guard: 300 nested parens exceeds MAX_DEPTH.
    let deep = "(".repeat(300) + "1" + &")".repeat(300);
    let (result, d) = eval(&deep);
    let err = result.expect_err("300 levels is over the limit");
    println!("  {:<12} -> {err} (reached depth {d})", "300 nested (");

    println!();
    println!("what to take from this:");
    println!();
    println!("  1. the code IS the grammar. Three rules, three functions. When the");
    println!("     input's definition is recursive, so is the natural reader --");
    println!("     this is the shape from example 11's mutually recursive data.");
    println!();
    println!("  2. PRECEDENCE comes from the call graph, not from any precedence");
    println!("     table: expr calls term, so * binds tighter than +. Look at");
    println!("     '2 + 3 * 4' = 14 and '(2 + 3) * 4' = 20.");
    println!();
    println!("  3. ASSOCIATIVITY comes from using a LOOP for the repetition. If expr");
    println!("     recursed on itself for the tail, '10 - 2 - 3' would group as");
    println!("     10 - (2 - 3) = 11 instead of the correct 5.");
    println!();
    println!("  4. the depth guard from example 15 costs four lines and turns a");
    println!("     hostile input from a process kill into an error value.");
    println!();
    println!("that is Part 1. From here on the structures do the work -- and every");
    println!("one of them is built out of what these five lessons measured.");
}
```

**Rust output:**

```
recursive-descent evaluation -- one function per grammar rule:

  expr   := term   (('+' | '-') term)*
  term   := factor (('*' | '/') factor)*
  factor := NUMBER | '-' factor | '(' expr ')'

expression                         result  max depth
1 + 2                                   3          3
2 + 3 * 4                              14          3
(2 + 3) * 4                            20          6
10 - 2 - 3                              5          3
2 * (3 + (4 - 1) * 2)                  18          9
-5 + 3                                 -2          4
-(2 + 3) * -2                          10          7
100 / 4 / 5                             5          3
1 + 2 * 3 - 4 / 2                       5          3

errors are refused, not crashed on:

  1 +          -> unexpected end of expression
  (1 + 2       -> missing closing parenthesis
  1 / 0        -> division by zero
  2 $ 3        -> unexpected "$"
  300 nested ( -> expression nested deeper than 200 (reached depth 201)

what to take from this:

  1. the code IS the grammar. Three rules, three functions. When the
     input's definition is recursive, so is the natural reader --
     this is the shape from example 11's mutually recursive data.

  2. PRECEDENCE comes from the call graph, not from any precedence
     table: expr calls term, so * binds tighter than +. Look at
     '2 + 3 * 4' = 14 and '(2 + 3) * 4' = 20.

  3. ASSOCIATIVITY comes from using a LOOP for the repetition. If expr
     recursed on itself for the tail, '10 - 2 - 3' would group as
     10 - (2 - 3) = 11 instead of the correct 5.

  4. the depth guard from example 15 costs four lines and turns a
     hostile input from a process kill into an error value.

that is Part 1. From here on the structures do the work -- and every
one of them is built out of what these five lessons measured.
```

> **Go vs Rust:** The output is identical to Go, character for character, including every max-depth figure and "reached depth 201". The shape changes where Rust has types for it: each rule returns `Result<f64, ParseError>` and `?` carries the first error up, replacing Go's sticky `p.err` field and its `for p.err == nil` checks; `peek`/`next` return `Option<&str>` instead of `""`; and `defer p.leave()` becomes a `nested` helper that always gives the depth back, since an RAII guard would keep the parser borrowed while a rule needs `&mut self`.

**Complexity:** Θ(tokens) time, Θ(nesting depth) stack · the guard caps the stack at a number you chose rather than one the input chose

---

> That closes **Part 1**. Everything from here builds structures with what these five lessons
> measured — starting at [06 — Dynamic Arrays & Slices](../../06-arrays-slices.md).

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
