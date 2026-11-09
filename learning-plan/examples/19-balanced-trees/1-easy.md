# Step 19 — Balanced Trees & B-Trees · 🟢 Easy

Examples **1–6**: how much disorder protects a plain BST, the rotation, AVL's four cases and its tallest tree, red-black's rules
with a checker, and the coin flip that replaces rebalancing.

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

## 1. How much disorder protects a BST

`🟢 easy` · *Height vs disorder*

A plain BST has no repair step, so its height is decided by the order its keys arrive in, and lesson 18 example 6 already counted the two extremes: a sorted build is a path that costs n(n−1)/2 comparisons, a random one is shallow. Real keys — timestamps, IDs, merged logs — are neither, they arrive *nearly* sorted, so the question that motivates the whole lesson is how much disorder a plain BST needs before it stops being a path. A build costs Θ(n·h) comparisons (Θ(h) per insert), somewhere between Θ(n log n) and Θ(n²), and counting height and comparisons instead of time answers the question exactly and the same on every run.

**Steps:**

1. Take one control row for each extreme at n = 1,000 (sorted, and one shuffle, to compare with lesson 18 example 6), then measure how much "random" itself varies over 200 shuffles at n = 1,000 and n = 5,000.
2. The key move: start from the sorted order at n = 5,000 and add disorder two ways, k random swaps and local jitter (key i arrives up to w slots late), building the tree each time and counting height and comparisons.
3. Read the tables against the floor `⌈lg(n+1)⌉` and the random median, and let the closing lines compute the first row that gets within 3× of random.

```go
package main

import (
	"cmp"
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"unsafe"
)

// A plain BST (lesson 18): no rebalancing, so its shape is decided entirely by
// the ORDER the keys arrive in. Lesson 18 example 6 measured the two extremes
// (sorted = a path, random = shallow). Real keys -- timestamps, IDs, merged
// logs -- are neither: they are NEARLY sorted. This example asks how much
// disorder a plain BST needs before it stops being a path.
type node struct {
	key         int
	left, right *node
}

// insert is iterative (a sorted build is a path n deep; see lesson 05 / 17 for
// why we never recurse on one). It returns the comparisons it spent.
// Keys here are always distinct, so there is no "equal" case.
func insert(root **node, key int) int {
	cmps := 0
	link := root
	for *link != nil {
		cmps++
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	*link = &node{key: key}
	return cmps
}

// height = nodes on the longest root-to-nil path, counted level by level.
func height(root *node) int {
	var level []*node
	if root != nil {
		level = append(level, root)
	}
	h := 0
	for len(level) > 0 {
		h++
		var next []*node
		for _, n := range level {
			if n.left != nil {
				next = append(next, n.left)
			}
			if n.right != nil {
				next = append(next, n.right)
			}
		}
		level = next
	}
	return h
}

// build inserts keys in the given order into an empty tree.
func build(order []int) (h, cmps int) {
	var root *node
	for _, k := range order {
		cmps += insert(&root, k)
	}
	return height(root), cmps
}

func sortedKeys(n int) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = i + 1
	}
	return a
}

// shuffle is a hand-written Fisher-Yates so the Rust twin draws the same numbers.
func shuffle(rng *rand.Rand, a []int) {
	for i := len(a) - 1; i >= 1; i-- {
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
}

func floor(n int) int { return bits.Len(uint(n)) } // ceil(lg(n+1)): the shortest any tree can be

// displaced counts keys that are not at their sorted position; maxMove is the
// farthest any key is from home.
func displaced(a []int) (count, maxMove int) {
	for i, k := range a {
		d := k - (i + 1)
		if d < 0 {
			d = -d
		}
		if d > 0 {
			count++
		}
		maxMove = max(maxMove, d)
	}
	return
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	fmt.Printf("node size: %d bytes\n", unsafe.Sizeof(node{}))

	fmt.Println("\n== (a) control: the two extremes at n = 1,000 (lesson 18 example 6 sweeps n and four orders) ==")
	fmt.Printf("%-8s %7s %6s %12s %11s %12s\n", "order", "height", "floor", "comparisons", "per insert", "n(n-1)/2")
	const n0 = 1000
	rnd := sortedKeys(n0)
	shuffle(rng, rnd)
	for _, c := range []struct {
		name  string
		order []int
	}{{"sorted", sortedKeys(n0)}, {"random", rnd}} {
		h, cmps := build(c.order)
		fmt.Printf("%-8s %7d %6d %12d %11.2f %12d\n", c.name, h, floor(n0), cmps, float64(cmps)/n0, n0*(n0-1)/2)
	}

	fmt.Println("\n== (b) how much does 'random' vary? 200 shuffles per n ==")
	fmt.Printf("%6s %6s %5s %7s %5s %9s %12s\n", "n", "floor", "min", "median", "max", "max/floor", "median/floor")
	medians := map[int]int{}
	for _, n := range []int{1000, 5000} {
		var hs []int
		for t := 0; t < 200; t++ {
			a := sortedKeys(n)
			shuffle(rng, a)
			h, _ := build(a)
			hs = append(hs, h)
		}
		slices.Sort(hs)
		medians[n] = hs[len(hs)/2]
		f := float64(floor(n))
		fmt.Printf("%6d %6d %5d %7d %5d %9.2f %12.2f\n", n, floor(n), hs[0], hs[len(hs)/2], hs[len(hs)-1], float64(hs[len(hs)-1])/f, float64(hs[len(hs)/2])/f)
	}

	// The core: start from the sorted array and add disorder.
	const n = 5000
	fmt.Printf("\n== (c) k random swaps applied to the sorted order, n = %d (floor %d, random median %d) ==\n", n, floor(n), medians[n])
	fmt.Println("swaps are cumulative: each row continues from the array of the row above")
	fmt.Printf("%6s %10s %7s %12s %12s %11s\n", "k", "displaced", "height", "height/floor", "comparisons", "per insert")
	a := sortedKeys(n)
	done := 0
	firstK, firstKMoved := -1, 0
	for _, k := range []int{0, 1, 10, 100, 1000, 2500, 5000} {
		for ; done < k; done++ {
			i, j := rng.IntN(n), rng.IntN(n)
			a[i], a[j] = a[j], a[i]
		}
		moved, _ := displaced(a)
		h, cmps := build(a)
		fmt.Printf("%6d %10d %7d %12.1f %12d %11.1f\n", k, moved, h, float64(h)/float64(floor(n)), cmps, float64(cmps)/n)
		if firstK < 0 && h <= 3*medians[n] {
			firstK, firstKMoved = k, moved
		}
	}

	fmt.Printf("\n== (d) local disorder: key i sorts at slot i + IntN(w+1), n = %d (late arrivals) ==\n", n)
	fmt.Printf("%6s %10s %9s %7s %12s %12s %11s\n", "w", "displaced", "max move", "height", "height/floor", "comparisons", "per insert")
	idx := make([]int, n)
	firstW, firstWMoved, lastW, lastH := -1, 0, 0, 0
	for _, w := range []int{2, 4, 16, 64, 256, 1024, 4096} {
		jit := make([]int, n)
		for i := range jit {
			jit[i] = rng.IntN(w + 1)
		}
		for i := range idx {
			idx[i] = i
		}
		// Stable sort: keys with equal slots keep their sorted order, so a key only
		// moves by jumping STRICTLY past another one (that is why w = 1 would
		// change nothing, and why the farthest move stays below w).
		slices.SortStableFunc(idx, func(x, y int) int { return cmp.Compare(x+jit[x], y+jit[y]) })
		order := make([]int, n)
		for pos, i := range idx {
			order[pos] = i + 1
		}
		moved, far := displaced(order)
		h, cmps := build(order)
		fmt.Printf("%6d %10d %9d %7d %12.1f %12d %11.1f\n", w, moved, far, h, float64(h)/float64(floor(n)), cmps, float64(cmps)/n)
		lastW, lastH = w, h
		if firstW < 0 && h <= 3*medians[n] {
			firstW, firstWMoved = w, moved
		}
	}

	fmt.Printf("\n== (e) how much disorder is enough? (height within 3x of the random median, %d) ==\n", 3*medians[n])
	report := func(what string, v, moved int) {
		if v < 0 {
			fmt.Printf("%s: none tried got there; the largest, w = %d, ends at height %d (%.1fx the median)\n",
				what, lastW, lastH, float64(lastH)/float64(medians[n]))
			return
		}
		fmt.Printf("%s: %d  (%d of %d keys displaced = %.0f%%)\n", what, v, moved, n, 100*float64(moved)/n)
	}
	report("smallest k tried (swaps)", firstK, firstKMoved)
	report("smallest w tried (jitter)", firstW, firstWMoved)
}
```

**Output:**

```
node size: 24 bytes

== (a) control: the two extremes at n = 1,000 (lesson 18 example 6 sweeps n and four orders) ==
order     height  floor  comparisons  per insert     n(n-1)/2
sorted      1000     10       499500      499.50       499500
random        23     10        11600       11.60       499500

== (b) how much does 'random' vary? 200 shuffles per n ==
     n  floor   min  median   max max/floor median/floor
  1000     10    18      22    28      2.80         2.20
  5000     13    24      28    38      2.92         2.15

== (c) k random swaps applied to the sorted order, n = 5000 (floor 13, random median 28) ==
swaps are cumulative: each row continues from the array of the row above
     k  displaced  height height/floor  comparisons  per insert
     0          0    5000        384.6     12497500      2499.5
     1          2    4360        335.4     10183262      2036.7
    10         20    2104        161.8      6137824      1227.6
   100        199     623         47.9       874930       175.0
  1000       1650     253         19.5       162621        32.5
  2500       3158      54          4.2        78969        15.8
  5000       4321      33          2.5        73662        14.7

== (d) local disorder: key i sorts at slot i + IntN(w+1), n = 5000 (late arrivals) ==
     w  displaced  max move  height height/floor  comparisons  per insert
     2       1126         1    4437        341.3     11084247      2216.8
     4       2914         3    3488        268.3      8718657      1743.7
    16       4663        12    1709        131.5      4250724       850.1
    64       4924        44     839         64.5      2102667       420.5
   256       4982       148     394         30.3       967726       193.5
  1024       4995       536     194         14.9       497101        99.4
  4096       4998      2043     120          9.2       290949        58.2

== (e) how much disorder is enough? (height within 3x of the random median, 84) ==
smallest k tried (swaps): 2500  (3158 of 5000 keys displaced = 63%)
smallest w tried (jitter): none tried got there; the largest, w = 4096, ends at height 120 (4.3x the median)
```

**Rust:**

```rust
use std::mem::size_of;

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the numbers the Go program draws (seed 1, 2).
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

    fn idx(&mut self, n: usize) -> usize {
        self.int_n(n as u64) as usize
    }
}

// A plain BST (lesson 18): no rebalancing, so its shape is decided entirely by
// the ORDER the keys arrive in. Lesson 18 example 6 measured the two extremes
// (sorted = a path, random = shallow). Real keys -- timestamps, IDs, merged
// logs -- are neither: they are NEARLY sorted. This example asks how much
// disorder a plain BST needs before it stops being a path.
struct Node {
    key: usize,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Dropping a Box chain recurses once per level, and a sorted build is a path
// n deep. This Drop hands the children to a heap-allocated stack instead
// (lesson 05 example 15). Go's collector needs nothing like it.
impl Drop for Node {
    fn drop(&mut self) {
        let mut stack: Vec<Box<Node>> = Vec::new();
        stack.extend(self.left.take());
        stack.extend(self.right.take());
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
        }
    }
}

// insert is iterative: `cur` is a cursor on the link that holds the subtree we
// are looking at -- Rust's spelling of Go's `link := &root`. It returns the
// comparisons it spent. Keys here are always distinct, so there is no "equal" case.
fn insert(root: &mut Link, key: usize) -> u64 {
    let mut cmps = 0;
    let mut cur = root;
    while let Some(n) = cur {
        cmps += 1;
        cur = if key < n.key {
            &mut n.left
        } else {
            &mut n.right
        };
    }
    *cur = Some(Box::new(Node {
        key,
        left: None,
        right: None,
    }));
    cmps
}

// height = nodes on the longest root-to-nil path, counted level by level.
fn height(root: &Link) -> usize {
    let mut level: Vec<&Node> = root.as_deref().into_iter().collect();
    let mut h = 0;
    while !level.is_empty() {
        h += 1;
        level = level
            .iter()
            .flat_map(|n| [n.left.as_deref(), n.right.as_deref()])
            .flatten()
            .collect();
    }
    h
}

// build inserts keys in the given order into an empty tree.
fn build(order: &[usize]) -> (usize, u64) {
    let mut root: Link = None;
    let mut cmps = 0;
    for &k in order {
        cmps += insert(&mut root, k);
    }
    (height(&root), cmps)
}

fn sorted_keys(n: usize) -> Vec<usize> {
    (1..=n).collect()
}

// shuffle is a hand-written Fisher-Yates so both programs draw the same numbers.
fn shuffle(rng: &mut Pcg, a: &mut [usize]) {
    for i in (1..a.len()).rev() {
        let j = rng.idx(i + 1);
        a.swap(i, j);
    }
}

// ceil(lg(n+1)): the shortest any tree can be
fn floor(n: usize) -> usize {
    (usize::BITS - n.leading_zeros()) as usize
}

// displaced counts keys that are not at their sorted position; the second
// value is the farthest any key is from home.
fn displaced(a: &[usize]) -> (usize, usize) {
    let (mut count, mut max_move) = (0, 0);
    for (i, &k) in a.iter().enumerate() {
        let d = k.abs_diff(i + 1);
        if d > 0 {
            count += 1;
        }
        max_move = max_move.max(d);
    }
    (count, max_move)
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    println!("node size: {} bytes", size_of::<Node>());

    println!(
        "\n== (a) control: the two extremes at n = 1,000 (lesson 18 example 6 sweeps n and four orders) =="
    );
    println!(
        "{:<8} {:>7} {:>6} {:>12} {:>11} {:>12}",
        "order", "height", "floor", "comparisons", "per insert", "n(n-1)/2"
    );
    const N0: usize = 1000;
    let mut rnd = sorted_keys(N0);
    shuffle(&mut rng, &mut rnd);
    for (name, order) in [("sorted", sorted_keys(N0)), ("random", rnd)] {
        let (h, cmps) = build(&order);
        println!(
            "{:<8} {:>7} {:>6} {:>12} {:>11.2} {:>12}",
            name,
            h,
            floor(N0),
            cmps,
            cmps as f64 / N0 as f64,
            N0 * (N0 - 1) / 2
        );
    }

    println!("\n== (b) how much does 'random' vary? 200 shuffles per n ==");
    println!(
        "{:>6} {:>6} {:>5} {:>7} {:>5} {:>9} {:>12}",
        "n", "floor", "min", "median", "max", "max/floor", "median/floor"
    );
    let mut median_5000 = 0;
    for n in [1000, 5000] {
        let mut hs: Vec<usize> = Vec::new();
        for _ in 0..200 {
            let mut a = sorted_keys(n);
            shuffle(&mut rng, &mut a);
            hs.push(build(&a).0);
        }
        hs.sort_unstable();
        let median = hs[hs.len() / 2];
        if n == 5000 {
            median_5000 = median;
        }
        let f = floor(n) as f64;
        println!(
            "{:>6} {:>6} {:>5} {:>7} {:>5} {:>9.2} {:>12.2}",
            n,
            floor(n),
            hs[0],
            median,
            hs[hs.len() - 1],
            hs[hs.len() - 1] as f64 / f,
            median as f64 / f
        );
    }

    // The core: start from the sorted array and add disorder.
    const N: usize = 5000;
    println!(
        "\n== (c) k random swaps applied to the sorted order, n = {N} (floor {}, random median {median_5000}) ==",
        floor(N)
    );
    println!("swaps are cumulative: each row continues from the array of the row above");
    println!(
        "{:>6} {:>10} {:>7} {:>12} {:>12} {:>11}",
        "k", "displaced", "height", "height/floor", "comparisons", "per insert"
    );
    let mut a = sorted_keys(N);
    let mut done = 0;
    let mut first_k: Option<(usize, usize)> = None;
    for k in [0, 1, 10, 100, 1000, 2500, 5000] {
        while done < k {
            let (i, j) = (rng.idx(N), rng.idx(N));
            a.swap(i, j);
            done += 1;
        }
        let (moved, _) = displaced(&a);
        let (h, cmps) = build(&a);
        println!(
            "{:>6} {:>10} {:>7} {:>12.1} {:>12} {:>11.1}",
            k,
            moved,
            h,
            h as f64 / floor(N) as f64,
            cmps,
            cmps as f64 / N as f64
        );
        if first_k.is_none() && h <= 3 * median_5000 {
            first_k = Some((k, moved));
        }
    }

    println!(
        "\n== (d) local disorder: key i sorts at slot i + IntN(w+1), n = {N} (late arrivals) =="
    );
    println!(
        "{:>6} {:>10} {:>9} {:>7} {:>12} {:>12} {:>11}",
        "w", "displaced", "max move", "height", "height/floor", "comparisons", "per insert"
    );
    let mut first_w: Option<(usize, usize)> = None;
    let (mut last_w, mut last_h) = (0, 0);
    for w in [2, 4, 16, 64, 256, 1024, 4096] {
        let jit: Vec<usize> = (0..N).map(|_| rng.idx(w + 1)).collect();
        let mut idx: Vec<usize> = (0..N).collect();
        // sort_by_key is the STABLE sort: keys with equal slots keep their
        // sorted order, so a key only moves by jumping STRICTLY past another
        // one (that is why w = 1 would change nothing, and why the farthest
        // move stays below w).
        idx.sort_by_key(|&i| i + jit[i]);
        let order: Vec<usize> = idx.iter().map(|&i| i + 1).collect();
        let (moved, far) = displaced(&order);
        let (h, cmps) = build(&order);
        println!(
            "{:>6} {:>10} {:>9} {:>7} {:>12.1} {:>12} {:>11.1}",
            w,
            moved,
            far,
            h,
            h as f64 / floor(N) as f64,
            cmps,
            cmps as f64 / N as f64
        );
        (last_w, last_h) = (w, h);
        if first_w.is_none() && h <= 3 * median_5000 {
            first_w = Some((w, moved));
        }
    }

    println!(
        "\n== (e) how much disorder is enough? (height within 3x of the random median, {}) ==",
        3 * median_5000
    );
    let report = |what: &str, found: Option<(usize, usize)>| match found {
        None => println!(
            "{what}: none tried got there; the largest, w = {last_w}, ends at height {last_h} ({:.1}x the median)",
            last_h as f64 / median_5000 as f64
        ),
        Some((v, moved)) => println!(
            "{what}: {v}  ({moved} of {N} keys displaced = {:.0}%)",
            100.0 * moved as f64 / N as f64
        ),
    };
    report("smallest k tried (swaps)", first_k);
    report("smallest w tried (jitter)", first_w);
}
```

