# Step 03 — Complexity Analysis · 🟡 Medium

Examples **7–11**: amortized analysis, space (including the call stack), and recurrences.

Examples 7, 8, 10 and 11 are **deterministic** — they count, they don't time. Example 9 measures real
goroutine stack memory, so its numbers move a little between runs.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Why append is amortized O(1)

`🟡 medium` · *The aggregate method*

`append` copies the entire backing array when it resizes, which is Θ(n). So how is it "O(1)"? Count
the **total** work across all n appends and divide. The answer converges to a constant, and this
example also shows the capacity sequence Go really uses — which is not plain doubling.

**Steps:**

1. Detect each resize by watching `cap` change, and record how many elements were copied.
2. Tabulate copies-per-append as n grows by 10×.
3. Print the capacity sequence, then compare against a hypothetical `+1` growth policy.

```go
package main

import "fmt"

// growthOf appends n elements and records exactly what the runtime did:
// every capacity change, and how many elements had to be copied to the new array.
func growthOf(n int) (copies int, caps []int) {
	var xs []int
	for i := 0; i < n; i++ {
		before := cap(xs)
		oldLen := len(xs)
		xs = append(xs, i)
		if cap(xs) != before {
			// The runtime allocated a bigger array and copied the old contents.
			copies += oldLen
			caps = append(caps, cap(xs))
		}
	}
	return copies, caps
}

func main() {
	fmt.Println("append is O(n) when it resizes. So how is it 'O(1) amortized'?")
	fmt.Println("Answer: count the TOTAL work over all n appends, then divide.")
	fmt.Println()

	fmt.Printf("%10s %14s %14s %12s\n", "n", "total copies", "copies/append", "resizes")
	for _, n := range []int{10, 100, 1_000, 100_000, 1_000_000} {
		copies, caps := growthOf(n)
		fmt.Printf("%10d %14d %14.2f %12d\n", n, copies, float64(copies)/float64(n), len(caps))
	}

	fmt.Println()
	fmt.Println("copies/append converges to a constant instead of growing -> O(1) amortized.")
	fmt.Println("resizes grow like log(n), not n: each one is rarer than the last, in proportion")
	fmt.Println("to how much more it costs.")

	_, caps := growthOf(5000)
	fmt.Println()
	fmt.Println("the capacity sequence Go actually used:")
	fmt.Printf("  %v\n", caps)
	fmt.Println()
	fmt.Println("small slices double; above ~256 the factor tapers toward 1.25 to waste less memory.")
	fmt.Println("The trick is not the number 2 -- it is that the factor is a CONSTANT above 1.")
	fmt.Println("Growing by a factor means the copies you just paid for are spread over a")
	fmt.Println("proportional number of free appends. Grow by a fixed +1 instead and every")
	fmt.Println("single append copies everything: the total becomes n^2/2.")

	// Prove that last claim.
	fixedGrowth := 0
	for i := 0; i < 10000; i++ {
		fixedGrowth += i // a +1 growth policy copies i elements on every append
	}
	copies, _ := growthOf(10000)
	fmt.Println()
	fmt.Printf("at n=10000:  doubling policy = %d copies,  +1 policy = %d copies  (%.0fx worse)\n",
		copies, fixedGrowth, float64(fixedGrowth)/float64(copies))
}
```

**Output:**

```
append is O(n) when it resizes. So how is it 'O(1) amortized'?
Answer: count the TOTAL work over all n appends, then divide.

         n   total copies  copies/append      resizes
        10             12           1.20            3
       100            127           1.27            8
      1000           1871           1.87           12
    100000         402079           4.02           28
   1000000        4154015           4.15           38

copies/append converges to a constant instead of growing -> O(1) amortized.
resizes grow like log(n), not n: each one is rarer than the last, in proportion
to how much more it costs.

the capacity sequence Go actually used:
  [4 8 16 32 64 128 256 512 848 1280 1792 2560 3408 5120]

small slices double; above ~256 the factor tapers toward 1.25 to waste less memory.
The trick is not the number 2 -- it is that the factor is a CONSTANT above 1.
Growing by a factor means the copies you just paid for are spread over a
proportional number of free appends. Grow by a fixed +1 instead and every
single append copies everything: the total becomes n^2/2.

at n=10000:  doubling policy = 32412 copies,  +1 policy = 49995000 copies  (1542x worse)
```

Read the capacity sequence: `4 8 16 32 64 128 256` is exact doubling, then `512 848 1280 1792 2560
3408 5120` — a factor drifting toward **1.25**. That is why copies/append settles near 4 rather than
the textbook 1: with growth factor `g` the geometric series sums to `n/(g−1)`, and `1/(1.25−1) = 4`.

**Rust:**

```rust
// grow_of appends n elements and records exactly what Vec did: every capacity
// change, and how many elements a move to the new buffer would have to copy.
// (The allocator may extend a buffer in place and copy nothing; the count is
// what the growth policy forces in the worst case, which is what Go always pays.)
fn grow_of(n: usize) -> (usize, Vec<usize>) {
    let mut copies = 0;
    let mut caps = Vec::new();
    let mut xs: Vec<usize> = Vec::new();
    for i in 0..n {
        let before = xs.capacity();
        let old_len = xs.len();
        xs.push(i);
        if xs.capacity() != before {
            // Vec asked for a bigger buffer and moved the old contents over.
            copies += old_len;
            caps.push(xs.capacity());
        }
    }
    (copies, caps)
}

fn main() {
    println!("push is O(n) when it resizes. So how is it 'O(1) amortized'?");
    println!("Answer: count the TOTAL work over all n pushes, then divide.");
    println!();

    println!(
        "{:>10} {:>14} {:>14} {:>12}",
        "n", "total copies", "copies/push", "resizes"
    );
    for n in [10, 100, 1_000, 100_000, 1_000_000] {
        let (copies, caps) = grow_of(n);
        println!(
            "{:>10} {:>14} {:>14.2} {:>12}",
            n,
            copies,
            copies as f64 / n as f64,
            caps.len()
        );
    }

    println!();
    println!("copies/push stays between 1 and 2 instead of growing -> O(1) amortized.");
    println!("resizes grow like log(n), not n: each one is rarer than the last, in proportion");
    println!("to how much more it costs.");

    let (_, caps) = grow_of(5000);
    println!();
    println!("the capacity sequence Vec actually used:");
    println!("  {caps:?}");
    println!();
    println!("every step above is an exact doubling, starting from 4 (for 8-byte elements).");
    println!("The trick is not the number 2 -- it is that the factor is a CONSTANT above 1.");
    println!("Growing by a factor means the copies you just paid for are spread over a");
    println!("proportional number of free pushes. Grow by a fixed +1 instead and every");
    println!("single push copies everything: the total becomes n^2/2.");

    // Why the constant wobbles (1.02, 1.31, 1.05 above): with doubling, the copies
    // are 4 + 8 + ... + (final capacity / 2) = final capacity - 4, and the final
    // capacity is the next power of two at or above n.
    let (full, _) = grow_of(65_536);
    let (just_past, _) = grow_of(65_537);
    println!();
    println!("where n falls between two powers of two sets the constant:");
    println!(
        "  n = 65536 (buffer just filled):       {full} copies = {:.2} per push",
        full as f64 / 65_536.0
    );
    println!(
        "  n = 65537 (one push past a resize):   {just_past} copies = {:.2} per push",
        just_past as f64 / 65_537.0
    );

    // Prove that last claim.
    let fixed_growth: u64 = (0..10_000u64).sum(); // a +1 growth policy copies i elements on every push
    let (copies, _) = grow_of(10_000);
    println!();
    println!(
        "at n=10000:  doubling policy = {copies} copies,  +1 policy = {fixed_growth} copies  ({:.0}x worse)",
        fixed_growth as f64 / copies as f64
    );
}
```

**Rust output:**

```
push is O(n) when it resizes. So how is it 'O(1) amortized'?
Answer: count the TOTAL work over all n pushes, then divide.

         n   total copies    copies/push      resizes
        10             12           1.20            3
       100            124           1.24            6
      1000           1020           1.02            9
    100000         131068           1.31           16
   1000000        1048572           1.05           19

copies/push stays between 1 and 2 instead of growing -> O(1) amortized.
resizes grow like log(n), not n: each one is rarer than the last, in proportion
to how much more it costs.

the capacity sequence Vec actually used:
  [4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192]

every step above is an exact doubling, starting from 4 (for 8-byte elements).
The trick is not the number 2 -- it is that the factor is a CONSTANT above 1.
Growing by a factor means the copies you just paid for are spread over a
proportional number of free pushes. Grow by a fixed +1 instead and every
single push copies everything: the total becomes n^2/2.

where n falls between two powers of two sets the constant:
  n = 65536 (buffer just filled):       65532 copies = 1.00 per push
  n = 65537 (one push past a resize):   131068 copies = 2.00 per push

at n=10000:  doubling policy = 16380 copies,  +1 policy = 49995000 copies  (3052x worse)
```

> **Go vs Rust:** The counts differ because the growth policies differ, not the language: Go tapers from 2x toward 1.25x (4.02 copies per append at n=100,000, 28 resizes, capacity 5120 after 5,000 elements), while `Vec` doubles at every size (1.31 copies per push at the same n, 16 resizes, capacity 8192 after 5,000). Fewer copies costs more spare capacity; both are O(1) amortized. The "copies" here are what a move to the new buffer would copy; Rust's `realloc` can sometimes extend in place, which this count does not capture.

**Complexity:** one append Θ(n) worst case · n appends Θ(n) total → **Θ(1) amortized** · the `+1` policy is Θ(n²) total, 1542× more copies at n=10,000

---

## 8. Amortized is not average, and is not worst case

`🟡 medium` · *Three different claims about the same operation*

The same 100,000 appends, viewed as a distribution instead of a total. **99.975%** of them are free;
**25** of them copy; one copies **88,064 elements**. "Amortized O(1)" and "worst case O(n)" are both
true, simultaneously, about the same line of code.

**Steps:**

1. Record the copy cost of every individual append.
2. Report the amortized cost, the worst single cost, and how many appends were expensive.
3. State the aggregate-method bound, and work out why the constant is ~4 and not ~1.

```go
package main

import "fmt"

// perAppendCost returns the copy cost of every single append, in order.
func perAppendCost(n int) []int {
	costs := make([]int, 0, n)
	var xs []int
	for i := 0; i < n; i++ {
		before := cap(xs)
		oldLen := len(xs)
		xs = append(xs, i)
		if cap(xs) != before {
			costs = append(costs, oldLen) // resized: copied oldLen elements
		} else {
			costs = append(costs, 0) // free
		}
	}
	return costs
}

func main() {
	const n = 100_000
	costs := perAppendCost(n)

	total, max, expensive := 0, 0, 0
	maxAt := 0
	for i, c := range costs {
		total += c
		if c > 0 {
			expensive++
		}
		if c > max {
			max, maxAt = c, i
		}
	}

	fmt.Printf("%d appends:\n\n", n)
	fmt.Printf("  amortized cost (total/n)     %10.2f copies\n", float64(total)/float64(n))
	fmt.Printf("  worst single append          %10d copies  (at append #%d)\n", max, maxAt)
	fmt.Printf("  appends that copied anything %10d  (%.3f%% of them)\n",
		expensive, 100*float64(expensive)/float64(n))
	fmt.Printf("  appends that were free       %10d\n", n-expensive)

	fmt.Println()
	fmt.Println("AMORTIZED is not AVERAGE and it is definitely not WORST CASE:")
	fmt.Printf("  - amortized O(1)  : any sequence of n appends costs O(n) total. Guaranteed.\n")
	fmt.Printf("  - worst case O(n) : one unlucky append copied %d elements.\n", max)
	fmt.Println()
	fmt.Println("both statements are true at the same time. Amortized analysis bounds the")
	fmt.Println("SEQUENCE; it says nothing about any individual operation.")

	fmt.Println()
	fmt.Println("why it matters: if you care about throughput, amortized O(1) is the number.")
	fmt.Println("If you care about p99 LATENCY, that one expensive append is your tail --")
	fmt.Println("and example 15 measures it in nanoseconds.")

	fmt.Println()
	fmt.Println("the aggregate method, in one line:")
	fmt.Printf("  total copies %d <= 5n = %d, so cost per operation <= 5 = O(1).\n", total, 5*n)
	fmt.Println()
	fmt.Println("why 5 and not 2? With a growth factor g, the copies form a geometric series")
	fmt.Println("summing to n/(g-1). Textbook doubling (g=2) gives n, i.e. 1 copy per append.")
	fmt.Printf("Go tapers g toward 1.25 for large slices, so 1/(1.25-1) = 4 -- and we measured %.2f.\n",
		float64(total)/float64(n))
	fmt.Println("A smaller factor wastes less memory and copies more. Both are still O(1):")
	fmt.Println("what matters for amortization is only that g is a CONSTANT greater than 1.")
}
```

**Output:**

```
100000 appends:

  amortized cost (total/n)           4.02 copies
  worst single append               88064 copies  (at append #88064)
  appends that copied anything         25  (0.025% of them)
  appends that were free            99975

AMORTIZED is not AVERAGE and it is definitely not WORST CASE:
  - amortized O(1)  : any sequence of n appends costs O(n) total. Guaranteed.
  - worst case O(n) : one unlucky append copied 88064 elements.

both statements are true at the same time. Amortized analysis bounds the
SEQUENCE; it says nothing about any individual operation.

why it matters: if you care about throughput, amortized O(1) is the number.
If you care about p99 LATENCY, that one expensive append is your tail --
and example 15 measures it in nanoseconds.

the aggregate method, in one line:
  total copies 402076 <= 5n = 500000, so cost per operation <= 5 = O(1).

why 5 and not 2? With a growth factor g, the copies form a geometric series
summing to n/(g-1). Textbook doubling (g=2) gives n, i.e. 1 copy per append.
Go tapers g toward 1.25 for large slices, so 1/(1.25-1) = 4 -- and we measured 4.02.
A smaller factor wastes less memory and copies more. Both are still O(1):
what matters for amortization is only that g is a CONSTANT greater than 1.
```