**Rust output:**

```
node size: 24 bytes

== (a) control: the two extremes at n = 1,000 (lesson 18 example 6 sweeps n and four orders) ==
order     height  floor  comparisons  per insert     n(n-1)/2
sorted      1000     10       499500      499.50       499500
random        23     10        11600       11.60       499500

== (b) how much does 'random' vary? 200 shuffles per n ==
     n  floor   min  median   max max/floor median/floor
  1000     10    18      22    28      2.80         2.20
  5000     13    24      28    38      2.92         2.15

== (c) k random swaps applied to the sorted order, n = 5000 (floor 13, random median 28) ==
swaps are cumulative: each row continues from the array of the row above
     k  displaced  height height/floor  comparisons  per insert
     0          0    5000        384.6     12497500      2499.5
     1          2    4360        335.4     10183262      2036.7
    10         20    2104        161.8      6137824      1227.6
   100        199     623         47.9       874930       175.0
  1000       1650     253         19.5       162621        32.5
  2500       3158      54          4.2        78969        15.8
  5000       4321      33          2.5        73662        14.7

== (d) local disorder: key i sorts at slot i + IntN(w+1), n = 5000 (late arrivals) ==
     w  displaced  max move  height height/floor  comparisons  per insert
     2       1126         1    4437        341.3     11084247      2216.8
     4       2914         3    3488        268.3      8718657      1743.7
    16       4663        12    1709        131.5      4250724       850.1
    64       4924        44     839         64.5      2102667       420.5
   256       4982       148     394         30.3       967726       193.5
  1024       4995       536     194         14.9       497101        99.4
  4096       4998      2043     120          9.2       290949        58.2

== (e) how much disorder is enough? (height within 3x of the random median, 84) ==
smallest k tried (swaps): 2500  (3158 of 5000 keys displaced = 63%)
smallest w tried (jitter): none tried got there; the largest, w = 4096, ends at height 120 (4.3x the median)
```

> **Go vs Rust:** Identical to Go line for line (diffed): the PCG-DXSM port draws the same shuffles and swaps, so every height and comparison count matches, and `size_of::<Node>()` is 24 bytes like Go's `unsafe.Sizeof`. The insert cursor is `let mut cur = &mut root; while let Some(n) = cur { cur = &mut n.left … }`, Rust's spelling of Go's `link := &root`, and `sort_by_key` is the stable sort (Go needs `slices.SortStableFunc` for the same tie-breaking). The iterative `Drop` has no Go counterpart: a path-shaped tree is what Rust's default recursive drop cannot survive at large n (lesson 17 example 13), so the tree gets one even though n = 5,000 would fit.

**Complexity:** a plain BST's height tracks how sorted the input is, not n: a sorted build is Θ(n²) (exactly n(n−1)/2 = 499,500 comparisons at n = 1,000) and a random one Θ(n log n) expected (11,600 comparisons, height 23 against a floor of 10), and the random height itself ranges from 18 to 28 over 200 shuffles. At n = 5,000 (floor 13, random median 28), 100 swaps that displace 199 keys still leave height 623 and 1,000 swaps (1,650 displaced) leave 253, the first row within 3× of random is 2,500 swaps (63% of the keys displaced), and local jitter never gets there: even w = 4096 ends at height 120. A plain BST needs disorder on the scale of the array itself, which is why the rest of the lesson is the repair.

---

## 2. Rotations: reshape, never reorder

`🟢 easy` · *Rotation*

A rotation is the one repair operation every balanced tree in this lesson is made of: promote a child, demote its parent, and hand the child's inner subtree across to the parent — three pointer writes, Θ(1). It is the right tool because it changes depths but cannot change the in-order sequence (the three subtrees keep their left-to-right order), so a tree stays a valid BST no matter how many rotations hit it, and the only thing left to decide is *where* to rotate. This example does it by hand on five nodes, shows that a rotation and its mirror undo each other, then fires 100,000 rotations at random and checks the order after every one.

**Steps:**

1. Build `y(x(A, B), C)`, print its shape, in-order and the depth of each subtree root, rotate right at `y`, and print again with the pointer writes counted.
2. On a random 50-node tree, rotate at every node in both directions, rotate back, and compare the shape strings.
3. Apply 100,000 random rotations to a random 1,000-node tree, check in-order against `1..n` after every performed one, and watch the height.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"strings"
)

// A rotation is the one repair every balanced tree below is built from.
// Rotate RIGHT at y, with x = y.left:
//
//	    y                x
//	   / \              / \
//	  x   C    ==>     A   y        three subtrees keep their left-to-right
//	 / \                  / \       order A < x < B < y < C, so BST order is
//	A   B                B   C      preserved by construction.
type node struct {
	key         int
	left, right *node
}

// writes counts pointer writes (the cost of a rotation), threaded by pointer.
type counters struct{ writes int }

// rotateRight returns the new root of the subtree; the CALLER must store it
// where y used to hang. With no left child there is nothing to do.
func rotateRight(y *node, c *counters) *node {
	x := y.left
	if x == nil {
		return y
	}
	y.left = x.right // B moves across
	c.writes++
	x.right = y // y goes down
	c.writes++
	return x
}

func rotateLeft(x *node, c *counters) *node { // the mirror image, and the inverse
	y := x.right
	if y == nil {
		return x
	}
	x.right = y.left
	c.writes++
	y.left = x
	c.writes++
	return y
}

// rotateAt rotates at the node holding key, if it has the child the rotation
// needs. It returns the key of the child it promoted (now the subtree root),
// or ok = false when the node is missing or lacks that child. Storing the new
// subtree root in the link that held the old one is the third pointer write.
func rotateAt(root **node, key int, right bool, c *counters) (promoted int, ok bool) {
	link := root
	for *link != nil && (*link).key != key {
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	n := *link
	if n == nil || (right && n.left == nil) || (!right && n.right == nil) {
		return 0, false
	}
	if right {
		*link = rotateRight(n, c)
	} else {
		*link = rotateLeft(n, c)
	}
	c.writes++
	return (*link).key, true
}

func move(d int) string {
	switch {
	case d < 0:
		return fmt.Sprintf("up %d level", -d)
	case d > 0:
		return fmt.Sprintf("down %d level", d)
	}
	return "same depth"
}

func insert(root **node, key int) {
	link := root
	for *link != nil {
		if key < (*link).key {
			link = &(*link).left
		} else {
			link = &(*link).right
		}
	}
	*link = &node{key: key}
}

// shape prints a tree as an s-expression: a leaf is its key, "." is a missing child.
func shape(n *node) string {
	switch {
	case n == nil:
		return "."
	case n.left == nil && n.right == nil:
		return fmt.Sprint(n.key)
	}
	return "(" + fmt.Sprint(n.key) + " " + shape(n.left) + " " + shape(n.right) + ")"
}

func depth(root *node, key int) int { // the root has depth 0
	d := 0
	for n := root; n != nil; d++ {
		switch {
		case key == n.key:
			return d
		case key < n.key:
			n = n.left
		default:
			n = n.right
		}
	}
	return -1
}

type frame struct {
	n     *node
	depth int
}

// walk returns the keys in order and the height (nodes on the longest path),
// iteratively, so it is safe on any shape.
func walk(root *node) (keys []int, h int) {
	var stack []frame
	cur, d := root, 1
	for cur != nil || len(stack) > 0 {
		for cur != nil {
			stack = append(stack, frame{cur, d})
			cur, d = cur.left, d+1
		}
		f := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		keys = append(keys, f.n.key)
		h = max(h, f.depth)
		cur, d = f.n.right, f.depth+1
	}
	return
}

func randomTree(rng *rand.Rand, n int) *node {
	a := make([]int, n)
	for i := range a {
		a[i] = i + 1
	}
	for i := n - 1; i >= 1; i-- { // hand-written Fisher-Yates, as in example 1
		j := rng.IntN(i + 1)
		a[i], a[j] = a[j], a[i]
	}
	var root *node
	for _, k := range a {
		insert(&root, k)
	}
	return root
}

func main() {
	var c counters

	fmt.Println("== (a) rotate right at y = 4 on the five-node tree y(x(A, B), C) ==")
	// y=4, x=2, A=1, B=3, C=5
	root := &node{4, &node{2, &node{1, nil, nil}, &node{3, nil, nil}}, &node{5, nil, nil}}
	names := []struct {
		label string
		key   int
	}{{"y=4", 4}, {"x=2", 2}, {"A=1", 1}, {"B=3", 3}, {"C=5", 5}}
	show := func(title string) (depths []int) {
		keys, h := walk(root)
		fmt.Printf("%-18s %-14s in-order %v  height %d\n", title, shape(root), keys, h)
		var parts []string
		for _, nm := range names {
			depths = append(depths, depth(root, nm.key))
			parts = append(parts, fmt.Sprintf("%s:%d", nm.label, depths[len(depths)-1]))
		}
		fmt.Printf("%-18s depths  %s\n", "", strings.Join(parts, "  "))
		return
	}
	before := show("before:")
	original := shape(root)
	rotateAt(&root, 4, true, &c)
	after := show("after rotateRight:")
	fmt.Printf("pointer writes: %d\n", c.writes)
	for i, nm := range names[2:] { // the three subtrees A, B, C
		fmt.Printf("  subtree %s: %s\n", nm.label, move(after[i+2]-before[i+2]))
	}
	rotateAt(&root, 2, false, &c)
	fmt.Printf("rotateLeft at x=2 restores %s: %v\n", original, shape(root) == original)

	fmt.Println("\n== (b) every rotation has an inverse: 50-node random tree, each node, both directions ==")
	rng := rand.New(rand.NewPCG(1, 2))
	const small = 50
	t := randomTree(rng, small)
	checks, restored, skipped := 0, 0, 0
	for key := 1; key <= small; key++ {
		for _, right := range []bool{true, false} {
			before := shape(t)
			promoted, ok := rotateAt(&t, key, right, &c)
			if !ok {
				skipped++
				continue
			}
			rotateAt(&t, promoted, !right, &c) // the promoted child is rotated back down
			checks++
			if shape(t) == before {
				restored++
			}
		}
	}
	fmt.Printf("rotate + inverse: %d checks, %d restored the exact shape, %d skipped (no child on that side)\n", checks, restored, skipped)
	fmt.Printf("checks == n-1 (= %d): %v; every non-root node is the child of exactly one node\n", small-1, checks == small-1)

	fmt.Println("\n== (c) 100,000 random rotations on a 1,000-node tree ==")
	const n = 1000
	t = randomTree(rng, n)
	c = counters{}
	_, startH := walk(t)
	performed, skippedC, failures := 0, 0, 0
	minH, maxH, sumH, prevH := startH, startH, 0, startH
	raised, lowered := 0, 0
	for range 100000 {
		key := rng.IntN(n) + 1
		right := rng.IntN(2) == 0
		if _, ok := rotateAt(&t, key, right, &c); !ok {
			skippedC++
			continue
		}
		performed++
		keys, h := walk(t)
		for i, k := range keys {
			if k != i+1 {
				failures++
				break
			}
		}
		minH, maxH, sumH = min(minH, h), max(maxH, h), sumH+h
		if h > prevH {
			raised++
		} else if h < prevH {
			lowered++
		}
		prevH = h
	}
	fmt.Printf("performed %d, skipped %d (node lacked the child), in-order failures %d\n", performed, skippedC, failures)
	fmt.Printf("pointer writes %d = 3 x %d performed: %v\n", c.writes, performed, c.writes == 3*performed)
	fmt.Printf("height: started %d, walked between %d and %d (mean %.1f); the floor for n = %d is %d\n",
		startH, minH, maxH, float64(sumH)/float64(performed), n, bits.Len(n))
	fmt.Printf("of %d rotations: %d raised the height, %d lowered it, %d left it as it was\n",
		performed, raised, lowered, performed-raised-lowered)
}
```

**Output:**

```
== (a) rotate right at y = 4 on the five-node tree y(x(A, B), C) ==
before:            (4 (2 1 3) 5)  in-order [1 2 3 4 5]  height 3
                   depths  y=4:0  x=2:1  A=1:2  B=3:2  C=5:1
after rotateRight: (2 1 (4 3 5))  in-order [1 2 3 4 5]  height 3
                   depths  y=4:1  x=2:0  A=1:1  B=3:2  C=5:2
pointer writes: 3
  subtree A=1: up 1 level
  subtree B=3: same depth
  subtree C=5: down 1 level
rotateLeft at x=2 restores (4 (2 1 3) 5): true

== (b) every rotation has an inverse: 50-node random tree, each node, both directions ==
rotate + inverse: 49 checks, 49 restored the exact shape, 51 skipped (no child on that side)
checks == n-1 (= 49): true; every non-root node is the child of exactly one node

== (c) 100,000 random rotations on a 1,000-node tree ==
performed 49875, skipped 50125 (node lacked the child), in-order failures 0
pointer writes 149625 = 3 x 49875 performed: true
height: started 22, walked between 22 and 93 (mean 60.4); the floor for n = 1000 is 10
of 49875 rotations: 1434 raised the height, 1366 lowered it, 47075 left it as it was
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the numbers the Go program draws (seed 1, 2).
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

    fn idx(&mut self, n: usize) -> usize {
        self.int_n(n as u64) as usize
    }
}