Three numbers, three different truths about one operation: **4.02** (amortized), **88,064** (worst
case), **0.025%** (how often it matters). Which one you quote depends entirely on whether you are
optimizing throughput or latency.

**Rust:**

```rust
// per_push_cost returns the copy cost of every single push, in order, plus the
// final capacity. The very first push allocates but copies nothing (old_len = 0),
// so it is counted as free.
fn per_push_cost(n: usize) -> (Vec<usize>, usize) {
    let mut costs = Vec::with_capacity(n);
    let mut xs: Vec<usize> = Vec::new();
    for i in 0..n {
        let before = xs.capacity();
        let old_len = xs.len();
        xs.push(i);
        if xs.capacity() != before {
            costs.push(old_len); // resized: moved old_len elements
        } else {
            costs.push(0); // free
        }
    }
    (costs, xs.capacity())
}

fn main() {
    const N: usize = 100_000;
    let (costs, final_cap) = per_push_cost(N);

    let total: usize = costs.iter().sum();
    let expensive = costs.iter().filter(|&&c| c > 0).count();
    // max_by_key returns the LAST maximum on ties; costs are strictly increasing
    // among the nonzero entries, so the maximum is unique.
    let (max_at, &max) = costs
        .iter()
        .enumerate()
        .max_by_key(|&(_, &c)| c)
        .expect("N > 0");

    println!("{N} pushes:\n");
    println!(
        "  amortized cost (total/n)     {:>10.2} copies",
        total as f64 / N as f64
    );
    println!("  worst single push            {max:>10} copies  (at push index {max_at})");
    println!(
        "  pushes that copied anything  {expensive:>10}  ({:.3}% of them)",
        100.0 * expensive as f64 / N as f64
    );
    println!("  pushes that were free        {:>10}", N - expensive);

    println!();
    println!("AMORTIZED is not AVERAGE and it is definitely not WORST CASE:");
    println!("  - amortized O(1)  : any sequence of n pushes costs O(n) total. Guaranteed.");
    println!("  - worst case O(n) : one unlucky push copied {max} elements.");
    println!();
    println!("both statements are true at the same time. Amortized analysis bounds the");
    println!("SEQUENCE; it says nothing about any individual operation.");

    println!();
    println!("why it matters: if you care about throughput, amortized O(1) is the number.");
    println!("If you care about p99 LATENCY, that one expensive push is your tail --");
    println!("and example 15 measures it in nanoseconds.");

    println!();
    println!("the aggregate method, in one line:");
    println!(
        "  total copies {total} <= 2n = {}, so cost per operation <= 2 = O(1).",
        2 * N
    );
    println!();
    println!("why 2? With a growth factor g, each resize copies g times more than the one");
    println!("before it, so the copies form a geometric series dominated by its last term:");
    println!("between n/(g-1) and n*g/(g-1) in total. Vec doubles (g=2), so that is between");
    println!("1 and 2 copies per push. Here the copies are 4 + 8 + ... + 65536, which is");
    println!(
        "final capacity - 4 = {} - 4 = {} for n={N}, so we measured {:.2}.",
        final_cap,
        final_cap - 4,
        total as f64 / N as f64
    );
    println!("A smaller factor wastes less memory and copies more. Both are still O(1):");
    println!("what matters for amortization is only that g is a CONSTANT greater than 1.");
}
```

**Rust output:**

```
100000 pushes:

  amortized cost (total/n)           1.31 copies
  worst single push                 65536 copies  (at push index 65536)
  pushes that copied anything          15  (0.015% of them)
  pushes that were free             99985

AMORTIZED is not AVERAGE and it is definitely not WORST CASE:
  - amortized O(1)  : any sequence of n pushes costs O(n) total. Guaranteed.
  - worst case O(n) : one unlucky push copied 65536 elements.

both statements are true at the same time. Amortized analysis bounds the
SEQUENCE; it says nothing about any individual operation.

why it matters: if you care about throughput, amortized O(1) is the number.
If you care about p99 LATENCY, that one expensive push is your tail --
and example 15 measures it in nanoseconds.

the aggregate method, in one line:
  total copies 131068 <= 2n = 200000, so cost per operation <= 2 = O(1).

why 2? With a growth factor g, each resize copies g times more than the one
before it, so the copies form a geometric series dominated by its last term:
between n/(g-1) and n*g/(g-1) in total. Vec doubles (g=2), so that is between
1 and 2 copies per push. Here the copies are 4 + 8 + ... + 65536, which is
final capacity - 4 = 131072 - 4 = 131068 for n=100000, so we measured 1.31.
A smaller factor wastes less memory and copies more. Both are still O(1):
what matters for amortization is only that g is a CONSTANT greater than 1.
```

> **Go vs Rust:** Same shape, different growth policy: `Vec` doubles, so the amortized cost is 1.31 copies per push (Go: 4.02), the worst single push copies 65,536 elements (Go: 88,064), and only 15 pushes copy anything (Go: 25). The three-way split of amortized, worst case and "how often" is identical; only the constants moved, and the bound I print is 2n instead of Go's 5n. Both languages report the failing push by its 0-based index.

**Complexity:** amortized Θ(1) · worst-case single op Θ(n) · number of expensive ops Θ(log n) · total Θ(n)

---

## 9. The call stack is space

`🟡 medium` · *Measuring recursion depth in bytes*

Recursion allocates memory without any line of code saying so. This measures how much: ~33–52 bytes
per frame, so a million-deep recursion is holding **33 MB** of goroutine stack.

The measurement detail matters as much as the result: each depth is measured in a **fresh goroutine**.
Go grows stacks but is slow to shrink them, so measuring on the main goroutine means the first deep
call sets a high-water mark that hides every later measurement behind it.

**Steps:**

1. Read `runtime.MemStats.StackInuse` as a baseline after a GC.
2. Recurse to a given depth inside a new goroutine, and read the counter at the bottom.
3. Divide by the depth to get bytes per frame.

```go
package main

import (
	"fmt"
	"runtime"
)

// sumRecursive uses one stack frame per element: O(n) space, invisible in the code.
func sumRecursive(xs []int) int {
	if len(xs) == 0 {
		return 0
	}
	return xs[0] + sumRecursive(xs[1:])
}

// sumIterative does the same arithmetic in O(1) space.
func sumIterative(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

// stackInUse reports how many bytes of goroutine stack the runtime holds.
func stackInUse() uint64 {
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.StackInuse
}

func deepest(depth int, atBottom func()) {
	if depth == 0 {
		atBottom()
		return
	}
	deepest(depth-1, atBottom)
}

// measureAtDepth recurses in a FRESH goroutine, which starts with a small stack.
// Measuring on the main goroutine does not work: Go grows stacks but is slow to
// shrink them, so the first deep call sets a high-water mark that hides every
// later measurement behind it.
func measureAtDepth(depth int) uint64 {
	var peak uint64
	done := make(chan struct{})
	go func() {
		defer close(done)
		deepest(depth, func() { peak = stackInUse() })
	}()
	<-done
	return peak
}

func main() {
	runtime.GC()
	base := stackInUse()

	fmt.Println("a recursion's stack frames are memory, even though no line of code")
	fmt.Println("in it says 'allocate'. Each frame is measured here in a fresh goroutine:")
	fmt.Println()
	fmt.Printf("goroutine stacks in use, baseline: %8.1f KB\n\n", float64(base)/1024)
	fmt.Printf("%12s %14s %18s\n", "depth", "stacks (KB)", "bytes per frame")
	for _, depth := range []int{1_000, 10_000, 100_000, 1_000_000} {
		peak := measureAtDepth(depth)
		fmt.Printf("%12d %14.1f %18.0f\n",
			depth, float64(peak)/1024, float64(peak-base)/float64(depth))
		runtime.GC() // let the finished goroutine's stack go back
	}

	xs := make([]int, 100_000)
	for i := range xs {
		xs[i] = 1
	}
	fmt.Println()
	fmt.Println("both of these compute the same sum in the same O(n) time:")
	fmt.Printf("  sumRecursive = %d   (O(n) space -- 100,000 live frames)\n", sumRecursive(xs))
	fmt.Printf("  sumIterative = %d   (O(1) space)\n", sumIterative(xs))

	fmt.Println()
	fmt.Println("space complexity counts the CALL STACK. It is the cost people forget,")
	fmt.Println("because unlike a make() it never appears in the source.")
	fmt.Println()
	fmt.Println("Go grows goroutine stacks dynamically (8 KB at first, doubling, to a 1 GB")
	fmt.Println("default limit), so deep recursion does not fail fast the way it does in C --")
	fmt.Println("it quietly consumes memory instead. Past the limit the runtime does not")
	fmt.Println("panic, it dies: 'fatal error: stack overflow', which no recover() can catch.")
	fmt.Println()
	fmt.Println("rule: recursion depth IS space. Balanced-tree recursion is O(log n) and fine;")
	fmt.Println("recursion down a list or an array is O(n) -- write the loop instead.")
}
```

**Sample output** (stack accounting shifts a little between runs):

```
a recursion's stack frames are memory, even though no line of code
in it says 'allocate'. Each frame is measured here in a fresh goroutine:

goroutine stacks in use, baseline:    352.0 KB

       depth    stacks (KB)    bytes per frame
        1000          384.0                 33
       10000          864.0                 52
      100000         4448.0                 42
     1000000        33120.0                 34

both of these compute the same sum in the same O(n) time:
  sumRecursive = 100000   (O(n) space -- 100,000 live frames)
  sumIterative = 100000   (O(1) space)

space complexity counts the CALL STACK. It is the cost people forget,
because unlike a make() it never appears in the source.

Go grows goroutine stacks dynamically (8 KB at first, doubling, to a 1 GB
default limit), so deep recursion does not fail fast the way it does in C --
it quietly consumes memory instead. Past the limit the runtime does not
panic, it dies: 'fatal error: stack overflow', which no recover() can catch.

rule: recursion depth IS space. Balanced-tree recursion is O(log n) and fine;
recursion down a list or an array is O(n) -- write the loop instead.
```

Bytes-per-frame wobbles (33, 52, 42, 34) because stacks grow by **doubling**, so the measurement
catches a different amount of slack each time. The trend is what matters: **stack memory is linear in
depth**, and at depth 1,000,000 it is 33 MB.

**Rust:**

```rust
use std::hint::black_box;
use std::thread;

// A fresh stack for every measurement. Unlike Go's goroutine stacks, a Rust
// thread's stack does not grow: it is one fixed-size region, reserved when the
// thread is created (virtual memory; only the pages a recursion touches are
// used). The main thread gets ~8 MiB from the OS and a spawned thread 2 MiB by
// default, so a million-deep recursion needs a thread with an explicit size.
const STACK: usize = 256 << 20;

fn on_fresh_stack<R: Send>(f: impl FnOnce() -> R + Send) -> R {
    thread::scope(|s| {
        thread::Builder::new()
            .stack_size(STACK)
            .spawn_scoped(s, f)
            .expect("spawn")
            .join()
            .expect("join")
    })
}

// Go reads runtime.MemStats.StackInuse; Rust has no such counter, so we read the
// stack POINTER instead: the address of a local variable in the current frame.
// (A pointer-to-integer cast is safe Rust; the stack grows toward lower
// addresses, abs_diff makes the subtraction direction-agnostic.)
#[inline(never)]
fn stack_addr() -> usize {
    let marker = 0u8;
    black_box(&marker) as *const u8 as usize
}

// stack_used is the distance in bytes between the stack pointer here and the one
// that `f` records at the bottom of its recursion.
fn stack_used(f: impl FnOnce(&mut usize)) -> usize {
    let top = stack_addr();
    let mut bottom = top;
    f(&mut bottom);
    top.abs_diff(bottom)
}

// sum_recursive uses one stack frame per element: O(n) space, invisible in the code.
// `bottom` is the measuring probe. With PIN = true, black_box(()) after the call
// makes the recursive call NOT the last thing the function does. With PIN = false
// it is the natural shape, `xs[0] + sum_recursive(rest)`, which LLVM may rewrite
// into a loop: Rust does not promise tail-call elimination either way.
#[inline(never)]
fn sum_recursive<const PIN: bool>(xs: &[i64], bottom: &mut usize) -> i64 {
    if xs.is_empty() {
        *bottom = stack_addr();
        return 0;
    }
    let sum = xs[0] + sum_recursive::<PIN>(&xs[1..], bottom);
    if PIN {
        black_box(());
    }
    sum
}

// sum_iterative does the same arithmetic in O(1) space.
fn sum_iterative(xs: &[i64]) -> i64 {
    xs.iter().sum()
}

// deepest recurses `depth` frames and calls at_bottom() at the bottom. The
// black_box(()) pins the frames, as above.
#[inline(never)]
fn deepest(depth: usize, at_bottom: &mut dyn FnMut()) {
    if depth == 0 {
        at_bottom();
        return;
    }
    deepest(depth - 1, at_bottom);
    black_box(());
}

fn main() {
    // Baseline: the distance between two probes with NO recursion in between
    // (the probe's own frames). Subtracted below, like Go's baseline StackInuse.
    let base = on_fresh_stack(|| stack_used(|b| deepest(0, &mut || *b = stack_addr())));

    println!("a recursion's stack frames are memory, even though no line of code");
    println!("in it says 'allocate'. Each depth is measured on a fresh thread stack,");
    println!("by comparing the stack pointer at the top and at the bottom:");
    println!();
    println!("probe overhead, baseline: {base:>8} bytes\n");
    println!(
        "{:>12} {:>14} {:>18}",
        "depth", "stack (KB)", "bytes per frame"
    );
    let mut per_frame = 0;
    for depth in [1_000, 10_000, 100_000, 1_000_000] {
        let used = on_fresh_stack(|| stack_used(|b| deepest(depth, &mut || *b = stack_addr())));
        per_frame = (used - base) / depth;
        println!(
            "{:>12} {:>14.1} {:>18}",
            depth,
            (used - base) as f64 / 1024.0,
            per_frame
        );
    }

    let xs = vec![1i64; 100_000];
    // The same recursion, measured with its depth-0 baseline subtracted.
    let measure = |pin: bool| {
        on_fresh_stack(|| {
            let run = |xs: &[i64]| {
                let mut sum = 0;
                let used = stack_used(|b| {
                    sum = if pin {
                        sum_recursive::<true>(xs, b)
                    } else {
                        sum_recursive::<false>(xs, b)
                    }
                });
                (sum, used)
            };
            let (_, base) = run(&xs[..0]);
            let (sum, used) = run(&xs);
            (sum, used - base)
        })
    };
    let (pinned, pinned_bytes) = measure(true);
    let (natural, natural_bytes) = measure(false);
    let verdict = if natural_bytes == 0 {
        "rustc -O turned the recursion into a loop"
    } else {
        "the recursion kept its frames"
    };

    println!();
    println!("both of these compute the same sum in the same O(n) time:");
    println!(
        "  sum_recursive, frames pinned = {pinned}   (O(n) space -- 100,000 live frames: {} KB, {} bytes each)",
        pinned_bytes / 1024,
        pinned_bytes / xs.len()
    );
    println!(
        "  sum_recursive, natural shape = {natural}   ({verdict}: {natural_bytes} bytes of stack)"
    );
    println!(
        "  sum_iterative                = {}   (O(1) space)",
        sum_iterative(&xs)
    );

    println!();
    println!("space complexity counts the CALL STACK. It is the cost people forget,");
    println!("because unlike a Vec::with_capacity it never appears in the source.");
    println!();
    println!("A Rust thread's stack is one fixed-size region, so deep recursion cannot grow");
    println!("into more memory the way it does in Go: it runs off the end, and the process");
    println!("aborts with 'thread has overflowed its stack' (std's documented behaviour; not");
    println!("triggered here, since it kills the program, and catch_unwind cannot catch it).");
    println!("At the {per_frame} bytes per frame measured in the table:");
    println!(
        "  the main thread's usual 8 MiB (set by the OS) holds about {} frames,",
        (8usize << 20) / per_frame
    );
    println!(
        "  a spawned thread's default 2 MiB holds about {} frames.",
        (2usize << 20) / per_frame
    );
    println!();
    println!("rule: recursion depth IS space. Balanced-tree recursion is O(log n) and fine;");
    println!("recursion down a list or an array is O(n) -- write the loop instead.");
    println!("An optimizer may remove the recursion for you, but it is not a promise.");
}
```