// A rotation is the one repair every balanced tree below is built from.
// Rotate RIGHT at y, with x = y.left:
//
//	    y                x
//	   / \              / \
//	  x   C    ==>     A   y        three subtrees keep their left-to-right
//	 / \                  / \       order A < x < B < y < C, so BST order is
//	A   B                B   C      preserved by construction.
struct Node {
    key: usize,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// Iterative Drop (lesson 05 example 15): safe for a tree of any shape, and the
// random rotations in (c) can make this one tall.
impl Drop for Node {
    fn drop(&mut self) {
        let mut stack: Vec<Box<Node>> = Vec::new();
        stack.extend(self.left.take());
        stack.extend(self.right.take());
        while let Some(mut n) = stack.pop() {
            stack.extend(n.left.take());
            stack.extend(n.right.take());
        }
    }
}

// writes counts pointer writes (the cost of a rotation), threaded by &mut.
// Go's `y.left = x.right` moves a pointer and leaves x.right as it was; Rust
// must MOVE the Box out with take() (leaving None behind) before it can store
// it elsewhere. That hidden store is not counted: this counts the three
// pointer assignments of the algorithm.
#[derive(Default)]
struct Counters {
    writes: usize,
}

// rot_right takes ownership of y and returns the new root of the subtree; the
// CALLER must store it where y used to hang. With no left child there is
// nothing to do.
fn rot_right(mut y: Box<Node>, c: &mut Counters) -> Box<Node> {
    let Some(mut x) = y.left.take() else {
        return y;
    };
    y.left = x.right.take(); // B moves across
    c.writes += 1;
    x.right = Some(y); // y goes down
    c.writes += 1;
    x
}

// the mirror image, and the inverse
fn rot_left(mut x: Box<Node>, c: &mut Counters) -> Box<Node> {
    let Some(mut y) = x.right.take() else {
        return x;
    };
    x.right = y.left.take();
    c.writes += 1;
    y.left = Some(x);
    c.writes += 1;
    y
}

// rotate_at rotates at the node holding key, if it has the child the rotation
// needs. It returns the key of the child it promoted (now the subtree root),
// or None when the node is missing or lacks that child. `cur` is a cursor on
// the link that holds the node, so a rotation at the ROOT needs no special
// case: storing the new subtree root in that link is the third pointer write.
fn rotate_at(root: &mut Link, key: usize, right: bool, c: &mut Counters) -> Option<usize> {
    let mut cur = root;
    while cur.as_ref().is_some_and(|n| n.key != key) {
        let n = cur.as_mut()?;
        cur = if key < n.key {
            &mut n.left
        } else {
            &mut n.right
        };
    }
    let n = cur.take()?;
    if (right && n.left.is_none()) || (!right && n.right.is_none()) {
        *cur = Some(n); // put it back untouched
        return None;
    }
    let new_root = if right {
        rot_right(n, c)
    } else {
        rot_left(n, c)
    };
    let promoted = new_root.key;
    *cur = Some(new_root);
    c.writes += 1;
    Some(promoted)
}

fn insert(root: &mut Link, key: usize) {
    let mut cur = root;
    while let Some(n) = cur {
        cur = if key < n.key {
            &mut n.left
        } else {
            &mut n.right
        };
    }
    *cur = Some(Box::new(Node {
        key,
        left: None,
        right: None,
    }));
}

fn leaf(key: usize) -> Link {
    Some(Box::new(Node {
        key,
        left: None,
        right: None,
    }))
}

// shape prints a tree as an s-expression: a leaf is its key, "." is a missing child.
fn shape(n: &Link) -> String {
    match n {
        None => ".".to_string(),
        Some(n) if n.left.is_none() && n.right.is_none() => n.key.to_string(),
        Some(n) => format!("({} {} {})", n.key, shape(&n.left), shape(&n.right)),
    }
}

// the root has depth 0
fn depth(root: &Link, key: usize) -> Option<usize> {
    let mut cur = root.as_deref();
    let mut d = 0;
    while let Some(n) = cur {
        if key == n.key {
            return Some(d);
        }
        cur = if key < n.key {
            n.left.as_deref()
        } else {
            n.right.as_deref()
        };
        d += 1;
    }
    None
}

fn move_text(d: isize) -> String {
    match d {
        d if d < 0 => format!("up {} level", -d),
        d if d > 0 => format!("down {d} level"),
        _ => "same depth".to_string(),
    }
}

// walk returns the keys in order and the height (nodes on the longest path),
// iteratively, so it is safe on any shape.
fn walk(root: &Link) -> (Vec<usize>, usize) {
    let mut keys = Vec::new();
    let mut h = 0;
    let mut stack: Vec<(&Node, usize)> = Vec::new();
    let (mut cur, mut d) = (root.as_deref(), 1);
    while cur.is_some() || !stack.is_empty() {
        while let Some(n) = cur {
            stack.push((n, d));
            cur = n.left.as_deref();
            d += 1;
        }
        let (n, depth) = stack.pop().expect("loop condition guarantees an entry");
        keys.push(n.key);
        h = h.max(depth);
        cur = n.right.as_deref();
        d = depth + 1;
    }
    (keys, h)
}

fn go_slice(a: &[usize]) -> String {
    // Go prints a slice as [1 2 3]; Rust's {:?} prints [1, 2, 3]. Format by hand.
    let parts: Vec<String> = a.iter().map(|k| k.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn random_tree(rng: &mut Pcg, n: usize) -> Link {
    let mut a: Vec<usize> = (1..=n).collect();
    for i in (1..n).rev() {
        // hand-written Fisher-Yates, as in example 1
        let j = rng.idx(i + 1);
        a.swap(i, j);
    }
    let mut root: Link = None;
    for k in a {
        insert(&mut root, k);
    }
    root
}

fn main() {
    let mut c = Counters::default();

    println!("== (a) rotate right at y = 4 on the five-node tree y(x(A, B), C) ==");
    // y=4, x=2, A=1, B=3, C=5
    let mut root: Link = Some(Box::new(Node {
        key: 4,
        left: Some(Box::new(Node {
            key: 2,
            left: leaf(1),
            right: leaf(3),
        })),
        right: leaf(5),
    }));
    let names = [("y=4", 4), ("x=2", 2), ("A=1", 1), ("B=3", 3), ("C=5", 5)];
    let show = |title: &str, root: &Link| -> Vec<usize> {
        let (keys, h) = walk(root);
        println!(
            "{:<18} {:<14} in-order {}  height {}",
            title,
            shape(root),
            go_slice(&keys),
            h
        );
        let depths: Vec<usize> = names
            .iter()
            .map(|&(_, key)| depth(root, key).expect("key is in the tree"))
            .collect();
        let parts: Vec<String> = names
            .iter()
            .zip(&depths)
            .map(|((label, _), d)| format!("{label}:{d}"))
            .collect();
        println!("{:<18} depths  {}", "", parts.join("  "));
        depths
    };
    let before = show("before:", &root);
    let original = shape(&root);
    rotate_at(&mut root, 4, true, &mut c);
    let after = show("after rotateRight:", &root);
    println!("pointer writes: {}", c.writes);
    for (i, (label, _)) in names.iter().enumerate().skip(2) {
        // the three subtrees A, B, C
        println!(
            "  subtree {label}: {}",
            move_text(after[i] as isize - before[i] as isize)
        );
    }
    rotate_at(&mut root, 2, false, &mut c);
    println!(
        "rotateLeft at x=2 restores {original}: {}",
        shape(&root) == original
    );

    println!(
        "\n== (b) every rotation has an inverse: 50-node random tree, each node, both directions =="
    );
    let mut rng = Pcg::new(1, 2);
    const SMALL: usize = 50;
    let mut t = random_tree(&mut rng, SMALL);
    let (mut checks, mut restored, mut skipped) = (0, 0, 0);
    for key in 1..=SMALL {
        for right in [true, false] {
            let before = shape(&t);
            let Some(promoted) = rotate_at(&mut t, key, right, &mut c) else {
                skipped += 1;
                continue;
            };
            rotate_at(&mut t, promoted, !right, &mut c); // the promoted child is rotated back down
            checks += 1;
            if shape(&t) == before {
                restored += 1;
            }
        }
    }
    println!(
        "rotate + inverse: {checks} checks, {restored} restored the exact shape, {skipped} skipped (no child on that side)"
    );
    println!(
        "checks == n-1 (= {}): {}; every non-root node is the child of exactly one node",
        SMALL - 1,
        checks == SMALL - 1
    );

    println!("\n== (c) 100,000 random rotations on a 1,000-node tree ==");
    const N: usize = 1000;
    t = random_tree(&mut rng, N);
    c = Counters::default();
    let (_, start_h) = walk(&t);
    let (mut performed, mut skipped_c, mut failures) = (0, 0, 0);
    let (mut min_h, mut max_h, mut sum_h, mut prev_h) = (start_h, start_h, 0, start_h);
    let (mut raised, mut lowered) = (0, 0);
    for _ in 0..100000 {
        let key = rng.idx(N) + 1;
        let right = rng.idx(2) == 0;
        if rotate_at(&mut t, key, right, &mut c).is_none() {
            skipped_c += 1;
            continue;
        }
        performed += 1;
        let (keys, h) = walk(&t);
        if keys.iter().enumerate().any(|(i, &k)| k != i + 1) {
            failures += 1;
        }
        (min_h, max_h, sum_h) = (min_h.min(h), max_h.max(h), sum_h + h);
        if h > prev_h {
            raised += 1;
        } else if h < prev_h {
            lowered += 1;
        }
        prev_h = h;
    }
    println!(
        "performed {performed}, skipped {skipped_c} (node lacked the child), in-order failures {failures}"
    );
    println!(
        "pointer writes {} = 3 x {performed} performed: {}",
        c.writes,
        c.writes == 3 * performed
    );
    println!(
        "height: started {start_h}, walked between {min_h} and {max_h} (mean {:.1}); the floor for n = {N} is {}",
        sum_h as f64 / performed as f64,
        usize::BITS - N.leading_zeros()
    );
    println!(
        "of {performed} rotations: {raised} raised the height, {lowered} lowered it, {} left it as it was",
        performed - raised - lowered
    );
}
```

**Rust output:**

```
== (a) rotate right at y = 4 on the five-node tree y(x(A, B), C) ==
before:            (4 (2 1 3) 5)  in-order [1 2 3 4 5]  height 3
                   depths  y=4:0  x=2:1  A=1:2  B=3:2  C=5:1
after rotateRight: (2 1 (4 3 5))  in-order [1 2 3 4 5]  height 3
                   depths  y=4:1  x=2:0  A=1:1  B=3:2  C=5:2
pointer writes: 3
  subtree A=1: up 1 level
  subtree B=3: same depth
  subtree C=5: down 1 level
rotateLeft at x=2 restores (4 (2 1 3) 5): true

== (b) every rotation has an inverse: 50-node random tree, each node, both directions ==
rotate + inverse: 49 checks, 49 restored the exact shape, 51 skipped (no child on that side)
checks == n-1 (= 49): true; every non-root node is the child of exactly one node

== (c) 100,000 random rotations on a 1,000-node tree ==
performed 49875, skipped 50125 (node lacked the child), in-order failures 0
pointer writes 149625 = 3 x 49875 performed: true
height: started 22, walked between 22 and 93 (mean 60.4); the floor for n = 1000 is 10
of 49875 rotations: 1434 raised the height, 1366 lowered it, 47075 left it as it was
```

> **Go vs Rust:** Identical to Go line for line (diffed), including the pointer-write total and the heights of the random walk. Go's `y.left = x.right` is one store; Rust cannot copy a `Box`, so it must move it out with `take()` (leaving `None` behind) before storing it elsewhere, and that hidden store is not counted: the count is the algorithm's three assignments. `rot_right(y: Box<Node>) -> Box<Node>` takes ownership and returns the new subtree root, which the caller must store where `y` hung; `rotate_at` finds that place with a `&mut Option<Box<Node>>` cursor, so a rotation at the root needs no special case, and its store into that link is the third write. Go's `%v` prints `[1 2 3 4 5]`, Rust's `{:?}` would print commas, so the slice is formatted by hand.

**Complexity:** a rotation is Θ(1) — three pointer writes, counted: 149,625 = 3 × 49,875 performed rotations — and it never reorders: 0 in-order failures over all 49,875, and every rotation was undone exactly by its mirror (49 of 49 checks restored the shape). It does change depth, which is the lever: the same random rotations took a tree of height 22 to between 22 and 93 (mean 60.4), because they were chosen blindly (1,434 raised the height, 1,366 lowered it, 47,075 left it unchanged). Balancing is the question of *which* rotation to make, which is what the next examples answer.

---

## 3. AVL: the four rebalance cases

`🟢 easy` · *AVL rebalancing*

An AVL tree keeps |height(left) − height(right)| ≤ 1 at every node by storing heights and repairing on the way back up from an insert: the first node whose balance factor reaches ±2 gets one of four fixes, chosen by the **sign of its heavy child's factor** — left-left and right-right take one rotation, left-right and right-left (a zig-zag) take two. That makes each repair Θ(1), so an insert stays Θ(log n) in all. This example shows the four cases on three keys, then exhausts every insertion order of 1..6 to show that the case table is the whole story and that an insert never repairs more than once.

**Steps:**

1. Insert three keys in each of the four orders with repair switched off, print the factor at the root and at its heavy child, then switch repair on and read the case, the rotation count and the result.
2. Insert every permutation of 1..n for n = 3 to 6 into a fresh tree, running the AVL checker after every insert, and tally the cases and the fix-ups per insert.
3. Print the decision table (factor, child factor) → case → rotations with an observed count per row, including the two rows where the heavy child is balanced.

```go
package main

import (
	"fmt"
	"slices"
)

// AVL invariant: at every node |height(left) - height(right)| <= 1.
// Each node stores its height (nil has height 0, a leaf 1).
type node struct {
	key, h      int
	left, right *node
}

func ht(n *node) int {
	if n == nil {
		return 0
	}
	return n.h
}

func upd(n *node)      { n.h = 1 + max(ht(n.left), ht(n.right)) }
func bf(n *node) int   { return ht(n.left) - ht(n.right) } // balance factor
func leaf(k int) *node { return &node{key: k, h: 1} }

// A rotation (example 2) that keeps heights right: recompute the node that
// went DOWN first, then the one that went up.
func rotateRight(y *node) *node {
	x := y.left
	y.left = x.right
	upd(y)
	x.right = y
	upd(x)
	return x
}

func rotateLeft(x *node) *node {
	y := x.right
	x.right = y.left
	upd(x)
	y.left = x
	upd(y)
	return y
}

// The decision table: row 0 is a node with factor +2 (left heavy), row 1 is
// -2; the column is the heavy child's factor -1, 0, +1.
var caseName = [2][3]string{
	{"left-right", "left-left", "left-left"},
	{"right-right", "right-right", "right-left"},
}

type stats struct {
	noFix    bool // grow the BST and keep heights, but never repair
	fixups   int  // nodes repaired
	rots     int  // rotations performed
	table    [2][3]int
	lastCase string // for printing example (a)
	lastRots int
}

// rebalance repairs n if its factor reached +-2. The first node on the way
// back up that gets there is the one to fix; the case is chosen by the SIGN of
// the heavy child's factor.
func rebalance(n *node, st *stats) *node {
	upd(n)
	b := bf(n)
	if (b < 2 && b > -2) || st.noFix {
		return n
	}
	row, cb := 0, 0
	if b == 2 {
		cb = bf(n.left)
	} else {
		row, cb = 1, bf(n.right)
	}
	st.fixups++
	st.table[row][cb+1]++
	st.lastCase = caseName[row][cb+1]
	st.lastRots = 1
	switch {
	case b == 2 && cb == -1: // left-right: straighten the zig-zag first
		n.left = rotateLeft(n.left)
		st.rots++
		st.lastRots = 2
	case b == -2 && cb == 1: // right-left
		n.right = rotateRight(n.right)
		st.rots++
		st.lastRots = 2
	}
	st.rots++
	if b == 2 {
		return rotateRight(n)
	}
	return rotateLeft(n)
}

func insert(n *node, key int, st *stats) *node {
	if n == nil {
		return leaf(key)
	}
	switch {
	case key < n.key:
		n.left = insert(n.left, key, st)
	case key > n.key:
		n.right = insert(n.right, key, st)
	default:
		return n
	}
	return rebalance(n, st)
}

// check is the AVL checker: BST order, stored height == recomputed height, |factor| <= 1.
func check(n *node, lo, hi int) (int, error) {
	if n == nil {
		return 0, nil
	}
	if n.key <= lo || n.key >= hi {
		return 0, fmt.Errorf("key %d out of order", n.key)
	}
	lh, err := check(n.left, lo, n.key)
	if err != nil {
		return 0, err
	}
	rh, err := check(n.right, n.key, hi)
	if err != nil {
		return 0, err
	}
	if h := 1 + max(lh, rh); n.h != h {
		return 0, fmt.Errorf("key %d stores height %d, real %d", n.key, n.h, h)
	} else if lh-rh > 1 || rh-lh > 1 {
		return 0, fmt.Errorf("key %d has factor %d", n.key, lh-rh)
	} else {
		return h, nil
	}
}

func verdict(err error) string {
	if err != nil {
		return err.Error()
	}
	return "ok"
}

func shape(n *node) string {
	switch {
	case n == nil:
		return "."
	case n.left == nil && n.right == nil:
		return fmt.Sprint(n.key)
	}
	return "(" + fmt.Sprint(n.key) + " " + shape(n.left) + " " + shape(n.right) + ")"
}

func nextPerm(a []int) bool { // lexicographic next permutation, in place
	i := len(a) - 2
	for i >= 0 && a[i] >= a[i+1] {
		i--
	}
	if i < 0 {
		return false
	}
	j := len(a) - 1
	for a[j] <= a[i] {
		j--
	}
	a[i], a[j] = a[j], a[i]
	slices.Reverse(a[i+1:])
	return true
}

func main() {
	fmt.Println("== (a) the four cases on three keys ==")
	for _, seq := range [][]int{{3, 2, 1}, {1, 2, 3}, {3, 1, 2}, {1, 3, 2}} {
		var plain, fixed *node
		var p, f stats
		p.noFix = true
		for _, k := range seq {
			plain = insert(plain, k, &p)
			fixed = insert(fixed, k, &f)
		}
		heavy := plain.left
		if bf(plain) < 0 {
			heavy = plain.right
		}
		fmt.Printf("insert %v\n", seq)
		fmt.Printf("  BST step : %-12s factor at root %+d, at its heavy child %+d\n", shape(plain), bf(plain), bf(heavy))
		fmt.Printf("  case     : %s, %d rotation(s)\n", f.lastCase, f.lastRots)
		fmt.Printf("  repaired : %s\n", shape(fixed))
		_, errPlain := check(plain, 0, 10)
		_, errFixed := check(fixed, 0, 10)
		fmt.Printf("  checker  : BST step -> %s; repaired -> %s\n", verdict(errPlain), verdict(errFixed))
	}

	fmt.Println("\n== (b) every permutation of 1..n, checker after every insert ==")
	fmt.Printf("%2s %6s %8s %7s %5s %5s %5s %5s %12s %9s\n", "n", "perms", "inserts", "fixups", "LL", "RR", "LR", "RL", "max per ins", "failures")
	var total stats
	allInserts, allMax, allFailures := 0, 0, 0
	for n := 3; n <= 6; n++ {
		perm := make([]int, n)
		for i := range perm {
			perm[i] = i + 1
		}
		var st stats
		perms, inserts, maxFix, failures := 0, 0, 0, 0
		for ok := true; ok; ok = nextPerm(perm) {
			perms++
			var root *node
			for _, k := range perm {
				before := st.fixups
				root = insert(root, k, &st)
				inserts++
				maxFix = max(maxFix, st.fixups-before)
				if _, err := check(root, 0, n+1); err != nil {
					failures++
				}
			}
		}
		fmt.Printf("%2d %6d %8d %7d %5d %5d %5d %5d %12d %9d\n", n, perms, inserts, st.fixups,
			st.table[0][1]+st.table[0][2], st.table[1][0]+st.table[1][1], st.table[0][0], st.table[1][2], maxFix, failures)
		for r := range st.table {
			for c := range st.table[r] {
				total.table[r][c] += st.table[r][c]
			}
		}
		total.fixups += st.fixups
		allInserts, allMax, allFailures = allInserts+inserts, max(allMax, maxFix), allFailures+failures
		total.rots += st.rots
	}

	fmt.Printf("over all %d inserts: at most %d fix-up(s) in any one insert, %d checker failures\n", allInserts, allMax, allFailures)

	fmt.Println("\n== (c) the decision table, counted over all of (b) ==")
	fmt.Printf("%-12s %-12s %-12s %9s %9s\n", "factor", "child factor", "case", "rotations", "observed")
	sum := 0
	for r, b := range []int{2, -2} {
		for c, cb := range []int{-1, 0, 1} {
			rots := 1
			if (b == 2 && cb == -1) || (b == -2 && cb == 1) {
				rots = 2
			}
			fmt.Printf("%+-12d %+-12d %-12s %9d %9d\n", b, cb, caseName[r][c], rots, total.table[r][c])
			sum += total.table[r][c]
		}
	}
	doubles := total.table[0][0] + total.table[1][2]
	fmt.Printf("rows sum to %d fix-ups: %v\n", sum, sum == total.fixups)
	fmt.Printf("rotations performed %d = singles %d + 2 x doubles %d: %v\n", total.rots, sum-doubles, doubles, total.rots == sum+doubles)
	fmt.Printf("inserts that reached a factor-0 heavy child (the two middle rows): %d\n", total.table[0][1]+total.table[1][1])
}
```

**Output:**

```
== (a) the four cases on three keys ==
insert [3 2 1]
  BST step : (3 (2 1 .) .) factor at root +2, at its heavy child +1
  case     : left-left, 1 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 3 has factor 2; repaired -> ok
insert [1 2 3]
  BST step : (1 . (2 . 3)) factor at root -2, at its heavy child -1
  case     : right-right, 1 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 1 has factor -2; repaired -> ok
insert [3 1 2]
  BST step : (3 (1 . 2) .) factor at root +2, at its heavy child -1
  case     : left-right, 2 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 3 has factor 2; repaired -> ok
insert [1 3 2]
  BST step : (1 . (3 2 .)) factor at root -2, at its heavy child +1
  case     : right-left, 2 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 1 has factor -2; repaired -> ok

== (b) every permutation of 1..n, checker after every insert ==
 n  perms  inserts  fixups    LL    RR    LR    RL  max per ins  failures
 3      6       18       4     1     1     1     1            1         0
 4     24       96      16     4     4     4     4            1         0
 5    120      600     128    32    32    32    32            1         0
 6    720     4320    1248   312   312   312   312            1         0
over all 5034 inserts: at most 1 fix-up(s) in any one insert, 0 checker failures

== (c) the decision table, counted over all of (b) ==
factor       child factor case         rotations  observed
+2           -1           left-right           2       349
+2           +0           left-left            1         0
+2           +1           left-left            1       349
-2           -1           right-right          1       349
-2           +0           right-right          1         0
-2           +1           right-left           2       349
rows sum to 1396 fix-ups: true
rotations performed 2094 = singles 698 + 2 x doubles 698: true
inserts that reached a factor-0 heavy child (the two middle rows): 0
```

**Rust:**

```rust
// AVL invariant: at every node |height(left) - height(right)| <= 1.
// Each node stores its height (nil has height 0, a leaf 1).
struct Node {
    key: usize,
    h: i32,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

fn ht(l: &Link) -> i32 {
    l.as_ref().map_or(0, |n| n.h)
}

fn upd(n: &mut Node) {
    n.h = 1 + ht(&n.left).max(ht(&n.right));
}

// balance factor
fn bf(n: &Node) -> i32 {
    ht(&n.left) - ht(&n.right)
}

fn leaf(key: usize) -> Box<Node> {
    Box::new(Node {
        key,
        h: 1,
        left: None,
        right: None,
    })
}

// A rotation (example 2) that keeps heights right: recompute the node that
// went DOWN first, then the one that went up. Each rotation takes ownership
// of the subtree root and returns the new root; take() moves a Box out of a
// field behind &mut, which Go's `y.left = x.right` does not need.
fn rot_right(mut y: Box<Node>) -> Box<Node> {
    let Some(mut x) = y.left.take() else {
        return y;
    };
    y.left = x.right.take();
    upd(&mut y);
    x.right = Some(y);
    upd(&mut x);
    x
}

fn rot_left(mut x: Box<Node>) -> Box<Node> {
    let Some(mut y) = x.right.take() else {
        return x;
    };
    x.right = y.left.take();
    upd(&mut x);
    y.left = Some(x);
    upd(&mut y);
    y
}

// The decision table: row 0 is a node with factor +2 (left heavy), row 1 is
// -2; the column is the heavy child's factor -1, 0, +1.
const CASE_NAME: [[&str; 3]; 2] = [
    ["left-right", "left-left", "left-left"],
    ["right-right", "right-right", "right-left"],
];

#[derive(Default)]
struct Stats {
    no_fix: bool,  // grow the BST and keep heights, but never repair
    fixups: usize, // nodes repaired
    rots: usize,   // rotations performed
    table: [[usize; 3]; 2],
    last_case: &'static str, // for printing example (a)
    last_rots: usize,
}

// rebalance repairs n if its factor reached +-2. The first node on the way
// back up that gets there is the one to fix; the case is chosen by the SIGN of
// the heavy child's factor.
fn rebalance(mut n: Box<Node>, st: &mut Stats) -> Box<Node> {
    upd(&mut n);
    let b = bf(&n);
    if (-1..=1).contains(&b) || st.no_fix {
        return n;
    }
    let (row, cb) = if b == 2 {
        (0, n.left.as_deref().map_or(0, bf))
    } else {
        (1, n.right.as_deref().map_or(0, bf))
    };
    st.fixups += 1;
    st.table[row][(cb + 1) as usize] += 1;
    st.last_case = CASE_NAME[row][(cb + 1) as usize];
    st.last_rots = 1;
    if b == 2 && cb == -1 {
        // left-right: straighten the zig-zag first
        n.left = n.left.take().map(rot_left);
        st.rots += 1;
        st.last_rots = 2;
    } else if b == -2 && cb == 1 {
        // right-left
        n.right = n.right.take().map(rot_right);
        st.rots += 1;
        st.last_rots = 2;
    }
    st.rots += 1;
    if b == 2 { rot_right(n) } else { rot_left(n) }
}

// insert returns the new root of the subtree: rebalancing replaces the node
// you are standing on, so the recursion hands the replacement back up.
fn insert(link: Link, key: usize, st: &mut Stats) -> Link {
    let Some(mut n) = link else {
        return Some(leaf(key));
    };
    if key < n.key {
        n.left = insert(n.left.take(), key, st);
    } else if key > n.key {
        n.right = insert(n.right.take(), key, st);
    } else {
        return Some(n);
    }
    Some(rebalance(n, st))
}

// check is the AVL checker: BST order, stored height == recomputed height, |factor| <= 1.
fn check(n: &Link, lo: usize, hi: usize) -> Result<i32, String> {
    let Some(n) = n else {
        return Ok(0);
    };
    if n.key <= lo || n.key >= hi {
        return Err(format!("key {} out of order", n.key));
    }
    let lh = check(&n.left, lo, n.key)?;
    let rh = check(&n.right, n.key, hi)?;
    let h = 1 + lh.max(rh);
    if n.h != h {
        Err(format!("key {} stores height {}, real {}", n.key, n.h, h))
    } else if (lh - rh).abs() > 1 {
        Err(format!("key {} has factor {}", n.key, lh - rh))
    } else {
        Ok(h)
    }
}

fn verdict(r: Result<i32, String>) -> String {
    r.map_or_else(|e| e, |_| "ok".to_string())
}

fn shape(n: &Link) -> String {
    match n {
        None => ".".to_string(),
        Some(n) if n.left.is_none() && n.right.is_none() => n.key.to_string(),
        Some(n) => format!("({} {} {})", n.key, shape(&n.left), shape(&n.right)),
    }
}

// lexicographic next permutation, in place (Rust's std has none)
fn next_perm(a: &mut [usize]) -> bool {
    let Some(i) = (0..a.len().saturating_sub(1))
        .rev()
        .find(|&i| a[i] < a[i + 1])
    else {
        return false;
    };
    let j = (i + 1..a.len())
        .rev()
        .find(|&j| a[j] > a[i])
        .expect("a[i] < a[i+1], so a larger element exists");
    a.swap(i, j);
    a[i + 1..].reverse();
    true
}

// Go prints a slice as [3 2 1]; Rust's {:?} prints [3, 2, 1]. Format by hand.
fn go_slice(a: &[usize]) -> String {
    let parts: Vec<String> = a.iter().map(|k| k.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    println!("== (a) the four cases on three keys ==");
    for seq in [[3, 2, 1], [1, 2, 3], [3, 1, 2], [1, 3, 2]] {
        let (mut plain, mut fixed): (Link, Link) = (None, None);
        let mut p = Stats {
            no_fix: true,
            ..Stats::default()
        };
        let mut f = Stats::default();
        for k in seq {
            plain = insert(plain, k, &mut p);
            fixed = insert(fixed, k, &mut f);
        }
        let root = plain.as_deref().expect("three keys were inserted");
        let heavy = if bf(root) < 0 {
            &root.right
        } else {
            &root.left
        };
        println!("insert {}", go_slice(&seq));
        println!(
            "  BST step : {:<12} factor at root {:+}, at its heavy child {:+}",
            shape(&plain),
            bf(root),
            heavy.as_deref().map_or(0, bf)
        );
        println!("  case     : {}, {} rotation(s)", f.last_case, f.last_rots);
        println!("  repaired : {}", shape(&fixed));
        println!(
            "  checker  : BST step -> {}; repaired -> {}",
            verdict(check(&plain, 0, 10)),
            verdict(check(&fixed, 0, 10))
        );
    }

    println!("\n== (b) every permutation of 1..n, checker after every insert ==");
    println!(
        "{:>2} {:>6} {:>8} {:>7} {:>5} {:>5} {:>5} {:>5} {:>12} {:>9}",
        "n", "perms", "inserts", "fixups", "LL", "RR", "LR", "RL", "max per ins", "failures"
    );
    let mut total = Stats::default();
    let (mut all_inserts, mut all_max, mut all_failures) = (0, 0, 0);
    for n in 3..=6 {
        let mut perm: Vec<usize> = (1..=n).collect();
        let mut st = Stats::default();
        let (mut perms, mut inserts, mut max_fix, mut failures) = (0, 0, 0, 0);
        loop {
            perms += 1;
            let mut root: Link = None;
            for &k in &perm {
                let before = st.fixups;
                root = insert(root, k, &mut st);
                inserts += 1;
                max_fix = max_fix.max(st.fixups - before);
                if check(&root, 0, n + 1).is_err() {
                    failures += 1;
                }
            }
            if !next_perm(&mut perm) {
                break;
            }
        }
        println!(
            "{:>2} {:>6} {:>8} {:>7} {:>5} {:>5} {:>5} {:>5} {:>12} {:>9}",
            n,
            perms,
            inserts,
            st.fixups,
            st.table[0][1] + st.table[0][2],
            st.table[1][0] + st.table[1][1],
            st.table[0][0],
            st.table[1][2],
            max_fix,
            failures
        );
        for (tr, sr) in total.table.iter_mut().zip(&st.table) {
            for (t, s) in tr.iter_mut().zip(sr) {
                *t += s;
            }
        }
        total.fixups += st.fixups;
        total.rots += st.rots;
        all_inserts += inserts;
        all_max = all_max.max(max_fix);
        all_failures += failures;
    }
    println!(
        "over all {all_inserts} inserts: at most {all_max} fix-up(s) in any one insert, {all_failures} checker failures"
    );

    println!("\n== (c) the decision table, counted over all of (b) ==");
    println!(
        "{:<12} {:<12} {:<12} {:>9} {:>9}",
        "factor", "child factor", "case", "rotations", "observed"
    );
    let mut sum = 0;
    for (r, b) in [2, -2].into_iter().enumerate() {
        for (c, cb) in [-1, 0, 1].into_iter().enumerate() {
            let rots = if (b == 2 && cb == -1) || (b == -2 && cb == 1) {
                2
            } else {
                1
            };
            println!(
                "{:<+12} {:<+12} {:<12} {:>9} {:>9}",
                b, cb, CASE_NAME[r][c], rots, total.table[r][c]
            );
            sum += total.table[r][c];
        }
    }
    let doubles = total.table[0][0] + total.table[1][2];
    println!("rows sum to {sum} fix-ups: {}", sum == total.fixups);
    println!(
        "rotations performed {} = singles {} + 2 x doubles {doubles}: {}",
        total.rots,
        sum - doubles,
        total.rots == sum + doubles
    );
    println!(
        "inserts that reached a factor-0 heavy child (the two middle rows): {}",
        total.table[0][1] + total.table[1][1]
    );
}
```

**Rust output:**

```
== (a) the four cases on three keys ==
insert [3 2 1]
  BST step : (3 (2 1 .) .) factor at root +2, at its heavy child +1
  case     : left-left, 1 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 3 has factor 2; repaired -> ok
insert [1 2 3]
  BST step : (1 . (2 . 3)) factor at root -2, at its heavy child -1
  case     : right-right, 1 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 1 has factor -2; repaired -> ok
insert [3 1 2]
  BST step : (3 (1 . 2) .) factor at root +2, at its heavy child -1
  case     : left-right, 2 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 3 has factor 2; repaired -> ok
insert [1 3 2]
  BST step : (1 . (3 2 .)) factor at root -2, at its heavy child +1
  case     : right-left, 2 rotation(s)
  repaired : (2 1 3)
  checker  : BST step -> key 1 has factor -2; repaired -> ok

== (b) every permutation of 1..n, checker after every insert ==
 n  perms  inserts  fixups    LL    RR    LR    RL  max per ins  failures
 3      6       18       4     1     1     1     1            1         0
 4     24       96      16     4     4     4     4            1         0
 5    120      600     128    32    32    32    32            1         0
 6    720     4320    1248   312   312   312   312            1         0
over all 5034 inserts: at most 1 fix-up(s) in any one insert, 0 checker failures

== (c) the decision table, counted over all of (b) ==
factor       child factor case         rotations  observed
+2           -1           left-right           2       349
+2           +0           left-left            1         0
+2           +1           left-left            1       349
-2           -1           right-right          1       349
-2           +0           right-right          1         0
-2           +1           right-left           2       349
rows sum to 1396 fix-ups: true
rotations performed 2094 = singles 698 + 2 x doubles 698: true
inserts that reached a factor-0 heavy child (the two middle rows): 0
```

> **Go vs Rust:** Identical to Go line for line (diffed). `insert(link: Link, key, st) -> Link` returns the new subtree root, because rebalancing replaces the node you are standing on; `take()` moves each child out and the recursion puts the result back. The double rotation is `n.left = n.left.take().map(rot_left)`, which needs no `unwrap`: a factor of +2 implies a left child, so the `None` arm is unreachable (and the checker would flag the tree if it were not). Std has no next-permutation, so it is hand-written in both languages; `{:+}` and `{:<+12}` replace Go's `%+d` and `%+-12d`, and `[3 2 1]` is formatted by hand.

**Complexity:** one fix-up is Θ(1) — one rotation for left-left and right-right, two for left-right and right-left — and over all 5,034 inserts of every permutation of 1..6 no insert needed more than one fix-up (1,396 inserts needed one: 349 of each case, 2,094 rotations in all, 0 checker failures). The two rows with a balanced heavy child (factor 0) were reached 0 times by inserts; a delete is the only way to reach them (examples 7 and 12 delete).

---

## 4. The sparsest AVL tree is a Fibonacci tree

`🟢 easy` · *The 1.44·lg n bound*

Why can an AVL tree never be much taller than the ideal ⌈lg(n+1)⌉? Count the fewest nodes N(h) an AVL tree of height h can have: its sparsest form hangs a sparsest tree of height h−1 and one of height h−2 off the root, so N(h) = N(h−1) + N(h−2) + 1 = F(h+2) − 1, a Fibonacci number minus one, and inverting that gives h < 1.4405·lg(n+2) − 0.3277. This is pure shape and arithmetic, with no insert code at all, and it is the reason AVL search and insert are Θ(log n) in the worst case rather than only on average.

**Steps:**

1. Compute N(h) from the recurrence and compare it with Fibonacci numbers from an independent loop, then build the sparsest tree for h = 1 to 16 and run a checker on each.
2. Prove the minimum by exhaustion: enumerate every AVL shape of height up to 5 as a node count (108,675 shapes at h = 5) and check that the smallest is N(h) and the largest 2^h − 1.
3. Invert the recurrence into the exact tallest height for n up to 10⁶ and compare it with the continuous bound and with the ideal height.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
)

// How tall can an AVL tree get? Let N(h) be the FEWEST nodes in an AVL tree of
// height h (height = nodes on the longest root-to-nil path; empty = 0).
// The sparsest tree hangs a sparsest tree of height h-1 and one of height h-2
// off its root (the factor allows a difference of 1, never more):
//
//	N(0) = 0, N(1) = 1, N(h) = N(h-1) + N(h-2) + 1   which is F(h+2) - 1.
type node struct {
	key, h      int
	left, right *node
}

// fibTree builds that sparsest tree, handing out keys 1, 2, 3, ... in order.
func fibTree(h int, next *int) *node {
	if h <= 0 {
		return nil
	}
	n := &node{h: h}
	n.left = fibTree(h-1, next)
	*next++
	n.key = *next
	n.right = fibTree(h-2, next)
	return n
}

// check returns the node count and an error if the tree is not a valid AVL
// tree: stored heights must be the real ones, |factor| <= 1, keys ascending.
func check(n *node, last *int) (int, int, error) {
	if n == nil {
		return 0, 0, nil
	}
	ls, lh, err := check(n.left, last)
	if err != nil {
		return 0, 0, err
	}
	if n.key != *last+1 {
		return 0, 0, fmt.Errorf("key %d out of order", n.key)
	}
	*last = n.key
	rs, rh, err := check(n.right, last)
	if err != nil {
		return 0, 0, err
	}
	if n.h != 1+max(lh, rh) {
		return 0, 0, fmt.Errorf("key %d stores height %d, real %d", n.key, n.h, 1+max(lh, rh))
	}
	if lh-rh > 1 || rh-lh > 1 {
		return 0, 0, fmt.Errorf("key %d has factor %d", n.key, lh-rh)
	}
	return 1 + ls + rs, n.h, nil
}

func main() {
	// minimal[h] = N(h), from the recurrence
	minimal := []int{0, 1}
	for h := 2; h <= 40; h++ {
		minimal = append(minimal, minimal[h-1]+minimal[h-2]+1)
	}

	fmt.Println("== (a) N(h) from the recurrence against Fibonacci, h = 1..12 ==")
	fmt.Printf("%3s %6s %10s %6s %9s\n", "h", "N(h)", "F(h+2)-1", "match", "2^h - 1")
	fib := []int{0, 1} // F(0), F(1), ... by an independent loop
	for len(fib) < 45 {
		fib = append(fib, fib[len(fib)-1]+fib[len(fib)-2])
	}
	for h := 1; h <= 12; h++ {
		fmt.Printf("%3d %6d %10d %6v %9d\n", h, minimal[h], fib[h+2]-1, minimal[h] == fib[h+2]-1, 1<<h-1)
	}

	fmt.Println("\n== (b) build the sparsest tree and run the checker, h = 1..16 ==")
	fmt.Printf("%3s %7s %7s %6s\n", "h", "nodes", "N(h)", "valid")
	for h := 1; h <= 16; h++ {
		next, last := 0, 0
		t := fibTree(h, &next)
		size, height, err := check(t, &last)
		fmt.Printf("%3d %7d %7d %6v\n", h, size, minimal[h], err == nil && height == h && size == minimal[h])
	}
	path := &node{key: 1, h: 3, right: &node{key: 2, h: 2, right: &node{key: 3, h: 1}}}
	last := 0
	_, _, err := check(path, &last)
	fmt.Printf("control, the path 1-2-3 (not AVL): %v\n", err)

	fmt.Println("\n== (c) every AVL shape of height h, as a list of node counts (h <= 5) ==")
	// sizes[h] has one entry per shape: the subtrees' heights are (h-1, h-1), (h-1, h-2) or (h-2, h-1).
	sizes := [][]int{{0}, {1}}
	for h := 2; h <= 5; h++ {
		var s []int
		for _, a := range sizes[h-1] {
			for _, b := range sizes[h-1] {
				s = append(s, 1+a+b)
			}
			for _, b := range sizes[h-2] {
				s = append(s, 1+a+b, 1+b+a) // (h-1, h-2) and the mirror (h-2, h-1)
			}
		}
		sizes = append(sizes, s)
	}
	fmt.Printf("%3s %9s %9s %6s %5s %6s %5s %8s\n", "h", "shapes", "formula", "match", "min", "N(h)", "max", "2^h - 1")
	t := []int{1, 1} // T(0), T(1); T(h) = T(h-1)^2 + 2 T(h-1) T(h-2)
	for h := 2; h <= 5; h++ {
		t = append(t, t[h-1]*t[h-1]+2*t[h-1]*t[h-2])
	}
	for h := 1; h <= 5; h++ {
		lo, hi := sizes[h][0], sizes[h][0]
		for _, v := range sizes[h] {
			lo, hi = min(lo, v), max(hi, v)
		}
		fmt.Printf("%3d %9d %9d %6v %5d %6d %5d %8d\n", h, len(sizes[h]), t[h], len(sizes[h]) == t[h], lo, minimal[h], hi, 1<<h-1)
	}

	fmt.Println("\n== (d) the tallest AVL tree for n nodes, against the continuous bound ==")
	fmt.Println("bound = 1.4405 * lg(n+2) - 0.3277; tallest = the largest h with N(h) <= n")
	fmt.Printf("%9s %6s %8s %7s %13s %13s\n", "n", "ideal", "tallest", "bound", "tallest/ideal", "tallest<bound")
	var tallest, n int
	for _, n = range []int{10, 100, 1000, 10000, 1000000} {
		tallest = 0
		for minimal[tallest+1] <= n {
			tallest++
		}
		bound := 1.4405*math.Log2(float64(n+2)) - 0.3277
		ideal := bits.Len(uint(n))
		fmt.Printf("%9d %6d %8d %7.2f %13.2f %13v\n", n, ideal, tallest, bound, float64(tallest)/float64(ideal), float64(tallest) < bound)
	}
	fmt.Printf("check by hand for n = %d: N(%d) = %d <= n < N(%d) = %d\n", n, tallest, minimal[tallest], tallest+1, minimal[tallest+1])
}
```

**Output:**

```
== (a) N(h) from the recurrence against Fibonacci, h = 1..12 ==
  h   N(h)   F(h+2)-1  match   2^h - 1
  1      1          1   true         1
  2      2          2   true         3
  3      4          4   true         7
  4      7          7   true        15
  5     12         12   true        31
  6     20         20   true        63
  7     33         33   true       127
  8     54         54   true       255
  9     88         88   true       511
 10    143        143   true      1023
 11    232        232   true      2047
 12    376        376   true      4095

== (b) build the sparsest tree and run the checker, h = 1..16 ==
  h   nodes    N(h)  valid
  1       1       1   true
  2       2       2   true
  3       4       4   true
  4       7       7   true
  5      12      12   true
  6      20      20   true
  7      33      33   true
  8      54      54   true
  9      88      88   true
 10     143     143   true
 11     232     232   true
 12     376     376   true
 13     609     609   true
 14     986     986   true
 15    1596    1596   true
 16    2583    2583   true
control, the path 1-2-3 (not AVL): key 1 has factor -2

== (c) every AVL shape of height h, as a list of node counts (h <= 5) ==
  h    shapes   formula  match   min   N(h)   max  2^h - 1
  1         1         1   true     1      1     1        1
  2         3         3   true     2      2     3        3
  3        15        15   true     4      4     7        7
  4       315       315   true     7      7    15       15
  5    108675    108675   true    12     12    31       31

== (d) the tallest AVL tree for n nodes, against the continuous bound ==
bound = 1.4405 * lg(n+2) - 0.3277; tallest = the largest h with N(h) <= n
        n  ideal  tallest   bound tallest/ideal tallest<bound
       10      4        4    4.84          1.00          true
      100      7        9    9.28          1.29          true
     1000     10       14   14.03          1.40          true
    10000     14       18   18.81          1.29          true
  1000000     20       28   28.38          1.40          true
check by hand for n = 1000000: N(28) = 832039 <= n < N(29) = 1346268
```

**Rust:**

```rust
// How tall can an AVL tree get? Let N(h) be the FEWEST nodes in an AVL tree of
// height h (height = nodes on the longest root-to-nil path; empty = 0).
// The sparsest tree hangs a sparsest tree of height h-1 and one of height h-2
// off its root (the factor allows a difference of 1, never more):
//
//	N(0) = 0, N(1) = 1, N(h) = N(h-1) + N(h-2) + 1   which is F(h+2) - 1.
struct Node {
    key: usize,
    h: usize,
    left: Link,
    right: Link,
}
type Link = Option<Box<Node>>;

// fib_tree builds that sparsest tree, handing out keys 1, 2, 3, ... in order.
// (Depth is at most h <= 16 here, so the default recursive Drop is safe.)
fn fib_tree(h: usize, next: &mut usize) -> Link {
    if h == 0 {
        return None;
    }
    let left = fib_tree(h - 1, next);
    *next += 1;
    let key = *next;
    // Go passes h-2 = -1 for h = 1 and treats h <= 0 as nil; usize cannot go
    // below 0, so the subtraction is spelled out.
    let right = fib_tree(h.saturating_sub(2), next);
    Some(Box::new(Node {
        key,
        h,
        left,
        right,
    }))
}

// check returns (node count, height) or an error if the tree is not a valid
// AVL tree: stored heights must be the real ones, |factor| <= 1, keys ascending.
fn check(n: &Link, last: &mut usize) -> Result<(usize, usize), String> {
    let Some(n) = n else {
        return Ok((0, 0));
    };
    let (ls, lh) = check(&n.left, last)?;
    if n.key != *last + 1 {
        return Err(format!("key {} out of order", n.key));
    }
    *last = n.key;
    let (rs, rh) = check(&n.right, last)?;
    if n.h != 1 + lh.max(rh) {
        return Err(format!(
            "key {} stores height {}, real {}",
            n.key,
            n.h,
            1 + lh.max(rh)
        ));
    }
    if lh.abs_diff(rh) > 1 {
        return Err(format!(
            "key {} has factor {}",
            n.key,
            lh as isize - rh as isize
        ));
    }
    Ok((1 + ls + rs, n.h))
}

fn main() {
    // minimal[h] = N(h), from the recurrence
    let mut minimal: Vec<u64> = vec![0, 1];
    for h in 2..=40 {
        minimal.push(minimal[h - 1] + minimal[h - 2] + 1);
    }

    println!("== (a) N(h) from the recurrence against Fibonacci, h = 1..12 ==");
    println!(
        "{:>3} {:>6} {:>10} {:>6} {:>9}",
        "h", "N(h)", "F(h+2)-1", "match", "2^h - 1"
    );
    let mut fib: Vec<u64> = vec![0, 1]; // F(0), F(1), ... by an independent loop
    while fib.len() < 45 {
        fib.push(fib[fib.len() - 1] + fib[fib.len() - 2]);
    }
    for h in 1..=12 {
        println!(
            "{:>3} {:>6} {:>10} {:>6} {:>9}",
            h,
            minimal[h],
            fib[h + 2] - 1,
            minimal[h] == fib[h + 2] - 1,
            (1u64 << h) - 1
        );
    }

    println!("\n== (b) build the sparsest tree and run the checker, h = 1..16 ==");
    println!("{:>3} {:>7} {:>7} {:>6}", "h", "nodes", "N(h)", "valid");
    for (h, &n_min) in minimal.iter().enumerate().take(17).skip(1) {
        let (mut next, mut last) = (0, 0);
        let t = fib_tree(h, &mut next);
        let res = check(&t, &mut last);
        let valid = matches!(res, Ok((size, height)) if height == h && size as u64 == n_min);
        let size = res.map_or(0, |(size, _)| size);
        println!("{:>3} {:>7} {:>7} {:>6}", h, size, n_min, valid);
    }
    let path: Link = Some(Box::new(Node {
        key: 1,
        h: 3,
        left: None,
        right: Some(Box::new(Node {
            key: 2,
            h: 2,
            left: None,
            right: Some(Box::new(Node {
                key: 3,
                h: 1,
                left: None,
                right: None,
            })),
        })),
    }));
    let mut last = 0;
    println!(
        "control, the path 1-2-3 (not AVL): {}",
        check(&path, &mut last).expect_err("a path of 3 nodes is not AVL")
    );

    println!("\n== (c) every AVL shape of height h, as a list of node counts (h <= 5) ==");
    // sizes[h] has one entry per shape: the subtrees' heights are (h-1, h-1), (h-1, h-2) or (h-2, h-1).
    let mut sizes: Vec<Vec<u64>> = vec![vec![0], vec![1]];
    for h in 2..=5 {
        let mut s = Vec::new();
        for &a in &sizes[h - 1] {
            for &b in &sizes[h - 1] {
                s.push(1 + a + b);
            }
            for &b in &sizes[h - 2] {
                s.push(1 + a + b); // (h-1, h-2)
                s.push(1 + b + a); // and the mirror (h-2, h-1)
            }
        }
        sizes.push(s);
    }
    println!(
        "{:>3} {:>9} {:>9} {:>6} {:>5} {:>6} {:>5} {:>8}",
        "h", "shapes", "formula", "match", "min", "N(h)", "max", "2^h - 1"
    );
    let mut t: Vec<usize> = vec![1, 1]; // T(0), T(1); T(h) = T(h-1)^2 + 2 T(h-1) T(h-2)
    for h in 2..=5 {
        t.push(t[h - 1] * t[h - 1] + 2 * t[h - 1] * t[h - 2]);
    }
    for h in 1..=5 {
        let lo = sizes[h].iter().min().expect("every height has a shape");
        let hi = sizes[h].iter().max().expect("every height has a shape");
        println!(
            "{:>3} {:>9} {:>9} {:>6} {:>5} {:>6} {:>5} {:>8}",
            h,
            sizes[h].len(),
            t[h],
            sizes[h].len() == t[h],
            lo,
            minimal[h],
            hi,
            (1u64 << h) - 1
        );
    }

    println!("\n== (d) the tallest AVL tree for n nodes, against the continuous bound ==");
    println!("bound = 1.4405 * lg(n+2) - 0.3277; tallest = the largest h with N(h) <= n");
    println!(
        "{:>9} {:>6} {:>8} {:>7} {:>13} {:>13}",
        "n", "ideal", "tallest", "bound", "tallest/ideal", "tallest<bound"
    );
    let (mut tallest, mut last_n) = (0, 0);
    for n in [10u64, 100, 1000, 10000, 1000000] {
        tallest = 0;
        while minimal[tallest + 1] <= n {
            tallest += 1;
        }
        last_n = n;
        let bound = 1.4405 * ((n + 2) as f64).log2() - 0.3277;
        let ideal = u64::BITS - n.leading_zeros();
        println!(
            "{:>9} {:>6} {:>8} {:>7.2} {:>13.2} {:>13}",
            n,
            ideal,
            tallest,
            bound,
            tallest as f64 / f64::from(ideal),
            (tallest as f64) < bound
        );
    }
    println!(
        "check by hand for n = {last_n}: N({tallest}) = {} <= n < N({}) = {}",
        minimal[tallest],
        tallest + 1,
        minimal[tallest + 1]
    );
}
```

**Rust output:**

```
== (a) N(h) from the recurrence against Fibonacci, h = 1..12 ==
  h   N(h)   F(h+2)-1  match   2^h - 1
  1      1          1   true         1
  2      2          2   true         3
  3      4          4   true         7
  4      7          7   true        15
  5     12         12   true        31
  6     20         20   true        63
  7     33         33   true       127
  8     54         54   true       255
  9     88         88   true       511
 10    143        143   true      1023
 11    232        232   true      2047
 12    376        376   true      4095

== (b) build the sparsest tree and run the checker, h = 1..16 ==
  h   nodes    N(h)  valid
  1       1       1   true
  2       2       2   true
  3       4       4   true
  4       7       7   true
  5      12      12   true
  6      20      20   true
  7      33      33   true
  8      54      54   true
  9      88      88   true
 10     143     143   true
 11     232     232   true
 12     376     376   true
 13     609     609   true
 14     986     986   true
 15    1596    1596   true
 16    2583    2583   true
control, the path 1-2-3 (not AVL): key 1 has factor -2

== (c) every AVL shape of height h, as a list of node counts (h <= 5) ==
  h    shapes   formula  match   min   N(h)   max  2^h - 1
  1         1         1   true     1      1     1        1
  2         3         3   true     2      2     3        3
  3        15        15   true     4      4     7        7
  4       315       315   true     7      7    15       15
  5    108675    108675   true    12     12    31       31

== (d) the tallest AVL tree for n nodes, against the continuous bound ==
bound = 1.4405 * lg(n+2) - 0.3277; tallest = the largest h with N(h) <= n
        n  ideal  tallest   bound tallest/ideal tallest<bound
       10      4        4    4.84          1.00          true
      100      7        9    9.28          1.29          true
     1000     10       14   14.03          1.40          true
    10000     14       18   18.81          1.29          true
  1000000     20       28   28.38          1.40          true
check by hand for n = 1000000: N(28) = 832039 <= n < N(29) = 1346268
```

> **Go vs Rust:** Identical to Go line for line (diffed). Where Go builds `fibTree(h-2)` with `h-2 = -1` for `h = 1` and treats `h <= 0` as nil, Rust's `usize` cannot go below 0, so the call is `h.saturating_sub(2)`. A `bool` pads like a string under `{:>6}`, which reproduces Go's `%6v` columns, and the shape lists are plain `Vec<u64>` of node counts (one entry per shape, 108,675 at h = 5), not trees.

**Complexity:** the fewest nodes in an AVL tree of height h is N(h) = F(h+2) − 1 (matched for h = 1..12, a tree built and checked at every h up to 16 where it has 2,583 nodes, and the minimum confirmed over all 108,675 shapes of height 5), so the tallest AVL tree is about 1.44·lg n high against the ideal lg n: 28 levels at n = 10⁶ against an ideal of 20 (1.40×, bound 28.38), and the bound is nearly tight, since the tallest tree at n = 1,000 has height 14 against a bound of 14.03.

---

## 5. Red-black: five rules, and a checker

`🟢 easy` · *Black height*

Red-black balance is an invariant, a handful of rules about colours, so the first thing to write is the checker: a Θ(n) walk that returns the black height or names the rule a tree breaks. With it in hand the rules stop being a list to memorise. Equal black height b forces at least 2^b − 1 nodes, and no two reds in a row keeps height within 2b, which together give height ≤ 2·lg(n+1), a looser bound than AVL's; and gluing each black node to its red children turns the tree into a 2-3-4 tree, the bridge to B-trees in examples 11–16.

**Steps:**

1. Hand-build a valid 10-node tree plus four copies that each break exactly one rule (root red, red under red, an extra black on one path, keys out of order) and let the checker name the rule.
2. Build the two extreme valid trees for black height b = 1 to 6, one with no reds and one with alternating black and red levels, check them, and derive the height bound for n = 10⁶.
3. Collapse every black node with its red children into one node of 1 to 3 keys, print the 2-3-4 trees, and read the leaf depths: equal black height is exactly "all leaves at the same depth".

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
	"strings"
)

// The five red-black rules are an INVARIANT, so write the checker first:
//  1. every node is red or black        (a bool: true by construction)
//  2. the root is black                 -> "root-red"
//  3. every nil leaf is black           (nil counts as black: true by construction)
//  4. a red node has only black children -> "red-red"
//  5. every path from a node down to nil crosses the same number of black
//     nodes, its black height           -> "black-height"
//
// plus plain BST order                  -> "order"
type node struct {
	key         int
	red         bool
	left, right *node
}

func n(key int, red bool, l, r *node) *node { return &node{key, red, l, r} }
func isRed(x *node) bool                    { return x != nil && x.red }

// walk returns the black height of x (black nodes on a path down, nil
// excluded) or the first rule that x's subtree breaks, as "rule at key K ...".
func walk(x *node, lo, hi int) (int, string) {
	if x == nil {
		return 0, ""
	}
	if x.key <= lo || x.key >= hi {
		return 0, fmt.Sprintf("order at key %d", x.key)
	}
	if x.red && (isRed(x.left) || isRed(x.right)) {
		return 0, fmt.Sprintf("red-red at key %d", x.key)
	}
	lb, v := walk(x.left, lo, x.key)
	if v != "" {
		return 0, v
	}
	rb, v := walk(x.right, x.key, hi)
	if v != "" {
		return 0, v
	}
	if lb != rb {
		return 0, fmt.Sprintf("black-height at key %d (left %d, right %d)", x.key, lb, rb)
	}
	if !x.red {
		lb++
	}
	return lb, ""
}

func check(root *node) (int, string) {
	if isRed(root) {
		return 0, fmt.Sprintf("root-red at key %d", root.key)
	}
	return walk(root, 0, math.MaxInt)
}

func find(x *node, key int) *node {
	for x.key != key {
		if key < x.key {
			x = x.left
		} else {
			x = x.right
		}
	}
	return x
}

// valid builds a 10-node tree by hand; black height 3.
func valid() *node {
	return n(4, false,
		n(2, false, n(1, false, nil, nil), n(3, false, nil, nil)),
		n(8, false,
			n(6, true, n(5, false, nil, nil), n(7, false, nil, nil)),
			n(9, false, nil, n(10, true, nil, nil))))
}

// perfect builds a perfect tree of the given height, keys 1, 2, 3... in order;
// red(depth) colours a whole level (the root has depth 1).
func perfect(height, depth int, next *int, red func(int) bool) *node {
	if depth > height {
		return nil
	}
	l := perfect(height, depth+1, next, red)
	*next++
	x := &node{key: *next, red: red(depth), left: l}
	x.right = perfect(height, depth+1, next, red)
	return x
}

func count(x *node) int {
	if x == nil {
		return 0
	}
	return 1 + count(x.left) + count(x.right)
}

// A red-black tree is a 2-3-4 tree in disguise: glue each black node to its
// red children and you get a node of 1, 2 or 3 keys.
type cnode struct {
	keys []int
	kids []*cnode
}

func collapse(b *node) *cnode {
	c := &cnode{}
	var slots []*node // the black subtrees (or nil) hanging below the glued group
	var gather func(x *node, top bool)
	gather = func(x *node, top bool) {
		if x == nil || (!x.red && !top) {
			slots = append(slots, x)
			return
		}
		gather(x.left, false)
		c.keys = append(c.keys, x.key)
		gather(x.right, false)
	}
	gather(b, true)
	for _, s := range slots {
		if s != nil {
			c.kids = append(c.kids, collapse(s))
		}
	}
	return c
}

func (c *cnode) String() string {
	s := fmt.Sprint(c.keys)
	if len(c.kids) == 0 {
		return s
	}
	var parts []string
	for _, k := range c.kids {
		parts = append(parts, k.String())
	}
	return s + "(" + strings.Join(parts, " ") + ")"
}

// levels walks the collapsed tree level by level and returns the node count
// per level, the depths at which leaves occur, the fewest and most keys in any
// node, and whether every internal node has keys+1 children.
func levels(c *cnode) (counts, leafDepths []int, minKeys, maxKeys int, fanOutOK bool) {
	fanOutOK, minKeys = true, len(c.keys)
	cur := []*cnode{c}
	for d := 0; len(cur) > 0; d++ {
		counts = append(counts, len(cur))
		var next []*cnode
		for _, x := range cur {
			minKeys, maxKeys = min(minKeys, len(x.keys)), max(maxKeys, len(x.keys))
			if len(x.kids) == 0 {
				if len(leafDepths) == 0 || leafDepths[len(leafDepths)-1] != d {
					leafDepths = append(leafDepths, d)
				}
			} else if len(x.kids) != len(x.keys)+1 {
				fanOutOK = false
			}
			next = append(next, x.kids...)
		}
		cur = next
	}
	return
}

func main() {
	fmt.Println("== (a) a valid tree, then four copies that each break exactly one rule ==")
	bh, v := check(valid())
	fmt.Printf("%-34s valid=%v, black height %d, %d nodes\n", "the 10-node tree:", v == "", bh, count(valid()))
	cases := []struct {
		what, rule string
		mutate     func(root *node)
	}{
		{"make the root red", "root-red", func(r *node) { r.red = true }},
		{"hang a red child under red 10", "red-red", func(r *node) { find(r, 10).right = n(11, true, nil, nil) }},
		{"colour red node 6 black (extra black)", "black-height", func(r *node) { find(r, 6).red = false }},
		{"swap the keys of leaves 1 and 3", "order", func(r *node) {
			a, b := find(r, 1), find(r, 3)
			a.key, b.key = b.key, a.key
		}},
	}
	for _, c := range cases {
		t := valid()
		c.mutate(t)
		_, v := check(t)
		fmt.Printf("%-38s -> %-42s names %-12s %v\n", c.what, v, c.rule, strings.HasPrefix(v, c.rule))
	}

	fmt.Println("\n== (b) two extreme valid trees for each black height b (built and checked here; CLRS ch. 13 proves they are the extremes) ==")
	fmt.Println("all-black = a perfect tree with no red nodes; alternating = a perfect tree whose levels alternate black, red, black, ...")
	fmt.Printf("%2s | %9s %8s %7s | %11s %8s %12s %7s\n", "b", "all-black", "height", "valid", "alternating", "height", "black height", "valid")
	allBlack := func(int) bool { return false }
	alternating := func(depth int) bool { return depth%2 == 0 }
	for b := 1; b <= 6; b++ {
		next := 0
		t1 := perfect(b, 1, &next, allBlack)
		bh1, v1 := check(t1)
		next = 0
		t2 := perfect(2*b, 1, &next, alternating)
		bh2, v2 := check(t2)
		fmt.Printf("%2d | %9d %8d %7v | %11d %8d %12d %7v\n",
			b, count(t1), b, v1 == "" && bh1 == b, count(t2), 2*b, bh2, v2 == "" && bh2 == b)
	}
	const big = 1000000
	maxB := bits.Len(uint(big+1)) - 1
	fmt.Printf("for n = %d: n >= 2^b - 1 gives b <= floor(lg(n+1)) = %d; no two reds in a row gives height <= 2b = %d\n", big, maxB, 2*maxB)
	fmt.Printf("  2*lg(n+1) = %.2f; the ideal height is %d\n", 2*math.Log2(big+1), bits.Len(uint(big)))

	fmt.Println("\n== (c) collapse red children into their black parent: a 2-3-4 tree ==")
	c := collapse(valid())
	counts, leaves, lo, hi, fan := levels(c)
	fmt.Printf("valid tree     : %v\n", c)
	fmt.Printf("                 nodes per level %v, keys per node %d..%d, leaves only at depth %v, internal nodes have keys+1 children: %v\n", counts, lo, hi, leaves, fan)
	for _, t := range []struct {
		name string
		red  func(int) bool
		h    int
	}{{"all-black b=3", allBlack, 3}, {"alternating b=3", alternating, 6}} {
		next := 0
		counts, leaves, lo, hi, fan := levels(collapse(perfect(t.h, 1, &next, t.red)))
		fmt.Printf("%-15s: nodes per level %v, keys per node %d..%d, leaves only at depth %v, internal nodes have keys+1 children: %v\n",
			t.name, counts, lo, hi, leaves, fan)
	}
}
```

**Output:**

```
== (a) a valid tree, then four copies that each break exactly one rule ==
the 10-node tree:                  valid=true, black height 3, 10 nodes
make the root red                      -> root-red at key 4                          names root-red     true
hang a red child under red 10          -> red-red at key 10                          names red-red      true
colour red node 6 black (extra black)  -> black-height at key 8 (left 2, right 1)    names black-height true
swap the keys of leaves 1 and 3        -> order at key 3                             names order        true

== (b) two extreme valid trees for each black height b (built and checked here; CLRS ch. 13 proves they are the extremes) ==
all-black = a perfect tree with no red nodes; alternating = a perfect tree whose levels alternate black, red, black, ...
 b | all-black   height   valid | alternating   height black height   valid
 1 |         1        1    true |           3        2            1    true
 2 |         3        2    true |          15        4            2    true
 3 |         7        3    true |          63        6            3    true
 4 |        15        4    true |         255        8            4    true
 5 |        31        5    true |        1023       10            5    true
 6 |        63        6    true |        4095       12            6    true
for n = 1000000: n >= 2^b - 1 gives b <= floor(lg(n+1)) = 19; no two reds in a row gives height <= 2b = 38
  2*lg(n+1) = 39.86; the ideal height is 20

== (c) collapse red children into their black parent: a 2-3-4 tree ==
valid tree     : [4]([2]([1] [3]) [6 8]([5] [7] [9 10]))
                 nodes per level [1 2 5], keys per node 1..2, leaves only at depth [2], internal nodes have keys+1 children: true
all-black b=3  : nodes per level [1 2 4], keys per node 1..1, leaves only at depth [2], internal nodes have keys+1 children: true
alternating b=3: nodes per level [1 4 16], keys per node 3..3, leaves only at depth [2], internal nodes have keys+1 children: true
```

**Rust:**

```rust
use std::fmt;

// The five red-black rules are an INVARIANT, so write the checker first:
//  1. every node is red or black        (an enum: true by construction)
//  2. the root is black                 -> "root-red"
//  3. every nil leaf is black           (the sentinel is black: true by construction)
//  4. a red node has only black children -> "red-red"
//  5. every path from a node down to nil crosses the same number of black
//     nodes, its black height           -> "black-height"
//
// plus plain BST order                  -> "order"
#[derive(Clone, Copy, PartialEq)]
enum Color {
    Red,
    Black,
}
use Color::{Black, Red};

#[derive(Clone)]
struct Node {
    key: usize,
    color: Color,
    left: u32,
    right: u32,
}

// The arena: nodes live in one Vec and refer to each other by u32 index. Index
// 0 is the sentinel, the single black nil leaf every missing child points at
// (CLRS does the same). Go's tree uses *node pointers and treats nil as
// black; a Box tree could not give a red-black fix-up its parent links, which
// is why the later red-black examples need this arena.
struct Tree {
    nodes: Vec<Node>,
    root: u32,
}

impl Tree {
    fn new() -> Self {
        let nil = Node {
            key: 0,
            color: Black,
            left: 0,
            right: 0,
        };
        Tree {
            nodes: vec![nil],
            root: 0,
        }
    }

    fn n(&mut self, key: usize, color: Color, left: u32, right: u32) -> u32 {
        self.nodes.push(Node {
            key,
            color,
            left,
            right,
        });
        (self.nodes.len() - 1) as u32
    }

    fn at(&self, i: u32) -> &Node {
        &self.nodes[i as usize]
    }

    fn is_red(&self, i: u32) -> bool {
        self.at(i).color == Red
    }

    fn find(&self, key: usize) -> u32 {
        let mut x = self.root;
        while self.at(x).key != key {
            x = if key < self.at(x).key {
                self.at(x).left
            } else {
                self.at(x).right
            };
        }
        x
    }

    fn count(&self, x: u32) -> usize {
        if x == 0 {
            return 0;
        }
        1 + self.count(self.at(x).left) + self.count(self.at(x).right)
    }
}

// The rule a tree breaks, printed by name.
enum Violation {
    RootRed(usize),
    RedRed(usize),
    Order(usize),
    BlackHeight {
        key: usize,
        left: usize,
        right: usize,
    },
}

impl Violation {
    fn rule(&self) -> &'static str {
        match self {
            Violation::RootRed(_) => "root-red",
            Violation::RedRed(_) => "red-red",
            Violation::Order(_) => "order",
            Violation::BlackHeight { .. } => "black-height",
        }
    }
}

impl fmt::Display for Violation {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Violation::RootRed(k) | Violation::RedRed(k) | Violation::Order(k) => {
                write!(f, "{} at key {k}", self.rule())
            }
            Violation::BlackHeight { key, left, right } => {
                write!(f, "black-height at key {key} (left {left}, right {right})")
            }
        }
    }
}

// walk returns the black height of x (black nodes on a path down, nil
// excluded) or the first rule that x's subtree breaks.
fn walk(t: &Tree, x: u32, lo: usize, hi: usize) -> Result<usize, Violation> {
    if x == 0 {
        return Ok(0);
    }
    let n = t.at(x);
    if n.key <= lo || n.key >= hi {
        return Err(Violation::Order(n.key));
    }
    if t.is_red(x) && (t.is_red(n.left) || t.is_red(n.right)) {
        return Err(Violation::RedRed(n.key));
    }
    let lb = walk(t, n.left, lo, n.key)?;
    let rb = walk(t, n.right, n.key, hi)?;
    if lb != rb {
        return Err(Violation::BlackHeight {
            key: n.key,
            left: lb,
            right: rb,
        });
    }
    Ok(lb + usize::from(n.color == Black))
}

fn check(t: &Tree) -> Result<usize, Violation> {
    if t.is_red(t.root) {
        return Err(Violation::RootRed(t.at(t.root).key));
    }
    walk(t, t.root, 0, usize::MAX)
}

// valid builds a 10-node tree by hand; black height 3.
fn valid() -> Tree {
    let mut t = Tree::new();
    let (n1, n3) = (t.n(1, Black, 0, 0), t.n(3, Black, 0, 0));
    let n2 = t.n(2, Black, n1, n3);
    let (n5, n7) = (t.n(5, Black, 0, 0), t.n(7, Black, 0, 0));
    let n6 = t.n(6, Red, n5, n7);
    let n10 = t.n(10, Red, 0, 0);
    let n9 = t.n(9, Black, 0, n10);
    let n8 = t.n(8, Black, n6, n9);
    t.root = t.n(4, Black, n2, n8);
    t
}

// a colouring decides, per depth, whether a whole level is red
type Colouring = fn(usize) -> bool;

// perfect builds a perfect tree of the given height, keys 1, 2, 3... in order;
// red(depth) colours a whole level (the root has depth 1).
fn perfect(t: &mut Tree, height: usize, depth: usize, next: &mut usize, red: Colouring) -> u32 {
    if depth > height {
        return 0;
    }
    let l = perfect(t, height, depth + 1, next, red);
    *next += 1;
    let key = *next;
    let r = perfect(t, height, depth + 1, next, red);
    t.n(key, if red(depth) { Red } else { Black }, l, r)
}

fn perfect_tree(height: usize, red: Colouring) -> Tree {
    let mut t = Tree::new();
    let mut next = 0;
    t.root = perfect(&mut t, height, 1, &mut next, red);
    t
}

// A red-black tree is a 2-3-4 tree in disguise: glue each black node to its
// red children and you get a node of 1, 2 or 3 keys.
struct CNode {
    keys: Vec<usize>,
    kids: Vec<CNode>,
}

fn collapse(t: &Tree, b: u32) -> CNode {
    // the black subtrees (or nil) hanging below the glued group
    fn gather(t: &Tree, x: u32, top: bool, keys: &mut Vec<usize>, slots: &mut Vec<u32>) {
        if x == 0 || (!t.is_red(x) && !top) {
            slots.push(x);
            return;
        }
        gather(t, t.at(x).left, false, keys, slots);
        keys.push(t.at(x).key);
        gather(t, t.at(x).right, false, keys, slots);
    }
    let (mut keys, mut slots) = (Vec::new(), Vec::new());
    gather(t, b, true, &mut keys, &mut slots);
    let kids = slots
        .into_iter()
        .filter(|&s| s != 0)
        .map(|s| collapse(t, s))
        .collect();
    CNode { keys, kids }
}

// Go prints a slice as [6 8]; Rust's {:?} prints [6, 8]. Format by hand.
fn go_slice(a: &[usize]) -> String {
    let parts: Vec<String> = a.iter().map(|k| k.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

impl fmt::Display for CNode {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", go_slice(&self.keys))?;
        if !self.kids.is_empty() {
            let parts: Vec<String> = self.kids.iter().map(|k| k.to_string()).collect();
            write!(f, "({})", parts.join(" "))?;
        }
        Ok(())
    }
}

struct Levels {
    counts: Vec<usize>,
    leaf_depths: Vec<usize>,
    min_keys: usize,
    max_keys: usize,
    fan_out_ok: bool,
}

// levels walks the collapsed tree level by level and returns the node count
// per level, the depths at which leaves occur, the fewest and most keys in any
// node, and whether every internal node has keys+1 children.
fn levels(c: &CNode) -> Levels {
    let mut r = Levels {
        counts: Vec::new(),
        leaf_depths: Vec::new(),
        min_keys: c.keys.len(),
        max_keys: 0,
        fan_out_ok: true,
    };
    let mut cur = vec![c];
    let mut d = 0;
    while !cur.is_empty() {
        r.counts.push(cur.len());
        let mut next = Vec::new();
        for x in cur {
            r.min_keys = r.min_keys.min(x.keys.len());
            r.max_keys = r.max_keys.max(x.keys.len());
            if x.kids.is_empty() {
                if r.leaf_depths.last() != Some(&d) {
                    r.leaf_depths.push(d);
                }
            } else if x.kids.len() != x.keys.len() + 1 {
                r.fan_out_ok = false;
            }
            next.extend(x.kids.iter());
        }
        cur = next;
        d += 1;
    }
    r
}

fn main() {
    println!("== (a) a valid tree, then four copies that each break exactly one rule ==");
    let t = valid();
    let (bh, v) = match check(&t) {
        Ok(bh) => (bh, true),
        Err(_) => (0, false),
    };
    println!(
        "{:<34} valid={}, black height {}, {} nodes",
        "the 10-node tree:",
        v,
        bh,
        t.count(t.root)
    );
    struct Case {
        what: &'static str,
        rule: &'static str,
        mutate: fn(&mut Tree),
    }
    let cases = [
        Case {
            what: "make the root red",
            rule: "root-red",
            mutate: |t| {
                let r = t.root as usize;
                t.nodes[r].color = Red;
            },
        },
        Case {
            what: "hang a red child under red 10",
            rule: "red-red",
            mutate: |t| {
                let i = t.find(10) as usize;
                let c = t.n(11, Red, 0, 0);
                t.nodes[i].right = c;
            },
        },
        Case {
            what: "colour red node 6 black (extra black)",
            rule: "black-height",
            mutate: |t| {
                let i = t.find(6) as usize;
                t.nodes[i].color = Black;
            },
        },
        Case {
            what: "swap the keys of leaves 1 and 3",
            rule: "order",
            mutate: |t| {
                let (a, b) = (t.find(1) as usize, t.find(3) as usize);
                let ka = t.nodes[a].key;
                t.nodes[a].key = t.nodes[b].key;
                t.nodes[b].key = ka;
            },
        },
    ];
    for c in &cases {
        let mut t = valid();
        (c.mutate)(&mut t);
        let v = check(&t).expect_err("each copy breaks one rule");
        println!(
            "{:<38} -> {:<42} names {:<12} {}",
            c.what,
            v.to_string(),
            c.rule,
            v.rule() == c.rule
        );
    }

    println!(
        "\n== (b) two extreme valid trees for each black height b (built and checked here; CLRS ch. 13 proves they are the extremes) =="
    );
    println!(
        "all-black = a perfect tree with no red nodes; alternating = a perfect tree whose levels alternate black, red, black, ..."
    );
    println!(
        "{:>2} | {:>9} {:>8} {:>7} | {:>11} {:>8} {:>12} {:>7}",
        "b", "all-black", "height", "valid", "alternating", "height", "black height", "valid"
    );
    let all_black: Colouring = |_| false;
    let alternating: Colouring = |depth| depth % 2 == 0;
    for b in 1..=6 {
        let t1 = perfect_tree(b, all_black);
        let r1 = check(&t1);
        let t2 = perfect_tree(2 * b, alternating);
        let r2 = check(&t2);
        println!(
            "{:>2} | {:>9} {:>8} {:>7} | {:>11} {:>8} {:>12} {:>7}",
            b,
            t1.count(t1.root),
            b,
            matches!(r1, Ok(bh) if bh == b),
            t2.count(t2.root),
            2 * b,
            r2.as_ref().map_or(0, |&bh| bh),
            matches!(r2, Ok(bh) if bh == b)
        );
    }
    const BIG: u64 = 1_000_000;
    let max_b = (u64::BITS - (BIG + 1).leading_zeros()) - 1;
    println!(
        "for n = {BIG}: n >= 2^b - 1 gives b <= floor(lg(n+1)) = {max_b}; no two reds in a row gives height <= 2b = {}",
        2 * max_b
    );
    println!(
        "  2*lg(n+1) = {:.2}; the ideal height is {}",
        2.0 * ((BIG + 1) as f64).log2(),
        u64::BITS - BIG.leading_zeros()
    );

    println!("\n== (c) collapse red children into their black parent: a 2-3-4 tree ==");
    let t = valid();
    let c = collapse(&t, t.root);
    let l = levels(&c);
    println!("valid tree     : {c}");
    println!(
        "                 nodes per level {}, keys per node {}..{}, leaves only at depth {}, internal nodes have keys+1 children: {}",
        go_slice(&l.counts),
        l.min_keys,
        l.max_keys,
        go_slice(&l.leaf_depths),
        l.fan_out_ok
    );
    let families: [(&str, Colouring, usize); 2] = [
        ("all-black b=3", all_black, 3),
        ("alternating b=3", alternating, 6),
    ];
    for (name, red, h) in families {
        let t = perfect_tree(h, red);
        let l = levels(&collapse(&t, t.root));
        println!(
            "{:<15}: nodes per level {}, keys per node {}..{}, leaves only at depth {}, internal nodes have keys+1 children: {}",
            name,
            go_slice(&l.counts),
            l.min_keys,
            l.max_keys,
            go_slice(&l.leaf_depths),
            l.fan_out_ok
        );
    }
}
```

**Rust output:**

```
== (a) a valid tree, then four copies that each break exactly one rule ==
the 10-node tree:                  valid=true, black height 3, 10 nodes
make the root red                      -> root-red at key 4                          names root-red     true
hang a red child under red 10          -> red-red at key 10                          names red-red      true
colour red node 6 black (extra black)  -> black-height at key 8 (left 2, right 1)    names black-height true
swap the keys of leaves 1 and 3        -> order at key 3                             names order        true

== (b) two extreme valid trees for each black height b (built and checked here; CLRS ch. 13 proves they are the extremes) ==
all-black = a perfect tree with no red nodes; alternating = a perfect tree whose levels alternate black, red, black, ...
 b | all-black   height   valid | alternating   height black height   valid
 1 |         1        1    true |           3        2            1    true
 2 |         3        2    true |          15        4            2    true
 3 |         7        3    true |          63        6            3    true
 4 |        15        4    true |         255        8            4    true
 5 |        31        5    true |        1023       10            5    true
 6 |        63        6    true |        4095       12            6    true
for n = 1000000: n >= 2^b - 1 gives b <= floor(lg(n+1)) = 19; no two reds in a row gives height <= 2b = 38
  2*lg(n+1) = 39.86; the ideal height is 20

== (c) collapse red children into their black parent: a 2-3-4 tree ==
valid tree     : [4]([2]([1] [3]) [6 8]([5] [7] [9 10]))
                 nodes per level [1 2 5], keys per node 1..2, leaves only at depth [2], internal nodes have keys+1 children: true
all-black b=3  : nodes per level [1 2 4], keys per node 1..1, leaves only at depth [2], internal nodes have keys+1 children: true
alternating b=3: nodes per level [1 4 16], keys per node 3..3, leaves only at depth [2], internal nodes have keys+1 children: true
```

> **Go vs Rust:** Identical to Go line for line (diffed). Go's tree is `*node` pointers with `nil` read as black; Rust's is an arena, a `Vec<Node>` addressed by `u32` with index 0 as the shared black sentinel (CLRS's `T.nil`), because the red-black fix-up of examples 8 and later needs parent links that a `Box` tree cannot express. Rust spells the checker's answer as `Result<usize, Violation>` with an enum whose `rule()` is the name that gets printed, instead of Go's `(int, string)` with an empty string meaning valid, and `go_slice` formats `[6 8]` by hand because `{:?}` would print commas.

**Complexity:** the checker is Θ(n) and named the broken rule for each of the four mutants (`root-red`, `red-red`, `black-height`, `order`). Black height b allows from 2^b − 1 to 4^b − 1 nodes (b = 6: 63 to 4,095, both trees valid), so b ≤ lg(n+1) and height ≤ 2b ≤ 2·lg(n+1): at n = 10⁶ that allows 38 levels (the bound is 39.86) against an ideal of 20. Every valid tree collapsed to a 2-3-4 tree with all leaves at one depth: the 10-node tree to levels of 1, 2 and 5 nodes, the b = 3 alternating tree to a complete 4-ary tree of 1, 4 and 16 nodes, and the all-black tree to 2-nodes only.

---

## 6. A coin flip per node: skip-list levels

`🟢 easy` · *Geometric levels*

A skip list needs no rebalancing because each node's height is decided by coin flips: a node reaches level k with probability p^(k−1), so its expected height is 1/(1−p) pointers and the tallest of n nodes is about log_{1/p} n. This example looks at that distribution alone, before any list exists (example 10 builds it), and compares it with the deterministic *perfect* skip list whose level is 1 plus the trailing zeros of the position, which is what a balanced structure would have to maintain at great cost. Drawing a level takes 1/(1−p) flips on average, and the whole claim is that a random histogram lands close to the perfect one for free.

**Steps:**

1. Draw 100,000 levels for p = 1/2 and p = 1/4 and print each histogram next to the expected count `n·p^(k−1)(1−p)`, with the mean level and the tallest node against `log_{1/p} n`.
2. Repeat "the tallest node among 1,000" 10,000 times to see how much the maximum itself varies, and how often it exceeds twice `log_{1/p} n`.
3. Count the perfect list's levels (`1 + trailing zeros of i`) for the same n and set the random histogram beside it, level by level.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// A skip list needs no rebalancing because each node's height is a run of coin
// flips: level k with probability p^(k-1). This example looks at that
// distribution alone -- no list yet (example 10 builds it) -- and compares it
// with the DETERMINISTIC perfect skip list, level(i) = 1 + trailing zeros of i,
// which is what a balanced structure would have to maintain at great cost.

// level keeps promoting a node while a 1-in-2^pBits event happens (pBits = 1
// is p = 1/2, pBits = 2 is p = 1/4); at most 32 levels.
func level(rng *rand.Rand, pBits int) int {
	lvl := 1
	for lvl < 32 && rng.IntN(1<<pBits) == 0 {
		lvl++
	}
	return lvl
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	const n = 100000
	var half []int // the p = 1/2 histogram, kept for part (c)
	var halfSum int

	fmt.Printf("== (a) levels of %d random nodes ==\n", n)
	for _, pBits := range []int{1, 2} {
		p := math.Pow(2, -float64(pBits))
		hist := make([]int, 33)
		sum, top := 0, 0
		for range n {
			l := level(rng, pBits)
			hist[l]++
			sum += l
			top = max(top, l)
		}
		fmt.Printf("p = 1/%d\n", 1<<pBits)
		fmt.Printf("%6s %8s %10s %7s\n", "level", "count", "expected", "ratio")
		for k := 1; k <= top; k++ {
			want := n * math.Pow(p, float64(k-1)) * (1 - p)
			fmt.Printf("%6d %8d %10.1f %7.3f\n", k, hist[k], want, float64(hist[k])/want)
		}
		fmt.Printf("mean level %.3f vs 1/(1-p) = %.3f pointers per node; max level %d vs log_{1/p} n = %.2f\n",
			float64(sum)/n, 1/(1-p), top, math.Log2(n)/float64(pBits))
		if pBits == 1 {
			half, halfSum = hist, sum
		}
	}

	const trials, size = 10000, 1000
	fmt.Printf("\n== (b) the tallest node among %d nodes, over %d trials ==\n", size, trials)
	fmt.Printf("%-6s %5s %7s %5s %14s %24s\n", "p", "min", "median", "max", "log_{1/p}(n)", "trials above twice that")
	for _, pBits := range []int{1, 2} {
		var tops []int
		for range trials {
			top := 0
			for range size {
				top = max(top, level(rng, pBits))
			}
			tops = append(tops, top)
		}
		slices.Sort(tops)
		lg := math.Log2(size) / float64(pBits)
		over := 0
		for _, t := range tops {
			if float64(t) > 2*lg {
				over++
			}
		}
		fmt.Printf("1/%-4d %5d %7d %5d %14.2f %17d = %.2f%%\n", 1<<pBits, tops[0], tops[trials/2], tops[trials-1], lg, over, 100*float64(over)/trials)
	}

	fmt.Printf("\n== (c) the perfect list, level(i) = 1 + trailing zeros of i, i = 1..%d ==\n", n)
	perfect := make([]int, 33)
	for i := 1; i <= n; i++ {
		perfect[1+bits.TrailingZeros(uint(i))]++
	}
	fmt.Println("formula = floor(n / 2^(k-1)) - floor(n / 2^k); random = the p = 1/2 histogram of part (a)")
	fmt.Printf("%6s %8s %8s %6s %8s %10s\n", "level", "perfect", "formula", "match", "random", "deviation")
	worst, worstK := 0.0, 0
	for k := 1; k < len(perfect) && (perfect[k] > 0 || half[k] > 0); k++ {
		formula := n>>(k-1) - n>>k
		devText, dev := "n/a", 0.0 // no perfect node at this level to compare with
		if perfect[k] > 0 {
			dev = float64(half[k]-perfect[k]) / float64(perfect[k])
			devText = fmt.Sprintf("%+.1f%%", 100*dev)
		}
		fmt.Printf("%6d %8d %8d %6v %8d %10s\n", k, perfect[k], formula, formula == perfect[k], half[k], devText)
		if perfect[k] >= 1000 && math.Abs(dev) > worst {
			worst, worstK = math.Abs(dev), k
		}
	}
	fmt.Printf("largest deviation at a level holding at least 1,000 perfect nodes: %.1f%% (level %d)\n", 100*worst, worstK)
	perfectSum := 0
	for k, c := range perfect {
		perfectSum += k * c
	}
	fmt.Printf("pointers per node: perfect list %.3f, random list %.3f\n", float64(perfectSum)/n, float64(halfSum)/n)
}
```

**Output:**

```
== (a) levels of 100000 random nodes ==
p = 1/2
 level    count   expected   ratio
     1    50238    50000.0   1.005
     2    24975    25000.0   0.999
     3    12376    12500.0   0.990
     4     6250     6250.0   1.000
     5     3070     3125.0   0.982
     6     1554     1562.5   0.995
     7      697      781.2   0.892
     8      452      390.6   1.157
     9      187      195.3   0.957
    10      103       97.7   1.055
    11       49       48.8   1.004
    12       28       24.4   1.147
    13       11       12.2   0.901
    14        4        6.1   0.655
    15        4        3.1   1.311
    16        1        1.5   0.655
    17        1        0.8   1.311
mean level 1.994 vs 1/(1-p) = 2.000 pointers per node; max level 17 vs log_{1/p} n = 16.61
p = 1/4
 level    count   expected   ratio
     1    75035    75000.0   1.000
     2    18681    18750.0   0.996
     3     4684     4687.5   0.999
     4     1216     1171.9   1.038
     5      296      293.0   1.010
     6       64       73.2   0.874
     7       21       18.3   1.147
     8        1        4.6   0.218
     9        1        1.1   0.874
    10        0        0.3   0.000
    11        1        0.1  13.981
mean level 1.334 vs 1/(1-p) = 1.333 pointers per node; max level 11 vs log_{1/p} n = 8.30

== (b) the tallest node among 1000 nodes, over 10000 trials ==
p        min  median   max   log_{1/p}(n)  trials above twice that
1/2        7      11    24           9.97                26 = 0.26%
1/4        4       6    13           4.98                37 = 0.37%

== (c) the perfect list, level(i) = 1 + trailing zeros of i, i = 1..100000 ==
formula = floor(n / 2^(k-1)) - floor(n / 2^k); random = the p = 1/2 histogram of part (a)
 level  perfect  formula  match   random  deviation
     1    50000    50000   true    50238      +0.5%
     2    25000    25000   true    24975      -0.1%
     3    12500    12500   true    12376      -1.0%
     4     6250     6250   true     6250      +0.0%
     5     3125     3125   true     3070      -1.8%
     6     1563     1563   true     1554      -0.6%
     7      781      781   true      697     -10.8%
     8      391      391   true      452     +15.6%
     9      195      195   true      187      -4.1%
    10       98       98   true      103      +5.1%
    11       49       49   true       49      +0.0%
    12       24       24   true       28     +16.7%
    13       12       12   true       11      -8.3%
    14        6        6   true        4     -33.3%
    15        3        3   true        4     +33.3%
    16        2        2   true        1     -50.0%
    17        1        1   true        1      +0.0%
largest deviation at a level holding at least 1,000 perfect nodes: 1.8% (level 5)
pointers per node: perfect list 2.000, random list 1.994
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and of its IntN, so this
// program draws exactly the numbers the Go program draws (seed 1, 2).
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

// A skip list needs no rebalancing because each node's height is a run of coin
// flips: level k with probability p^(k-1). This example looks at that
// distribution alone -- no list yet (example 10 builds it) -- and compares it
// with the DETERMINISTIC perfect skip list, level(i) = 1 + trailing zeros of i,
// which is what a balanced structure would have to maintain at great cost.

// level keeps promoting a node while a 1-in-2^p_bits event happens (p_bits = 1
// is p = 1/2, p_bits = 2 is p = 1/4); at most 32 levels.
fn level(rng: &mut Pcg, p_bits: u32) -> usize {
    let mut lvl = 1;
    while lvl < 32 && rng.int_n(1 << p_bits) == 0 {
        lvl += 1;
    }
    lvl
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    const N: usize = 100000;
    let mut half: Vec<usize> = Vec::new(); // the p = 1/2 histogram, kept for part (c)
    let mut half_sum = 0;

    println!("== (a) levels of {N} random nodes ==");
    for p_bits in [1u32, 2] {
        let p = 0.5f64.powi(p_bits as i32);
        let mut hist = vec![0usize; 33];
        let (mut sum, mut top) = (0, 0);
        for _ in 0..N {
            let l = level(&mut rng, p_bits);
            hist[l] += 1;
            sum += l;
            top = top.max(l);
        }
        println!("p = 1/{}", 1 << p_bits);
        println!(
            "{:>6} {:>8} {:>10} {:>7}",
            "level", "count", "expected", "ratio"
        );
        for (k, &count) in hist.iter().enumerate().take(top + 1).skip(1) {
            let want = N as f64 * p.powi(k as i32 - 1) * (1.0 - p);
            println!(
                "{:>6} {:>8} {:>10.1} {:>7.3}",
                k,
                count,
                want,
                count as f64 / want
            );
        }
        println!(
            "mean level {:.3} vs 1/(1-p) = {:.3} pointers per node; max level {} vs log_{{1/p}} n = {:.2}",
            sum as f64 / N as f64,
            1.0 / (1.0 - p),
            top,
            (N as f64).log2() / f64::from(p_bits)
        );
        if p_bits == 1 {
            half = hist;
            half_sum = sum;
        }
    }

    const TRIALS: usize = 10000;
    const SIZE: usize = 1000;
    println!("\n== (b) the tallest node among {SIZE} nodes, over {TRIALS} trials ==");
    println!(
        "{:<6} {:>5} {:>7} {:>5} {:>14} {:>24}",
        "p", "min", "median", "max", "log_{1/p}(n)", "trials above twice that"
    );
    for p_bits in [1u32, 2] {
        let mut tops: Vec<usize> = Vec::new();
        for _ in 0..TRIALS {
            let mut top = 0;
            for _ in 0..SIZE {
                top = top.max(level(&mut rng, p_bits));
            }
            tops.push(top);
        }
        tops.sort_unstable();
        let lg = (SIZE as f64).log2() / f64::from(p_bits);
        let over = tops.iter().filter(|&&t| t as f64 > 2.0 * lg).count();
        println!(
            "{:<6} {:>5} {:>7} {:>5} {:>14.2} {:>17} = {:.2}%",
            format!("1/{}", 1 << p_bits),
            tops[0],
            tops[TRIALS / 2],
            tops[TRIALS - 1],
            lg,
            over,
            100.0 * over as f64 / TRIALS as f64
        );
    }

    println!("\n== (c) the perfect list, level(i) = 1 + trailing zeros of i, i = 1..{N} ==");
    let mut perfect = vec![0usize; 33];
    for i in 1..=N {
        perfect[1 + i.trailing_zeros() as usize] += 1;
    }
    println!(
        "formula = floor(n / 2^(k-1)) - floor(n / 2^k); random = the p = 1/2 histogram of part (a)"
    );
    println!(
        "{:>6} {:>8} {:>8} {:>6} {:>8} {:>10}",
        "level", "perfect", "formula", "match", "random", "deviation"
    );
    let (mut worst, mut worst_k) = (0.0f64, 0);
    for k in 1..perfect.len() {
        if perfect[k] == 0 && half[k] == 0 {
            break;
        }
        let formula = (N >> (k - 1)) - (N >> k);
        // no perfect node at this level to compare with -> n/a
        let (mut dev_text, mut dev) = ("n/a".to_string(), 0.0);
        if perfect[k] > 0 {
            dev = (half[k] as f64 - perfect[k] as f64) / perfect[k] as f64;
            dev_text = format!("{:+.1}%", 100.0 * dev);
        }
        println!(
            "{:>6} {:>8} {:>8} {:>6} {:>8} {:>10}",
            k,
            perfect[k],
            formula,
            formula == perfect[k],
            half[k],
            dev_text
        );
        if perfect[k] >= 1000 && dev.abs() > worst {
            (worst, worst_k) = (dev.abs(), k);
        }
    }
    println!(
        "largest deviation at a level holding at least 1,000 perfect nodes: {:.1}% (level {worst_k})",
        100.0 * worst
    );
    let perfect_sum: usize = perfect.iter().enumerate().map(|(k, c)| k * c).sum();
    println!(
        "pointers per node: perfect list {:.3}, random list {:.3}",
        perfect_sum as f64 / N as f64,
        half_sum as f64 / N as f64
    );
}
```

**Rust output:**

```
== (a) levels of 100000 random nodes ==
p = 1/2
 level    count   expected   ratio
     1    50238    50000.0   1.005
     2    24975    25000.0   0.999
     3    12376    12500.0   0.990
     4     6250     6250.0   1.000
     5     3070     3125.0   0.982
     6     1554     1562.5   0.995
     7      697      781.2   0.892
     8      452      390.6   1.157
     9      187      195.3   0.957
    10      103       97.7   1.055
    11       49       48.8   1.004
    12       28       24.4   1.147
    13       11       12.2   0.901
    14        4        6.1   0.655
    15        4        3.1   1.311
    16        1        1.5   0.655
    17        1        0.8   1.311
mean level 1.994 vs 1/(1-p) = 2.000 pointers per node; max level 17 vs log_{1/p} n = 16.61
p = 1/4
 level    count   expected   ratio
     1    75035    75000.0   1.000
     2    18681    18750.0   0.996
     3     4684     4687.5   0.999
     4     1216     1171.9   1.038
     5      296      293.0   1.010
     6       64       73.2   0.874
     7       21       18.3   1.147
     8        1        4.6   0.218
     9        1        1.1   0.874
    10        0        0.3   0.000
    11        1        0.1  13.981
mean level 1.334 vs 1/(1-p) = 1.333 pointers per node; max level 11 vs log_{1/p} n = 8.30

== (b) the tallest node among 1000 nodes, over 10000 trials ==
p        min  median   max   log_{1/p}(n)  trials above twice that
1/2        7      11    24           9.97                26 = 0.26%
1/4        4       6    13           4.98                37 = 0.37%

== (c) the perfect list, level(i) = 1 + trailing zeros of i, i = 1..100000 ==
formula = floor(n / 2^(k-1)) - floor(n / 2^k); random = the p = 1/2 histogram of part (a)
 level  perfect  formula  match   random  deviation
     1    50000    50000   true    50238      +0.5%
     2    25000    25000   true    24975      -0.1%
     3    12500    12500   true    12376      -1.0%
     4     6250     6250   true     6250      +0.0%
     5     3125     3125   true     3070      -1.8%
     6     1563     1563   true     1554      -0.6%
     7      781      781   true      697     -10.8%
     8      391      391   true      452     +15.6%
     9      195      195   true      187      -4.1%
    10       98       98   true      103      +5.1%
    11       49       49   true       49      +0.0%
    12       24       24   true       28     +16.7%
    13       12       12   true       11      -8.3%
    14        6        6   true        4     -33.3%
    15        3        3   true        4     +33.3%
    16        2        2   true        1     -50.0%
    17        1        1   true        1      +0.0%
largest deviation at a level holding at least 1,000 perfect nodes: 1.8% (level 5)
pointers per node: perfect list 2.000, random list 1.994
```

> **Go vs Rust:** Identical to Go line for line (diffed): `Pcg::int_n(1 << p_bits)` is a power of two, so it is a mask of one 64-bit draw exactly like Go's `IntN`, and `u64::trailing_zeros` is `bits.TrailingZeros`. The expected counts use `powi` where Go uses `math.Pow`; for p = 1/2 and 1/4 both are exact, so the printed decimals agree. Rust's `{:+.1}` and `{:>10}` reproduce Go's `%+.1f` and `%10s` columns.

**Complexity:** P(level ≥ k) = p^(k−1), so the mean level is 1/(1−p) pointers per node (measured 1.994 for p = 1/2 and 1.334 for p = 1/4 over 100,000 nodes) and the tallest node is about log_{1/p} n, but the tail is real: the tallest of 100,000 nodes was 17 against 16.61 for p = 1/2 and 11 against 8.30 for p = 1/4, and among 1,000 nodes it ranged from 7 to 24 for p = 1/2 against log₂ 1000 = 9.97, exceeding twice that in 0.26% of 10,000 trials. The perfect list has exactly ⌊n/2^(k−1)⌋ − ⌊n/2^k⌋ nodes at level k; the random list matched it within 1.8% at every level holding at least 1,000 nodes and used 1.994 pointers per node against the perfect list's 2.000, with no maintenance at all.

---

> Next tier: [🟡 medium](2-medium.md) — AVL, red-black, a treap, a skip list and a B-tree, built and proved.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