**Rust sample output:**

```
a recursion's stack frames are memory, even though no line of code
in it says 'allocate'. Each depth is measured on a fresh thread stack,
by comparing the stack pointer at the top and at the bottom:

probe overhead, baseline:       32 bytes

       depth     stack (KB)    bytes per frame
        1000           15.6                 16
       10000          156.2                 16
      100000         1562.5                 16
     1000000        15625.0                 16

both of these compute the same sum in the same O(n) time:
  sum_recursive, frames pinned = 100000   (O(n) space -- 100,000 live frames: 3125 KB, 32 bytes each)
  sum_recursive, natural shape = 100000   (rustc -O turned the recursion into a loop: 0 bytes of stack)
  sum_iterative                = 100000   (O(1) space)

space complexity counts the CALL STACK. It is the cost people forget,
because unlike a Vec::with_capacity it never appears in the source.

A Rust thread's stack is one fixed-size region, so deep recursion cannot grow
into more memory the way it does in Go: it runs off the end, and the process
aborts with 'thread has overflowed its stack' (std's documented behaviour; not
triggered here, since it kills the program, and catch_unwind cannot catch it).
At the 16 bytes per frame measured in the table:
  the main thread's usual 8 MiB (set by the OS) holds about 524288 frames,
  a spawned thread's default 2 MiB holds about 131072 frames.

rule: recursion depth IS space. Balanced-tree recursion is O(log n) and fine;
recursion down a list or an array is O(n) -- write the loop instead.
An optimizer may remove the recursion for you, but it is not a promise.
```

> **Go vs Rust:** Go reads `runtime.MemStats.StackInuse` and needs a fresh goroutine because its stacks grow and shrink lazily; a Rust thread has one fixed-size stack, so the twin reads the stack pointer at the top and bottom of the recursion and gets a flat 16 bytes per frame for `deepest` and 32 for `sum_recursive` (Go: 33-52, wobbling). Rust adds a twist Go does not have: under `-O` the natural `xs[0] + sum_recursive(&xs[1..])` was compiled into a loop (0 bytes of stack), a `split_first` spelling of the same recursion kept all its frames in my test, and a build without `-O` kept them too (about 64 bytes each) -- so a recursion's space cost is not always readable from the source. The stack-overflow abort described in the output was not triggered, because it kills the process.

**Complexity:** `sumRecursive` Θ(n) time, **Θ(n) space** · `sumIterative` Θ(n) time, **Θ(1) space** · at ~40 bytes/frame, a 1 GB stack limit is reached around depth 25,000,000

---

## 10. Auxiliary space vs total space

`🟡 medium` · *Θ(1), Θ(log n), Θ(n) side by side*

"Space complexity" nearly always means **auxiliary** space — what the algorithm needs *beyond* its
input. Three sorts, three different answers, and the difference is the entire argument for in-place
sorting when the data is large.

**Steps:**

1. Instrument allocation with a counter that tracks the **peak**, not the total (freed memory doesn't count twice).
2. Run merge sort (buffer), insertion sort (in place), and quicksort (recursion only).
3. Compare how each scales with n.

```go
package main

import "fmt"

// A counter for extra memory: every int the algorithm allocates beyond its input.
var extraInts int

// peakExtra tracks the high-water mark, because auxiliary space is a PEAK,
// not a total -- memory that is freed does not count twice.
var peakExtra int

func alloc(n int) []int {
	extraInts += n
	if extraInts > peakExtra {
		peakExtra = extraInts
	}
	return make([]int, n)
}

func free(n int) { extraInts -= n }

// mergeSort allocates a scratch buffer at every merge: O(n) auxiliary space.
func mergeSort(xs []int) []int {
	if len(xs) <= 1 {
		return xs
	}
	mid := len(xs) / 2
	left := mergeSort(xs[:mid])
	right := mergeSort(xs[mid:])

	out := alloc(len(xs))
	i, j := 0, 0
	for k := range out {
		switch {
		case i >= len(left):
			out[k] = right[j]
			j++
		case j >= len(right):
			out[k] = left[i]
			i++
		case left[i] <= right[j]:
			out[k] = left[i]
			i++
		default:
			out[k] = right[j]
			j++
		}
	}
	if len(left) > 1 {
		free(len(left))
	}
	if len(right) > 1 {
		free(len(right))
	}
	return out
}

// insertionSort sorts in place: O(1) auxiliary space, whatever n is.
func insertionSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		xs[j+1] = v
	}
}

// quickSortDepth reports the maximum recursion depth -- quicksort's auxiliary
// space is its stack, O(log n) when the pivots split evenly.
func quickSortDepth(xs []int, depth int) int {
	if len(xs) <= 1 {
		return depth
	}
	mid := len(xs) / 2 // pretend the pivot always splits evenly
	l := quickSortDepth(xs[:mid], depth+1)
	r := quickSortDepth(xs[mid:], depth+1)
	if l > r {
		return l
	}
	return r
}

func input(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = (i * 7919) % n
	}
	return xs
}

func main() {
	fmt.Println("'space complexity' almost always means AUXILIARY space:")
	fmt.Println("extra memory beyond the input itself. The input does not count.")
	fmt.Println()

	fmt.Printf("%10s %18s %18s %18s\n", "n", "merge sort peak", "insertion sort", "quicksort depth")
	for _, n := range []int{16, 256, 4096, 65536} {
		extraInts, peakExtra = 0, 0
		mergeSort(input(n))
		mergePeak := peakExtra

		xs := input(n)
		insertionSort(xs) // touches nothing but xs

		depth := quickSortDepth(input(n), 0)

		fmt.Printf("%10d %18d %18d %18d\n", n, mergePeak, 0, depth)
	}

	fmt.Println()
	fmt.Println("  merge sort      O(n) auxiliary  -- the scratch buffer scales with the input")
	fmt.Println("  insertion sort  O(1) auxiliary  -- swaps in place, allocates nothing")
	fmt.Println("  quicksort       O(log n)        -- no buffer, but the recursion IS space")
	fmt.Println()
	fmt.Println("this is the trade behind 'in-place': merge sort is the faster, stable sort,")
	fmt.Println("and it costs a whole extra copy of your data. At 4 GB of input that decides it.")
	fmt.Println()
	fmt.Println("(quicksort's O(log n) holds only while the pivots split evenly. Bad pivots")
	fmt.Println(" give depth n -- O(n) stack, which is the other half of its worst case.)")
}
```

**Output:**

```
'space complexity' almost always means AUXILIARY space:
extra memory beyond the input itself. The input does not count.

         n    merge sort peak     insertion sort    quicksort depth
        16                 32                  0                  4
       256                512                  0                  8
      4096               8192                  0                 12
     65536             131072                  0                 16

  merge sort      O(n) auxiliary  -- the scratch buffer scales with the input
  insertion sort  O(1) auxiliary  -- swaps in place, allocates nothing
  quicksort       O(log n)        -- no buffer, but the recursion IS space

this is the trade behind 'in-place': merge sort is the faster, stable sort,
and it costs a whole extra copy of your data. At 4 GB of input that decides it.

(quicksort's O(log n) holds only while the pivots split evenly. Bad pivots
 give depth n -- O(n) stack, which is the other half of its worst case.)
```

Merge sort's peak is exactly **2n** here (this naive version holds a level's buffers while building
the next); the quicksort depth column is exactly **log₂n**. Both are the classes you'd write down.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::borrow::Cow;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Go's version counted extra memory by hand (alloc/free helpers and two package
// globals). Rust has no mutable globals and needs no helpers: a counting global
// allocator sees every heap byte the program really asks for, and tracks the
// PEAK, because auxiliary space is a peak, not a total -- memory that is freed
// does not count twice.
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live
static PEAK: AtomicUsize = AtomicUsize::new(0); // high-water mark of LIVE

unsafe impl GlobalAlloc for Counting {
    // `unsafe` is required by the GlobalAlloc trait; every call just forwards to System.
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        let live = LIVE.fetch_add(l.size(), Relaxed) + l.size();
        PEAK.fetch_max(live, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// Peak extra heap, in i64 elements, that `f` needs beyond what was live when it started.
fn peak_extra_elems<R>(f: impl FnOnce() -> R) -> usize {
    let base = LIVE.load(Relaxed);
    PEAK.store(base, Relaxed);
    drop(f());
    (PEAK.load(Relaxed) - base) / size_of::<i64>()
}

// merge_sort allocates a scratch buffer at every merge: O(n) auxiliary space.
// A slice of length <= 1 is returned borrowed (Cow::Borrowed): no allocation, like
// Go returning the same slice. There are no free() calls: `left` and `right` are
// dropped at the end of the function, which is exactly when Go freed them by hand.
fn merge_sort(xs: &[i64]) -> Cow<'_, [i64]> {
    if xs.len() <= 1 {
        return Cow::Borrowed(xs);
    }
    let (l, r) = xs.split_at(xs.len() / 2);
    let left = merge_sort(l);
    let right = merge_sort(r);

    let mut out = Vec::with_capacity(xs.len());
    let (mut i, mut j) = (0, 0);
    while out.len() < xs.len() {
        if j >= right.len() || (i < left.len() && left[i] <= right[j]) {
            out.push(left[i]);
            i += 1;
        } else {
            out.push(right[j]);
            j += 1;
        }
    }
    Cow::Owned(out)
}

// insertion_sort sorts in place: O(1) auxiliary space, whatever n is.
fn insertion_sort(xs: &mut [i64]) {
    for i in 1..xs.len() {
        let v = xs[i];
        let mut j = i;
        while j > 0 && xs[j - 1] > v {
            xs[j] = xs[j - 1];
            j -= 1;
        }
        xs[j] = v;
    }
}

// quick_sort_depth reports the maximum recursion depth -- quicksort's auxiliary
// space is its stack, O(log n) when the pivots split evenly.
fn quick_sort_depth(xs: &[i64], depth: usize) -> usize {
    if xs.len() <= 1 {
        return depth;
    }
    let (l, r) = xs.split_at(xs.len() / 2); // pretend the pivot always splits evenly
    quick_sort_depth(l, depth + 1).max(quick_sort_depth(r, depth + 1))
}

fn input(n: usize) -> Vec<i64> {
    (0..n).map(|i| ((i * 7919) % n) as i64).collect()
}

fn main() {
    println!("'space complexity' almost always means AUXILIARY space:");
    println!("extra memory beyond the input itself. The input does not count.");
    println!();

    println!(
        "{:>10} {:>18} {:>18} {:>18}",
        "n", "merge sort peak", "insertion sort", "quicksort depth"
    );
    for n in [16, 256, 4096, 65536] {
        let xs = input(n); // allocated BEFORE measuring: the input is not auxiliary
        let merge_peak = peak_extra_elems(|| merge_sort(&xs).into_owned());

        let mut ys = input(n);
        let insertion_peak = peak_extra_elems(|| insertion_sort(&mut ys)); // measured, not assumed

        let depth = quick_sort_depth(&xs, 0);

        println!("{n:>10} {merge_peak:>18} {insertion_peak:>18} {depth:>18}");
    }

    println!();
    println!("  merge sort      O(n) auxiliary  -- the scratch buffer scales with the input");
    println!("  insertion sort  O(1) auxiliary  -- swaps in place, allocates nothing");
    println!("  quicksort       O(log n)        -- no buffer, but the recursion IS space");
    println!();
    println!("this is the trade behind 'in-place': merge sort is the faster, stable sort,");
    println!("and it costs a whole extra copy of your data. At 4 GB of input that decides it.");
    println!();
    println!("(quicksort's O(log n) holds only while the pivots split evenly. Bad pivots");
    println!(" give depth n -- O(n) stack, which is the other half of its worst case.)");
}
```

**Rust output:**

```
'space complexity' almost always means AUXILIARY space:
extra memory beyond the input itself. The input does not count.

         n    merge sort peak     insertion sort    quicksort depth
        16                 32                  0                  4
       256                512                  0                  8
      4096               8192                  0                 12
     65536             131072                  0                 16

  merge sort      O(n) auxiliary  -- the scratch buffer scales with the input
  insertion sort  O(1) auxiliary  -- swaps in place, allocates nothing
  quicksort       O(log n)        -- no buffer, but the recursion IS space

this is the trade behind 'in-place': merge sort is the faster, stable sort,
and it costs a whole extra copy of your data. At 4 GB of input that decides it.

(quicksort's O(log n) holds only while the pivots split evenly. Bad pivots
 give depth n -- O(n) stack, which is the other half of its worst case.)
```

> **Go vs Rust:** Go tracked extra memory by hand (two package-level globals plus `alloc`/`free` helpers) and typed the `0` for insertion sort into its `Printf`. The Rust twin has no mutable globals: a counting `#[global_allocator]` records the real peak of live heap bytes, so the insertion-sort `0` is measured, and `left`/`right` are freed by `Drop` where Go called `free()` by hand. The table is identical to Go's (merge sort peaks at exactly 2n; the unit is `i64` elements, 8 bytes each), which shows these numbers belong to the algorithm, not the language.

**Complexity:** merge sort Θ(n) auxiliary · insertion sort Θ(1) · quicksort Θ(log n) average stack, Θ(n) worst — full treatment in [13 — Efficient Sorts](../../13-efficient-sorts.md)

---

## 11. Recursion trees

`🟡 medium` · *Solving recurrences by drawing them*

A recurrence becomes arithmetic you can do by hand once you draw the tree: level k has `a^k` nodes of
size `n/b^k`. Three recurrences, three completely different shapes — and the shape tells you the
answer without any theorem.

**Steps:**

1. For each recurrence, print nodes / size / work at every level.
2. Ask which level dominates: all equal, or the top, or the bottom?
3. Then handle a recurrence that isn't divide-and-conquer at all.

```go
package main

import "fmt"

// A recursion tree turns a recurrence into arithmetic you can do by hand:
// draw the levels, cost each level, add them up.
//
// For T(n) = a*T(n/b) + f(n):
//   level k has a^k nodes, each of size n/b^k, so it costs a^k * f(n/b^k).
//   the tree is log_b(n) levels deep.

type recurrence struct {
	name  string
	fname string
	a     int
	b     int
	f     func(int) int
}

func (r recurrence) tree(n int) {
	fmt.Printf("\n%s\n  T(n) = %d*T(n/%d) + %s,  n = %d\n\n", r.name, r.a, r.b, r.fname, n)
	fmt.Printf("  %-7s %10s %12s %14s\n", "level", "nodes", "size each", "work at level")

	total := 0
	nodes := 1
	size := n
	level := 0
	for size >= 1 {
		work := nodes * r.f(size)
		total += work
		fmt.Printf("  %-7d %10d %12d %14d\n", level, nodes, size, work)
		nodes *= r.a
		size /= r.b
		level++
	}
	fmt.Printf("  %-7s %10s %12s %14d\n", "", "", "TOTAL", total)
}

func main() {
	n := 64

	recs := []recurrence{
		{"merge sort -- work is the same at every level", "n", 2, 2, func(k int) int { return k }},
		{"binary search -- one path down the tree", "1", 1, 2, func(k int) int { return 1 }},
		{"tree traversal -- work explodes at the leaves", "1", 2, 2, func(k int) int { return 1 }},
	}
	for _, r := range recs {
		r.tree(n)
	}

	fmt.Println("\nreading the three tables:")
	fmt.Printf("  merge sort:     %d levels x n work each  = n log n  (%d = %d*%d)\n", 7, 448, 64, 7)
	fmt.Println("  binary search:  1 unit per level, log n levels  = log n")
	fmt.Println("  tree traversal: the last level dominates -- 64 of the 127 nodes are leaves = n")
	fmt.Println()
	fmt.Println("that is the whole method: if every level costs the same, multiply by the depth;")
	fmt.Println("if one level dominates, that level IS the answer.")

	// A recurrence that is not divide-and-conquer at all.
	fmt.Println("\n----------------------------------------------------------------")
	fmt.Println("\nnot every recurrence is a/b-shaped. Selection sort peels one element:")
	fmt.Println("  T(n) = T(n-1) + n")
	fmt.Println()
	fmt.Printf("  %-10s %14s\n", "n", "expanding")
	fmt.Println("  T(n)       = n + T(n-1)")
	fmt.Println("  T(n-1)     = (n-1) + T(n-2)")
	fmt.Println("  ...")
	fmt.Println("  T(1)       = 1")
	fmt.Println()
	total := 0
	for i := 1; i <= n; i++ {
		total += i
	}
	fmt.Printf("  so T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = %d at n=%d  -> Theta(n^2)\n", total, n)
	fmt.Println()
	fmt.Println("shrinking by a FACTOR gives a log-deep tree; shrinking by a CONSTANT gives")
	fmt.Println("an n-deep one. That single difference is most of the gap between n log n and n^2.")
}
```

**Output:**

```
merge sort -- work is the same at every level
  T(n) = 2*T(n/2) + n,  n = 64

  level        nodes    size each  work at level
  0                1           64             64
  1                2           32             64
  2                4           16             64
  3                8            8             64
  4               16            4             64
  5               32            2             64
  6               64            1             64
                            TOTAL            448

binary search -- one path down the tree
  T(n) = 1*T(n/2) + 1,  n = 64

  level        nodes    size each  work at level
  0                1           64              1
  1                1           32              1
  2                1           16              1
  3                1            8              1
  4                1            4              1
  5                1            2              1
  6                1            1              1
                            TOTAL              7

tree traversal -- work explodes at the leaves
  T(n) = 2*T(n/2) + 1,  n = 64

  level        nodes    size each  work at level
  0                1           64              1
  1                2           32              2
  2                4           16              4
  3                8            8              8
  4               16            4             16
  5               32            2             32
  6               64            1             64
                            TOTAL            127

reading the three tables:
  merge sort:     7 levels x n work each  = n log n  (448 = 64*7)
  binary search:  1 unit per level, log n levels  = log n
  tree traversal: the last level dominates -- 64 of the 127 nodes are leaves = n

that is the whole method: if every level costs the same, multiply by the depth;
if one level dominates, that level IS the answer.

----------------------------------------------------------------

not every recurrence is a/b-shaped. Selection sort peels one element:
  T(n) = T(n-1) + n

  n              expanding
  T(n)       = n + T(n-1)
  T(n-1)     = (n-1) + T(n-2)
  ...
  T(1)       = 1

  so T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = 2080 at n=64  -> Theta(n^2)

shrinking by a FACTOR gives a log-deep tree; shrinking by a CONSTANT gives
an n-deep one. That single difference is most of the gap between n log n and n^2.
```

Three shapes, and they are the only three: **every level equal** (multiply by depth), **root
dominates** (the top level is the answer), **leaves dominate** (the bottom level is the answer).
Example 12 names them Cases 2, 3 and 1.

**Rust:**

```rust
// A recursion tree turns a recurrence into arithmetic you can do by hand:
// draw the levels, cost each level, add them up.
//
// For T(n) = a*T(n/b) + f(n):
//   level k has a^k nodes, each of size n/b^k, so it costs a^k * f(n/b^k).
//   the tree is log_b(n) levels deep.

struct Recurrence {
    name: &'static str,
    fname: &'static str,
    a: u64,
    b: u64,
    f: fn(u64) -> u64,
}

// What tree() found, so the summary can quote it instead of retyping it.
struct Tree {
    levels: u32,
    total_work: u64,
    total_nodes: u64,
    leaves: u64,
}

impl Recurrence {
    fn tree(&self, n: u64) -> Tree {
        println!(
            "\n{}\n  T(n) = {}*T(n/{}) + {},  n = {}\n",
            self.name, self.a, self.b, self.fname, n
        );
        println!(
            "  {:<7} {:>10} {:>12} {:>14}",
            "level", "nodes", "size each", "work at level"
        );

        let (mut total, mut total_nodes) = (0, 0);
        let (mut nodes, mut size, mut level) = (1, n, 0);
        while size >= 1 {
            let work = nodes * (self.f)(size);
            total += work;
            total_nodes += nodes;
            println!("  {level:<7} {nodes:>10} {size:>12} {work:>14}");
            nodes *= self.a;
            size /= self.b;
            level += 1;
        }
        println!("  {:<7} {:>10} {:>12} {:>14}", "", "", "TOTAL", total);
        Tree {
            levels: level,
            total_work: total,
            total_nodes,
            leaves: nodes / self.a, // the last level's node count
        }
    }
}

fn main() {
    let n = 64;

    let recs = [
        Recurrence {
            name: "merge sort -- work is the same at every level",
            fname: "n",
            a: 2,
            b: 2,
            f: |k| k,
        },
        Recurrence {
            name: "binary search -- one path down the tree",
            fname: "1",
            a: 1,
            b: 2,
            f: |_| 1,
        },
        Recurrence {
            name: "tree traversal -- work explodes at the leaves",
            fname: "1",
            a: 2,
            b: 2,
            f: |_| 1,
        },
    ];
    let trees: Vec<Tree> = recs.iter().map(|r| r.tree(n)).collect();
    let [merge, _binary, traversal] = &trees[..] else {
        unreachable!("three recurrences")
    };

    // Go typed these numbers into its Printf; here they come from the trees above.
    println!("\nreading the three tables:");
    println!(
        "  merge sort:     {} levels x n work each  = n log n  ({} = {}*{})",
        merge.levels, merge.total_work, n, merge.levels
    );
    println!("  binary search:  1 unit per level, log n levels  = log n");
    println!(
        "  tree traversal: the last level dominates -- {} of the {} nodes are leaves = n",
        traversal.leaves, traversal.total_nodes
    );
    println!();
    println!("that is the whole method: if every level costs the same, multiply by the depth;");
    println!("if one level dominates, that level IS the answer.");

    // A recurrence that is not divide-and-conquer at all.
    println!("\n----------------------------------------------------------------");
    println!("\nnot every recurrence is a/b-shaped. Selection sort peels one element:");
    println!("  T(n) = T(n-1) + n");
    println!();
    println!("  {:<10} {:>14}", "n", "expanding");
    println!("  T(n)       = n + T(n-1)");
    println!("  T(n-1)     = (n-1) + T(n-2)");
    println!("  ...");
    println!("  T(1)       = 1");
    println!();
    let total: u64 = (1..=n).sum();
    println!("  so T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = {total} at n={n}  -> Theta(n^2)");
    println!();
    println!("shrinking by a FACTOR gives a log-deep tree; shrinking by a CONSTANT gives");
    println!("an n-deep one. That single difference is most of the gap between n log n and n^2.");
}
```

**Rust output:**

```

merge sort -- work is the same at every level
  T(n) = 2*T(n/2) + n,  n = 64

  level        nodes    size each  work at level
  0                1           64             64
  1                2           32             64
  2                4           16             64
  3                8            8             64
  4               16            4             64
  5               32            2             64
  6               64            1             64
                            TOTAL            448

binary search -- one path down the tree
  T(n) = 1*T(n/2) + 1,  n = 64

  level        nodes    size each  work at level
  0                1           64              1
  1                1           32              1
  2                1           16              1
  3                1            8              1
  4                1            4              1
  5                1            2              1
  6                1            1              1
                            TOTAL              7

tree traversal -- work explodes at the leaves
  T(n) = 2*T(n/2) + 1,  n = 64

  level        nodes    size each  work at level
  0                1           64              1
  1                2           32              2
  2                4           16              4
  3                8            8              8
  4               16            4             16
  5               32            2             32
  6               64            1             64
                            TOTAL            127

reading the three tables:
  merge sort:     7 levels x n work each  = n log n  (448 = 64*7)
  binary search:  1 unit per level, log n levels  = log n
  tree traversal: the last level dominates -- 64 of the 127 nodes are leaves = n

that is the whole method: if every level costs the same, multiply by the depth;
if one level dominates, that level IS the answer.

----------------------------------------------------------------

not every recurrence is a/b-shaped. Selection sort peels one element:
  T(n) = T(n-1) + n

  n               expanding
  T(n)       = n + T(n-1)
  T(n-1)     = (n-1) + T(n-2)
  ...
  T(1)       = 1

  so T(n) = n + (n-1) + ... + 1 = n(n+1)/2 = 2080 at n=64  -> Theta(n^2)

shrinking by a FACTOR gives a log-deep tree; shrinking by a CONSTANT gives
an n-deep one. That single difference is most of the gap between n log n and n^2.
```

> **Go vs Rust:** The output is identical to Go's, because every number here is arithmetic on the recurrence. One difference in the source: Go's "reading the three tables" line had `7`, `448` and `64 of the 127` typed into its `Printf`, while the Rust twin returns a `Tree` from each `tree()` call and prints those figures from it, so the summary cannot drift from the tables.

**Complexity:** `2T(n/2)+n` → Θ(n log n) · `T(n/2)+1` → Θ(log n) · `2T(n/2)+1` → Θ(n) · `T(n−1)+n` → Θ(n²)

---

> Next tier: [🔴 hard](3-hard.md) — the master theorem, hidden quadratics, and measuring complexity for real.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
