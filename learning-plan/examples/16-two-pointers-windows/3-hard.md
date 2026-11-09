# Step 16 — Two Pointers, Sliding Window & Prefix Sums · 🔴 Hard

Examples **12–16**: counting windows as a difference of two passes, 3Sum, a brute-force audit of when a
sliding window is legal, the operations a prefix array cannot answer, and the capstone toolkit.

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

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Counting subarrays: exactly k as a difference

`🔴 hard` · *Variable window + counting*

"Exactly k" is not monotone: a window can stop being valid by growing *or* by shrinking, so the longest/shortest templates of examples 9 and 10 do not apply. "At most k" is monotone, though: for a fixed right edge every start in `[left, right]` is valid, so one pass adds `right - left + 1` per step, and a window with exactly k is a window with at most k that is not a window with at most k−1, so `exactly(k) = atMost(k) − atMost(k−1)`. This example counts two kinds of subarray that way, exactly k distinct values and sum exactly k over non-negative values (zeros allowed), checks both against brute force on 20,000 PCG-seeded cases, shows the tempting single-pass "exact" window undercounting, and puts the window beside lesson 10's prefix-sum map, which tolerates negatives. Each at-most pass is Θ(n) with at most 2n pointer moves and Θ(1) extra space for a fixed alphabet.

**Steps:**

1. Write `atMostDistinct` (a count per value) and `atMostSum` (a running sum); both add `right - left + 1` per right edge, and subtract two passes to get "exactly".
2. Write the oracles (try every start and end), the naive one-pass exact window and the prefix-sum map, then run all of them on the same random cases, first on non-negative values, then with negatives.
3. Read the table of example (a), then the mismatch counts and the first counterexamples, then the pointer-move and hash-operation counts at n = 100,000.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"strings"
)

// "At most k" is monotone: shrink a valid window and it stays valid. So for a
// fixed right edge, EVERY start in [left, right] is valid, and one pass can add
// right-left+1 per step. "Exactly k" has no such interval, but
//
//	exactly(k) = atMost(k) - atMost(k-1)
//
// because a window with exactly k is a window with at most k that is not a window
// with at most k-1. Two problems below: exactly k DISTINCT values, and sum
// exactly k over NON-NEGATIVE values (zeros allowed).

// cost counts the work: pointer moves for the windows, hash operations for the map.
type cost struct{ moves, lookups, inserts int }

type step struct{ left, add int }

// atMostDistinct counts subarrays with at most k distinct values (values 0..7).
// The window keeps a count per value: Theta(1) space for a fixed alphabet.
func atMostDistinct(a []int, k int, c *cost, trace *[]step) int {
	var cnt [8]int
	distinct, left, total := 0, 0, 0
	for right, v := range a {
		if cnt[v] == 0 {
			distinct++
		}
		cnt[v]++
		c.moves++ // right moved
		for distinct > k {
			u := a[left]
			cnt[u]--
			if cnt[u] == 0 {
				distinct--
			}
			left++
			c.moves++ // left moved
		}
		add := right - left + 1 // every start in [left, right] is valid
		total += add
		if trace != nil {
			*trace = append(*trace, step{left, add})
		}
	}
	return total
}

// atMostSum counts subarrays with sum <= k. Correct ONLY when no value is negative:
// then growing never lowers the sum and shrinking never raises it.
func atMostSum(a []int, k int, c *cost) int {
	if k < 0 { // non-negative values: no window has a negative sum
		return 0
	}
	sum, left, total := 0, 0, 0
	for right, v := range a {
		sum += v
		c.moves++
		for sum > k && left <= right {
			sum -= a[left]
			left++
			c.moves++
		}
		total += right - left + 1
	}
	return total
}

// naiveExactSum is the tempting one-pass version: shrink while sum > k, then
// count ONE window per right edge if the sum is exactly k.
func naiveExactSum(a []int, k int) int {
	sum, left, count := 0, 0, 0
	for right, v := range a {
		sum += v
		for sum > k && left <= right {
			sum -= a[left]
			left++
		}
		if left <= right && sum == k {
			count++
		}
	}
	return count
}

// prefixMap is lesson 10 example 13: count earlier prefixes equal to sum-k.
// Valid for any integers; costs a map.
func prefixMap(a []int, k int, c *cost) (count, entries int) {
	seen := map[int]int{0: 1}
	c.inserts++ // the seed: the empty prefix
	sum := 0
	for _, v := range a {
		sum += v
		c.lookups++
		count += seen[sum-k]
		seen[sum]++
		c.inserts++
	}
	return count, len(seen)
}

// The oracles: try every start and every end.
func bruteSum(a []int, k int) int {
	count := 0
	for i := range a {
		sum := 0
		for j := i; j < len(a); j++ {
			sum += a[j]
			if sum == k {
				count++
			}
		}
	}
	return count
}

func bruteDistinct(a []int, k int) int {
	count := 0
	for i := range a {
		var seen [8]bool
		d := 0
		for j := i; j < len(a); j++ {
			if !seen[a[j]] {
				seen[a[j]] = true
				d++
			}
			if d == k {
				count++
			}
		}
	}
	return count
}

// startsWith lists the starts s <= right whose window a[s..right] has exactly k distinct values.
func startsWith(a []int, right, k int) []int {
	var out []int
	for s := 0; s <= right; s++ {
		seen := map[int]bool{}
		for _, v := range a[s : right+1] {
			seen[v] = true
		}
		if len(seen) == k {
			out = append(out, s)
		}
	}
	return out
}

// tally records disagreements with the brute force: how many, the first, and the shortest.
type tally struct {
	n, over      int // mismatches, and how many of them overcounted
	first, short *failure
}

type failure struct {
	a         []int
	k         int
	want, got int
}

func (t *tally) note(a []int, k, want, got int) {
	if want == got {
		return
	}
	t.n++
	if got > want {
		t.over++
	}
	f := &failure{append([]int(nil), a...), k, want, got}
	if t.first == nil {
		t.first = f
	}
	if t.short == nil || len(a) < len(t.short.a) {
		t.short = f
	}
}

func (f *failure) String(who string) string {
	return fmt.Sprintf("%v k=%d: brute force %d, %s %d", f.a, f.k, f.want, who, f.got)
}

func gen(rng *rand.Rand, maxLen, alphabet, offset int) []int {
	a := make([]int, rng.IntN(maxLen+1))
	for i := range a {
		a[i] = rng.IntN(alphabet) + offset
	}
	return a
}

func letters(a []int) string {
	var sb strings.Builder
	for _, v := range a {
		sb.WriteByte(byte('a' + v))
	}
	return sb.String()
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	var scratch cost

	// (a) the table: [1 2 1 2 3], exactly 2 distinct.
	a := []int{1, 2, 1, 2, 3}
	const k = 2
	var t2, t1 []step
	var c2, c1 cost
	at2 := atMostDistinct(a, k, &c2, &t2)
	at1 := atMostDistinct(a, k-1, &c1, &t1)
	fmt.Printf("(a) %v: subarrays with exactly %d distinct values\n", a, k)
	fmt.Println("right a[r] | atMost(2): left add total | atMost(1): left add total | diff | starts with exactly 2 (brute force)")
	tot2, tot1 := 0, 0
	for r := range a {
		tot2 += t2[r].add
		tot1 += t1[r].add
		fmt.Printf("%5d %4d | %16d %3d %5d | %16d %3d %5d | %4d | %v\n",
			r, a[r], t2[r].left, t2[r].add, tot2, t1[r].left, t1[r].add, tot1,
			t2[r].add-t1[r].add, startsWith(a, r, k))
	}
	fmt.Printf("atMost(2) = %d, atMost(1) = %d, difference = %d, brute force = %d\n",
		at2, at1, at2-at1, bruteDistinct(a, k))
	fmt.Println()

	// (b) sums over NON-NEGATIVE arrays (zeros occur): window difference vs map vs brute force.
	const cases = 20000
	var winB, mapB, naiveB tally
	var total int
	for range cases {
		x := gen(rng, 15, 4, 0)
		kk := rng.IntN(9)
		want := bruteSum(x, kk)
		total += want
		winB.note(x, kk, want, atMostSum(x, kk, &scratch)-atMostSum(x, kk-1, &scratch))
		got, _ := prefixMap(x, kk, &scratch)
		mapB.note(x, kk, want, got)
		naiveB.note(x, kk, want, naiveExactSum(x, kk))
	}
	fmt.Printf("(b) sum exactly k, values 0..3, %d random cases (length 0-15, k 0-8), %d subarrays in total\n", cases, total)
	fmt.Printf("    atMost(k) - atMost(k-1) window : %d mismatches with brute force\n", winB.n)
	fmt.Printf("    prefix-sum map                 : %d mismatches\n", mapB.n)
	fmt.Printf("    naive one-pass 'exact' window  : %d mismatches (%.1f%% of the cases), %d of them overcounts\n", naiveB.n, 100*float64(naiveB.n)/cases, naiveB.over)
	fmt.Println("    first naive failure  :", naiveB.first.String("naive window"))
	fmt.Println("    shortest naive failure:", naiveB.short.String("naive window"))
	fmt.Println()

	// (c) the same with negative values allowed (-2..3): the window argument is void.
	var winC, mapC tally
	for range cases {
		x := gen(rng, 15, 6, -2)
		kk := rng.IntN(8) + 1
		want := bruteSum(x, kk)
		winC.note(x, kk, want, atMostSum(x, kk, &scratch)-atMostSum(x, kk-1, &scratch))
		got, _ := prefixMap(x, kk, &scratch)
		mapC.note(x, kk, want, got)
	}
	fmt.Printf("(c) sum exactly k, values -2..3, %d random cases (k 1-8)\n", cases)
	fmt.Printf("    atMost(k) - atMost(k-1) window : %d mismatches (%.1f%% of the cases)\n", winC.n, 100*float64(winC.n)/cases)
	fmt.Printf("    prefix-sum map                 : %d mismatches\n", mapC.n)
	fmt.Println("    shortest window failure:", winC.short.String("window"))
	fmt.Println()

	// (d) exactly k distinct letters over "abcd".
	var winD tally
	var totalD, nonEmpty int
	var firstD string
	for range cases {
		x := gen(rng, 15, 4, 0)
		kk := rng.IntN(4) + 1
		want := bruteDistinct(x, kk)
		totalD += want
		got := atMostDistinct(x, kk, &scratch, nil) - atMostDistinct(x, kk-1, &scratch, nil)
		if len(x) > 0 {
			nonEmpty++
			if firstD == "" {
				firstD = fmt.Sprintf("%q k=%d: window %d, brute force %d", letters(x), kk, got, want)
			}
		}
		winD.note(x, kk, want, got)
	}
	fmt.Printf("(d) exactly k distinct letters over \"abcd\", %d random strings (length 0-15, k 1-4), %d of them non-empty\n", cases, nonEmpty)
	fmt.Printf("    atMost(k) - atMost(k-1) window : %d mismatches with brute force over %d subarrays\n", winD.n, totalD)
	fmt.Println("    first non-empty case:", firstD)
	fmt.Println()

	// (e) cost at n = 100,000.
	const n = 100000
	big := make([]int, n)
	for i := range big {
		big[i] = rng.IntN(4)
	}
	const kBig = 100
	var cw1, cw2, cm cost
	wcount := atMostSum(big, kBig, &cw1) - atMostSum(big, kBig-1, &cw2)
	mcount, entries := prefixMap(big, kBig, &cm)
	fmt.Printf("(e) n = %d, values 0..3, sum exactly %d: %d subarrays\n", n, kBig, wcount)
	fmt.Printf("    window  : pointer moves %d + %d = %d (bound 4n = %d), extra space: 2 integers (sum, left)\n",
		cw1.moves, cw2.moves, cw1.moves+cw2.moves, 4*n)
	fmt.Printf("    map     : %d lookups + %d inserts = %d hash operations, extra space: %d entries; same count: %v\n",
		cm.lookups, cm.inserts, cm.lookups+cm.inserts, entries, mcount == wcount)
	dist := make([]int, n)
	for i := range dist {
		dist[i] = rng.IntN(8)
	}
	var cd1, cd2 cost
	dcount := atMostDistinct(dist, 3, &cd1, nil) - atMostDistinct(dist, 2, &cd2, nil)
	fmt.Printf("    exactly 3 distinct of 8 values: %d subarrays, pointer moves %d + %d, extra space: 8 counters (plus left)\n",
		dcount, cd1.moves, cd2.moves)
	const maxSub = n * (n + 1) / 2
	cmp := "fits in"
	if maxSub > 1<<32-1 {
		cmp = "does not fit in"
	}
	fmt.Printf("    n(n+1)/2 = %d possible subarrays: that %s a 32-bit unsigned counter (max %d)\n", maxSub, cmp, uint32(1<<32-1))
	fmt.Printf("    window: no table at all, but wrong in %d of %d cases with negatives; map: %d entries, wrong in %d\n",
		winC.n, cases, entries, mapC.n)
}
```

**Output:**

```
(a) [1 2 1 2 3]: subarrays with exactly 2 distinct values
right a[r] | atMost(2): left add total | atMost(1): left add total | diff | starts with exactly 2 (brute force)
    0    1 |                0   1     1 |                0   1     1 |    0 | []
    1    2 |                0   2     3 |                1   1     2 |    1 | [0]
    2    1 |                0   3     6 |                2   1     3 |    2 | [0 1]
    3    2 |                0   4    10 |                3   1     4 |    3 | [0 1 2]
    4    3 |                3   2    12 |                4   1     5 |    1 | [3]
atMost(2) = 12, atMost(1) = 5, difference = 7, brute force = 7

(b) sum exactly k, values 0..3, 20000 random cases (length 0-15, k 0-8), 64761 subarrays in total
    atMost(k) - atMost(k-1) window : 0 mismatches with brute force
    prefix-sum map                 : 0 mismatches
    naive one-pass 'exact' window  : 6740 mismatches (33.7% of the cases), 0 of them overcounts
    first naive failure  : [2 0 0 1 2 1 0 3] k=4: brute force 7, naive window 3
    shortest naive failure: [0 1] k=1: brute force 2, naive window 1

(c) sum exactly k, values -2..3, 20000 random cases (k 1-8)
    atMost(k) - atMost(k-1) window : 9358 mismatches (46.8% of the cases)
    prefix-sum map                 : 0 mismatches
    shortest window failure: [2 -2] k=2: brute force 1, window 2

(d) exactly k distinct letters over "abcd", 20000 random strings (length 0-15, k 1-4), 18749 of them non-empty
    atMost(k) - atMost(k-1) window : 0 mismatches with brute force over 212191 subarrays
    first non-empty case: "dabbcdcdbca" k=2: window 14, brute force 14

(e) n = 100000, values 0..3, sum exactly 100: 66606 subarrays
    window  : pointer moves 199934 + 199935 = 399869 (bound 4n = 400000), extra space: 2 integers (sum, left)
    map     : 100000 lookups + 100001 inserts = 200001 hash operations, extra space: 75065 entries; same count: true
    exactly 3 distinct of 8 values: 159715 subarrays, pointer moves 199996 + 199997, extra space: 8 counters (plus left)
    n(n+1)/2 = 5000050000 possible subarrays: that does not fit in a 32-bit unsigned counter (max 4294967295)
    window: no table at all, but wrong in 9358 of 20000 cases with negatives; map: 75065 entries, wrong in 0
```

**Rust:**

```rust
use std::collections::{HashMap, HashSet};

// Pcg: the port of Go's math/rand/v2 PCG-DXSM generator and its IntN (the same
// ~30 lines as in lesson 01 example 14), so seed (1, 2) feeds this program the
// same arrays Go sees and every count below comes out identical.
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

// "At most k" is monotone: shrink a valid window and it stays valid. So for a
// fixed right edge, EVERY start in [left, right] is valid, and one pass can add
// right-left+1 per step. "Exactly k" has no such interval, but
//
//     exactly(k) = atMost(k) - atMost(k-1)
//
// because a window with exactly k is a window with at most k that is not a window
// with at most k-1. Two problems below: exactly k DISTINCT values, and sum
// exactly k over NON-NEGATIVE values (zeros allowed).
//
// All counts are i64, not usize/u64: n(n+1)/2 outgrows u32, and the difference of
// two counts is a signed quantity if a broken window ever makes it negative.

/// cost counts the work: pointer moves for the windows, hash operations for the map.
#[derive(Default)]
struct Cost {
    moves: usize,
    lookups: usize,
    inserts: usize,
}

struct Step {
    left: usize,
    add: usize,
}

/// atMostDistinct counts subarrays with at most k distinct values (values 0..7).
/// The window keeps a count per value: Theta(1) space for a fixed alphabet.
fn at_most_distinct(a: &[i64], k: usize, c: &mut Cost, mut trace: Option<&mut Vec<Step>>) -> i64 {
    let mut cnt = [0usize; 8];
    let (mut distinct, mut left, mut total) = (0, 0, 0i64);
    for (right, &v) in a.iter().enumerate() {
        let v = v as usize;
        if cnt[v] == 0 {
            distinct += 1;
        }
        cnt[v] += 1;
        c.moves += 1; // right moved
        while distinct > k {
            let u = a[left] as usize;
            cnt[u] -= 1;
            if cnt[u] == 0 {
                distinct -= 1;
            }
            left += 1;
            c.moves += 1; // left moved
        }
        // every start in [left, right] is valid. `right - left + 1` would
        // underflow a usize when the window is empty (left == right + 1), so the
        // +1 goes first.
        let add = right + 1 - left;
        total += add as i64;
        if let Some(t) = trace.as_mut() {
            t.push(Step { left, add });
        }
    }
    total
}

/// atMostSum counts subarrays with sum <= k. Correct ONLY when no value is negative:
/// then growing never lowers the sum and shrinking never raises it.
fn at_most_sum(a: &[i64], k: i64, c: &mut Cost) -> i64 {
    if k < 0 {
        // non-negative values: no window has a negative sum
        return 0;
    }
    let (mut sum, mut left, mut total) = (0i64, 0, 0i64);
    for (right, &v) in a.iter().enumerate() {
        sum += v;
        c.moves += 1;
        while sum > k && left <= right {
            sum -= a[left];
            left += 1;
            c.moves += 1;
        }
        total += (right + 1 - left) as i64;
    }
    total
}

/// naiveExactSum is the tempting one-pass version: shrink while sum > k, then
/// count ONE window per right edge if the sum is exactly k.
fn naive_exact_sum(a: &[i64], k: i64) -> i64 {
    let (mut sum, mut left, mut count) = (0i64, 0, 0i64);
    for (right, &v) in a.iter().enumerate() {
        sum += v;
        while sum > k && left <= right {
            sum -= a[left];
            left += 1;
        }
        if left <= right && sum == k {
            count += 1;
        }
    }
    count
}

/// prefixMap is lesson 10 example 13: count earlier prefixes equal to sum-k.
/// Valid for any integers; costs a map. Returns (count, map entries).
fn prefix_map(a: &[i64], k: i64, c: &mut Cost) -> (i64, usize) {
    let mut seen: HashMap<i64, i64> = HashMap::from([(0, 1)]);
    c.inserts += 1; // the seed: the empty prefix
    let (mut sum, mut count) = (0i64, 0i64);
    for &v in a {
        sum += v;
        c.lookups += 1;
        count += seen.get(&(sum - k)).copied().unwrap_or(0);
        *seen.entry(sum).or_insert(0) += 1;
        c.inserts += 1;
    }
    (count, seen.len())
}

// The oracles: try every start and every end.
fn brute_sum(a: &[i64], k: i64) -> i64 {
    let mut count = 0;
    for i in 0..a.len() {
        let mut sum = 0;
        for &v in &a[i..] {
            sum += v;
            if sum == k {
                count += 1;
            }
        }
    }
    count
}

fn brute_distinct(a: &[i64], k: usize) -> i64 {
    let mut count = 0;
    for i in 0..a.len() {
        let mut seen = [false; 8];
        let mut d = 0;
        for &v in &a[i..] {
            if !seen[v as usize] {
                seen[v as usize] = true;
                d += 1;
            }
            if d == k {
                count += 1;
            }
        }
    }
    count
}

/// startsWith lists the starts s <= right whose window a[s..=right] has exactly k distinct values.
fn starts_with(a: &[i64], right: usize, k: usize) -> Vec<usize> {
    (0..=right)
        .filter(|&s| a[s..=right].iter().collect::<HashSet<_>>().len() == k)
        .collect()
}

/// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
/// so the output reads the same in both languages.
fn go_slice<T: ToString>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

struct Failure {
    a: Vec<i64>,
    k: i64,
    want: i64,
    got: i64,
}

impl Failure {
    fn describe(&self, who: &str) -> String {
        format!(
            "{} k={}: brute force {}, {who} {}",
            go_slice(&self.a),
            self.k,
            self.want,
            self.got
        )
    }
}

/// tally records disagreements with the brute force: how many (and how many
/// overcounted), the first, and the shortest.
#[derive(Default)]
struct Tally {
    n: usize,
    over: usize,
    first: Option<Failure>,
    short: Option<Failure>,
}

impl Tally {
    fn note(&mut self, a: &[i64], k: i64, want: i64, got: i64) {
        if want == got {
            return;
        }
        self.n += 1;
        if got > want {
            self.over += 1;
        }
        let make = || Failure {
            a: a.to_vec(),
            k,
            want,
            got,
        };
        if self.first.is_none() {
            self.first = Some(make());
        }
        if self.short.as_ref().is_none_or(|s| a.len() < s.a.len()) {
            self.short = Some(make());
        }
    }
}

fn gen_array(rng: &mut Pcg, max_len: u64, alphabet: u64, offset: i64) -> Vec<i64> {
    let len = rng.int_n(max_len + 1) as usize;
    (0..len)
        .map(|_| rng.int_n(alphabet) as i64 + offset)
        .collect()
}

fn letters(a: &[i64]) -> String {
    a.iter().map(|&v| (b'a' + v as u8) as char).collect()
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    let mut scratch = Cost::default();

    // (a) the table: [1 2 1 2 3], exactly 2 distinct.
    let a: Vec<i64> = vec![1, 2, 1, 2, 3];
    const K: usize = 2;
    let (mut t2, mut t1) = (Vec::new(), Vec::new());
    let (mut c2, mut c1) = (Cost::default(), Cost::default());
    let at2 = at_most_distinct(&a, K, &mut c2, Some(&mut t2));
    let at1 = at_most_distinct(&a, K - 1, &mut c1, Some(&mut t1));
    println!(
        "(a) {}: subarrays with exactly {K} distinct values",
        go_slice(&a)
    );
    println!(
        "right a[r] | atMost(2): left add total | atMost(1): left add total | diff | starts with exactly 2 (brute force)"
    );
    let (mut tot2, mut tot1) = (0, 0);
    for r in 0..a.len() {
        tot2 += t2[r].add;
        tot1 += t1[r].add;
        println!(
            "{r:>5} {:>4} | {:>16} {:>3} {tot2:>5} | {:>16} {:>3} {tot1:>5} | {:>4} | {}",
            a[r],
            t2[r].left,
            t2[r].add,
            t1[r].left,
            t1[r].add,
            t2[r].add - t1[r].add,
            go_slice(&starts_with(&a, r, K)),
        );
    }
    println!(
        "atMost(2) = {at2}, atMost(1) = {at1}, difference = {}, brute force = {}",
        at2 - at1,
        brute_distinct(&a, K)
    );
    println!();

    // (b) sums over NON-NEGATIVE arrays (zeros occur): window difference vs map vs brute force.
    const CASES: usize = 20000;
    let (mut win_b, mut map_b, mut naive_b) =
        (Tally::default(), Tally::default(), Tally::default());
    let mut total = 0;
    for _ in 0..CASES {
        let x = gen_array(&mut rng, 15, 4, 0);
        let kk = rng.int_n(9) as i64;
        let want = brute_sum(&x, kk);
        total += want;
        let window = at_most_sum(&x, kk, &mut scratch) - at_most_sum(&x, kk - 1, &mut scratch);
        win_b.note(&x, kk, want, window);
        let (got, _) = prefix_map(&x, kk, &mut scratch);
        map_b.note(&x, kk, want, got);
        naive_b.note(&x, kk, want, naive_exact_sum(&x, kk));
    }
    println!(
        "(b) sum exactly k, values 0..3, {CASES} random cases (length 0-15, k 0-8), {total} subarrays in total"
    );
    println!(
        "    atMost(k) - atMost(k-1) window : {} mismatches with brute force",
        win_b.n
    );
    println!(
        "    prefix-sum map                 : {} mismatches",
        map_b.n
    );
    println!(
        "    naive one-pass 'exact' window  : {} mismatches ({:.1}% of the cases), {} of them overcounts",
        naive_b.n,
        100.0 * naive_b.n as f64 / CASES as f64,
        naive_b.over
    );
    if let (Some(first), Some(short)) = (&naive_b.first, &naive_b.short) {
        println!(
            "    first naive failure  : {}",
            first.describe("naive window")
        );
        println!(
            "    shortest naive failure: {}",
            short.describe("naive window")
        );
    }
    println!();

    // (c) the same with negative values allowed (-2..3): the window argument is void.
    let (mut win_c, mut map_c) = (Tally::default(), Tally::default());
    for _ in 0..CASES {
        let x = gen_array(&mut rng, 15, 6, -2);
        let kk = rng.int_n(8) as i64 + 1;
        let want = brute_sum(&x, kk);
        let window = at_most_sum(&x, kk, &mut scratch) - at_most_sum(&x, kk - 1, &mut scratch);
        win_c.note(&x, kk, want, window);
        let (got, _) = prefix_map(&x, kk, &mut scratch);
        map_c.note(&x, kk, want, got);
    }
    println!("(c) sum exactly k, values -2..3, {CASES} random cases (k 1-8)");
    println!(
        "    atMost(k) - atMost(k-1) window : {} mismatches ({:.1}% of the cases)",
        win_c.n,
        100.0 * win_c.n as f64 / CASES as f64
    );
    println!(
        "    prefix-sum map                 : {} mismatches",
        map_c.n
    );
    if let Some(short) = &win_c.short {
        println!("    shortest window failure: {}", short.describe("window"));
    }
    println!();

    // (d) exactly k distinct letters over "abcd".
    let mut win_d = Tally::default();
    let (mut total_d, mut non_empty) = (0, 0);
    let mut first_d = String::new();
    for _ in 0..CASES {
        let x = gen_array(&mut rng, 15, 4, 0);
        let kk = rng.int_n(4) as usize + 1;
        let want = brute_distinct(&x, kk);
        total_d += want;
        let got = at_most_distinct(&x, kk, &mut scratch, None)
            - at_most_distinct(&x, kk - 1, &mut scratch, None);
        if !x.is_empty() {
            non_empty += 1;
            if first_d.is_empty() {
                first_d = format!("{:?} k={kk}: window {got}, brute force {want}", letters(&x));
            }
        }
        win_d.note(&x, kk as i64, want, got);
    }
    println!(
        "(d) exactly k distinct letters over \"abcd\", {CASES} random strings (length 0-15, k 1-4), {non_empty} of them non-empty"
    );
    println!(
        "    atMost(k) - atMost(k-1) window : {} mismatches with brute force over {total_d} subarrays",
        win_d.n
    );
    println!("    first non-empty case: {first_d}");
    println!();

    // (e) cost at n = 100,000.
    const N: usize = 100000;
    let big: Vec<i64> = (0..N).map(|_| rng.int_n(4) as i64).collect();
    const K_BIG: i64 = 100;
    let (mut cw1, mut cw2, mut cm) = (Cost::default(), Cost::default(), Cost::default());
    let wcount = at_most_sum(&big, K_BIG, &mut cw1) - at_most_sum(&big, K_BIG - 1, &mut cw2);
    let (mcount, entries) = prefix_map(&big, K_BIG, &mut cm);
    println!("(e) n = {N}, values 0..3, sum exactly {K_BIG}: {wcount} subarrays");
    println!(
        "    window  : pointer moves {} + {} = {} (bound 4n = {}), extra space: 2 integers (sum, left)",
        cw1.moves,
        cw2.moves,
        cw1.moves + cw2.moves,
        4 * N
    );
    println!(
        "    map     : {} lookups + {} inserts = {} hash operations, extra space: {entries} entries; same count: {}",
        cm.lookups,
        cm.inserts,
        cm.lookups + cm.inserts,
        mcount == wcount
    );
    let dist: Vec<i64> = (0..N).map(|_| rng.int_n(8) as i64).collect();
    let (mut cd1, mut cd2) = (Cost::default(), Cost::default());
    let dcount =
        at_most_distinct(&dist, 3, &mut cd1, None) - at_most_distinct(&dist, 2, &mut cd2, None);
    println!(
        "    exactly 3 distinct of 8 values: {dcount} subarrays, pointer moves {} + {}, extra space: 8 counters (plus left)",
        cd1.moves, cd2.moves
    );
    const MAX_SUB: u64 = (N * (N + 1) / 2) as u64;
    let cmp = if MAX_SUB > u64::from(u32::MAX) {
        "does not fit in"
    } else {
        "fits in"
    };
    println!(
        "    n(n+1)/2 = {MAX_SUB} possible subarrays: that {cmp} a 32-bit unsigned counter (max {})",
        u32::MAX
    );
    println!(
        "    window: no table at all, but wrong in {} of {CASES} cases with negatives; map: {entries} entries, wrong in {}",
        win_c.n, map_c.n
    );
}
```

**Rust output:**

```
(a) [1 2 1 2 3]: subarrays with exactly 2 distinct values
right a[r] | atMost(2): left add total | atMost(1): left add total | diff | starts with exactly 2 (brute force)
    0    1 |                0   1     1 |                0   1     1 |    0 | []
    1    2 |                0   2     3 |                1   1     2 |    1 | [0]
    2    1 |                0   3     6 |                2   1     3 |    2 | [0 1]
    3    2 |                0   4    10 |                3   1     4 |    3 | [0 1 2]
    4    3 |                3   2    12 |                4   1     5 |    1 | [3]
atMost(2) = 12, atMost(1) = 5, difference = 7, brute force = 7

(b) sum exactly k, values 0..3, 20000 random cases (length 0-15, k 0-8), 64761 subarrays in total
    atMost(k) - atMost(k-1) window : 0 mismatches with brute force
    prefix-sum map                 : 0 mismatches
    naive one-pass 'exact' window  : 6740 mismatches (33.7% of the cases), 0 of them overcounts
    first naive failure  : [2 0 0 1 2 1 0 3] k=4: brute force 7, naive window 3
    shortest naive failure: [0 1] k=1: brute force 2, naive window 1

(c) sum exactly k, values -2..3, 20000 random cases (k 1-8)
    atMost(k) - atMost(k-1) window : 9358 mismatches (46.8% of the cases)
    prefix-sum map                 : 0 mismatches
    shortest window failure: [2 -2] k=2: brute force 1, window 2

(d) exactly k distinct letters over "abcd", 20000 random strings (length 0-15, k 1-4), 18749 of them non-empty
    atMost(k) - atMost(k-1) window : 0 mismatches with brute force over 212191 subarrays
    first non-empty case: "dabbcdcdbca" k=2: window 14, brute force 14

(e) n = 100000, values 0..3, sum exactly 100: 66606 subarrays
    window  : pointer moves 199934 + 199935 = 399869 (bound 4n = 400000), extra space: 2 integers (sum, left)
    map     : 100000 lookups + 100001 inserts = 200001 hash operations, extra space: 75065 entries; same count: true
    exactly 3 distinct of 8 values: 159715 subarrays, pointer moves 199996 + 199997, extra space: 8 counters (plus left)
    n(n+1)/2 = 5000050000 possible subarrays: that does not fit in a 32-bit unsigned counter (max 4294967295)
    window: no table at all, but wrong in 9358 of 20000 cases with negatives; map: 75065 entries, wrong in 0
```

> **Go vs Rust:** Identical output, byte for byte (diffed against the Go run): the `Pcg` port and the same sequence of `int_n` calls give both programs the same 20,000 arrays per section, so every mismatch count and pointer-move count matches. Two Rust-specific details: the add per right edge is written `right + 1 - left`, because on a `usize` `right - left + 1` underflows whenever the window is empty (`left == right + 1`, which happens as soon as one value exceeds k; Go's `int` would just compute 0 either way), and counts are `i64` so that the difference of two broken window counts could never wrap. A missing map key needs `seen.get(&key).copied().unwrap_or(0)` where Go reads `seen[key]` as 0, and `*seen.entry(sum).or_insert(0) += 1` replaces `seen[sum]++`.

**Complexity:** Θ(n) per at-most pass (at most 2n pointer moves; two passes took 399,869 of the 4n = 400,000 at n = 100,000) and Θ(1) extra space for a fixed alphabet. `exactly(k) = atMost(k) − atMost(k−1)` had 0 mismatches in 20,000 cases for non-negative sums and for distinct letters, the naive one-pass "exact" window undercounted in 6,740 of 20,000 cases and never overcounted, and with negatives the window was wrong in 9,358 of 20,000 cases while the prefix-sum map (n lookups, n+1 inserts, 75,065 entries here) stayed at 0 mismatches.

---

## 13. 3Sum: two pointers inside a loop

`🔴 hard` · *Θ(n³) → Θ(n²)*

3Sum asks for every distinct triple of values that adds up to 0. Brute force examines all n(n−1)(n−2)/6 index triples, Θ(n³). The better idea is pair sum (example 1) inside a loop: sort, fix `i`, run the converging pointers on the suffix `[i+1, n)` looking for `a[lo] + a[hi] == -a[i]`, and skip equal neighbours at `i` and after every hit so each distinct triple is reported once. That is Θ(n²) after the Θ(n log n) sort, with no structure beyond the sorted array and the output. This example traces the classic input step by step, counts brute-force triples against pointer steps as n grows (running brute force only up to n = 400), checks the algorithm and its two dedupe bugs against a brute-force set on 20,000 random arrays, and compares the hash-set formulation, which is also Θ(n²) but allocates and needs its own dedupe, in hash operations and allocations.

**Steps:**

1. Write `scan` over a sorted array with the two dedupe rules switchable, and `bruteSet` (every triple, collected into a set of sorted triples) as the oracle.
2. Trace `[-1 0 1 2 -1 -4]`, then count sums evaluated against C(n,3) at n = 50 to 4,000, and run the three variants (correct, no skip at `i`, no skip after a hit) over 20,000 random arrays.
3. Read the counts, the counterexamples, and the allocation line for the hash-set version against the pointers.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// 3Sum: all distinct triples of values that add up to 0.
//
// Sort, fix i, and run example 1's converging pointers on the suffix [i+1, n)
// looking for a[lo] + a[hi] == -a[i]. Duplicates are removed by skipping equal
// neighbours: at i, and on both sides after a hit. Sorting makes equal values
// adjacent, which is what makes skipping enough (no set of triples needed).

type counters struct{ steps, moves int } // sums evaluated; pointer moves

type row struct {
	i, lo, hi, sum int
	action         string
}

// scan runs the pointers over an already SORTED array, appending triples to out.
// skipI and skipHit switch the two dedupe rules off, to show what each one buys.
func scan(a []int, skipI, skipHit bool, out [][3]int, c *counters, tr *[]row) [][3]int {
	n := len(a)
	for i := 0; i+2 < n; i++ {
		if skipI && i > 0 && a[i] == a[i-1] {
			if tr != nil {
				*tr = append(*tr, row{i: i, lo: -1, hi: -1, action: fmt.Sprintf("a[%d] == a[%d]: skip this i", i, i-1)})
			}
			continue
		}
		lo, hi := i+1, n-1
		for lo < hi {
			sum := a[i] + a[lo] + a[hi]
			c.steps++
			switch {
			case sum < 0:
				if tr != nil {
					*tr = append(*tr, row{i, lo, hi, sum, "too small: lo++"})
				}
				lo++
				c.moves++
			case sum > 0:
				if tr != nil {
					*tr = append(*tr, row{i, lo, hi, sum, "too big: hi--"})
				}
				hi--
				c.moves++
			default:
				t := [3]int{a[i], a[lo], a[hi]}
				out = append(out, t)
				lo0, hi0 := lo, hi
				lo++
				hi--
				c.moves += 2
				skipped := 0
				if skipHit {
					for lo < hi && a[lo] == a[lo-1] {
						lo++
						skipped++
						c.moves++
					}
					for lo < hi && a[hi] == a[hi+1] {
						hi--
						skipped++
						c.moves++
					}
				}
				if tr != nil {
					*tr = append(*tr, row{i, lo0, hi0, 0, fmt.Sprintf("hit %v: lo++ hi--, then skip %d equal", t, skipped)})
				}
			}
		}
	}
	return out
}

// threeSum is the whole algorithm: copy, sort, scan.
func threeSum(a []int) [][3]int {
	s := slices.Clone(a)
	slices.Sort(s)
	var c counters
	return scan(s, true, true, nil, &c, nil)
}

func cmpTriple(x, y [3]int) int { return slices.Compare(x[:], y[:]) }

func sorted3(x, y, z int) [3]int {
	t := [3]int{x, y, z}
	slices.Sort(t[:])
	return t
}

// bruteSet is the oracle: every index triple i<j<k, and the SET of sorted value
// triples that sum to 0. It also reports how many index triples it examined.
func bruteSet(a []int) (out [][3]int, examined int) {
	set := map[[3]int]struct{}{}
	for i := 0; i < len(a); i++ {
		for j := i + 1; j < len(a); j++ {
			for k := j + 1; k < len(a); k++ {
				examined++
				if a[i]+a[j]+a[k] == 0 {
					set[sorted3(a[i], a[j], a[k])] = struct{}{}
				}
			}
		}
	}
	for t := range set {
		out = append(out, t)
	}
	slices.SortFunc(out, cmpTriple)
	return out, examined
}

// asSet sorts a result list and drops repeats, to ask "same triples, ignoring repeats?".
func asSet(ts [][3]int) [][3]int {
	s := slices.Clone(ts)
	slices.SortFunc(s, cmpTriple)
	return slices.Compact(s)
}

// hashCost counts the hash-table operations of the hash-set version.
type hashCost struct{ lookups, seenInserts, foundInserts int }

// threeSumHash is the other classic: for each i, scan j > i with a set of the
// values already seen, and dedupe whole triples through a second set. No sort of
// the array, but a fresh map per i and a map of triples.
func threeSumHash(a []int, c *hashCost) [][3]int {
	found := map[[3]int]struct{}{}
	for i := 0; i < len(a); i++ {
		seen := map[int]struct{}{}
		for j := i + 1; j < len(a); j++ {
			want := -a[i] - a[j]
			c.lookups++
			if _, ok := seen[want]; ok {
				found[sorted3(a[i], a[j], want)] = struct{}{}
				c.foundInserts++
			}
			seen[a[j]] = struct{}{}
			c.seenInserts++
		}
	}
	out := make([][3]int, 0, len(found))
	for t := range found {
		out = append(out, t)
	}
	slices.SortFunc(out, cmpTriple)
	return out
}

func randArray(rng *rand.Rand, n, span, offset int) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = rng.IntN(span) + offset
	}
	return a
}

// variant is one version of the scan: which dedupe rules it keeps, and what it got wrong.
type variant struct {
	name            string
	skipI, skipHit  bool
	bad, onlyRepeat int   // wrong arrays; of those, how many had the right triples with repeats
	first, short    []int // the first failing input, and the shortest
}

// run sorts a copy of x and scans it with this variant's rules.
func (v *variant) run(x []int) [][3]int {
	s := slices.Clone(x)
	slices.Sort(s)
	var c counters
	return scan(s, v.skipI, v.skipHit, nil, &c, nil)
}

func choose3(n int) int { return n * (n - 1) * (n - 2) / 6 }

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	// (a) the classic example, with a trace of every step.
	in := []int{-1, 0, 1, 2, -1, -4}
	s := slices.Clone(in)
	slices.Sort(s)
	fmt.Printf("(a) input %v, sorted %v\n", in, s)
	var tr []row
	var ca counters
	res := scan(s, true, true, nil, &ca, &tr)
	fmt.Println("    i  lo  hi | a[i] a[lo] a[hi] | sum | action")
	for _, r := range tr {
		if r.lo < 0 {
			fmt.Printf("  %3d   -   - | %4d    -    - |   - | %s\n", r.i, s[r.i], r.action)
			continue
		}
		fmt.Printf("  %3d %3d %3d | %4d %4d %4d | %3d | %s\n", r.i, r.lo, r.hi, s[r.i], s[r.lo], s[r.hi], r.sum, r.action)
	}
	oracle, _ := bruteSet(in)
	fmt.Printf("    result %v, %d sums evaluated, %d pointer moves; brute-force set %v\n\n", res, ca.steps, ca.moves, oracle)

	// (b) counting: brute force vs pointers on random arrays, values in -n..n.
	fmt.Println("(b) triples examined by brute force vs sums evaluated by the pointers (values -n..n; the sort, Theta(n log n), is not counted)")
	fmt.Println("        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples | brute set agrees")
	for _, n := range []int{50, 100, 200, 400} {
		a := randArray(rng, n, 2*n+1, -n)
		slices.Sort(a)
		var c counters
		got := scan(a, true, true, nil, &c, nil)
		want, examined := bruteSet(a)
		fmt.Printf("  %7d | %14d | %12d | %18d | %14.1f | %8.3f | %16d | %v\n",
			n, examined, c.steps, (n-1)*(n-2)/2, float64(examined)/float64(c.steps),
			float64(c.steps)/float64(n*n), len(got), slices.Equal(got, want))
	}
	fmt.Println("    pointers only; C(n,3) is the analytic count of triples brute force would examine (not run):")
	fmt.Println("        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples")
	last := 0
	for _, n := range []int{1000, 2000, 4000} {
		a := randArray(rng, n, 2*n+1, -n)
		slices.Sort(a)
		var c counters
		got := scan(a, true, true, nil, &c, nil)
		fmt.Printf("  %7d | %14d | %12d | %18d | %14.1f | %8.3f | %16d\n",
			n, choose3(n), c.steps, (n-1)*(n-2)/2, float64(choose3(n))/float64(c.steps),
			float64(c.steps)/float64(n*n), len(got))
		if n == 4000 {
			fmt.Printf("    doubling n from 2000 to 4000 multiplied the pointer sums by %.2f (a quadratic gives 4, a cubic 8)\n", float64(c.steps)/float64(last))
		}
		last = c.steps
	}
	fmt.Println()

	// (c) the oracle, and the two dedupe bugs.
	const cases = 20000
	vs := []*variant{
		{name: "both skips (correct)", skipI: true, skipHit: true},
		{name: "no skip at i", skipI: false, skipHit: true},
		{name: "no skip after a hit", skipI: true, skipHit: false},
	}
	var found int
	for range cases {
		x := randArray(rng, rng.IntN(11), 7, -3)
		want, _ := bruteSet(x)
		found += len(want)
		for _, v := range vs {
			got := v.run(x)
			if slices.Equal(got, want) {
				continue
			}
			v.bad++
			if slices.Equal(asSet(got), want) {
				v.onlyRepeat++
			}
			if v.first == nil {
				v.first = x
			}
			if v.short == nil || len(x) < len(v.short) {
				v.short = x
			}
		}
	}
	fmt.Printf("(c) %d random arrays (length 0-10, values -3..3) against the brute-force set of sorted triples (%d triples in all)\n", cases, found)
	for _, v := range vs {
		fmt.Printf("    %-22s %5d wrong", v.name, v.bad)
		if v.bad > 0 {
			fmt.Printf(", %d of them with the right triples but repeats", v.onlyRepeat)
		}
		fmt.Println()
		if v.bad > 0 {
			sf, _ := bruteSet(v.first)
			ss, _ := bruteSet(v.short)
			fmt.Printf("      first    %v: oracle %v, got %v\n", v.first, sf, v.run(v.first))
			fmt.Printf("      shortest %v: oracle %v, got %v\n", v.short, ss, v.run(v.short))
		}
	}
	fmt.Println()

	// (d) the hash-set formulation: same answers, counted in hash operations and allocations.
	const n = 200
	a := randArray(rng, n, 2*n+1, -n)
	var hc hashCost
	hres := threeSumHash(a, &hc)
	sorted := slices.Clone(a)
	slices.Sort(sorted)
	var pc counters
	pres := scan(sorted, true, true, nil, &pc, nil)
	buf := make([][3]int, 0, len(pres)) // preallocated output: the scan itself needs no heap
	pAllocs := testing.AllocsPerRun(1, func() {
		var c counters
		buf = scan(sorted, true, true, buf[:0], &c, nil)
	})
	hAllocs := testing.AllocsPerRun(1, func() {
		var c hashCost
		hres = threeSumHash(a, &c)
	})
	fmt.Printf("(d) n = %d, values -%d..%d: two pointers vs the hash-set version\n", n, n, n)
	fmt.Printf("    same %d distinct triples: %v\n", len(pres), slices.Equal(hres, pres))
	fmt.Printf("    pointers : %6d sums evaluated, %d allocations (array already sorted, output buffer preallocated)\n", pc.steps, int(pAllocs))
	fmt.Printf("    hash sets: %6d lookups + %d inserts into the per-i sets + %d inserts of triples, %d allocations\n",
		hc.lookups, hc.seenInserts, hc.foundInserts, int(hAllocs))
	fmt.Printf("    the hash version made %d hits for %d distinct triples: its triple set is what dedupes\n", hc.foundInserts, len(pres))
}
```

**Sample output:**

```
(a) input [-1 0 1 2 -1 -4], sorted [-4 -1 -1 0 1 2]
    i  lo  hi | a[i] a[lo] a[hi] | sum | action
    0   1   5 |   -4   -1    2 |  -3 | too small: lo++
    0   2   5 |   -4   -1    2 |  -3 | too small: lo++
    0   3   5 |   -4    0    2 |  -2 | too small: lo++
    0   4   5 |   -4    1    2 |  -1 | too small: lo++
    1   2   5 |   -1   -1    2 |   0 | hit [-1 -1 2]: lo++ hi--, then skip 0 equal
    1   3   4 |   -1    0    1 |   0 | hit [-1 0 1]: lo++ hi--, then skip 0 equal
    2   -   - |   -1    -    - |   - | a[2] == a[1]: skip this i
    3   4   5 |    0    1    2 |   3 | too big: hi--
    result [[-1 -1 2] [-1 0 1]], 7 sums evaluated, 9 pointer moves; brute-force set [[-1 -1 2] [-1 0 1]]

(b) triples examined by brute force vs sums evaluated by the pointers (values -n..n; the sort, Theta(n log n), is not counted)
        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples | brute set agrees
       50 |          19600 |          909 |               1176 |           21.6 |    0.364 |               77 | true
      100 |         161700 |         3338 |               4851 |           48.4 |    0.334 |              250 | true
      200 |        1313400 |        14571 |              19701 |           90.1 |    0.364 |             1382 | true
      400 |       10586800 |        54941 |              79401 |          192.7 |    0.343 |             5127 | true
    pointers only; C(n,3) is the analytic count of triples brute force would examine (not run):
        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples
     1000 |      166167000 |       344840 |             498501 |          481.9 |    0.345 |            30704
     2000 |     1331334000 |      1384323 |            1997001 |          961.7 |    0.346 |           120019
     4000 |    10658668000 |      5527674 |            7994001 |         1928.2 |    0.345 |           479691
    doubling n from 2000 to 4000 multiplied the pointer sums by 3.99 (a quadratic gives 4, a cubic 8)

(c) 20000 random arrays (length 0-10, values -3..3) against the brute-force set of sorted triples (20788 triples in all)
    both skips (correct)       0 wrong
    no skip at i            5282 wrong, 5282 of them with the right triples but repeats
      first    [3 0 3 2 2 -1 -3 -2 -1 -1]: oracle [[-3 0 3] [-2 -1 3] [-2 0 2] [-1 -1 2]], got [[-3 0 3] [-2 -1 3] [-2 0 2] [-1 -1 2] [-1 -1 2]]
      shortest [1 -1 0 -1]: oracle [[-1 0 1]], got [[-1 0 1] [-1 0 1]]
    no skip after a hit     2313 wrong, 2313 of them with the right triples but repeats
      first    [3 0 3 2 2 -1 -3 -2 -1 -1]: oracle [[-3 0 3] [-2 -1 3] [-2 0 2] [-1 -1 2]], got [[-3 0 3] [-2 -1 3] [-2 -1 3] [-2 0 2] [-1 -1 2] [-1 -1 2]]
      shortest [0 0 3 3 -3]: oracle [[-3 0 3]], got [[-3 0 3] [-3 0 3]]

(d) n = 200, values -200..200: two pointers vs the hash-set version
    same 1310 distinct triples: true
    pointers :  13683 sums evaluated, 0 allocations (array already sorted, output buffer preallocated)
    hash sets:  19900 lookups + 19900 inserts into the per-i sets + 2172 inserts of triples, 1726 allocations
    the hash version made 2172 hits for 1310 distinct triples: its triple set is what dedupes
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashSet;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Pcg: the port of Go's math/rand/v2 PCG-DXSM generator and its IntN (the same
// ~30 lines as in lesson 01 example 14), so seed (1, 2) feeds this program the
// same arrays Go sees and every count below comes out identical.
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

// A counting allocator: stand-in for Go's testing.AllocsPerRun. It counts every
// allocation event (a fresh allocation or a growing realloc) and forwards the
// real work to the system allocator. Std has no safe way to hook the allocator.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

// SAFETY: every method forwards to `System` with the arguments it received, so
// `System`'s contract (layout validity, matching alloc/dealloc) is passed through.
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

/// Allocation events while running `f` (like AllocsPerRun, one measured run).
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let before = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - before, r)
}

// 3Sum: all distinct triples of values that add up to 0.
//
// Sort, fix i, and run example 1's converging pointers on the suffix [i+1, n)
// looking for a[lo] + a[hi] == -a[i]. Duplicates are removed by skipping equal
// neighbours: at i, and on both sides after a hit. Sorting makes equal values
// adjacent, which is what makes skipping enough (no set of triples needed).
//
// Sums are i64: the values here are small (|a| <= 4000), so a[i] + a[lo] + a[hi]
// cannot overflow. For arbitrary i64 input you would use checked_add, or widen
// to i128, because -O silently wraps and a debug build panics.

/// sums evaluated; pointer moves
#[derive(Default)]
struct Counters {
    steps: usize,
    moves: usize,
}

struct Row {
    i: usize,
    ends: Option<(usize, usize)>, // (lo, hi); None for a skipped i
    sum: i64,
    action: String,
}

/// scan runs the pointers over an already SORTED array, appending triples to out.
/// skip_i and skip_hit switch the two dedupe rules off, to show what each one buys.
fn scan(
    a: &[i64],
    skip_i: bool,
    skip_hit: bool,
    out: &mut Vec<[i64; 3]>,
    c: &mut Counters,
    mut tr: Option<&mut Vec<Row>>,
) {
    let n = a.len();
    for i in 0..n.saturating_sub(2) {
        if skip_i && i > 0 && a[i] == a[i - 1] {
            if let Some(t) = tr.as_mut() {
                t.push(Row {
                    i,
                    ends: None,
                    sum: 0,
                    action: format!("a[{i}] == a[{}]: skip this i", i - 1),
                });
            }
            continue;
        }
        let (mut lo, mut hi) = (i + 1, n - 1);
        while lo < hi {
            let sum = a[i] + a[lo] + a[hi];
            c.steps += 1;
            // `hi -= 1` cannot underflow: inside the loop hi > lo >= 1.
            if sum < 0 {
                if let Some(t) = tr.as_mut() {
                    t.push(Row {
                        i,
                        ends: Some((lo, hi)),
                        sum,
                        action: "too small: lo++".to_string(),
                    });
                }
                lo += 1;
                c.moves += 1;
            } else if sum > 0 {
                if let Some(t) = tr.as_mut() {
                    t.push(Row {
                        i,
                        ends: Some((lo, hi)),
                        sum,
                        action: "too big: hi--".to_string(),
                    });
                }
                hi -= 1;
                c.moves += 1;
            } else {
                let t = [a[i], a[lo], a[hi]];
                out.push(t);
                let (lo0, hi0) = (lo, hi);
                lo += 1;
                hi -= 1;
                c.moves += 2;
                let mut skipped = 0;
                if skip_hit {
                    while lo < hi && a[lo] == a[lo - 1] {
                        lo += 1;
                        skipped += 1;
                        c.moves += 1;
                    }
                    while lo < hi && a[hi] == a[hi + 1] {
                        hi -= 1;
                        skipped += 1;
                        c.moves += 1;
                    }
                }
                if let Some(tr) = tr.as_mut() {
                    tr.push(Row {
                        i,
                        ends: Some((lo0, hi0)),
                        sum: 0,
                        action: format!(
                            "hit {}: lo++ hi--, then skip {skipped} equal",
                            go_slice(&t)
                        ),
                    });
                }
            }
        }
    }
}

fn sorted3(x: i64, y: i64, z: i64) -> [i64; 3] {
    let mut t = [x, y, z];
    t.sort_unstable();
    t
}

/// bruteSet is the oracle: every index triple i<j<k, and the SET of sorted value
/// triples that sum to 0. It also reports how many index triples it examined.
fn brute_set(a: &[i64]) -> (Vec<[i64; 3]>, usize) {
    let mut set = HashSet::new();
    let mut examined = 0;
    for i in 0..a.len() {
        for j in i + 1..a.len() {
            for k in j + 1..a.len() {
                examined += 1;
                if a[i] + a[j] + a[k] == 0 {
                    set.insert(sorted3(a[i], a[j], a[k]));
                }
            }
        }
    }
    let mut out: Vec<[i64; 3]> = set.into_iter().collect();
    out.sort_unstable(); // arrays compare lexicographically
    (out, examined)
}

/// asSet sorts a result list and drops repeats, to ask "same triples, ignoring repeats?".
fn as_set(ts: &[[i64; 3]]) -> Vec<[i64; 3]> {
    let mut s = ts.to_vec();
    s.sort_unstable();
    s.dedup();
    s
}

/// hashCost counts the hash-table operations of the hash-set version.
#[derive(Default)]
struct HashCost {
    lookups: usize,
    seen_inserts: usize,
    found_inserts: usize,
}

/// threeSumHash is the other classic: for each i, scan j > i with a set of the
/// values already seen, and dedupe whole triples through a second set. No sort of
/// the array, but a fresh set per i and a set of triples.
fn three_sum_hash(a: &[i64], c: &mut HashCost) -> Vec<[i64; 3]> {
    let mut found: HashSet<[i64; 3]> = HashSet::new();
    for i in 0..a.len() {
        let mut seen: HashSet<i64> = HashSet::new();
        for j in i + 1..a.len() {
            let want = -a[i] - a[j];
            c.lookups += 1;
            if seen.contains(&want) {
                found.insert(sorted3(a[i], a[j], want));
                c.found_inserts += 1;
            }
            seen.insert(a[j]);
            c.seen_inserts += 1;
        }
    }
    let mut out: Vec<[i64; 3]> = found.into_iter().collect();
    out.sort_unstable();
    out
}

/// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
/// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn go_triples(ts: &[[i64; 3]]) -> String {
    let items: Vec<String> = ts.iter().map(|t| go_slice(t)).collect();
    format!("[{}]", items.join(" "))
}

fn rand_array(rng: &mut Pcg, n: usize, span: u64, offset: i64) -> Vec<i64> {
    (0..n).map(|_| rng.int_n(span) as i64 + offset).collect()
}

fn choose3(n: usize) -> usize {
    n * (n - 1) * (n - 2) / 6
}

/// variant is one version of the scan: which dedupe rules it keeps, and what it got wrong.
struct Variant {
    name: &'static str,
    skip_i: bool,
    skip_hit: bool,
    bad: usize,
    only_repeat: usize, // wrong arrays whose triples were right but repeated
    first: Option<Vec<i64>>,
    short: Option<Vec<i64>>,
}

impl Variant {
    fn new(name: &'static str, skip_i: bool, skip_hit: bool) -> Self {
        Variant {
            name,
            skip_i,
            skip_hit,
            bad: 0,
            only_repeat: 0,
            first: None,
            short: None,
        }
    }

    /// run sorts a copy of x and scans it with this variant's rules.
    fn run(&self, x: &[i64]) -> Vec<[i64; 3]> {
        let mut s = x.to_vec();
        s.sort_unstable();
        let mut out = Vec::new();
        scan(
            &s,
            self.skip_i,
            self.skip_hit,
            &mut out,
            &mut Counters::default(),
            None,
        );
        out
    }
}

fn main() {
    let mut rng = Pcg::new(1, 2);

    // (a) the classic example, with a trace of every step.
    let input = [-1, 0, 1, 2, -1, -4];
    let mut s = input.to_vec();
    s.sort_unstable();
    println!("(a) input {}, sorted {}", go_slice(&input), go_slice(&s));
    let mut tr = Vec::new();
    let mut ca = Counters::default();
    let mut res = Vec::new();
    scan(&s, true, true, &mut res, &mut ca, Some(&mut tr));
    println!("    i  lo  hi | a[i] a[lo] a[hi] | sum | action");
    for r in &tr {
        match r.ends {
            None => println!(
                "  {:>3}   -   - | {:>4}    -    - |   - | {}",
                r.i, s[r.i], r.action
            ),
            Some((lo, hi)) => println!(
                "  {:>3} {lo:>3} {hi:>3} | {:>4} {:>4} {:>4} | {:>3} | {}",
                r.i, s[r.i], s[lo], s[hi], r.sum, r.action
            ),
        }
    }
    let (oracle, _) = brute_set(&input);
    println!(
        "    result {}, {} sums evaluated, {} pointer moves; brute-force set {}",
        go_triples(&res),
        ca.steps,
        ca.moves,
        go_triples(&oracle)
    );
    println!();

    // (b) counting: brute force vs pointers on random arrays, values in -n..n.
    println!(
        "(b) triples examined by brute force vs sums evaluated by the pointers (values -n..n; the sort, Theta(n log n), is not counted)"
    );
    println!(
        "        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples | brute set agrees"
    );
    for n in [50usize, 100, 200, 400] {
        let mut a = rand_array(&mut rng, n, 2 * n as u64 + 1, -(n as i64));
        a.sort_unstable();
        let mut c = Counters::default();
        let mut got = Vec::new();
        scan(&a, true, true, &mut got, &mut c, None);
        let (want, examined) = brute_set(&a);
        println!(
            "  {n:>7} | {examined:>14} | {:>12} | {:>18} | {:>14.1} | {:>8.3} | {:>16} | {}",
            c.steps,
            (n - 1) * (n - 2) / 2,
            examined as f64 / c.steps as f64,
            c.steps as f64 / (n * n) as f64,
            got.len(),
            got == want
        );
    }
    println!(
        "    pointers only; C(n,3) is the analytic count of triples brute force would examine (not run):"
    );
    println!(
        "        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples"
    );
    let mut last = 0;
    for n in [1000usize, 2000, 4000] {
        let mut a = rand_array(&mut rng, n, 2 * n as u64 + 1, -(n as i64));
        a.sort_unstable();
        let mut c = Counters::default();
        let mut got = Vec::new();
        scan(&a, true, true, &mut got, &mut c, None);
        println!(
            "  {n:>7} | {:>14} | {:>12} | {:>18} | {:>14.1} | {:>8.3} | {:>16}",
            choose3(n),
            c.steps,
            (n - 1) * (n - 2) / 2,
            choose3(n) as f64 / c.steps as f64,
            c.steps as f64 / (n * n) as f64,
            got.len()
        );
        if n == 4000 {
            println!(
                "    doubling n from 2000 to 4000 multiplied the pointer sums by {:.2} (a quadratic gives 4, a cubic 8)",
                c.steps as f64 / last as f64
            );
        }
        last = c.steps;
    }
    println!();

    // (c) the oracle, and the two dedupe bugs.
    const CASES: usize = 20000;
    let mut vs = [
        Variant::new("both skips (correct)", true, true),
        Variant::new("no skip at i", false, true),
        Variant::new("no skip after a hit", true, false),
    ];
    let mut found = 0;
    for _ in 0..CASES {
        let len = rng.int_n(11) as usize;
        let x = rand_array(&mut rng, len, 7, -3);
        let (want, _) = brute_set(&x);
        found += want.len();
        for v in vs.iter_mut() {
            let got = v.run(&x);
            if got == want {
                continue;
            }
            v.bad += 1;
            if as_set(&got) == want {
                v.only_repeat += 1;
            }
            if v.first.is_none() {
                v.first = Some(x.clone());
            }
            if v.short.as_ref().is_none_or(|s| x.len() < s.len()) {
                v.short = Some(x.clone());
            }
        }
    }
    println!(
        "(c) {CASES} random arrays (length 0-10, values -3..3) against the brute-force set of sorted triples ({found} triples in all)"
    );
    for v in &vs {
        print!("    {:<22} {:>5} wrong", v.name, v.bad);
        if v.bad > 0 {
            print!(
                ", {} of them with the right triples but repeats",
                v.only_repeat
            );
        }
        println!();
        if let (Some(first), Some(short)) = (&v.first, &v.short) {
            for (label, x) in [("first   ", first), ("shortest", short)] {
                println!(
                    "      {label} {}: oracle {}, got {}",
                    go_slice(x),
                    go_triples(&brute_set(x).0),
                    go_triples(&v.run(x))
                );
            }
        }
    }
    println!();

    // (d) the hash-set formulation: same answers, counted in hash operations and allocations.
    const N: usize = 200;
    let a = rand_array(&mut rng, N, 2 * N as u64 + 1, -(N as i64));
    let mut hc = HashCost::default();
    let hres = three_sum_hash(&a, &mut hc);
    let mut sorted = a.clone();
    sorted.sort_unstable();
    let mut pc = Counters::default();
    let mut pres = Vec::new();
    scan(&sorted, true, true, &mut pres, &mut pc, None);
    // preallocated output: the scan itself needs no heap
    let mut buf: Vec<[i64; 3]> = Vec::with_capacity(pres.len());
    let (p_allocs, ()) = allocs_of(|| {
        buf.clear();
        scan(
            &sorted,
            true,
            true,
            &mut buf,
            &mut Counters::default(),
            None,
        );
    });
    let (h_allocs, hres2) = allocs_of(|| three_sum_hash(&a, &mut HashCost::default()));
    println!("(d) n = {N}, values -{N}..{N}: two pointers vs the hash-set version");
    println!(
        "    same {} distinct triples: {}",
        pres.len(),
        hres == pres && hres2 == pres
    );
    println!(
        "    pointers : {:>6} sums evaluated, {p_allocs} allocations (array already sorted, output buffer preallocated)",
        pc.steps
    );
    println!(
        "    hash sets: {:>6} lookups + {} inserts into the per-i sets + {} inserts of triples, {h_allocs} allocations",
        hc.lookups, hc.seen_inserts, hc.found_inserts
    );
    println!(
        "    the hash version made {} hits for {} distinct triples: its triple set is what dedupes",
        hc.found_inserts,
        pres.len()
    );
}
```

**Rust sample output:**

```
(a) input [-1 0 1 2 -1 -4], sorted [-4 -1 -1 0 1 2]
    i  lo  hi | a[i] a[lo] a[hi] | sum | action
    0   1   5 |   -4   -1    2 |  -3 | too small: lo++
    0   2   5 |   -4   -1    2 |  -3 | too small: lo++
    0   3   5 |   -4    0    2 |  -2 | too small: lo++
    0   4   5 |   -4    1    2 |  -1 | too small: lo++
    1   2   5 |   -1   -1    2 |   0 | hit [-1 -1 2]: lo++ hi--, then skip 0 equal
    1   3   4 |   -1    0    1 |   0 | hit [-1 0 1]: lo++ hi--, then skip 0 equal
    2   -   - |   -1    -    - |   - | a[2] == a[1]: skip this i
    3   4   5 |    0    1    2 |   3 | too big: hi--
    result [[-1 -1 2] [-1 0 1]], 7 sums evaluated, 9 pointer moves; brute-force set [[-1 -1 2] [-1 0 1]]

(b) triples examined by brute force vs sums evaluated by the pointers (values -n..n; the sort, Theta(n log n), is not counted)
        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples | brute set agrees
       50 |          19600 |          909 |               1176 |           21.6 |    0.364 |               77 | true
      100 |         161700 |         3338 |               4851 |           48.4 |    0.334 |              250 | true
      200 |        1313400 |        14571 |              19701 |           90.1 |    0.364 |             1382 | true
      400 |       10586800 |        54941 |              79401 |          192.7 |    0.343 |             5127 | true
    pointers only; C(n,3) is the analytic count of triples brute force would examine (not run):
        n |   C(n,3) brute | pointer sums | bound (n-1)(n-2)/2 | brute/pointers | sums/n^2 | distinct triples
     1000 |      166167000 |       344840 |             498501 |          481.9 |    0.345 |            30704
     2000 |     1331334000 |      1384323 |            1997001 |          961.7 |    0.346 |           120019
     4000 |    10658668000 |      5527674 |            7994001 |         1928.2 |    0.345 |           479691
    doubling n from 2000 to 4000 multiplied the pointer sums by 3.99 (a quadratic gives 4, a cubic 8)

(c) 20000 random arrays (length 0-10, values -3..3) against the brute-force set of sorted triples (20788 triples in all)
    both skips (correct)       0 wrong
    no skip at i            5282 wrong, 5282 of them with the right triples but repeats
      first    [3 0 3 2 2 -1 -3 -2 -1 -1]: oracle [[-3 0 3] [-2 -1 3] [-2 0 2] [-1 -1 2]], got [[-3 0 3] [-2 -1 3] [-2 0 2] [-1 -1 2] [-1 -1 2]]
      shortest [1 -1 0 -1]: oracle [[-1 0 1]], got [[-1 0 1] [-1 0 1]]
    no skip after a hit     2313 wrong, 2313 of them with the right triples but repeats
      first    [3 0 3 2 2 -1 -3 -2 -1 -1]: oracle [[-3 0 3] [-2 -1 3] [-2 0 2] [-1 -1 2]], got [[-3 0 3] [-2 -1 3] [-2 -1 3] [-2 0 2] [-1 -1 2] [-1 -1 2]]
      shortest [0 0 3 3 -3]: oracle [[-3 0 3]], got [[-3 0 3] [-3 0 3]]

(d) n = 200, values -200..200: two pointers vs the hash-set version
    same 1310 distinct triples: true
    pointers :  13683 sums evaluated, 0 allocations (array already sorted, output buffer preallocated)
    hash sets:  19900 lookups + 19900 inserts into the per-i sets + 2172 inserts of triples, 1164 allocations
    the hash version made 2172 hits for 1310 distinct triples: its triple set is what dedupes
```

> **Go vs Rust:** Every line is identical to the Go output (diffed) except the last measured one: the same hash operations (19,900 lookups, 19,900 + 2,172 inserts) cost 1,164 allocation events here against 1,726 in Go, because each language's table grows on its own schedule (plausible mechanism, not measured); the scan allocates 0 in both. Std has no allocation counter, so `testing.AllocsPerRun` becomes a ~15-line counting `#[global_allocator]` that forwards to `System` (the `unsafe impl` is unavoidable and carries a SAFETY comment). Sorted triples compare as `[i64; 3]` arrays, which are `Ord` lexicographically, so the oracle needs no comparator and `dedup` replaces `slices.Compact`.

**Complexity:** Θ(n²) after an Θ(n log n) sort. The pointers evaluated 54,941 sums at n = 400 where brute force examines C(400,3) = 10,586,800 triples (about 193 times fewer), and 5,527,674 sums at n = 4,000, always below the bound (n−1)(n−2)/2; doubling n from 2,000 to 4,000 multiplied the sums by 3.99. Both skips are needed: without the skip at `i` 5,282 of 20,000 arrays reported a triple twice, without the skip after a hit 2,313 did, and no run ever reported a wrong triple. The hash-set version found the same 1,310 distinct triples at n = 200 with 19,900 lookups, 19,900 set inserts and 2,172 triple inserts, and 1,726 allocations on Go 1.27 (the same on 60 runs), against 0 for the scan.

---

## 14. Which technique? A monotonicity audit

`🔴 hard` · *Recognising the pattern*

"When is a sliding window legal?" answered by brute force instead of folklore. A window loop is correct when validity is monotone under containment: **shrink-closed** (a valid window stays valid when shrunk, so the *longest* template works) or **expand-closed** (it stays valid when grown, so the *shortest* template works). This example enumerates every array of length 0 to 7 over a small alphabet (3,280, 21,845 and 97,656 arrays), tests both properties directly on every (array, sub-window) pair for ten predicates, then runs both generic window templates against a brute-force oracle. It prints which templates are right for which predicates, checks whether the two properties predict that, flags every exception, and ends with a decision table that sends each cue to the example that built the technique. The audit is exhaustive, so the counts are exact and identical in both languages.

**Steps:**

1. Write the ten predicates as closures built by small constructors (`sumAtMost(3)`, `distinctExactly(2)`, ...), the LONGEST and SHORTEST templates, and their brute-force oracles.
2. Enumerate every array with a base-`size` counter and, for each predicate, record the first shrink-closure and expand-closure counterexample and every array where a template disagrees with brute force.
3. Read the table, then the first counterexamples, then the summary: which predicates are window-safe, and which pairs of (predicate, template) are exceptions to the rule.

```go
package main

import (
	"fmt"
	"slices"
	"strconv"
	"strings"
)

// When is a sliding window legal? Not folklore: a window loop is correct when
// validity is MONOTONE under containment. Two properties, tested here directly
// on every small array:
//
//	shrink-closed: a valid window stays valid when you shrink it  -> LONGEST template works
//	expand-closed: a valid window stays valid when you grow it    -> SHORTEST template works
//
// Then both templates run against brute force for ten predicates, to see
// whether the properties predict which templates are right.

// ---- the ten predicates (built by constructors that capture their parameter) ----

func sum(w []int) (s int) {
	for _, v := range w {
		s += v
	}
	return
}

func sumAtMost(k int) func([]int) bool  { return func(w []int) bool { return sum(w) <= k } }
func sumAtLeast(k int) func([]int) bool { return func(w []int) bool { return sum(w) >= k } }
func sumEquals(k int) func([]int) bool  { return func(w []int) bool { return sum(w) == k } }
func sumDivisibleBy(m int) func([]int) bool {
	return func(w []int) bool { return sum(w)%m == 0 }
}

func distinct(w []int) (d int) {
	var seen [8]bool // values are in -2..3: index v+2
	for _, v := range w {
		if !seen[v+2] {
			seen[v+2] = true
			d++
		}
	}
	return
}

func distinctAtMost(k int) func([]int) bool  { return func(w []int) bool { return distinct(w) <= k } }
func distinctExactly(k int) func([]int) bool { return func(w []int) bool { return distinct(w) == k } }
func allDistinct(w []int) bool               { return distinct(w) == len(w) }

// maxMinAtMost recomputes max and min from the slice: fine for an audit. A real
// O(n) version needs two monotonic deques (lesson 09 example 10).
func maxMinAtMost(k int) func([]int) bool {
	return func(w []int) bool { return slices.Max(w)-slices.Min(w) <= k }
}

type predicate struct {
	name     string
	lo, size int // alphabet: size values starting at lo
	ok       func([]int) bool
	intended string // the template this predicate is paired with: "LONGEST" or "SHORTEST"
}

var predicates = []predicate{
	{"sum <= 3", 0, 3, sumAtMost(3), "LONGEST"},
	{"sum <= 3", -2, 5, sumAtMost(3), "LONGEST"},
	{"sum >= 3", 0, 3, sumAtLeast(3), "SHORTEST"},
	{"sum >= 3", -2, 5, sumAtLeast(3), "SHORTEST"},
	{"sum == 3", 0, 3, sumEquals(3), "LONGEST"},
	{"at most 2 distinct", 0, 4, distinctAtMost(2), "LONGEST"},
	{"exactly 2 distinct", 0, 4, distinctExactly(2), "LONGEST"},
	{"all distinct", 0, 4, allDistinct, "LONGEST"},
	{"max - min <= 1", 0, 4, maxMinAtMost(1), "LONGEST"},
	{"sum divisible by 3", -2, 5, sumDivisibleBy(3), "LONGEST"},
}

// ---- the two generic templates, and their brute-force oracles ----
// Convention: no valid (non-empty) window -> 0.

func longest(a []int, ok func([]int) bool) int {
	best, left := 0, 0
	for right := range a {
		for left <= right && !ok(a[left:right+1]) {
			left++
		}
		best = max(best, right-left+1)
	}
	return best
}

func shortest(a []int, ok func([]int) bool) int {
	none := len(a) + 1 // longer than any window: "nothing found yet"
	best, left := none, 0
	for right := range a {
		for left <= right && ok(a[left:right+1]) {
			best = min(best, right-left+1)
			left++
		}
	}
	if best == none {
		return 0
	}
	return best
}

func bruteLongest(a []int, ok func([]int) bool) int {
	best := 0
	for l := range a {
		for r := l; r < len(a); r++ {
			if ok(a[l : r+1]) {
				best = max(best, r-l+1)
			}
		}
	}
	return best
}

func bruteShortest(a []int, ok func([]int) bool) int {
	none := len(a) + 1
	best := none
	for l := range a {
		for r := l; r < len(a); r++ {
			if ok(a[l : r+1]) {
				best = min(best, r-l+1)
			}
		}
	}
	if best == none {
		return 0
	}
	return best
}

// eachArray calls f on every array of length 0..maxLen over the alphabet lo..lo+size-1.
// The slice is reused between calls: f must clone anything it keeps.
func eachArray(lo, size, maxLen int, f func([]int)) {
	for n := 0; n <= maxLen; n++ {
		a := make([]int, n)
		total := 1
		for range n {
			total *= size
		}
		for code := 0; code < total; code++ {
			c := code
			for i := range a {
				a[i] = lo + c%size
				c /= size
			}
			f(a)
		}
	}
}

type result struct {
	arrays, pairs int
	shrinkCE      string    // first shrink-closure counterexample ("" = closed)
	expandCE      string    // first expand-closure counterexample
	wrong         [2]int    // arrays on which LONGEST / SHORTEST disagreed with brute force
	firstWrong    [2]string // the first such array
	singleBad     bool      // some one-element window is invalid
}

// closed reports whether the closure property template t relies on held on every array.
func (r result) closed(t int) bool {
	if t == longestT {
		return r.shrinkCE == ""
	}
	return r.expandCE == ""
}

const longestT, shortestT = 0, 1

func audit(p predicate, maxLen int) result {
	var r result
	eachArray(p.lo, p.size, maxLen, func(a []int) {
		r.arrays++
		n := len(a)
		vx := n > 0 && p.ok(a)
		if n == 1 && !vx {
			r.singleBad = true
		}
		// closure: compare the array x with every proper non-empty sub-window w.
		for l := 0; l < n; l++ {
			for rr := l; rr < n; rr++ {
				if l == 0 && rr == n-1 {
					continue
				}
				r.pairs++
				w := a[l : rr+1]
				vw := p.ok(w)
				if r.shrinkCE == "" && vx && !vw {
					r.shrinkCE = fmt.Sprintf("%v is valid but its sub-window %v is not", a, w)
				}
				if r.expandCE == "" && vw && !vx {
					r.expandCE = fmt.Sprintf("%v is valid but the larger %v is not", w, a)
				}
			}
		}
		// templates against brute force.
		got := [2]int{longest(a, p.ok), shortest(a, p.ok)}
		want := [2]int{bruteLongest(a, p.ok), bruteShortest(a, p.ok)}
		for t := range got {
			if got[t] != want[t] {
				r.wrong[t]++
				if r.firstWrong[t] == "" {
					r.firstWrong[t] = fmt.Sprintf("%v: template %d, brute force %d", a, got[t], want[t])
				}
			}
		}
	})
	return r
}

func yesNo(ce string) string {
	if ce == "" {
		return "yes"
	}
	return "NO"
}

func list(ns []int) string {
	if len(ns) == 0 {
		return "none"
	}
	parts := make([]string, len(ns))
	for i, n := range ns {
		parts[i] = strconv.Itoa(n)
	}
	return strings.Join(parts, ", ")
}

func main() {
	const maxLen = 7
	fmt.Printf("every array of length 0..%d over each alphabet, windows non-empty, no valid window -> 0\n", maxLen)
	for _, al := range [][2]int{{0, 3}, {-2, 5}, {0, 4}} {
		count := 0
		eachArray(al[0], al[1], maxLen, func([]int) { count++ })
		fmt.Printf("  alphabet {%d..%d}: %d arrays\n", al[0], al[0]+al[1]-1, count)
	}
	fmt.Println()

	res := make([]result, len(predicates))
	for i, p := range predicates {
		res[i] = audit(p, maxLen)
	}

	fmt.Println(" #  predicate           values     arrays  window pairs | shrink-closed expand-closed | LONGEST wrong  SHORTEST wrong | paired with")
	for i, p := range predicates {
		r := res[i]
		fmt.Printf("%2d  %-18s {%2d..%d}  %7d  %12d | %-13s %-13s | %13d  %14d | %s\n",
			i+1, p.name, p.lo, p.lo+p.size-1, r.arrays, r.pairs,
			yesNo(r.shrinkCE), yesNo(r.expandCE), r.wrong[longestT], r.wrong[shortestT], p.intended)
	}
	fmt.Println()

	fmt.Println("first counterexamples (enumeration order is by length, so each is a shortest one)")
	for i, p := range predicates {
		r := res[i]
		fmt.Printf("%2d  %s over {%d..%d}\n", i+1, p.name, p.lo, p.lo+p.size-1)
		if r.shrinkCE != "" {
			fmt.Printf("      not shrink-closed: %s\n", r.shrinkCE)
		}
		if r.expandCE != "" {
			fmt.Printf("      not expand-closed: %s\n", r.expandCE)
		}
		for t, name := range []string{"LONGEST", "SHORTEST"} {
			if r.firstWrong[t] != "" {
				fmt.Printf("      %-8s template wrong on %s\n", name, r.firstWrong[t])
			}
		}
	}
	fmt.Println()

	// What the audit shows. The rule to test: shrink-closed => LONGEST is right,
	// expand-closed => SHORTEST is right; and a predicate without the property
	// should break the template somewhere.
	names := []string{"LONGEST", "SHORTEST"}
	var safe, unsafe, trivial, nonTrivial []int
	var closedRight, closedWrong, openWrong, openRight int
	var exceptions []string
	for i, p := range predicates {
		r := res[i]
		for t := range names {
			switch {
			case r.closed(t) && r.wrong[t] == 0:
				closedRight++
			case r.closed(t):
				closedWrong++
			case r.wrong[t] > 0:
				openWrong++
			default:
				openRight++
				exceptions = append(exceptions, fmt.Sprintf("%d with %s", i+1, names[t]))
				if t == shortestT && !r.singleBad {
					trivial = append(trivial, i+1)
				} else {
					nonTrivial = append(nonTrivial, i+1)
				}
			}
		}
		t := longestT
		if p.intended == "SHORTEST" {
			t = shortestT
		}
		if r.wrong[t] == 0 {
			safe = append(safe, i+1)
		} else {
			unsafe = append(unsafe, i+1)
		}
	}
	fmt.Println("what the audit shows")
	fmt.Printf("  the template each predicate is paired with is right on every array for: %s\n", list(safe))
	fmt.Printf("  and wrong on some array for: %s\n", list(unsafe))
	fmt.Printf("  matches the hypothesis (1, 3, 6, 8, 9 safe; 2, 4, 5, 7, 10 not): %v\n",
		slices.Equal(safe, []int{1, 3, 6, 8, 9}) && slices.Equal(unsafe, []int{2, 4, 5, 7, 10}))
	fmt.Printf("  across all %d (predicate, template) pairs, the closure a template needs holds in %d pairs: %d right on every array, %d wrong somewhere\n",
		2*len(predicates), closedRight+closedWrong, closedRight, closedWrong)
	fmt.Printf("  it fails in %d pairs: %d wrong on some array, %d right on every array\n",
		openWrong+openRight, openWrong, openRight)
	if openRight > 0 {
		fmt.Printf("  those %d exceptions: predicate %s\n", openRight, strings.Join(exceptions, ", "))
		fmt.Printf("    for predicates %s every one-element window is valid, so the shortest window is trivially 1 (0 for the empty array)\n", list(trivial))
		fmt.Printf("    the others (%s) are real exceptions: not closed, yet right on every array up to length %d,\n", list(nonTrivial), maxLen)
		fmt.Println("    so the two properties are sufficient conditions for a template, not necessary ones")
	}
	var unsafeNames []string
	for _, n := range unsafe {
		p := predicates[n-1]
		unsafeNames = append(unsafeNames, fmt.Sprintf("%s over {%d..%d}", p.name, p.lo, p.lo+p.size-1))
	}
	fmt.Printf("  window loops are the wrong tool for: %s\n\n", strings.Join(unsafeNames, "; "))

	fmt.Println("decision table: the cue in the problem -> the technique (and the example that built it)")
	rows := [][3]string{
		{"sorted (or sortable) array, a pair or triple with a target", "converging pointers", "examples 1, 7, 13"},
		{"rewrite an array in place, keeping some elements", "read pointer chasing a write pointer", "example 2"},
		{"split into two or three classes in one pass", "three-way flag partition", "example 8"},
		{"every window of one fixed size k", "fixed window: add one, drop one", "example 3"},
		{"longest window, validity survives shrinking", "variable window, longest shape", "example 10"},
		{"shortest window, validity survives growing", "variable window, shortest shape", "example 9"},
		{"count windows with exactly k (non-negative / distinct)", "atMost(k) - atMost(k-1)", "example 12"},
		{"many range sums or XORs", "prefix array", "examples 4, 5, 11, 15"},
		{"many range updates, one read at the end", "difference array", "example 6"},
		{"sum equals k with negatives, or sum divisible by m", "prefix sums + a map of earlier prefixes", "examples 12, 15; lesson 10 ex 13"},
		{"unsorted array, a pair with a target", "hash map of seen values", "lesson 10 example 12"},
		{"window maximum or minimum", "monotonic deque", "lesson 09 example 10"},
	}
	for _, r := range rows {
		fmt.Printf("  %-58s -> %-40s [%s]\n", r[0], r[1], r[2])
	}
	fmt.Printf("  audit evidence: the paired window template was right on every array for predicates %s and wrong for %s\n", list(safe), list(unsafe))
}
```

**Output:**

```
every array of length 0..7 over each alphabet, windows non-empty, no valid window -> 0
  alphabet {0..2}: 3280 arrays
  alphabet {-2..2}: 97656 arrays
  alphabet {0..3}: 21845 arrays

 #  predicate           values     arrays  window pairs | shrink-closed expand-closed | LONGEST wrong  SHORTEST wrong | paired with
 1  sum <= 3           { 0..2}     3280         77913 | yes           NO            |             0               0 | LONGEST
 2  sum <= 3           {-2..2}    97656       2471925 | NO            NO            |         11760               0 | LONGEST
 3  sum >= 3           { 0..2}     3280         77913 | NO            yes           |          3160               0 | SHORTEST
 4  sum >= 3           {-2..2}    97656       2471925 | NO            NO            |         55206           24035 | SHORTEST
 5  sum == 3           { 0..2}     3280         77913 | NO            NO            |          2941            1229 | LONGEST
 6  at most 2 distinct { 0..3}    21845        541280 | yes           NO            |             0               0 | LONGEST
 7  exactly 2 distinct { 0..3}    21845        541280 | NO            NO            |         21816               0 | LONGEST
 8  all distinct       { 0..3}    21845        541280 | yes           NO            |             0               0 | LONGEST
 9  max - min <= 1     { 0..3}    21845        541280 | yes           NO            |             0               0 | LONGEST
10  sum divisible by 3 {-2..2}    97656       2471925 | NO            NO            |         96768           66696 | LONGEST

first counterexamples (enumeration order is by length, so each is a shortest one)
 1  sum <= 3 over {0..2}
      not expand-closed: [2] is valid but the larger [2 2] is not
 2  sum <= 3 over {-2..2}
      not shrink-closed: [2 2 -2] is valid but its sub-window [2 2] is not
      not expand-closed: [2] is valid but the larger [2 2] is not
      LONGEST  template wrong on [2 2 -2]: template 2, brute force 3
 3  sum >= 3 over {0..2}
      not shrink-closed: [2 1] is valid but its sub-window [2] is not
      LONGEST  template wrong on [2 1]: template 0, brute force 2
 4  sum >= 3 over {-2..2}
      not shrink-closed: [2 1] is valid but its sub-window [2] is not
      not expand-closed: [2 1] is valid but the larger [2 1 -2] is not
      LONGEST  template wrong on [2 1]: template 0, brute force 2
      SHORTEST template wrong on [-2 2 1]: template 0, brute force 2
 5  sum == 3 over {0..2}
      not shrink-closed: [2 1] is valid but its sub-window [2] is not
      not expand-closed: [2 1] is valid but the larger [2 1 1] is not
      LONGEST  template wrong on [2 1]: template 0, brute force 2
      SHORTEST template wrong on [2 2 1]: template 0, brute force 2
 6  at most 2 distinct over {0..3}
      not expand-closed: [2] is valid but the larger [2 1 0] is not
 7  exactly 2 distinct over {0..3}
      not shrink-closed: [1 0] is valid but its sub-window [1] is not
      not expand-closed: [2 1] is valid but the larger [2 1 0] is not
      LONGEST  template wrong on [1 0]: template 0, brute force 2
 8  all distinct over {0..3}
      not expand-closed: [0] is valid but the larger [0 0] is not
 9  max - min <= 1 over {0..3}
      not expand-closed: [2] is valid but the larger [2 0] is not
10  sum divisible by 3 over {-2..2}
      not shrink-closed: [-1 -2] is valid but its sub-window [-1] is not
      not expand-closed: [0] is valid but the larger [0 -2] is not
      LONGEST  template wrong on [-1 -2]: template 0, brute force 2
      SHORTEST template wrong on [-2 0]: template 0, brute force 1

what the audit shows
  the template each predicate is paired with is right on every array for: 1, 3, 6, 8, 9
  and wrong on some array for: 2, 4, 5, 7, 10
  matches the hypothesis (1, 3, 6, 8, 9 safe; 2, 4, 5, 7, 10 not): true
  across all 20 (predicate, template) pairs, the closure a template needs holds in 5 pairs: 5 right on every array, 0 wrong somewhere
  it fails in 15 pairs: 9 wrong on some array, 6 right on every array
  those 6 exceptions: predicate 1 with SHORTEST, 2 with SHORTEST, 6 with SHORTEST, 7 with SHORTEST, 8 with SHORTEST, 9 with SHORTEST
    for predicates 1, 2, 6, 8, 9 every one-element window is valid, so the shortest window is trivially 1 (0 for the empty array)
    the others (7) are real exceptions: not closed, yet right on every array up to length 7,
    so the two properties are sufficient conditions for a template, not necessary ones
  window loops are the wrong tool for: sum <= 3 over {-2..2}; sum >= 3 over {-2..2}; sum == 3 over {0..2}; exactly 2 distinct over {0..3}; sum divisible by 3 over {-2..2}

decision table: the cue in the problem -> the technique (and the example that built it)
  sorted (or sortable) array, a pair or triple with a target -> converging pointers                      [examples 1, 7, 13]
  rewrite an array in place, keeping some elements           -> read pointer chasing a write pointer     [example 2]
  split into two or three classes in one pass                -> three-way flag partition                 [example 8]
  every window of one fixed size k                           -> fixed window: add one, drop one          [example 3]
  longest window, validity survives shrinking                -> variable window, longest shape           [example 10]
  shortest window, validity survives growing                 -> variable window, shortest shape          [example 9]
  count windows with exactly k (non-negative / distinct)     -> atMost(k) - atMost(k-1)                  [example 12]
  many range sums or XORs                                    -> prefix array                             [examples 4, 5, 11, 15]
  many range updates, one read at the end                    -> difference array                         [example 6]
  sum equals k with negatives, or sum divisible by m         -> prefix sums + a map of earlier prefixes  [examples 12, 15; lesson 10 ex 13]
  unsorted array, a pair with a target                       -> hash map of seen values                  [lesson 10 example 12]
  window maximum or minimum                                  -> monotonic deque                          [lesson 09 example 10]
  audit evidence: the paired window template was right on every array for predicates 1, 3, 6, 8, 9 and wrong for 2, 4, 5, 7, 10
```

**Rust:**

```rust
// When is a sliding window legal? Not folklore: a window loop is correct when
// validity is MONOTONE under containment. Two properties, tested here directly
// on every small array:
//
//     shrink-closed: a valid window stays valid when you shrink it  -> LONGEST template works
//     expand-closed: a valid window stays valid when you grow it    -> SHORTEST template works
//
// Then both templates run against brute force for ten predicates, to see
// whether the properties predict which templates are right.
//
// No randomness and no hashing: every number below is a count over an exhaustive
// enumeration, so it is identical to the Go program's.

/// A predicate is a boxed closure over a window. `dyn Fn` is the trait-object
/// counterpart of Go's `func([]int) bool` value: one pointer to the closure's
/// captured state plus a vtable, and a call is an indirect call.
type Pred = Box<dyn Fn(&[i64]) -> bool>;

// ---- the ten predicates (built by constructors that capture their parameter) ----

fn sum(w: &[i64]) -> i64 {
    w.iter().sum()
}

fn sum_at_most(k: i64) -> Pred {
    Box::new(move |w| sum(w) <= k)
}
fn sum_at_least(k: i64) -> Pred {
    Box::new(move |w| sum(w) >= k)
}
fn sum_equals(k: i64) -> Pred {
    Box::new(move |w| sum(w) == k)
}
fn sum_divisible_by(m: i64) -> Pred {
    Box::new(move |w| sum(w) % m == 0)
}

fn distinct(w: &[i64]) -> usize {
    let mut seen = [false; 8]; // values are in -2..3: index v+2
    let mut d = 0;
    for &v in w {
        let slot = &mut seen[(v + 2) as usize];
        if !*slot {
            *slot = true;
            d += 1;
        }
    }
    d
}

fn distinct_at_most(k: usize) -> Pred {
    Box::new(move |w| distinct(w) <= k)
}
fn distinct_exactly(k: usize) -> Pred {
    Box::new(move |w| distinct(w) == k)
}
fn all_distinct() -> Pred {
    Box::new(|w| distinct(w) == w.len())
}

/// max_min_at_most recomputes max and min from the slice: fine for an audit. A real
/// O(n) version needs two monotonic deques (lesson 09 example 10).
fn max_min_at_most(k: i64) -> Pred {
    Box::new(move |w| {
        // windows are never empty here, so the fold's seeds are always replaced
        let (min, max) = w
            .iter()
            .fold((i64::MAX, i64::MIN), |(lo, hi), &v| (lo.min(v), hi.max(v)));
        max - min <= k
    })
}

#[derive(Clone, Copy)]
enum Template {
    Longest = 0,
    Shortest = 1,
}

const TEMPLATES: [Template; 2] = [Template::Longest, Template::Shortest];

impl Template {
    fn name(self) -> &'static str {
        match self {
            Template::Longest => "LONGEST",
            Template::Shortest => "SHORTEST",
        }
    }
}

struct Predicate {
    name: &'static str,
    lo: i64,
    size: u64, // alphabet: size values starting at lo
    ok: Pred,
    intended: Template, // the template this predicate is paired with
}

fn predicates() -> Vec<Predicate> {
    use Template::{Longest, Shortest};
    let p = |name, lo, size, ok, intended| Predicate {
        name,
        lo,
        size,
        ok,
        intended,
    };
    vec![
        p("sum <= 3", 0, 3, sum_at_most(3), Longest),
        p("sum <= 3", -2, 5, sum_at_most(3), Longest),
        p("sum >= 3", 0, 3, sum_at_least(3), Shortest),
        p("sum >= 3", -2, 5, sum_at_least(3), Shortest),
        p("sum == 3", 0, 3, sum_equals(3), Longest),
        p("at most 2 distinct", 0, 4, distinct_at_most(2), Longest),
        p("exactly 2 distinct", 0, 4, distinct_exactly(2), Longest),
        p("all distinct", 0, 4, all_distinct(), Longest),
        p("max - min <= 1", 0, 4, max_min_at_most(1), Longest),
        p("sum divisible by 3", -2, 5, sum_divisible_by(3), Longest),
    ]
}

// ---- the two generic templates, and their brute-force oracles ----
// Convention: no valid (non-empty) window -> 0.
// `right + 1 - left` (never `right - left + 1`): left can be right + 1, and
// that subtraction on a usize would underflow.

fn longest(a: &[i64], ok: &dyn Fn(&[i64]) -> bool) -> usize {
    let (mut best, mut left) = (0, 0);
    for right in 0..a.len() {
        while left <= right && !ok(&a[left..=right]) {
            left += 1;
        }
        best = best.max(right + 1 - left);
    }
    best
}

fn shortest(a: &[i64], ok: &dyn Fn(&[i64]) -> bool) -> usize {
    let none = a.len() + 1; // longer than any window: "nothing found yet"
    let (mut best, mut left) = (none, 0);
    for right in 0..a.len() {
        while left <= right && ok(&a[left..=right]) {
            best = best.min(right + 1 - left);
            left += 1;
        }
    }
    if best == none { 0 } else { best }
}

fn brute_longest(a: &[i64], ok: &dyn Fn(&[i64]) -> bool) -> usize {
    let mut best = 0;
    for l in 0..a.len() {
        for r in l..a.len() {
            if ok(&a[l..=r]) {
                best = best.max(r - l + 1);
            }
        }
    }
    best
}

fn brute_shortest(a: &[i64], ok: &dyn Fn(&[i64]) -> bool) -> usize {
    let none = a.len() + 1;
    let mut best = none;
    for l in 0..a.len() {
        for r in l..a.len() {
            if ok(&a[l..=r]) {
                best = best.min(r - l + 1);
            }
        }
    }
    if best == none { 0 } else { best }
}

/// each_array calls f on every array of length 0..=max_len over the alphabet lo..lo+size-1.
/// The slice is reused between calls (a Vec the closure only borrows).
fn each_array(lo: i64, size: u64, max_len: usize, mut f: impl FnMut(&[i64])) {
    for n in 0..=max_len {
        let mut a = vec![0i64; n];
        let total = size.pow(n as u32);
        for code in 0..total {
            let mut c = code;
            for slot in a.iter_mut() {
                *slot = lo + (c % size) as i64;
                c /= size;
            }
            f(&a);
        }
    }
}

/// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
/// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

#[derive(Default)]
struct AuditResult {
    arrays: usize,
    pairs: usize,
    shrink_ce: String,        // first shrink-closure counterexample ("" = closed)
    expand_ce: String,        // first expand-closure counterexample
    wrong: [usize; 2],        // arrays on which LONGEST / SHORTEST disagreed with brute force
    first_wrong: [String; 2], // the first such array
    single_bad: bool,         // some one-element window is invalid
}

impl AuditResult {
    /// closed reports whether the closure property template t relies on held on every array.
    fn closed(&self, t: Template) -> bool {
        match t {
            Template::Longest => self.shrink_ce.is_empty(),
            Template::Shortest => self.expand_ce.is_empty(),
        }
    }
}

fn audit(p: &Predicate, max_len: usize) -> AuditResult {
    let mut r = AuditResult::default();
    let ok = p.ok.as_ref();
    each_array(p.lo, p.size, max_len, |a| {
        r.arrays += 1;
        let n = a.len();
        let vx = n > 0 && ok(a);
        if n == 1 && !vx {
            r.single_bad = true;
        }
        // closure: compare the array x with every proper non-empty sub-window w.
        for l in 0..n {
            for rr in l..n {
                if l == 0 && rr == n - 1 {
                    continue;
                }
                r.pairs += 1;
                let w = &a[l..=rr];
                let vw = ok(w);
                if r.shrink_ce.is_empty() && vx && !vw {
                    r.shrink_ce = format!(
                        "{} is valid but its sub-window {} is not",
                        go_slice(a),
                        go_slice(w)
                    );
                }
                if r.expand_ce.is_empty() && vw && !vx {
                    r.expand_ce = format!(
                        "{} is valid but the larger {} is not",
                        go_slice(w),
                        go_slice(a)
                    );
                }
            }
        }
        // templates against brute force.
        let got = [longest(a, ok), shortest(a, ok)];
        let want = [brute_longest(a, ok), brute_shortest(a, ok)];
        for t in 0..2 {
            if got[t] != want[t] {
                r.wrong[t] += 1;
                if r.first_wrong[t].is_empty() {
                    r.first_wrong[t] = format!(
                        "{}: template {}, brute force {}",
                        go_slice(a),
                        got[t],
                        want[t]
                    );
                }
            }
        }
    });
    r
}

fn yes_no(ce: &str) -> &'static str {
    if ce.is_empty() { "yes" } else { "NO" }
}

fn list(ns: &[usize]) -> String {
    if ns.is_empty() {
        return "none".to_string();
    }
    let parts: Vec<String> = ns.iter().map(usize::to_string).collect();
    parts.join(", ")
}

fn main() {
    const MAX_LEN: usize = 7;
    println!(
        "every array of length 0..{MAX_LEN} over each alphabet, windows non-empty, no valid window -> 0"
    );
    for (lo, size) in [(0i64, 3u64), (-2, 5), (0, 4)] {
        let mut count = 0;
        each_array(lo, size, MAX_LEN, |_| count += 1);
        println!(
            "  alphabet {{{lo}..{}}}: {count} arrays",
            lo + size as i64 - 1
        );
    }
    println!();

    let preds = predicates();
    let res: Vec<AuditResult> = preds.iter().map(|p| audit(p, MAX_LEN)).collect();

    println!(
        " #  predicate           values     arrays  window pairs | shrink-closed expand-closed | LONGEST wrong  SHORTEST wrong | paired with"
    );
    for (i, (p, r)) in preds.iter().zip(&res).enumerate() {
        println!(
            "{:>2}  {:<18} {{{:>2}..{}}}  {:>7}  {:>12} | {:<13} {:<13} | {:>13}  {:>14} | {}",
            i + 1,
            p.name,
            p.lo,
            p.lo + p.size as i64 - 1,
            r.arrays,
            r.pairs,
            yes_no(&r.shrink_ce),
            yes_no(&r.expand_ce),
            r.wrong[Template::Longest as usize],
            r.wrong[Template::Shortest as usize],
            p.intended.name()
        );
    }
    println!();

    println!("first counterexamples (enumeration order is by length, so each is a shortest one)");
    for (i, (p, r)) in preds.iter().zip(&res).enumerate() {
        println!(
            "{:>2}  {} over {{{}..{}}}",
            i + 1,
            p.name,
            p.lo,
            p.lo + p.size as i64 - 1
        );
        if !r.shrink_ce.is_empty() {
            println!("      not shrink-closed: {}", r.shrink_ce);
        }
        if !r.expand_ce.is_empty() {
            println!("      not expand-closed: {}", r.expand_ce);
        }
        for t in TEMPLATES {
            let first = &r.first_wrong[t as usize];
            if !first.is_empty() {
                println!("      {:<8} template wrong on {first}", t.name());
            }
        }
    }
    println!();

    // What the audit shows. The rule to test: shrink-closed => LONGEST is right,
    // expand-closed => SHORTEST is right; and a predicate without the property
    // should break the template somewhere.
    let (mut safe, mut unsafe_, mut trivial, mut non_trivial) = (vec![], vec![], vec![], vec![]);
    let (mut closed_right, mut closed_wrong, mut open_wrong, mut open_right) = (0, 0, 0, 0);
    let mut exceptions: Vec<String> = Vec::new();
    for (i, (p, r)) in preds.iter().zip(&res).enumerate() {
        for t in TEMPLATES {
            let wrong = r.wrong[t as usize];
            match (r.closed(t), wrong) {
                (true, 0) => closed_right += 1,
                (true, _) => closed_wrong += 1,
                (false, 1..) => open_wrong += 1,
                (false, 0) => {
                    open_right += 1;
                    exceptions.push(format!("{} with {}", i + 1, t.name()));
                    if matches!(t, Template::Shortest) && !r.single_bad {
                        trivial.push(i + 1);
                    } else {
                        non_trivial.push(i + 1);
                    }
                }
            }
        }
        if r.wrong[p.intended as usize] == 0 {
            safe.push(i + 1);
        } else {
            unsafe_.push(i + 1);
        }
    }
    println!("what the audit shows");
    println!(
        "  the template each predicate is paired with is right on every array for: {}",
        list(&safe)
    );
    println!("  and wrong on some array for: {}", list(&unsafe_));
    println!(
        "  matches the hypothesis (1, 3, 6, 8, 9 safe; 2, 4, 5, 7, 10 not): {}",
        safe == [1, 3, 6, 8, 9] && unsafe_ == [2, 4, 5, 7, 10]
    );
    println!(
        "  across all {} (predicate, template) pairs, the closure a template needs holds in {} pairs: {closed_right} right on every array, {closed_wrong} wrong somewhere",
        2 * preds.len(),
        closed_right + closed_wrong
    );
    println!(
        "  it fails in {} pairs: {open_wrong} wrong on some array, {open_right} right on every array",
        open_wrong + open_right
    );
    if open_right > 0 {
        println!(
            "  those {open_right} exceptions: predicate {}",
            exceptions.join(", ")
        );
        println!(
            "    for predicates {} every one-element window is valid, so the shortest window is trivially 1 (0 for the empty array)",
            list(&trivial)
        );
        println!(
            "    the others ({}) are real exceptions: not closed, yet right on every array up to length {MAX_LEN},",
            list(&non_trivial)
        );
        println!(
            "    so the two properties are sufficient conditions for a template, not necessary ones"
        );
    }
    let unsafe_names: Vec<String> = unsafe_
        .iter()
        .map(|&n| {
            let p = &preds[n - 1];
            format!("{} over {{{}..{}}}", p.name, p.lo, p.lo + p.size as i64 - 1)
        })
        .collect();
    println!(
        "  window loops are the wrong tool for: {}\n",
        unsafe_names.join("; ")
    );

    println!(
        "decision table: the cue in the problem -> the technique (and the example that built it)"
    );
    let rows = [
        (
            "sorted (or sortable) array, a pair or triple with a target",
            "converging pointers",
            "examples 1, 7, 13",
        ),
        (
            "rewrite an array in place, keeping some elements",
            "read pointer chasing a write pointer",
            "example 2",
        ),
        (
            "split into two or three classes in one pass",
            "three-way flag partition",
            "example 8",
        ),
        (
            "every window of one fixed size k",
            "fixed window: add one, drop one",
            "example 3",
        ),
        (
            "longest window, validity survives shrinking",
            "variable window, longest shape",
            "example 10",
        ),
        (
            "shortest window, validity survives growing",
            "variable window, shortest shape",
            "example 9",
        ),
        (
            "count windows with exactly k (non-negative / distinct)",
            "atMost(k) - atMost(k-1)",
            "example 12",
        ),
        (
            "many range sums or XORs",
            "prefix array",
            "examples 4, 5, 11, 15",
        ),
        (
            "many range updates, one read at the end",
            "difference array",
            "example 6",
        ),
        (
            "sum equals k with negatives, or sum divisible by m",
            "prefix sums + a map of earlier prefixes",
            "examples 12, 15; lesson 10 ex 13",
        ),
        (
            "unsorted array, a pair with a target",
            "hash map of seen values",
            "lesson 10 example 12",
        ),
        (
            "window maximum or minimum",
            "monotonic deque",
            "lesson 09 example 10",
        ),
    ];
    for (cue, technique, built) in rows {
        println!("  {cue:<58} -> {technique:<40} [{built}]");
    }
    println!(
        "  audit evidence: the paired window template was right on every array for predicates {} and wrong for {}",
        list(&safe),
        list(&unsafe_)
    );
}
```

**Rust output:**

```
every array of length 0..7 over each alphabet, windows non-empty, no valid window -> 0
  alphabet {0..2}: 3280 arrays
  alphabet {-2..2}: 97656 arrays
  alphabet {0..3}: 21845 arrays

 #  predicate           values     arrays  window pairs | shrink-closed expand-closed | LONGEST wrong  SHORTEST wrong | paired with
 1  sum <= 3           { 0..2}     3280         77913 | yes           NO            |             0               0 | LONGEST
 2  sum <= 3           {-2..2}    97656       2471925 | NO            NO            |         11760               0 | LONGEST
 3  sum >= 3           { 0..2}     3280         77913 | NO            yes           |          3160               0 | SHORTEST
 4  sum >= 3           {-2..2}    97656       2471925 | NO            NO            |         55206           24035 | SHORTEST
 5  sum == 3           { 0..2}     3280         77913 | NO            NO            |          2941            1229 | LONGEST
 6  at most 2 distinct { 0..3}    21845        541280 | yes           NO            |             0               0 | LONGEST
 7  exactly 2 distinct { 0..3}    21845        541280 | NO            NO            |         21816               0 | LONGEST
 8  all distinct       { 0..3}    21845        541280 | yes           NO            |             0               0 | LONGEST
 9  max - min <= 1     { 0..3}    21845        541280 | yes           NO            |             0               0 | LONGEST
10  sum divisible by 3 {-2..2}    97656       2471925 | NO            NO            |         96768           66696 | LONGEST

first counterexamples (enumeration order is by length, so each is a shortest one)
 1  sum <= 3 over {0..2}
      not expand-closed: [2] is valid but the larger [2 2] is not
 2  sum <= 3 over {-2..2}
      not shrink-closed: [2 2 -2] is valid but its sub-window [2 2] is not
      not expand-closed: [2] is valid but the larger [2 2] is not
      LONGEST  template wrong on [2 2 -2]: template 2, brute force 3
 3  sum >= 3 over {0..2}
      not shrink-closed: [2 1] is valid but its sub-window [2] is not
      LONGEST  template wrong on [2 1]: template 0, brute force 2
 4  sum >= 3 over {-2..2}
      not shrink-closed: [2 1] is valid but its sub-window [2] is not
      not expand-closed: [2 1] is valid but the larger [2 1 -2] is not
      LONGEST  template wrong on [2 1]: template 0, brute force 2
      SHORTEST template wrong on [-2 2 1]: template 0, brute force 2
 5  sum == 3 over {0..2}
      not shrink-closed: [2 1] is valid but its sub-window [2] is not
      not expand-closed: [2 1] is valid but the larger [2 1 1] is not
      LONGEST  template wrong on [2 1]: template 0, brute force 2
      SHORTEST template wrong on [2 2 1]: template 0, brute force 2
 6  at most 2 distinct over {0..3}
      not expand-closed: [2] is valid but the larger [2 1 0] is not
 7  exactly 2 distinct over {0..3}
      not shrink-closed: [1 0] is valid but its sub-window [1] is not
      not expand-closed: [2 1] is valid but the larger [2 1 0] is not
      LONGEST  template wrong on [1 0]: template 0, brute force 2
 8  all distinct over {0..3}
      not expand-closed: [0] is valid but the larger [0 0] is not
 9  max - min <= 1 over {0..3}
      not expand-closed: [2] is valid but the larger [2 0] is not
10  sum divisible by 3 over {-2..2}
      not shrink-closed: [-1 -2] is valid but its sub-window [-1] is not
      not expand-closed: [0] is valid but the larger [0 -2] is not
      LONGEST  template wrong on [-1 -2]: template 0, brute force 2
      SHORTEST template wrong on [-2 0]: template 0, brute force 1

what the audit shows
  the template each predicate is paired with is right on every array for: 1, 3, 6, 8, 9
  and wrong on some array for: 2, 4, 5, 7, 10
  matches the hypothesis (1, 3, 6, 8, 9 safe; 2, 4, 5, 7, 10 not): true
  across all 20 (predicate, template) pairs, the closure a template needs holds in 5 pairs: 5 right on every array, 0 wrong somewhere
  it fails in 15 pairs: 9 wrong on some array, 6 right on every array
  those 6 exceptions: predicate 1 with SHORTEST, 2 with SHORTEST, 6 with SHORTEST, 7 with SHORTEST, 8 with SHORTEST, 9 with SHORTEST
    for predicates 1, 2, 6, 8, 9 every one-element window is valid, so the shortest window is trivially 1 (0 for the empty array)
    the others (7) are real exceptions: not closed, yet right on every array up to length 7,
    so the two properties are sufficient conditions for a template, not necessary ones
  window loops are the wrong tool for: sum <= 3 over {-2..2}; sum >= 3 over {-2..2}; sum == 3 over {0..2}; exactly 2 distinct over {0..3}; sum divisible by 3 over {-2..2}

decision table: the cue in the problem -> the technique (and the example that built it)
  sorted (or sortable) array, a pair or triple with a target -> converging pointers                      [examples 1, 7, 13]
  rewrite an array in place, keeping some elements           -> read pointer chasing a write pointer     [example 2]
  split into two or three classes in one pass                -> three-way flag partition                 [example 8]
  every window of one fixed size k                           -> fixed window: add one, drop one          [example 3]
  longest window, validity survives shrinking                -> variable window, longest shape           [example 10]
  shortest window, validity survives growing                 -> variable window, shortest shape          [example 9]
  count windows with exactly k (non-negative / distinct)     -> atMost(k) - atMost(k-1)                  [example 12]
  many range sums or XORs                                    -> prefix array                             [examples 4, 5, 11, 15]
  many range updates, one read at the end                    -> difference array                         [example 6]
  sum equals k with negatives, or sum divisible by m         -> prefix sums + a map of earlier prefixes  [examples 12, 15; lesson 10 ex 13]
  unsorted array, a pair with a target                       -> hash map of seen values                  [lesson 10 example 12]
  window maximum or minimum                                  -> monotonic deque                          [lesson 09 example 10]
  audit evidence: the paired window template was right on every array for predicates 1, 3, 6, 8, 9 and wrong for 2, 4, 5, 7, 10
```

> **Go vs Rust:** Identical output, byte for byte (diffed against the Go run): the audit is an exhaustive enumeration, so there is no randomness to port. The planned `Box<dyn Fn(&[i64]) -> bool>` compiled as written (checked): it is the trait-object counterpart of Go's `func([]int) bool` value, a closure built by a constructor such as `sum_at_most(3)` with `move` to capture `k`; capture-free predicates could be plain `fn` pointers. Sub-windows are `&a[l..=r]` slices (no allocation), the per-window add is `right + 1 - left` because `right - left + 1` underflows a `usize` when `left == right + 1`, and `match (r.closed(t), wrong)` makes the compiler check that the four classes of (closure, wrong-count) cover every case.

**Complexity:** the audit is exhaustive (3,280 + 97,656 + 21,845 arrays, each tried against ten predicates and both templates), so every count is exact. The paired template was right on every array for predicates 1, 3, 6, 8 and 9 and wrong on some array for 2, 4, 5, 7 and 10, exactly the hypothesis. Across all 20 (predicate, template) pairs the closure a template needs held in 5, and all 5 were right on every array; it failed in 15 and 9 of those were wrong somewhere. The other 6 were right everywhere despite the missing closure (predicates 1, 2, 6, 7, 8, 9 with SHORTEST): five because every one-element window is valid so the answer is trivially 1, and one for real, "exactly 2 distinct", so the two properties are sufficient for a window template, not necessary. A window needs monotone validity; sums over values that can be negative, equality and divisibility predicates do not have it, so they go to prefix sums.

---

## 15. Beyond sums: XOR, remainders, and what has no inverse

`🔴 hard` · *Prefix structures and the inverse*

A prefix array answers a range query exactly when the operation can be undone: `P[r] ^ P[l]` is the XOR of `a[l:r]` because XOR is its own inverse, so the same prefix-plus-map identity that counts subarrays with sum k counts subarrays with XOR k, in Θ(n) against Θ(n²) for brute force. "Sum divisible by m" works through residues, since two prefixes with the same residue differ by a multiple of m, but Go's `%` takes the sign of the dividend: with negative numbers the residues `-1` and `m-1` name the same class and land in different buckets, and the count comes out too low. One line fixes it. Then the operations with no inverse: a prefix max is the max of a longer range, so it overshoots, and a prefix product cannot be divided once a zero has entered. Every check runs against a brute-force oracle on PCG-seeded inputs, so the counts are deterministic.

**Steps:**

1. Build prefix XOR, check a few ranges by hand, then count subarrays with XOR `k` with a map, with and without the `counts[0] = 1` seed, on 5,000 random arrays against brute force.
2. Count subarrays with sum divisible by `m` as the sum of C(c, 2) over residue classes, using the raw `x % m` and the normalised `((x % m) + m) % m`, and read the mismatches and the shortest counterexample.
3. Ask a prefix max for a range max and a prefix product for a range product on random queries, and count the answers that are wrong or undefined.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"os"
)

// A prefix array answers a range query exactly when the operation can be UNDONE:
//
//	range(l, r) = P[r] "minus" P[l]
//
// + has an inverse (-), XOR is its own inverse, a count has one. max, min and
// gcd do not, and a product only does while nothing is zero. This program shows
// the three cases that work or almost work, then the ones that cannot. Every
// check runs against a brute-force oracle on PCG-seeded inputs, so the counts
// below are the same in every run (and in the Rust twin).

// mustBe aborts the run if a claim the program is about to print would be false.
func mustBe(ok bool, what string) {
	if !ok {
		fmt.Println("INTERNAL CHECK FAILED:", what)
		os.Exit(1)
	}
}

// randArray draws a length in [minLen, maxLen], then that many values in
// [lo, lo+span). The draw order is part of the contract: the Rust twin makes
// the same calls on the same generator.
func randArray(rng *rand.Rand, minLen, maxLen, lo, span int) []int {
	a := make([]int, minLen+rng.IntN(maxLen-minLen+1))
	for i := range a {
		a[i] = lo + rng.IntN(span)
	}
	return a
}

// ---------- (a) prefix XOR ----------

// prefixXor returns P with P[0] = 0 and P[i+1] = P[i] ^ a[i].
func prefixXor(a []int) []int {
	p := make([]int, len(a)+1)
	for i, v := range a {
		p[i+1] = p[i] ^ v
	}
	return p
}

// countXorMap counts subarrays with XOR == k in one pass: a[l:r] has XOR k
// exactly when P[l] == P[r] ^ k, so look that prefix up among the earlier ones.
// seed is the counts[0] = 1 that stands for P[0]. ops counts map operations.
func countXorMap(a []int, k int, seed bool, ops *int) int {
	counts := map[int]int{}
	if seed {
		counts[0] = 1
	}
	x, total := 0, 0
	for _, v := range a {
		x ^= v
		total += counts[x^k] // one lookup
		counts[x]++          // one update
		*ops += 2
	}
	return total
}

// countXorBrute tries every (l, r), extending a running XOR from each start.
func countXorBrute(a []int, k int, steps *int) int {
	total := 0
	for l := range a {
		x := 0
		for r := l; r < len(a); r++ {
			x ^= a[r]
			*steps++
			if x == k {
				total++
			}
		}
	}
	return total
}

// ---------- (b) sum divisible by m, with negatives ----------

// Go's % truncates toward zero, so the result takes the sign of the DIVIDEND.
func rawMod(x, m int) int { return x % m }

// normMod is the one-line fix: shift the negative remainder up by m.
func normMod(x, m int) int { return ((x % m) + m) % m }

// countDivisible counts subarrays with sum % m == 0. Two prefixes with the same
// residue differ by a multiple of m, so every pair inside a residue class is one
// subarray: the answer is the sum over classes of C(c, 2). The only thing that
// differs between the two versions is the function that names the class.
func countDivisible(a []int, m int, mod func(x, m int) int) int {
	classes := map[int]int{}
	p := 0
	classes[mod(p, m)]++ // P[0]
	for _, v := range a {
		p += v
		classes[mod(p, m)]++
	}
	total := 0
	for _, c := range classes {
		total += c * (c - 1) / 2
	}
	return total
}

// bruteDivisible tries every (l, r). `s % m == 0` is a correct divisibility
// test for either sign: only COMPARING two remainders goes wrong.
func bruteDivisible(a []int, m int) int {
	total := 0
	for l := range a {
		s := 0
		for r := l; r < len(a); r++ {
			s += a[r]
			if s%m == 0 {
				total++
			}
		}
	}
	return total
}

// ---------- (c) what has no inverse ----------

// prefixMax returns pm with pm[i] = max(a[0..=i]). It is all a prefix can store.
func prefixMax(a []int) []int {
	pm := make([]int, len(a))
	for i, v := range a {
		pm[i] = v
		if i > 0 {
			pm[i] = max(pm[i-1], v)
		}
	}
	return pm
}

func rangeMax(a []int, l, r int) int {
	best := a[l]
	for _, v := range a[l+1 : r] {
		best = max(best, v)
	}
	return best
}

// prefixProduct returns P with P[0] = 1 and P[i+1] = P[i] * a[i].
func prefixProduct(a []int) []int {
	p := make([]int, len(a)+1)
	p[0] = 1
	for i, v := range a {
		p[i+1] = p[i] * v
	}
	return p
}

// rangeProduct is P[r] / P[l]. Once a zero has entered the prefix every later
// entry is 0, and dividing by it panics in Go, so we look first and say "undefined".
func rangeProduct(p []int, l, r int) (int, bool) {
	if p[l] == 0 {
		return 0, false
	}
	return p[r] / p[l], true
}

func bruteProduct(a []int, l, r int) int {
	prod := 1
	for _, v := range a[l:r] {
		prod *= v
	}
	return prod
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	// ---- (a) ----
	fmt.Println("(a) prefix XOR: XOR is its own inverse")
	a := []int{3, 5, 6, 1, 7, 2}
	p := prefixXor(a)
	fmt.Println("  a =", a)
	fmt.Println("  P =", p)
	for _, rg := range [][2]int{{1, 4}, {0, 6}, {3, 3}} {
		l, r := rg[0], rg[1]
		direct := 0
		for _, v := range a[l:r] {
			direct ^= v
		}
		fmt.Printf("  a[%d:%d] XOR = %d,  P[%d]^P[%d] = %d^%d = %d\n", l, r, direct, r, l, p[r], p[l], p[r]^p[l])
		mustBe(direct == p[r]^p[l], "range XOR")
	}

	const cases = 5000
	var elems, mapOps, bruteSteps, bad, badNoSeed int
	for range cases {
		arr := randArray(rng, 0, 15, 0, 8)
		k := rng.IntN(8)
		elems += len(arr)
		want := countXorBrute(arr, k, &bruteSteps)
		if countXorMap(arr, k, true, &mapOps) != want {
			bad++
		}
		var unused int
		if countXorMap(arr, k, false, &unused) != want {
			badNoSeed++
		}
	}
	fmt.Printf("\n  count subarrays with XOR == k, %d arrays (len 0..15, values 0..7, k 0..7):\n", cases)
	fmt.Printf("    map version vs brute force: %d mismatches\n", bad)
	fmt.Printf("    same, without counts[0] = 1: %d mismatches (every subarray that starts at index 0 is lost)\n", badNoSeed)
	fmt.Printf("    work: %d map operations for %d elements; brute force ran %d XOR steps\n", mapOps, elems, bruteSteps)
	mustBe(bad == 0, "XOR map count")
	mustBe(mapOps == 2*elems, "two map operations per element")

	// ---- (b) ----
	fmt.Println()
	fmt.Println("(b) sum divisible by m, with negative numbers")
	fmt.Println("  Go spec: x % y takes the sign of x (truncated division)")
	for _, xy := range [][2]int{{5, 3}, {-5, 3}, {5, -3}, {-5, -3}} {
		x, y := xy[0], xy[1]
		fmt.Printf("    %2d %% %2d = %2d     (x / y = %2d)\n", x, y, x%y, x/y)
	}

	type counter struct {
		a        []int
		m        int
		raw      int
		norm     int
		brute    int
		prefixes []int
	}
	var first, shortest *counter
	var wrongRaw, wrongNorm, under, over int
	var negCases, negWrong, nonNegWrong int
	for range cases {
		arr := randArray(rng, 0, 15, -5, 11)
		m := 2 + rng.IntN(5)
		raw, norm, brute := countDivisible(arr, m, rawMod), countDivisible(arr, m, normMod), bruteDivisible(arr, m)
		sums := []int{0}
		anyNeg := false
		for _, v := range arr {
			sums = append(sums, sums[len(sums)-1]+v)
			anyNeg = anyNeg || sums[len(sums)-1] < 0
		}
		if norm != brute {
			wrongNorm++
		}
		if anyNeg {
			negCases++
		}
		if raw == brute {
			continue
		}
		wrongRaw++
		if anyNeg {
			negWrong++
		} else {
			nonNegWrong++
		}
		if raw < brute {
			under++
		} else {
			over++
		}
		c := &counter{arr, m, raw, norm, brute, sums}
		if first == nil {
			first = c
		}
		if shortest == nil || len(arr) < len(shortest.a) {
			shortest = c
		}
	}
	fmt.Printf("\n  %d arrays (len 0..15, values -5..5, m 2..6), three ways to count:\n", cases)
	fmt.Printf("    normalised ((x %% m) + m) %% m vs brute force: %d mismatches\n", wrongNorm)
	fmt.Printf("    raw x %% m vs brute force:                      %d mismatches (%d too low, %d too high)\n", wrongRaw, under, over)
	fmt.Printf("    arrays with some negative prefix: %d, and raw was wrong on %d of them;\n", negCases, negWrong)
	fmt.Printf("    arrays whose prefixes are all >= 0: %d, and raw was wrong on %d of them\n", cases-negCases, nonNegWrong)
	mustBe(wrongNorm == 0, "normalised residues")
	mustBe(over == 0 && nonNegWrong == 0, "raw only undercounts, and only with a negative prefix")
	show := func(label string, c *counter) {
		fmt.Printf("\n  %s: a = %v, m = %d\n", label, c.a, c.m)
		fmt.Printf("    prefixes         %v\n", c.prefixes)
		rawRes, normRes := make([]int, len(c.prefixes)), make([]int, len(c.prefixes))
		for i, s := range c.prefixes {
			rawRes[i], normRes[i] = rawMod(s, c.m), normMod(s, c.m)
		}
		fmt.Printf("    raw residues     %v\n", rawRes)
		fmt.Printf("    normalised       %v\n", normRes)
		fmt.Printf("    pairs counted: raw %d, normalised %d, brute force %d\n", c.raw, c.norm, c.brute)
	}
	show("first counterexample in the stream", first)
	if shortest != first {
		show("shortest counterexample in the stream", shortest)
	}

	// ---- (c) ----
	fmt.Println()
	fmt.Println("(c) what has no inverse")
	const queries = 20000
	var total, wrong [2]int // index 0: queries with l == 0, index 1: with l > 0
	var tooLow int
	for range queries {
		arr := randArray(rng, 1, 12, 0, 10)
		l := rng.IntN(len(arr))
		r := l + 1 + rng.IntN(len(arr)-l)
		guess, truth := prefixMax(arr)[r-1], rangeMax(arr, l, r) // all a prefix can offer for a[l:r]
		side := min(l, 1)
		total[side]++
		if guess != truth {
			wrong[side]++
		}
		if guess < truth {
			tooLow++
		}
	}
	fmt.Printf("  range max from a prefix max, pm[r-1], on %d random (array, l, r) queries:\n", queries)
	fmt.Printf("    l == 0: %d queries, %d wrong;  l > 0: %d queries, %d wrong (%.1f%%)\n", total[0], wrong[0], total[1], wrong[1], 100*float64(wrong[1])/float64(total[1]))
	fmt.Printf("    answers below the true max: %d (it can only overshoot: it is the max of a longer range)\n", tooLow)
	mustBe(wrong[0] == 0 && tooLow == 0 && wrong[1] > 0, "prefix max")

	fmt.Println()
	ex := []int{2, 0, 3}
	pe := prefixProduct(ex)
	fmt.Printf("  prefix products of %v are %v: the last range a[2:3] = [3] would be %d / %d\n", ex, pe, pe[3], pe[2])
	var defined, undefined, undefinedButNonzero, definedWrong int
	for range cases {
		arr := randArray(rng, 1, 12, 0, 5)
		pp := prefixProduct(arr)
		l := rng.IntN(len(arr))
		r := l + 1 + rng.IntN(len(arr)-l)
		truth := bruteProduct(arr, l, r)
		got, ok := rangeProduct(pp, l, r)
		if ok {
			defined++
			if got != truth {
				definedWrong++
			}
		} else {
			undefined++
			if truth != 0 {
				undefinedButNonzero++
			}
		}
	}
	fmt.Printf("  range product P[r]/P[l], %d random queries (len 1..12, values 0..4):\n", cases)
	fmt.Printf("    defined (P[l] != 0): %d, wrong: %d\n", defined, definedWrong)
	fmt.Printf("    undefined (P[l] == 0, Go would panic on the divide): %d; for %d of them the true product is not 0\n", undefined, undefinedButNonzero)
	mustBe(definedWrong == 0, "defined products are exact")

	fmt.Println()
	fmt.Println("  max, min and gcd have no inverse at all, and a product loses its inverse at 0;")
	fmt.Println("  a window gets its max from a monotonic deque (lesson 09, example 10); arbitrary ranges")
	fmt.Println("  need a sparse table or a tree, neither built in this lesson.")
}
```

**Output:**

```
(a) prefix XOR: XOR is its own inverse
  a = [3 5 6 1 7 2]
  P = [0 3 6 0 1 6 4]
  a[1:4] XOR = 2,  P[4]^P[1] = 1^3 = 2
  a[0:6] XOR = 4,  P[6]^P[0] = 4^0 = 4
  a[3:3] XOR = 0,  P[3]^P[3] = 0^0 = 0

  count subarrays with XOR == k, 5000 arrays (len 0..15, values 0..7, k 0..7):
    map version vs brute force: 0 mismatches
    same, without counts[0] = 1: 2801 mismatches (every subarray that starts at index 0 is lost)
    work: 74950 map operations for 37475 elements; brute force ran 212275 XOR steps

(b) sum divisible by m, with negative numbers
  Go spec: x % y takes the sign of x (truncated division)
     5 %  3 =  2     (x / y =  1)
    -5 %  3 = -2     (x / y = -1)
     5 % -3 =  2     (x / y = -1)
    -5 % -3 = -2     (x / y =  1)

  5000 arrays (len 0..15, values -5..5, m 2..6), three ways to count:
    normalised ((x % m) + m) % m vs brute force: 0 mismatches
    raw x % m vs brute force:                      1765 mismatches (1765 too low, 0 too high)
    arrays with some negative prefix: 3412, and raw was wrong on 1765 of them;
    arrays whose prefixes are all >= 0: 1588, and raw was wrong on 0 of them

  first counterexample in the stream: a = [-3 5 1 5 0 -5 -4], m = 6
    prefixes         [0 -3 2 3 8 8 3 -1]
    raw residues     [0 -3 2 3 2 2 3 -1]
    normalised       [0 3 2 3 2 2 3 5]
    pairs counted: raw 4, normalised 6, brute force 6

  shortest counterexample in the stream: a = [-1 3], m = 3
    prefixes         [0 -1 2]
    raw residues     [0 -1 2]
    normalised       [0 2 2]
    pairs counted: raw 0, normalised 1, brute force 1

(c) what has no inverse
  range max from a prefix max, pm[r-1], on 20000 random (array, l, r) queries:
    l == 0: 5178 queries, 0 wrong;  l > 0: 14822 queries, 7744 wrong (52.2%)
    answers below the true max: 0 (it can only overshoot: it is the max of a longer range)

  prefix products of [2 0 3] are [1 2 0 0]: the last range a[2:3] = [3] would be 0 / 0
  range product P[r]/P[l], 5000 random queries (len 1..12, values 0..4):
    defined (P[l] != 0): 3178, wrong: 0
    undefined (P[l] == 0, Go would panic on the divide): 1822; for 1161 of them the true product is not 0

  max, min and gcd have no inverse at all, and a product loses its inverse at 0;
  a window gets its max from a monotonic deque (lesson 09, example 10); arbitrary ranges
  need a sparse table or a tree, neither built in this lesson.
```

**Rust:**

```rust
use std::collections::HashMap;
use std::process::exit;

// A prefix array answers a range query exactly when the operation can be UNDONE:
//
//     range(l, r) = P[r] "minus" P[l]
//
// + has an inverse (-), XOR is its own inverse, a count has one. max, min and
// gcd do not, and a product only does while nothing is zero. This program shows
// the three cases that work or almost work, then the ones that cannot. Every
// check runs against a brute-force oracle on PCG-seeded inputs, so the counts
// below are the same in every run -- and the same as the Go program's.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) gives the same stream as the Go program.
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

    fn below(&mut self, n: usize) -> usize {
        self.int_n(n as u64) as usize
    }
}

// must_be aborts the run if a claim the program is about to print would be false.
fn must_be(ok: bool, what: &str) {
    if !ok {
        println!("INTERNAL CHECK FAILED: {what}");
        exit(1);
    }
}

// Go prints a slice as [1 2 3]; Rust's {:?} prints [1, 2, 3].
fn go_slice(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

// rand_array draws a length in [min_len, max_len], then that many values in
// [lo, lo+span). The draw order is part of the contract: it is the Go program's.
fn rand_array(rng: &mut Pcg, min_len: usize, max_len: usize, lo: i64, span: usize) -> Vec<i64> {
    let n = min_len + rng.below(max_len - min_len + 1);
    (0..n).map(|_| lo + rng.below(span) as i64).collect()
}

// ---------- (a) prefix XOR ----------

// prefix_xor returns P with P[0] = 0 and P[i+1] = P[i] ^ a[i].
fn prefix_xor(a: &[i64]) -> Vec<i64> {
    std::iter::once(0)
        .chain(a.iter().scan(0, |x, &v| {
            *x ^= v;
            Some(*x)
        }))
        .collect()
}

// count_xor_map counts subarrays with XOR == k in one pass: a[l:r] has XOR k
// exactly when P[l] == P[r] ^ k, so look that prefix up among the earlier ones.
// seed is the counts[0] = 1 that stands for P[0]. ops counts map operations.
fn count_xor_map(a: &[i64], k: i64, seed: bool, ops: &mut usize) -> usize {
    let mut counts: HashMap<i64, usize> = HashMap::new();
    if seed {
        counts.insert(0, 1);
    }
    let (mut x, mut total) = (0, 0);
    for &v in a {
        x ^= v;
        total += counts.get(&(x ^ k)).copied().unwrap_or(0); // one lookup
        *counts.entry(x).or_insert(0) += 1; // one update
        *ops += 2;
    }
    total
}

// count_xor_brute tries every (l, r), extending a running XOR from each start.
fn count_xor_brute(a: &[i64], k: i64, steps: &mut usize) -> usize {
    let mut total = 0;
    for l in 0..a.len() {
        let mut x = 0;
        for &v in &a[l..] {
            x ^= v;
            *steps += 1;
            if x == k {
                total += 1;
            }
        }
    }
    total
}

// ---------- (b) sum divisible by m, with negatives ----------

// Rust's % truncates toward zero exactly like Go's, so it takes the sign of the
// DIVIDEND too: the bug below reproduces with the same counts.
fn raw_mod(x: i64, m: i64) -> i64 {
    x % m
}

// norm_mod is the fix. Rust has it built in as rem_euclid; Go's int does not.
fn norm_mod(x: i64, m: i64) -> i64 {
    x.rem_euclid(m)
}

// norm_formula is the Go program's one-liner, kept to check it against rem_euclid.
fn norm_formula(x: i64, m: i64) -> i64 {
    ((x % m) + m) % m
}

// count_divisible counts subarrays with sum % m == 0. Two prefixes with the same
// residue differ by a multiple of m, so every pair inside a residue class is one
// subarray: the answer is the sum over classes of C(c, 2). The only thing that
// differs between the two versions is the function that names the class.
fn count_divisible(a: &[i64], m: i64, modulo: fn(i64, i64) -> i64) -> usize {
    let mut classes: HashMap<i64, usize> = HashMap::new();
    let mut p = 0;
    *classes.entry(modulo(p, m)).or_insert(0) += 1; // P[0]
    for &v in a {
        p += v;
        *classes.entry(modulo(p, m)).or_insert(0) += 1;
    }
    classes.values().map(|&c| c * (c - 1) / 2).sum()
}

// brute_divisible tries every (l, r). `s % m == 0` is a correct divisibility
// test for either sign: only COMPARING two remainders goes wrong.
fn brute_divisible(a: &[i64], m: i64) -> usize {
    let mut total = 0;
    for l in 0..a.len() {
        let mut s = 0;
        for &v in &a[l..] {
            s += v;
            if s % m == 0 {
                total += 1;
            }
        }
    }
    total
}

// ---------- (c) what has no inverse ----------

// prefix_max returns pm with pm[i] = max(a[0..=i]). It is all a prefix can store.
fn prefix_max(a: &[i64]) -> Vec<i64> {
    a.iter()
        .scan(i64::MIN, |best, &v| {
            *best = (*best).max(v);
            Some(*best)
        })
        .collect()
}

fn range_max(a: &[i64], l: usize, r: usize) -> i64 {
    a[l..r].iter().copied().max().unwrap()
}

// prefix_product returns P with P[0] = 1 and P[i+1] = P[i] * a[i].
fn prefix_product(a: &[i64]) -> Vec<i64> {
    std::iter::once(1)
        .chain(a.iter().scan(1, |p, &v| {
            *p *= v;
            Some(*p)
        }))
        .collect()
}

// range_product is P[r] / P[l]. Once a zero has entered the prefix every later
// entry is 0. Dividing by it panics in Rust as in Go, but checked_div turns
// that into None: the type says "undefined" and there is nothing to forget.
fn range_product(p: &[i64], l: usize, r: usize) -> Option<i64> {
    p[r].checked_div(p[l])
}

fn brute_product(a: &[i64], l: usize, r: usize) -> i64 {
    a[l..r].iter().product()
}

struct Counter {
    id: usize,
    a: Vec<i64>,
    m: i64,
    raw: usize,
    norm: usize,
    brute: usize,
    prefixes: Vec<i64>,
}

fn show(label: &str, c: &Counter) {
    println!("\n  {label}: a = {}, m = {}", go_slice(&c.a), c.m);
    println!("    prefixes         {}", go_slice(&c.prefixes));
    let raw_res: Vec<i64> = c.prefixes.iter().map(|&s| raw_mod(s, c.m)).collect();
    let norm_res: Vec<i64> = c.prefixes.iter().map(|&s| norm_mod(s, c.m)).collect();
    println!("    raw residues     {}", go_slice(&raw_res));
    println!("    normalised       {}", go_slice(&norm_res));
    println!(
        "    pairs counted: raw {}, normalised {}, brute force {}",
        c.raw, c.norm, c.brute
    );
}

fn main() {
    let mut rng = Pcg::new(1, 2);

    // ---- (a) ----
    println!("(a) prefix XOR: XOR is its own inverse");
    let a = [3, 5, 6, 1, 7, 2];
    let p = prefix_xor(&a);
    println!("  a = {}", go_slice(&a));
    println!("  P = {}", go_slice(&p));
    for (l, r) in [(1, 4), (0, 6), (3, 3)] {
        let direct = a[l..r].iter().fold(0, |x, &v| x ^ v);
        println!(
            "  a[{l}:{r}] XOR = {direct},  P[{r}]^P[{l}] = {}^{} = {}",
            p[r],
            p[l],
            p[r] ^ p[l]
        );
        must_be(direct == p[r] ^ p[l], "range XOR");
    }

    const CASES: usize = 5000;
    let (mut elems, mut map_ops, mut brute_steps, mut bad, mut bad_no_seed) = (0, 0, 0, 0, 0);
    for _ in 0..CASES {
        let arr = rand_array(&mut rng, 0, 15, 0, 8);
        let k = rng.below(8) as i64;
        elems += arr.len();
        let want = count_xor_brute(&arr, k, &mut brute_steps);
        if count_xor_map(&arr, k, true, &mut map_ops) != want {
            bad += 1;
        }
        let mut unused = 0;
        if count_xor_map(&arr, k, false, &mut unused) != want {
            bad_no_seed += 1;
        }
    }
    println!("\n  count subarrays with XOR == k, {CASES} arrays (len 0..15, values 0..7, k 0..7):");
    println!("    map version vs brute force: {bad} mismatches");
    println!(
        "    same, without counts[0] = 1: {bad_no_seed} mismatches (every subarray that starts at index 0 is lost)"
    );
    println!(
        "    work: {map_ops} map operations for {elems} elements; brute force ran {brute_steps} XOR steps"
    );
    must_be(bad == 0, "XOR map count");
    must_be(map_ops == 2 * elems, "two map operations per element");

    // ---- (b) ----
    println!();
    println!("(b) sum divisible by m, with negative numbers");
    println!("  In Rust, too, x % y takes the sign of x (rem_euclid does not)");
    for (x, y) in [(5i64, 3i64), (-5, 3), (5, -3), (-5, -3)] {
        println!(
            "    {x:>2} % {y:>2} = {:>2}     (x / y = {:>2})     rem_euclid = {}",
            x % y,
            x / y,
            x.rem_euclid(y)
        );
    }

    let mut first: Option<Counter> = None;
    let mut shortest: Option<Counter> = None;
    let (mut wrong_raw, mut wrong_norm, mut under, mut over) = (0, 0, 0, 0);
    let (mut neg_cases, mut neg_wrong, mut non_neg_wrong) = (0, 0, 0);
    let (mut prefixes_checked, mut formula_differs) = (0, 0);
    for id in 0..CASES {
        let arr = rand_array(&mut rng, 0, 15, -5, 11);
        let m = 2 + rng.below(5) as i64;
        let raw = count_divisible(&arr, m, raw_mod);
        let norm = count_divisible(&arr, m, norm_mod);
        let brute = brute_divisible(&arr, m);
        let mut sums = vec![0i64];
        for &v in &arr {
            sums.push(sums[sums.len() - 1] + v);
        }
        let any_neg = sums.iter().any(|&s| s < 0);
        for &s in &sums {
            prefixes_checked += 1;
            if norm_formula(s, m) != norm_mod(s, m) {
                formula_differs += 1;
            }
        }
        if norm != brute {
            wrong_norm += 1;
        }
        if any_neg {
            neg_cases += 1;
        }
        if raw == brute {
            continue;
        }
        wrong_raw += 1;
        if any_neg {
            neg_wrong += 1;
        } else {
            non_neg_wrong += 1;
        }
        if raw < brute {
            under += 1;
        } else {
            over += 1;
        }
        let make = || Counter {
            id,
            a: arr.clone(),
            m,
            raw,
            norm,
            brute,
            prefixes: sums.clone(),
        };
        if first.is_none() {
            first = Some(make());
        }
        if shortest.as_ref().is_none_or(|s| arr.len() < s.a.len()) {
            shortest = Some(make());
        }
    }
    println!("\n  {CASES} arrays (len 0..15, values -5..5, m 2..6), three ways to count:");
    println!("    normalised rem_euclid vs brute force:           {wrong_norm} mismatches");
    println!(
        "    raw x % m vs brute force:                      {wrong_raw} mismatches ({under} too low, {over} too high)"
    );
    println!(
        "    arrays with some negative prefix: {neg_cases}, and raw was wrong on {neg_wrong} of them;"
    );
    println!(
        "    arrays whose prefixes are all >= 0: {}, and raw was wrong on {non_neg_wrong} of them",
        CASES - neg_cases
    );
    println!(
        "    rem_euclid vs ((x % m) + m) % m on all {prefixes_checked} prefixes: {formula_differs} differ"
    );
    must_be(wrong_norm == 0, "normalised residues");
    must_be(
        over == 0 && non_neg_wrong == 0,
        "raw only undercounts, and only with a negative prefix",
    );
    must_be(formula_differs == 0, "rem_euclid is the Go formula");
    let (first, shortest) = (first.unwrap(), shortest.unwrap());
    show("first counterexample in the stream", &first);
    if shortest.id != first.id {
        show("shortest counterexample in the stream", &shortest);
    }

    // ---- (c) ----
    println!();
    println!("(c) what has no inverse");
    const QUERIES: usize = 20000;
    let mut total = [0usize; 2]; // index 0: queries with l == 0, index 1: with l > 0
    let mut wrong = [0usize; 2];
    let mut too_low = 0;
    for _ in 0..QUERIES {
        let arr = rand_array(&mut rng, 1, 12, 0, 10);
        let l = rng.below(arr.len());
        let r = l + 1 + rng.below(arr.len() - l);
        // all a prefix can offer for a[l:r]
        let (guess, truth) = (prefix_max(&arr)[r - 1], range_max(&arr, l, r));
        let side = l.min(1);
        total[side] += 1;
        if guess != truth {
            wrong[side] += 1;
        }
        if guess < truth {
            too_low += 1;
        }
    }
    println!("  range max from a prefix max, pm[r-1], on {QUERIES} random (array, l, r) queries:");
    println!(
        "    l == 0: {} queries, {} wrong;  l > 0: {} queries, {} wrong ({:.1}%)",
        total[0],
        wrong[0],
        total[1],
        wrong[1],
        100.0 * wrong[1] as f64 / total[1] as f64
    );
    println!(
        "    answers below the true max: {too_low} (it can only overshoot: it is the max of a longer range)"
    );
    must_be(wrong[0] == 0 && too_low == 0 && wrong[1] > 0, "prefix max");

    println!();
    let ex = [2, 0, 3];
    let pe = prefix_product(&ex);
    println!(
        "  prefix products of {} are {}: the last range a[2:3] = [3] would be {} / {}",
        go_slice(&ex),
        go_slice(&pe),
        pe[3],
        pe[2]
    );
    let (mut defined, mut undefined, mut undefined_but_nonzero, mut defined_wrong) = (0, 0, 0, 0);
    for _ in 0..CASES {
        let arr = rand_array(&mut rng, 1, 12, 0, 5);
        let pp = prefix_product(&arr);
        let l = rng.below(arr.len());
        let r = l + 1 + rng.below(arr.len() - l);
        let truth = brute_product(&arr, l, r);
        match range_product(&pp, l, r) {
            Some(got) => {
                defined += 1;
                if got != truth {
                    defined_wrong += 1;
                }
            }
            None => {
                undefined += 1;
                if truth != 0 {
                    undefined_but_nonzero += 1;
                }
            }
        }
    }
    println!("  range product P[r]/P[l], {CASES} random queries (len 1..12, values 0..4):");
    println!("    defined (P[l] != 0): {defined}, wrong: {defined_wrong}");
    println!(
        "    undefined (P[l] == 0, checked_div returns None): {undefined}; for {undefined_but_nonzero} of them the true product is not 0"
    );
    must_be(defined_wrong == 0, "defined products are exact");

    println!();
    println!("  max, min and gcd have no inverse at all, and a product loses its inverse at 0;");
    println!(
        "  a window gets its max from a monotonic deque (lesson 09, example 10); arbitrary ranges"
    );
    println!("  need a sparse table or a tree, neither built in this lesson.");
}
```

**Rust output:**

```
(a) prefix XOR: XOR is its own inverse
  a = [3 5 6 1 7 2]
  P = [0 3 6 0 1 6 4]
  a[1:4] XOR = 2,  P[4]^P[1] = 1^3 = 2
  a[0:6] XOR = 4,  P[6]^P[0] = 4^0 = 4
  a[3:3] XOR = 0,  P[3]^P[3] = 0^0 = 0

  count subarrays with XOR == k, 5000 arrays (len 0..15, values 0..7, k 0..7):
    map version vs brute force: 0 mismatches
    same, without counts[0] = 1: 2801 mismatches (every subarray that starts at index 0 is lost)
    work: 74950 map operations for 37475 elements; brute force ran 212275 XOR steps

(b) sum divisible by m, with negative numbers
  In Rust, too, x % y takes the sign of x (rem_euclid does not)
     5 %  3 =  2     (x / y =  1)     rem_euclid = 2
    -5 %  3 = -2     (x / y = -1)     rem_euclid = 1
     5 % -3 =  2     (x / y = -1)     rem_euclid = 2
    -5 % -3 = -2     (x / y =  1)     rem_euclid = 1

  5000 arrays (len 0..15, values -5..5, m 2..6), three ways to count:
    normalised rem_euclid vs brute force:           0 mismatches
    raw x % m vs brute force:                      1765 mismatches (1765 too low, 0 too high)
    arrays with some negative prefix: 3412, and raw was wrong on 1765 of them;
    arrays whose prefixes are all >= 0: 1588, and raw was wrong on 0 of them
    rem_euclid vs ((x % m) + m) % m on all 42073 prefixes: 0 differ

  first counterexample in the stream: a = [-3 5 1 5 0 -5 -4], m = 6
    prefixes         [0 -3 2 3 8 8 3 -1]
    raw residues     [0 -3 2 3 2 2 3 -1]
    normalised       [0 3 2 3 2 2 3 5]
    pairs counted: raw 4, normalised 6, brute force 6

  shortest counterexample in the stream: a = [-1 3], m = 3
    prefixes         [0 -1 2]
    raw residues     [0 -1 2]
    normalised       [0 2 2]
    pairs counted: raw 0, normalised 1, brute force 1

(c) what has no inverse
  range max from a prefix max, pm[r-1], on 20000 random (array, l, r) queries:
    l == 0: 5178 queries, 0 wrong;  l > 0: 14822 queries, 7744 wrong (52.2%)
    answers below the true max: 0 (it can only overshoot: it is the max of a longer range)

  prefix products of [2 0 3] are [1 2 0 0]: the last range a[2:3] = [3] would be 0 / 0
  range product P[r]/P[l], 5000 random queries (len 1..12, values 0..4):
    defined (P[l] != 0): 3178, wrong: 0
    undefined (P[l] == 0, checked_div returns None): 1822; for 1161 of them the true product is not 0

  max, min and gcd have no inverse at all, and a product loses its inverse at 0;
  a window gets its max from a monotonic deque (lesson 09, example 10); arbitrary ranges
  need a sparse table or a tree, neither built in this lesson.
```

> **Go vs Rust:** The counts match the Go run line for line (same PCG stream: 2,801 / 1,765 / 7,744 / 1,822), because Rust's `%` truncates exactly like Go's, so the raw-residue bug reproduces with the same counterexamples. Only the fix differs: Rust has `rem_euclid` built in (the run shows it agrees with Go's `((x % m) + m) % m` on all 42,073 prefixes), while Go's `int` has none. `checked_div` returns `None` for the zero prefix where Go must test `p[l] == 0` before dividing, which makes "undefined" a value the type forces you to handle.

**Complexity:** prefix XOR and prefix sum are Θ(n) to build and Θ(1) per range; counting by map or by residue class is Θ(n) time (74,950 map operations for 37,475 elements, against 212,275 brute-force XOR steps) and Θ(min(n, m)) space for residues. The XOR map version had 0 mismatches over 5,000 arrays, and dropping `counts[0] = 1` lost 2,801 of them. Raw `x % m` was wrong on 1,765 of 5,000 arrays, always too low and only when a prefix was negative (3,412 such arrays; 0 wrong among the 1,588 with all prefixes >= 0), and the shortest counterexample is `[-1 3]` with m = 3; the normalised residue had 0 mismatches. A prefix max returned a wrong range max on 7,744 of 14,822 queries with `l > 0` (never on `l == 0`, never too low), and a prefix product was undefined on 1,822 of 5,000 queries, 1,161 of which have a non-zero true product.

---

## 16. Capstone: the toolkit, built, proved, measured

`🔴 hard` · *Applied mini-build*

The lesson as one small toolkit in one `package main`: `PairSum` (converging pointers), `DedupeAtMost` (read/write pointers), `Partition3` (the Dutch national flag), `Prefix` (1-D, half-open, zero slot, rejects reversed ranges instead of wrapping), `Prefix2D`, `Diff` (range add in two writes), and `LongestValid`, one sliding window driven by `add` / `remove` / `valid` callbacks that also counts valid windows. Every tool returns the work it did, so the proof can assert the bound it claims (n − 1 steps, n reads, exactly n iterations, 2n moves, 2 or 4 reads per query) instead of trusting it, against a brute-force oracle: exhaustively on every small input and on PCG-seeded random ones, plus an overflow row with values near 2⁵⁹, where a range whose true sum fits must still be exact. Then one timing table puts each technique next to its brute-force version on the same input at n = 10³, 10⁴ and 10⁵, with the operation counts beside the nanoseconds. This is a capstone, so the Go program is much longer than the size guide for an example, and the timings change from run to run.

**Steps:**

1. Read the toolkit: each tool is a short function, each returns its step count, and `LongestValid` takes its state through three closures. The first section calls each one once.
2. Read the proof table: `cases`, `mismatches` against the oracle, the bound asserted, and the worst observed value minus the bound (0 means the bound is reached exactly, -1 means it never is). Then read the overflow lines.
3. Read the timing table: ns and operations per task for brute force and for the technique, and how the ratio changes as n grows. Rows where brute force would need more than 10⁹ operations are not run.

```go
package main

import (
	"fmt"
	"math"
	"math/big"
	"math/rand/v2"
	"os"
	"slices"
	"time"
)

// The lesson as one small toolkit, then proved, then measured:
//
//	PairSum       converging pointers on a sorted slice
//	DedupeAtMost  read/write pointers, keep at most k copies
//	Partition3    the Dutch national flag
//	Prefix        1-D prefix sums, half-open ranges, a zero slot
//	Prefix2D      2-D prefix sums, inclusion-exclusion
//	Diff          difference array: range add in Theta(1)
//	LongestValid  one sliding window, driven by add / remove / valid callbacks
//
// Every function returns the work it did (steps, reads, moves), so the proof can
// assert the bound each one claims instead of trusting it. This is a capstone, so
// the program is longer than the other examples.

// mustBe aborts the run if a claim the program is about to print would be false.
func mustBe(ok bool, what string) {
	if !ok {
		fmt.Println("INTERNAL CHECK FAILED:", what)
		os.Exit(1)
	}
}

// ======================= 1. THE TOOLKIT =======================

// PairSum finds i < j with a[i]+a[j] == target in a SORTED slice. steps counts
// loop iterations: each one moves a pointer, so there are at most n-1 of them.
func PairSum(a []int, target int) (i, j int, ok bool, steps int) {
	lo, hi := 0, len(a)-1
	for lo < hi { // not <=: an element must not pair with itself
		steps++
		switch s := a[lo] + a[hi]; {
		case s == target:
			return lo, hi, true, steps
		case s < target:
			lo++ // a[lo] cannot pair with anything left of hi
		default:
			hi-- // a[hi] cannot pair with anything right of lo
		}
	}
	return 0, 0, false, steps
}

// DedupeAtMost keeps at most k copies of each value of a sorted slice, in place,
// and returns the new length. It compares with a[write-k], k places back in the
// OUTPUT. reads counts elements read: exactly len(a). The tail is left stale.
func DedupeAtMost(a []int, k int) (n, reads int) {
	if k < 1 {
		return 0, 0
	}
	write := 0
	for _, v := range a {
		reads++
		if write < k || a[write-k] != v {
			a[write] = v
			write++
		}
	}
	return write, reads
}

// Partition3 sorts a slice of 0s, 1s and 2s in one pass. Regions: [0,lo) zeros,
// [lo,mid) ones, [mid,hi) unknown, [hi,n) twos. Every iteration shrinks the unknown
// region by one, so iterations == len(a). After the swap with hi, mid stays put:
// the element that arrived has not been looked at.
func Partition3(a []int) (iterations int) {
	lo, mid, hi := 0, 0, len(a)
	for mid < hi {
		iterations++
		switch a[mid] {
		case 0:
			a[lo], a[mid] = a[mid], a[lo]
			lo++
			mid++
		case 1:
			mid++
		default:
			hi--
			a[mid], a[hi] = a[hi], a[mid]
		}
	}
	return iterations
}

// Prefix holds P with P[0] = 0 and P[i+1] = P[i] + a[i]. Go's int wraps silently,
// so P may wrap; a range whose true sum fits is still exact (see the overflow row).
type Prefix struct {
	p     []int
	reads int // reads of p made by Range
}

func NewPrefix(a []int) *Prefix {
	p := make([]int, len(a)+1)
	for i, v := range a {
		p[i+1] = p[i] + v
	}
	return &Prefix{p: p}
}

// Range returns the sum of a[l:r] (half-open). A reversed or out-of-range pair is
// rejected instead of returning garbage.
func (pf *Prefix) Range(l, r int) (int, bool) {
	if l < 0 || l > r || r >= len(pf.p) {
		return 0, false
	}
	pf.reads += 2
	return pf.p[r] - pf.p[l], true
}

// Prefix2D is the 2-D table, flattened: P[i+1][j+1] = g[i][j] + P[i][j+1] + P[i+1][j] - P[i][j].
type Prefix2D struct {
	p          []int
	rows, cols int
	reads      int
}

func NewPrefix2D(g [][]int) *Prefix2D {
	rows, cols := len(g), 0
	if rows > 0 {
		cols = len(g[0])
	}
	w := cols + 1
	p := make([]int, (rows+1)*w)
	for i := range rows {
		for j := range cols {
			p[(i+1)*w+j+1] = g[i][j] + p[i*w+j+1] + p[(i+1)*w+j] - p[i*w+j]
		}
	}
	return &Prefix2D{p: p, rows: rows, cols: cols}
}

// Rect returns the sum over rows [r1,r2) and columns [c1,c2): four reads.
func (t *Prefix2D) Rect(r1, c1, r2, c2 int) (int, bool) {
	if r1 < 0 || r1 > r2 || r2 > t.rows || c1 < 0 || c1 > c2 || c2 > t.cols {
		return 0, false
	}
	w := t.cols + 1
	t.reads += 4
	return t.p[r2*w+c2] - t.p[r1*w+c2] - t.p[r2*w+c1] + t.p[r1*w+c1], true
}

// Diff records "add v to a[l:r]" as two writes; Values does the one prefix sum.
type Diff struct {
	d      []int
	writes int
}

func NewDiff(n int) *Diff { return &Diff{d: make([]int, n+1)} }

func (df *Diff) Add(l, r, v int) bool {
	if l < 0 || l > r || r >= len(df.d) {
		return false
	}
	df.d[l] += v
	df.d[r] -= v
	df.writes += 2
	return true
}

func (df *Diff) Values() []int {
	out := make([]int, len(df.d)-1)
	run := 0
	for i := range out {
		run += df.d[i]
		out[i] = run
	}
	return out
}

// DiffOf is the forward difference of a: the inverse of a prefix sum.
func DiffOf(a []int) []int {
	d := make([]int, len(a))
	for i, v := range a {
		d[i] = v
		if i > 0 {
			d[i] -= a[i-1]
		}
	}
	return d
}

// Windows is what one pass of LongestValid learns.
type Windows struct {
	Longest, Start int // longest valid window (earliest if tied)
	Count          int // number of valid windows (subarrays)
	Moves          int // pointer moves: at most 2n
}

// LongestValid slides one window over xs. The callbacks hold the state: add and
// remove update it, valid reads it in O(1). The predicate must be monotone: a valid
// window stays valid when it shrinks (at most k distinct, sum <= limit over
// non-negatives, no repeats). Then every start in [left, right] is valid for this
// right, so Count adds right-left+1 per step.
func LongestValid[T any](xs []T, add, remove func(T), valid func() bool) Windows {
	var w Windows
	left := 0
	for right, x := range xs {
		add(x)
		w.Moves++
		for left <= right && !valid() {
			remove(xs[left])
			left++
			w.Moves++
		}
		w.Count += right - left + 1
		if right-left+1 > w.Longest {
			w.Longest, w.Start = right-left+1, left
		}
	}
	return w
}

// atMostDistinct runs LongestValid with the "at most k distinct bytes" predicate.
func atMostDistinct(s []byte, k int) Windows {
	var counts [256]int
	distinct := 0
	return LongestValid(s,
		func(c byte) {
			counts[c]++
			if counts[c] == 1 {
				distinct++
			}
		},
		func(c byte) {
			counts[c]--
			if counts[c] == 0 {
				distinct--
			}
		},
		func() bool { return distinct <= k })
}

// sumAtMost runs LongestValid with "sum <= limit" over non-negative ints.
func sumAtMost(a []int, limit int) Windows {
	sum := 0
	return LongestValid(a,
		func(v int) { sum += v },
		func(v int) { sum -= v },
		func() bool { return sum <= limit })
}

// ======================= 2. THE ORACLES =======================

func hasPair(a []int, target int) bool {
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i]+a[j] == target {
				return true
			}
		}
	}
	return false
}

func bruteSum(a []int, l, r int) int {
	s := 0
	for _, v := range a[l:r] {
		s += v
	}
	return s
}

// keepAtMost: the first k copies of each value (right for sorted input).
func keepAtMost(a []int, k int) []int {
	seen := map[int]int{}
	var out []int
	for _, v := range a {
		if seen[v] < k {
			out = append(out, v)
		}
		seen[v]++
	}
	return out
}

// bruteWindows tries every (l, r): longest and count of windows with <= k distinct bytes.
func bruteWindows(s []byte, k int) (longest, start, count int) {
	for l := range s {
		var seen [256]bool
		distinct := 0
		for r := l; r < len(s); r++ {
			if !seen[s[r]] {
				seen[s[r]] = true
				distinct++
			}
			if distinct > k {
				break
			}
			count++
			if r-l+1 > longest {
				longest, start = r-l+1, l
			}
		}
	}
	return longest, start, count
}

func bruteRect(g [][]int, r1, c1, r2, c2 int) int {
	s := 0
	for i := r1; i < r2; i++ {
		for j := c1; j < c2; j++ {
			s += g[i][j]
		}
	}
	return s
}

// each calls f with a copy of every array of length <= maxLen over vals (only
// non-decreasing ones when sorted is true).
func each(maxLen int, vals []int, sorted bool, f func([]int)) {
	var rec func(a []int)
	rec = func(a []int) {
		f(slices.Clone(a))
		if len(a) == maxLen {
			return
		}
		for _, v := range vals {
			if sorted && len(a) > 0 && v < a[len(a)-1] {
				continue
			}
			rec(append(a, v))
		}
	}
	rec(nil)
}

// randInts draws a length in [minLen, maxLen], then that many values in [lo, lo+span).
// The draw order is part of the contract: the Rust twin makes the same calls.
func randInts(rng *rand.Rand, minLen, maxLen, lo, span int) []int {
	a := make([]int, minLen+rng.IntN(maxLen-minLen+1))
	for i := range a {
		a[i] = lo + rng.IntN(span)
	}
	return a
}

// ifValid is the work a query should have cost: n if it was accepted, 0 if rejected.
func ifValid(valid bool, n int) int {
	if valid {
		return n
	}
	return 0
}

// tally is one row of the proof table. Each check records whether the answer was
// right and how far the observed work was from the bound it claims.
type tally struct {
	name, bound string
	cases, bad  int
	lo, hi      int // min and max of (observed - bound)
}

func newTally(name, bound string) *tally {
	return &tally{name: name, bound: bound, lo: math.MaxInt, hi: math.MinInt}
}

func (t *tally) check(good bool, observed, bound int) {
	t.cases++
	if !good {
		t.bad++
	}
	t.lo, t.hi = min(t.lo, observed-bound), max(t.hi, observed-bound)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	smoke()
	prove(rng)
	measureAll(rng)
}

// smoke makes one call to each tool, so the API is visible before the proof.
func smoke() {
	fmt.Println("1. the toolkit, one call each")
	say := func(call, format string, args ...any) {
		fmt.Printf("  %-37s -> %s\n", call, fmt.Sprintf(format, args...))
	}
	i, j, ok, steps := PairSum([]int{1, 3, 4, 6, 9}, 12)
	say("PairSum([1 3 4 6 9], 12)", "i=%d j=%d ok=%v after %d steps", i, j, ok, steps)
	dd := []int{1, 1, 1, 2, 2, 3}
	n, reads := DedupeAtMost(dd, 2)
	say("DedupeAtMost([1 1 1 2 2 3], 2)", "%v after %d reads, stale tail %v", dd[:n], reads, dd[n:])
	fl := []int{2, 0, 1, 2, 0, 1}
	it := Partition3(fl)
	say("Partition3([2 0 1 2 0 1])", "%v in %d iterations", fl, it)
	pf := NewPrefix([]int{3, 1, 4, 1, 5})
	sum, _ := pf.Range(1, 4)
	_, accepted := pf.Range(4, 1)
	say("Prefix([3 1 4 1 5]).Range", "[1:4] = %d, [4:1] accepted = %v", sum, accepted)
	p2 := NewPrefix2D([][]int{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}})
	sum, _ = p2.Rect(1, 1, 3, 3)
	say("Prefix2D(1..9).Rect(1, 1, 3, 3)", "%d", sum)
	df := NewDiff(6)
	df.Add(1, 4, 5)
	df.Add(3, 6, 2)
	say("Diff(6): +5 on [1:4], +2 on [3:6]", "%v with %d writes", df.Values(), df.writes)
	w := atMostDistinct([]byte("eceba"), 2)
	say("LongestValid(\"eceba\", <= 2 distinct)", "longest %d at %d, %d valid windows, %d moves", w.Longest, w.Start, w.Count, w.Moves)
	fmt.Println()
}

// prove checks every tool against its oracle and asserts the bound it claims.
func prove(rng *rand.Rand) {
	pair := newTally("PairSum", "steps <= n-1")
	dedupe := newTally("DedupeAtMost (k 1..3)", "reads == n")
	flag := newTally("Partition3", "iterations == n")
	prefix := newTally("Prefix.Range", "2 reads per accepted query")
	diff := newTally("Diff.Add / Values", "2 writes per Add")
	inverse := newTally("diff <-> prefix inverses", "none (equality)")
	rect := newTally("Prefix2D.Rect", "4 reads per accepted query")
	window := newTally("LongestValid, <= k distinct", "moves <= 2n")
	wrap := newTally("Prefix near 2^59 (wraps)", "exact if the true sum fits")

	// PairSum: every sorted array of length <= 7 over {0..4} x every target, then PCG cases.
	noPair, noPairTight := 0, 0
	pairCase := func(a []int, target int) {
		i, j, ok, steps := PairSum(a, target)
		want := hasPair(a, target)
		good := ok == want && (!ok || (i < j && a[i]+a[j] == target))
		pair.check(good, steps, max(len(a)-1, 0))
		if !want && len(a) >= 1 {
			noPair++
			if steps == len(a)-1 {
				noPairTight++
			}
		}
	}
	each(7, []int{0, 1, 2, 3, 4}, true, func(a []int) {
		for target := 0; target <= 8; target++ {
			pairCase(a, target)
		}
	})
	for range 20000 {
		a := randInts(rng, 0, 30, 0, 20)
		slices.Sort(a)
		pairCase(a, rng.IntN(40))
	}

	// DedupeAtMost: every sorted array of length <= 7 over {0..3} x k, then PCG cases.
	dedupeCase := func(a []int, k int) {
		want := keepAtMost(a, k)
		got := slices.Clone(a)
		n, reads := DedupeAtMost(got, k)
		good := slices.Equal(got[:n], want)
		if k == 1 { // at k = 1 this is exactly slices.Compact
			good = good && slices.Equal(got[:n], slices.Compact(slices.Clone(a)))
		}
		dedupe.check(good, reads, len(a))
	}
	each(7, []int{0, 1, 2, 3}, true, func(a []int) {
		for k := 1; k <= 3; k++ {
			dedupeCase(a, k)
		}
	})
	for range 20000 {
		a := randInts(rng, 0, 30, 0, 6)
		slices.Sort(a)
		dedupeCase(a, 1+rng.IntN(3))
	}

	// Partition3: every array of length <= 8 over {0,1,2}.
	each(8, []int{0, 1, 2}, false, func(a []int) {
		want := slices.Sorted(slices.Values(a))
		it := Partition3(a)
		flag.check(slices.Equal(a, want), it, len(a))
	})

	// Prefix and Diff: every array of length <= 6 over {-1,0,1}, every (l, r) including
	// reversed and out-of-range ones, which must be rejected.
	each(6, []int{-1, 0, 1}, false, func(a []int) {
		n := len(a)
		pf := NewPrefix(a)
		for l := -1; l <= n+1; l++ {
			for r := -1; r <= n+1; r++ {
				valid := 0 <= l && l <= r && r <= n
				before := pf.reads
				got, ok := pf.Range(l, r)
				good := ok == valid && (!ok || got == bruteSum(a, l, r))
				prefix.check(good, pf.reads-before, ifValid(valid, 2))
			}
		}
		// prefix(diff(a)) == a and diff(prefix(a)) == a
		back := NewPrefix(DiffOf(a)).p[1:]
		inverse.check(slices.Equal(back, a), 0, 0)
		inverse.check(slices.Equal(DiffOf(NewPrefix(a).p[1:]), a), 0, 0)
	})
	// Diff: every (l, r, v) on arrays of length 0..6, including rejected pairs, then 5,000
	// random sequences of range adds against the naive loops.
	for n := 0; n <= 6; n++ {
		for l := -1; l <= n+1; l++ {
			for r := -1; r <= n+1; r++ {
				for v := -2; v <= 2; v++ {
					valid := 0 <= l && l <= r && r <= n
					df := NewDiff(n)
					ok := df.Add(l, r, v)
					want := make([]int, n)
					if valid {
						for x := l; x < r; x++ {
							want[x] += v
						}
					}
					diff.check(ok == valid && slices.Equal(df.Values(), want), df.writes, ifValid(valid, 2))
				}
			}
		}
	}
	for range 5000 {
		n := 1 + rng.IntN(20)
		updates := 1 + rng.IntN(10)
		df, want := NewDiff(n), make([]int, n)
		for range updates {
			l := rng.IntN(n + 1)
			r := l + rng.IntN(n+1-l)
			v := rng.IntN(11) - 5
			df.Add(l, r, v)
			for x := l; x < r; x++ {
				want[x] += v
			}
		}
		diff.check(slices.Equal(df.Values(), want), df.writes, 2*updates)
	}

	// Prefix2D: every rectangle of 20 random 5x5 grids (reversed ones must be rejected),
	// then 2,000 random rectangles of a 30x30 grid.
	rectCase := func(g [][]int, t *Prefix2D, r1, c1, r2, c2 int) {
		valid := r1 <= r2 && c1 <= c2
		before := t.reads
		got, ok := t.Rect(r1, c1, r2, c2)
		good := ok == valid && (!ok || got == bruteRect(g, r1, c1, r2, c2))
		rect.check(good, t.reads-before, ifValid(valid, 4))
	}
	randGrid := func(side, span int) [][]int {
		g := make([][]int, side)
		for i := range g {
			g[i] = make([]int, side)
			for j := range g[i] {
				g[i][j] = rng.IntN(span) - span/2
			}
		}
		return g
	}
	for range 20 {
		g := randGrid(5, 7)
		t := NewPrefix2D(g)
		for r1 := 0; r1 <= 5; r1++ {
			for r2 := 0; r2 <= 5; r2++ {
				for c1 := 0; c1 <= 5; c1++ {
					for c2 := 0; c2 <= 5; c2++ {
						rectCase(g, t, r1, c1, r2, c2)
					}
				}
			}
		}
	}
	g30 := randGrid(30, 19)
	t30 := NewPrefix2D(g30)
	for range 2000 {
		r1 := rng.IntN(31)
		r2 := r1 + rng.IntN(31-r1)
		c1 := rng.IntN(31)
		c2 := c1 + rng.IntN(31-c1)
		rectCase(g30, t30, r1, c1, r2, c2)
	}

	// LongestValid: 20,000 random strings of length 1..40, at most k distinct, against the O(n^2) oracle.
	for range 20000 {
		n := 1 + rng.IntN(40)
		sigma := 1 + rng.IntN(5)
		s := make([]byte, n)
		for i := range s {
			s[i] = byte('a' + rng.IntN(sigma))
		}
		k := 1 + rng.IntN(3)
		w := atMostDistinct(s, k)
		longest, start, count := bruteWindows(s, k)
		window.check(w.Longest == longest && w.Start == start && w.Count == count, w.Moves, 2*n)
	}

	// Overflow: values near 2^59 wrap the prefix at P[16]. A range whose true sum fits
	// in int64 must still be exact (math/big is the oracle); a reversed range is rejected.
	firstNegLo, firstNegHi, longestExact, notFit, notFitDiffer, reversed := math.MaxInt, 0, 0, 0, 0, 0
	for range 200 {
		a := make([]int, 20)
		for i := range a {
			a[i] = 1<<59 + rng.IntN(1<<20)
		}
		pf := NewPrefix(a)
		for i, v := range pf.p {
			if v < 0 {
				firstNegLo, firstNegHi = min(firstNegLo, i), max(firstNegHi, i)
				break
			}
		}
		for l := 0; l <= 20; l++ {
			trueSum := new(big.Int)
			for r := l; r <= 20; r++ {
				if r > l {
					trueSum.Add(trueSum, big.NewInt(int64(a[r-1])))
				}
				got, ok := pf.Range(l, r)
				if trueSum.IsInt64() {
					wrap.check(ok && int64(got) == trueSum.Int64(), 0, 0)
					longestExact = max(longestExact, r-l)
				} else {
					notFit++
					if big.NewInt(int64(got)).Cmp(trueSum) != 0 {
						notFitDiffer++
					}
				}
			}
			for r := 0; r < l; r++ {
				_, ok := pf.Range(l, r)
				wrap.check(!ok, 0, 0)
				reversed++
			}
		}
	}

	mustBe(firstNegLo != math.MaxInt, "every array wrapped")
	fmt.Println("2. proof against brute-force oracles (PCG(1, 2); every bound is asserted, not just printed)")
	fmt.Printf("  %-28s %8s %11s  %-27s %s\n", "operation", "cases", "mismatches", "bound asserted", "worst (observed - bound)")
	total, totalBad := 0, 0
	for _, t := range []*tally{pair, dedupe, flag, prefix, diff, inverse, rect, window, wrap} {
		fmt.Printf("  %-28s %8d %11d  %-27s %d\n", t.name, t.cases, t.bad, t.bound, t.hi)
		total += t.cases
		totalBad += t.bad
		mustBe(t.bad == 0, t.name+" matches its oracle")
		mustBe(t.hi <= 0, t.name+" stays within its bound")
	}
	mustBe(dedupe.lo == 0 && flag.lo == 0 && prefix.lo == 0 && diff.lo == 0 && rect.lo == 0, "equality bounds are exact")
	fmt.Printf("  %-28s %8d %11d\n", "all rows", total, totalBad)
	fmt.Printf("  PairSum with no pair (n >= 1): %d cases, exactly n-1 steps in %d of them\n", noPair, noPairTight)
	mustBe(noPair == noPairTight, "no pair means exactly n-1 steps")
	worst := "2n"
	if window.hi < 0 {
		worst = fmt.Sprintf("2n-%d", -window.hi)
	}
	fmt.Printf("  LongestValid: the most moves any string needed was %s (bound: 2n)\n", worst)
	fmt.Printf("  overflow row: in 200 arrays of 20 values near 2^59 the first negative prefix is P[%d]", firstNegLo)
	if firstNegLo != firstNegHi {
		fmt.Printf(" to P[%d]", firstNegHi)
	}
	fmt.Printf(";\n  the longest range whose true sum fits is %d elements, and all %d ranges that fit were exact\n", longestExact, wrap.cases-reversed)
	fmt.Printf("  %d ranges had a true sum above MaxInt64: %d came back as a different number; %d reversed ranges were rejected\n\n", notFit, notFitDiffer, reversed)

}

// measureAll is the one timing table.
func measureAll(rng *rand.Rand) {
	fmt.Println("3. measured: brute force against the technique, same input, best of 3 interleaved rounds")
	fmt.Printf("  n is the length, or the grid side for the 2-D rows; ns and ops are per task (a query, or one whole call);\n")
	fmt.Printf("  the technique's ops include building its table; brute force answers the first %d range queries, the first %d rectangles,\n", bruteRange, bruteRects)
	fmt.Printf("  and is skipped when it would need more than %d operations\n", bruteLimit)
	fmt.Printf("  %-34s %7s %14s %14s %10s %14s %14s\n", "task (per ...)", "n", "brute ns", "fast ns", "brute/fast", "brute ops", "fast ops")
	var results []benchRow
	for _, n := range []int{1000, 10000, 100000} {
		results = append(results, benchRangeSum(rng, n), benchPairSum(n), benchWindowMax(rng, n), benchCount(rng, n))
	}
	for _, side := range []int{100, 300, 1000} {
		results = append(results, bench2D(rng, side))
	}
	for _, r := range results {
		r.print()
	}
	fmt.Println()
	explain(results)
}

// ======================= 3. MEASURE =======================

var sink int // keeps timed results alive

const (
	bruteRange = 1000          // range-sum queries brute force answers (the technique answers n)
	bruteRects = 100           // rectangle queries brute force answers (the technique answers 10,000)
	bruteLimit = 1_000_000_000 // skip brute force above this many predicted operations
)

// measure returns ns per call of f, growing the repetition count until the loop
// takes at least 5 ms.
func measure(f func() (int, int)) float64 {
	for reps := 1; ; reps *= 4 {
		start := time.Now()
		for range reps {
			a, _ := f()
			sink += a
		}
		if el := time.Since(start); el >= 5*time.Millisecond || reps >= 1<<22 {
			return float64(el.Nanoseconds()) / float64(reps)
		}
	}
}

// benchRow is one measured line. Both closures return (answer, ops); the answer is
// a checksum over the tasks brute force answers, so the two sides can be compared.
type benchRow struct {
	task              string
	n                 int
	bruteNs, fastNs   float64 // ns per task (bruteNs is 0 when brute was skipped)
	bruteOps, fastOps float64 // operations per task
	bruteTotal        int     // predicted operations of one whole brute-force call
	skipped           bool
}

// run times brute (if it is cheap enough) against fast, interleaved, best of 3, and
// asserts that both give the same answer.
func run(task string, n, bruteTasks, fastTasks int, predictedBruteOps int, brute, fast func() (int, int)) benchRow {
	r := benchRow{task: task, n: n}
	fa, fops := fast()
	r.fastOps = float64(fops) / float64(fastTasks)
	r.fastNs = math.MaxFloat64
	r.skipped = predictedBruteOps > bruteLimit
	r.bruteOps = float64(predictedBruteOps) / float64(bruteTasks)
	r.bruteTotal = predictedBruteOps
	if !r.skipped {
		ba, bops := brute()
		mustBe(ba == fa, task+": brute force and technique agree")
		mustBe(bops == predictedBruteOps, task+": predicted brute-force op count is the counted one")
		r.bruteNs = math.MaxFloat64
	}
	for range 3 {
		if !r.skipped {
			r.bruteNs = min(r.bruteNs, measure(brute)/float64(bruteTasks))
		}
		r.fastNs = min(r.fastNs, measure(fast)/float64(fastTasks))
	}
	return r
}

func (r benchRow) print() {
	bn, ratio, bo := "-", "-", fmt.Sprintf("%.1f (not run)", r.bruteOps)
	if !r.skipped {
		bn, ratio, bo = fmt.Sprintf("%.1f", r.bruteNs), fmt.Sprintf("%.1fx", r.bruteNs/r.fastNs), fmt.Sprintf("%.1f", r.bruteOps)
	}
	fmt.Printf("  %-34s %7d %14s %14.1f %10s %14s %14.1f\n", r.task, r.n, bn, r.fastNs, ratio, bo, r.fastOps)
}

// explain prints conclusions that follow from the table above, computed from it.
func explain(rows []benchRow) {
	byTask := map[string][]benchRow{}
	var order []string
	for _, r := range rows {
		if _, seen := byTask[r.task]; !seen {
			order = append(order, r.task)
		}
		byTask[r.task] = append(byTask[r.task], r)
	}
	ran, faster := 0, 0
	for _, r := range rows {
		if !r.skipped {
			ran++
			if r.fastNs < r.bruteNs {
				faster++
			}
		}
	}
	fmt.Printf("  the technique was faster in %d of the %d rows where brute force ran\n", faster, ran)
	for _, task := range order {
		var done []benchRow
		for _, r := range byTask[task] {
			if !r.skipped {
				done = append(done, r)
			}
		}
		if len(done) < 2 {
			continue
		}
		a, b := done[0], done[len(done)-1]
		fmt.Printf("  %-34s brute/fast %.1fx at n=%d -> %.1fx at n=%d (brute ops/task x%.1f, fast ops/task x%.2f)\n",
			task, a.bruteNs/a.fastNs, a.n, b.bruteNs/b.fastNs, b.n, b.bruteOps/a.bruteOps, b.fastOps/a.fastOps)
	}
	for _, r := range rows {
		if r.skipped {
			fmt.Printf("  not run: %s at n=%d would need %d brute-force operations\n", r.task, r.n, r.bruteTotal)
		}
	}
}

// benchRangeSum: q = n range sums; brute force re-sums each range, the technique
// builds a prefix array once. Brute force answers the first bruteRange of the queries.
func benchRangeSum(rng *rand.Rand, n int) benchRow {
	a := randInts(rng, n, n, 0, 1000)
	ls, rs := make([]int, n), make([]int, n)
	for q := range n {
		ls[q] = rng.IntN(n)
		rs[q] = ls[q] + rng.IntN(n-ls[q]+1)
	}
	bq := min(n, bruteRange)
	predicted := 0
	for q := range bq {
		predicted += rs[q] - ls[q]
	}
	brute := func() (int, int) {
		sum, ops := 0, 0
		for q := range bq {
			sum += bruteSum(a, ls[q], rs[q])
			ops += rs[q] - ls[q]
		}
		return sum, ops
	}
	fast := func() (int, int) {
		pf := NewPrefix(a)
		sum, chk := 0, 0
		for q := range n {
			s, _ := pf.Range(ls[q], rs[q])
			if q < bq {
				chk += s
			}
			sum += s
		}
		return chk, n + pf.reads // n additions to build + the reads
	}
	return run("range sum (per query, q = n)", n, bq, n, predicted, brute, fast)
}

// benchPairSum: sorted even numbers and an odd target, so there is no pair.
func benchPairSum(n int) benchRow {
	a := make([]int, n)
	for i := range a {
		a[i] = 2 * i
	}
	target := 2*n - 1
	brute := func() (int, int) {
		ops := 0
		for i := range a {
			for j := i + 1; j < n; j++ {
				ops++
				if a[i]+a[j] == target {
					return 1, ops
				}
			}
		}
		return 0, ops
	}
	fast := func() (int, int) {
		_, _, ok, steps := PairSum(a, target)
		if ok {
			return 1, steps
		}
		return 0, steps
	}
	return run("pair sum, no pair (per call)", n, 1, 1, n*(n-1)/2, brute, fast)
}

// benchWindowMax: the best sum over every window of k = n/10 elements.
func benchWindowMax(rng *rand.Rand, n int) benchRow {
	a := randInts(rng, n, n, 0, 1000)
	k := n / 10
	brute := func() (int, int) {
		best, ops := 0, 0
		for s := 0; s+k <= n; s++ {
			sum := 0
			for _, v := range a[s : s+k] {
				sum += v
				ops++
			}
			best = max(best, sum)
		}
		return best, ops
	}
	fast := func() (int, int) {
		sum, ops := 0, 0
		for _, v := range a[:k] {
			sum += v
			ops++
		}
		best := sum
		for s := k; s < n; s++ {
			sum += a[s] - a[s-k] // one add, one drop
			ops += 2
			best = max(best, sum)
		}
		return best, ops
	}
	return run("fixed window max sum, k = n/10", n, 1, 1, (n-k+1)*k, brute, fast)
}

// benchCount: how many subarrays of non-negative numbers have sum <= limit. Brute
// force tries every one; the window uses LongestValid and its Count.
func benchCount(rng *rand.Rand, n int) benchRow {
	a := randInts(rng, n, n, 0, 10)
	const limit = 50
	brute := func() (int, int) {
		count, ops := 0, 0
		for l := range a {
			sum := 0
			for r := l; r < n; r++ {
				sum += a[r]
				ops++
				if sum <= limit {
					count++
				}
			}
		}
		return count, ops
	}
	fast := func() (int, int) {
		w := sumAtMost(a, limit)
		return w.Count, w.Moves
	}
	return run("count sums <= 50 (per call)", n, 1, 1, n*(n+1)/2, brute, fast)
}

// bench2D: q = 10,000 rectangle sums on a side x side grid. Brute force sums the
// cells and answers the first bruteRects; the technique builds the table once.
func bench2D(rng *rand.Rand, side int) benchRow {
	g := make([][]int, side)
	for i := range g {
		g[i] = randInts(rng, side, side, 0, 10)
	}
	const q, bq = 10000, bruteRects
	type rc struct{ r1, c1, r2, c2 int }
	qs := make([]rc, q)
	for i := range qs {
		r1 := rng.IntN(side + 1)
		c1 := rng.IntN(side + 1)
		qs[i] = rc{r1, c1, r1 + rng.IntN(side+1-r1), c1 + rng.IntN(side+1-c1)}
	}
	predicted := 0
	for _, x := range qs[:bq] {
		predicted += (x.r2 - x.r1) * (x.c2 - x.c1)
	}
	brute := func() (int, int) {
		sum, ops := 0, 0
		for _, x := range qs[:bq] {
			sum += bruteRect(g, x.r1, x.c1, x.r2, x.c2)
			ops += (x.r2 - x.r1) * (x.c2 - x.c1)
		}
		return sum, ops
	}
	fast := func() (int, int) {
		t := NewPrefix2D(g)
		chk := 0
		for i, x := range qs {
			s, _ := t.Rect(x.r1, x.c1, x.r2, x.c2)
			if i < bq {
				chk += s
			}
		}
		return chk, side*side + t.reads
	}
	return run("2-D rectangle sum (per query)", side, bq, q, predicted, brute, fast)
}
```

**Sample output:**

```
1. the toolkit, one call each
  PairSum([1 3 4 6 9], 12)              -> i=1 j=4 ok=true after 2 steps
  DedupeAtMost([1 1 1 2 2 3], 2)        -> [1 1 2 2 3] after 6 reads, stale tail [3]
  Partition3([2 0 1 2 0 1])             -> [0 0 1 1 2 2] in 6 iterations
  Prefix([3 1 4 1 5]).Range             -> [1:4] = 6, [4:1] accepted = false
  Prefix2D(1..9).Rect(1, 1, 3, 3)       -> 28
  Diff(6): +5 on [1:4], +2 on [3:6]     -> [0 5 5 7 2 2] with 4 writes
  LongestValid("eceba", <= 2 distinct)  -> longest 3 at 0, 10 valid windows, 8 moves

2. proof against brute-force oracles (PCG(1, 2); every bound is asserted, not just printed)
  operation                       cases  mismatches  bound asserted              worst (observed - bound)
  PairSum                         27128           0  steps <= n-1                0
  DedupeAtMost (k 1..3)           20990           0  reads == n                  0
  Partition3                       9841           0  iterations == n             0
  Prefix.Range                    79824           0  2 reads per accepted query  0
  Diff.Add / Values                6400           0  2 writes per Add            0
  diff <-> prefix inverses         2186           0  none (equality)             0
  Prefix2D.Rect                   27920           0  4 reads per accepted query  0
  LongestValid, <= k distinct     20000           0  moves <= 2n                 -1
  Prefix near 2^59 (wraps)        85200           0  exact if the true sum fits  0
  all rows                       279489           0
  PairSum with no pair (n >= 1): 11070 cases, exactly n-1 steps in 11070 of them
  LongestValid: the most moves any string needed was 2n-1 (bound: 2n)
  overflow row: in 200 arrays of 20 values near 2^59 the first negative prefix is P[16];
  the longest range whose true sum fits is 15 elements, and all 43200 ranges that fit were exact
  3000 ranges had a true sum above MaxInt64: 3000 came back as a different number; 42000 reversed ranges were rejected

3. measured: brute force against the technique, same input, best of 3 interleaved rounds
  n is the length, or the grid side for the 2-D rows; ns and ops are per task (a query, or one whole call);
  the technique's ops include building its table; brute force answers the first 1000 range queries, the first 100 rectangles,
  and is skipped when it would need more than 1000000000 operations
  task (per ...)                           n       brute ns        fast ns brute/fast      brute ops       fast ops
  range sum (per query, q = n)          1000           70.4            2.7      26.5x          244.9            3.0
  pair sum, no pair (per call)          1000       193123.0          638.9     302.3x       499500.0          999.0
  fixed window max sum, k = n/10        1000        27891.3          476.9      58.5x        90100.0         1900.0
  count sums <= 50 (per call)           1000       142471.4         3866.4      36.8x       500500.0         1988.0
  range sum (per query, q = n)         10000          633.9            2.5     251.1x         2437.5            3.0
  pair sum, no pair (per call)         10000     18904166.0         6292.3    3004.3x     49995000.0         9999.0
  fixed window max sum, k = n/10       10000      2361229.2         4848.8     487.0x      9001000.0        19000.0
  count sums <= 50 (per call)          10000     12701417.0        66057.8     192.3x     50005000.0        19993.0
  range sum (per query, q = n)        100000         6449.8            3.0    2167.7x        24261.3            3.0
  pair sum, no pair (per call)        100000              -        62883.1          - 4999950000.0 (not run)        99999.0
  fixed window max sum, k = n/10      100000    234755083.0        48599.8    4830.4x    900010000.0       190000.0
  count sums <= 50 (per call)         100000              -       902781.2          - 5000050000.0 (not run)       199990.0
  2-D rectangle sum (per query)          100          238.5            3.8      62.9x          563.3            5.0
  2-D rectangle sum (per query)          300         2289.7           12.0     191.0x         5351.7           13.0
  2-D rectangle sum (per query)         1000        30241.9          136.7     221.2x        74470.8          104.0

  the technique was faster in 13 of the 13 rows where brute force ran
  range sum (per query, q = n)       brute/fast 26.5x at n=1000 -> 2167.7x at n=100000 (brute ops/task x99.1, fast ops/task x1.00)
  pair sum, no pair (per call)       brute/fast 302.3x at n=1000 -> 3004.3x at n=10000 (brute ops/task x100.1, fast ops/task x10.01)
  fixed window max sum, k = n/10     brute/fast 58.5x at n=1000 -> 4830.4x at n=100000 (brute ops/task x9989.0, fast ops/task x100.00)
  count sums <= 50 (per call)        brute/fast 36.8x at n=1000 -> 192.3x at n=10000 (brute ops/task x99.9, fast ops/task x10.06)
  2-D rectangle sum (per query)      brute/fast 62.9x at n=100 -> 221.2x at n=1000 (brute ops/task x132.2, fast ops/task x20.80)
  not run: pair sum, no pair (per call) at n=100000 would need 4999950000 brute-force operations
  not run: count sums <= 50 (per call) at n=100000 would need 5000050000 brute-force operations
```

**Rust:**

```rust
use std::collections::HashMap;
use std::hint::black_box;
use std::process::exit;
use std::time::{Duration, Instant};

// The lesson as one small toolkit, then proved, then measured:
//
//   pair_sum        converging pointers on a sorted slice
//   dedupe_at_most  read/write pointers, keep at most k copies
//   partition3      the Dutch national flag
//   Prefix          1-D prefix sums, half-open ranges, a zero slot
//   Prefix2D        2-D prefix sums, inclusion-exclusion
//   Diff            difference array: range add in Theta(1)
//   longest_valid   one sliding window, driven by add / remove / valid callbacks
//
// Every function returns the work it did (steps, reads, moves), so the proof can
// assert the bound each one claims instead of trusting it. This is a capstone, so
// the program is longer than the other examples. It is the Go program's twin:
// same generator, same draws, so every count below matches the Go run.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) gives the same stream as the Go program.
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

    fn below(&mut self, n: usize) -> usize {
        self.int_n(n as u64) as usize
    }
}

// must_be aborts the run if a claim the program is about to print would be false.
fn must_be(ok: bool, what: &str) {
    if !ok {
        println!("INTERNAL CHECK FAILED: {what}");
        exit(1);
    }
}

// ======================= 1. THE TOOLKIT =======================

// pair_sum finds i < j with a[i] + a[j] == target in a SORTED slice. It returns
// Some((i, j)) or None, and steps counts loop iterations: each one moves a pointer,
// so there are at most n - 1 of them.
fn pair_sum(a: &[i64], target: i64) -> (Option<(usize, usize)>, usize) {
    let (mut lo, mut hi) = (0, a.len().saturating_sub(1));
    let mut steps = 0;
    while lo < hi {
        // not <=: an element must not pair with itself
        steps += 1;
        match (a[lo] + a[hi]).cmp(&target) {
            std::cmp::Ordering::Equal => return (Some((lo, hi)), steps),
            std::cmp::Ordering::Less => lo += 1, // a[lo] cannot pair with anything left of hi
            std::cmp::Ordering::Greater => hi -= 1, // a[hi] cannot pair with anything right of lo
        }
    }
    (None, steps)
}

// dedupe_at_most keeps at most k copies of each value of a sorted slice, in place,
// and returns the new length. It compares with a[write-k], k places back in the
// OUTPUT. reads counts elements read: exactly a.len(). The tail is left stale.
// (Vec::dedup_by is the std version of this loop, and it also drops the removed
// elements; a slice has no length to shrink, so we return the new length.)
fn dedupe_at_most(a: &mut [i64], k: usize) -> (usize, usize) {
    if k < 1 {
        return (0, 0);
    }
    let (mut write, mut reads) = (0, 0);
    for read in 0..a.len() {
        reads += 1;
        let v = a[read];
        if write < k || a[write - k] != v {
            a[write] = v;
            write += 1;
        }
    }
    (write, reads)
}

// partition3 sorts a slice of 0s, 1s and 2s in one pass. Regions: [0,lo) zeros,
// [lo,mid) ones, [mid,hi) unknown, [hi,n) twos. Every iteration shrinks the unknown
// region by one, so iterations == a.len(). After the swap with hi, mid stays put:
// the element that arrived has not been looked at.
fn partition3(a: &mut [i64]) -> usize {
    let (mut lo, mut mid, mut hi) = (0, 0, a.len());
    let mut iterations = 0;
    while mid < hi {
        iterations += 1;
        match a[mid] {
            0 => {
                a.swap(lo, mid);
                lo += 1;
                mid += 1;
            }
            1 => mid += 1,
            _ => {
                hi -= 1;
                a.swap(mid, hi);
            }
        }
    }
    iterations
}

// Prefix holds P with P[0] = 0 and P[i+1] = P[i] + a[i]. The build wraps ON
// PURPOSE (wrapping_add): plain `+` would wrap under -O but panic in a debug build.
// A range whose true sum fits is still exact (see the overflow row).
struct Prefix {
    p: Vec<i64>,
    reads: usize, // reads of p made by range
}

impl Prefix {
    fn new(a: &[i64]) -> Self {
        let mut p = vec![0i64; a.len() + 1];
        for (i, &v) in a.iter().enumerate() {
            p[i + 1] = p[i].wrapping_add(v);
        }
        Prefix { p, reads: 0 }
    }

    // range returns the sum of a[l:r] (half-open). A reversed or out-of-range pair is
    // rejected (None) instead of returning garbage; with usize there is no negative
    // index to reject, and `p[r] - p[l]` is a wrapping_sub only where we say so.
    fn range(&mut self, l: usize, r: usize) -> Option<i64> {
        if l > r || r >= self.p.len() {
            return None;
        }
        self.reads += 2;
        Some(self.p[r].wrapping_sub(self.p[l]))
    }
}

// Prefix2D is the 2-D table, flattened: P[i+1][j+1] = g[i][j] + P[i][j+1] + P[i+1][j] - P[i][j].
struct Prefix2D {
    p: Vec<i64>,
    rows: usize,
    cols: usize,
    reads: usize,
}

impl Prefix2D {
    fn new(g: &[Vec<i64>]) -> Self {
        let rows = g.len();
        let cols = g.first().map_or(0, Vec::len);
        let w = cols + 1;
        let mut p = vec![0i64; (rows + 1) * w];
        for (i, row) in g.iter().enumerate() {
            for (j, &v) in row.iter().enumerate() {
                p[(i + 1) * w + j + 1] = v + p[i * w + j + 1] + p[(i + 1) * w + j] - p[i * w + j];
            }
        }
        Prefix2D {
            p,
            rows,
            cols,
            reads: 0,
        }
    }

    // rect returns the sum over rows [r1,r2) and columns [c1,c2): four reads.
    fn rect(&mut self, r1: usize, c1: usize, r2: usize, c2: usize) -> Option<i64> {
        if r1 > r2 || r2 > self.rows || c1 > c2 || c2 > self.cols {
            return None;
        }
        let w = self.cols + 1;
        self.reads += 4;
        Some(self.p[r2 * w + c2] - self.p[r1 * w + c2] - self.p[r2 * w + c1] + self.p[r1 * w + c1])
    }
}

// Diff records "add v to a[l:r]" as two writes; values does the one prefix sum.
struct Diff {
    d: Vec<i64>,
    writes: usize,
}

impl Diff {
    fn new(n: usize) -> Self {
        Diff {
            d: vec![0; n + 1],
            writes: 0,
        }
    }

    fn add(&mut self, l: usize, r: usize, v: i64) -> bool {
        if l > r || r >= self.d.len() {
            return false;
        }
        self.d[l] += v;
        self.d[r] -= v;
        self.writes += 2;
        true
    }

    fn values(&self) -> Vec<i64> {
        self.d[..self.d.len() - 1]
            .iter()
            .scan(0, |run, &x| {
                *run += x;
                Some(*run)
            })
            .collect()
    }
}

// diff_of is the forward difference of a: the inverse of a prefix sum.
fn diff_of(a: &[i64]) -> Vec<i64> {
    (0..a.len())
        .map(|i| if i == 0 { a[0] } else { a[i] - a[i - 1] })
        .collect()
}

// Windows is what one pass of longest_valid learns.
struct Windows {
    longest: usize,
    start: usize, // longest valid window (earliest if tied)
    count: usize, // number of valid windows (subarrays)
    moves: usize, // pointer moves: at most 2n
}

// longest_valid slides one window over xs. The callbacks update and read the
// window's state, and the predicate must be monotone: a valid window stays valid
// when it shrinks. Then every start in [left, right] is valid for this right, so
// count adds right-left+1 per step.
//
// The Go version takes three closures that all capture the same variables. That
// does not compile here: two closures that mutate the same counters cannot both
// borrow them (E0499), and `valid` cannot read them while `add` holds them (E0502).
// So the state is a value that the toolkit owns and lends to each callback.
fn longest_valid<T: Copy, S>(
    xs: &[T],
    state: &mut S,
    add: impl Fn(&mut S, T),
    remove: impl Fn(&mut S, T),
    valid: impl Fn(&S) -> bool,
) -> Windows {
    let mut w = Windows {
        longest: 0,
        start: 0,
        count: 0,
        moves: 0,
    };
    let mut left = 0;
    for (right, &x) in xs.iter().enumerate() {
        add(state, x);
        w.moves += 1;
        while left <= right && !valid(state) {
            remove(state, xs[left]);
            left += 1;
            w.moves += 1;
        }
        w.count += right + 1 - left;
        if right + 1 - left > w.longest {
            w.longest = right + 1 - left;
            w.start = left;
        }
    }
    w
}

struct Distinct {
    counts: [usize; 256],
    distinct: usize,
}

// at_most_distinct runs longest_valid with the "at most k distinct bytes" predicate.
fn at_most_distinct(s: &[u8], k: usize) -> Windows {
    let mut st = Distinct {
        counts: [0; 256],
        distinct: 0,
    };
    longest_valid(
        s,
        &mut st,
        |st, c| {
            st.counts[c as usize] += 1;
            if st.counts[c as usize] == 1 {
                st.distinct += 1;
            }
        },
        |st, c| {
            st.counts[c as usize] -= 1;
            if st.counts[c as usize] == 0 {
                st.distinct -= 1;
            }
        },
        |st| st.distinct <= k,
    )
}

// sum_at_most runs longest_valid with "sum <= limit" over non-negative ints.
fn sum_at_most(a: &[i64], limit: i64) -> Windows {
    longest_valid(
        a,
        &mut 0i64,
        |sum, v| *sum += v,
        |sum, v| *sum -= v,
        |sum| *sum <= limit,
    )
}

// ======================= 2. THE ORACLES =======================

fn has_pair(a: &[i64], target: i64) -> bool {
    (0..a.len()).any(|i| (i + 1..a.len()).any(|j| a[i] + a[j] == target))
}

fn brute_sum(a: &[i64], l: usize, r: usize) -> i64 {
    a[l..r].iter().sum()
}

// keep_at_most: the first k copies of each value (right for sorted input).
fn keep_at_most(a: &[i64], k: usize) -> Vec<i64> {
    let mut seen: HashMap<i64, usize> = HashMap::new();
    let mut out = Vec::new();
    for &v in a {
        let c = seen.entry(v).or_insert(0);
        if *c < k {
            out.push(v);
        }
        *c += 1;
    }
    out
}

// brute_windows tries every (l, r): longest and count of windows with <= k distinct bytes.
fn brute_windows(s: &[u8], k: usize) -> (usize, usize, usize) {
    let (mut longest, mut start, mut count) = (0, 0, 0);
    for l in 0..s.len() {
        let mut seen = [false; 256];
        let mut distinct = 0;
        for r in l..s.len() {
            if !seen[s[r] as usize] {
                seen[s[r] as usize] = true;
                distinct += 1;
            }
            if distinct > k {
                break;
            }
            count += 1;
            if r - l + 1 > longest {
                longest = r - l + 1;
                start = l;
            }
        }
    }
    (longest, start, count)
}

fn brute_rect(g: &[Vec<i64>], r1: usize, c1: usize, r2: usize, c2: usize) -> i64 {
    g[r1..r2]
        .iter()
        .map(|row| row[c1..c2].iter().sum::<i64>())
        .sum()
}

// each calls f with every array of length <= max_len over vals (only non-decreasing
// ones when sorted is true). f gets a borrowed slice: clone it before mutating.
fn each(max_len: usize, vals: &[i64], sorted: bool, f: &mut dyn FnMut(&[i64])) {
    fn rec(
        a: &mut Vec<i64>,
        max_len: usize,
        vals: &[i64],
        sorted: bool,
        f: &mut dyn FnMut(&[i64]),
    ) {
        f(a);
        if a.len() == max_len {
            return;
        }
        for &v in vals {
            if sorted && a.last().is_some_and(|&last| v < last) {
                continue;
            }
            a.push(v);
            rec(a, max_len, vals, sorted, f);
            a.pop();
        }
    }
    rec(&mut Vec::new(), max_len, vals, sorted, f);
}

// rand_ints draws a length in [min_len, max_len], then that many values in [lo, lo+span).
// The draw order is part of the contract: it is the Go program's.
fn rand_ints(rng: &mut Pcg, min_len: usize, max_len: usize, lo: i64, span: usize) -> Vec<i64> {
    let n = min_len + rng.below(max_len - min_len + 1);
    (0..n).map(|_| lo + rng.below(span) as i64).collect()
}

// if_valid is the work a query should have cost: n if it was accepted, 0 if rejected.
fn if_valid(valid: bool, n: usize) -> usize {
    if valid { n } else { 0 }
}

// idx turns a signed test coordinate into an index. Go's int can be -1 and the
// toolkit rejects it; a usize cannot be negative, so here the rejection is the
// conversion itself.
fn idx(x: i64) -> Option<usize> {
    usize::try_from(x).ok()
}

// Tally is one row of the proof table. Each check records whether the answer was
// right and how far the observed work was from the bound it claims.
struct Tally {
    name: &'static str,
    bound: &'static str,
    cases: usize,
    bad: usize,
    lo: i64, // min of (observed - bound)
    hi: i64, // max of (observed - bound)
}

impl Tally {
    fn new(name: &'static str, bound: &'static str) -> Self {
        Tally {
            name,
            bound,
            cases: 0,
            bad: 0,
            lo: i64::MAX,
            hi: i64::MIN,
        }
    }

    fn check(&mut self, good: bool, observed: usize, bound: usize) {
        self.cases += 1;
        if !good {
            self.bad += 1;
        }
        let d = observed as i64 - bound as i64;
        self.lo = self.lo.min(d);
        self.hi = self.hi.max(d);
    }
}

fn main() {
    let mut rng = Pcg::new(1, 2);
    smoke();
    prove(&mut rng);
    measure_all(&mut rng);
}

// smoke makes one call to each tool, so the API is visible before the proof.
fn smoke() {
    fn say(call: &str, result: String) {
        println!("  {call:<37} -> {result}");
    }
    fn show(xs: &[i64]) -> String {
        let parts: Vec<String> = xs.iter().map(i64::to_string).collect();
        format!("[{}]", parts.join(" "))
    }
    println!("1. the toolkit, one call each");
    let (found, steps) = pair_sum(&[1, 3, 4, 6, 9], 12);
    let (i, j) = found.unwrap_or((0, 0));
    say(
        "pair_sum([1 3 4 6 9], 12)",
        format!("i={i} j={j} ok={} after {steps} steps", found.is_some()),
    );
    let mut dd = [1, 1, 1, 2, 2, 3];
    let (n, reads) = dedupe_at_most(&mut dd, 2);
    say(
        "dedupe_at_most([1 1 1 2 2 3], 2)",
        format!(
            "{} after {reads} reads, stale tail {}",
            show(&dd[..n]),
            show(&dd[n..])
        ),
    );
    let mut fl = [2, 0, 1, 2, 0, 1];
    let it = partition3(&mut fl);
    say(
        "partition3([2 0 1 2 0 1])",
        format!("{} in {it} iterations", show(&fl)),
    );
    let mut pf = Prefix::new(&[3, 1, 4, 1, 5]);
    let sum = pf.range(1, 4).unwrap();
    let accepted = pf.range(4, 1).is_some();
    say(
        "Prefix([3 1 4 1 5]).range",
        format!("[1:4] = {sum}, [4:1] accepted = {accepted}"),
    );
    let mut p2 = Prefix2D::new(&[vec![1, 2, 3], vec![4, 5, 6], vec![7, 8, 9]]);
    say(
        "Prefix2D(1..9).rect(1, 1, 3, 3)",
        format!("{}", p2.rect(1, 1, 3, 3).unwrap()),
    );
    let mut df = Diff::new(6);
    df.add(1, 4, 5);
    df.add(3, 6, 2);
    say(
        "Diff(6): +5 on [1:4], +2 on [3:6]",
        format!("{} with {} writes", show(&df.values()), df.writes),
    );
    let w = at_most_distinct(b"eceba", 2);
    say(
        "longest_valid(\"eceba\", <= 2 distinct)",
        format!(
            "longest {} at {}, {} valid windows, {} moves",
            w.longest, w.start, w.count, w.moves
        ),
    );
    println!();
}

// prove checks every tool against its oracle and asserts the bound it claims.
fn prove(rng: &mut Pcg) {
    let mut pair = Tally::new("pair_sum", "steps <= n-1");
    let mut dedupe = Tally::new("dedupe_at_most (k 1..3)", "reads == n");
    let mut flag = Tally::new("partition3", "iterations == n");
    let mut prefix = Tally::new("Prefix::range", "2 reads per accepted query");
    let mut diff = Tally::new("Diff::add / values", "2 writes per Add");
    let mut inverse = Tally::new("diff <-> prefix inverses", "none (equality)");
    let mut rect = Tally::new("Prefix2D::rect", "4 reads per accepted query");
    let mut window = Tally::new("longest_valid, <= k distinct", "moves <= 2n");
    let mut wrap = Tally::new("Prefix near 2^59 (wraps)", "exact if the true sum fits");

    // PairSum: every sorted array of length <= 7 over {0..4} x every target, then PCG cases.
    let (mut no_pair, mut no_pair_tight) = (0, 0);
    let mut pair_case = |a: &[i64], target: i64| {
        let (found, steps) = pair_sum(a, target);
        let want = has_pair(a, target);
        let good =
            found.is_some() == want && found.is_none_or(|(i, j)| i < j && a[i] + a[j] == target);
        pair.check(good, steps, a.len().saturating_sub(1));
        if !want && !a.is_empty() {
            no_pair += 1;
            if steps == a.len() - 1 {
                no_pair_tight += 1;
            }
        }
    };
    each(7, &[0, 1, 2, 3, 4], true, &mut |a| {
        for target in 0..=8 {
            pair_case(a, target);
        }
    });
    for _ in 0..20000 {
        let mut a = rand_ints(rng, 0, 30, 0, 20);
        a.sort_unstable();
        pair_case(&a, rng.below(40) as i64);
    }

    // DedupeAtMost: every sorted array of length <= 7 over {0..3} x k, then PCG cases.
    let mut dedupe_case = |a: &[i64], k: usize| {
        let want = keep_at_most(a, k);
        let mut got = a.to_vec();
        let (n, reads) = dedupe_at_most(&mut got, k);
        let mut good = got[..n] == want[..];
        if k == 1 {
            // at k = 1 this is exactly Vec::dedup
            let mut compact = a.to_vec();
            compact.dedup();
            good = good && got[..n] == compact[..];
        }
        dedupe.check(good, reads, a.len());
    };
    each(7, &[0, 1, 2, 3], true, &mut |a| {
        for k in 1..=3 {
            dedupe_case(a, k);
        }
    });
    for _ in 0..20000 {
        let mut a = rand_ints(rng, 0, 30, 0, 6);
        a.sort_unstable();
        dedupe_case(&a, 1 + rng.below(3));
    }

    // Partition3: every array of length <= 8 over {0,1,2}.
    each(8, &[0, 1, 2], false, &mut |a| {
        let mut want = a.to_vec();
        want.sort_unstable();
        let mut got = a.to_vec();
        let it = partition3(&mut got);
        flag.check(got == want, it, a.len());
    });

    // Prefix: every array of length <= 6 over {-1,0,1}, every (l, r) including reversed
    // and out-of-range ones, which must be rejected.
    each(6, &[-1, 0, 1], false, &mut |a| {
        let n = a.len() as i64;
        let mut pf = Prefix::new(a);
        for l in -1..=n + 1 {
            for r in -1..=n + 1 {
                let valid = 0 <= l && l <= r && r <= n;
                let before = pf.reads;
                let got = idx(l).zip(idx(r)).and_then(|(l, r)| pf.range(l, r));
                let good = got.is_some() == valid
                    && got.is_none_or(|g| g == brute_sum(a, l as usize, r as usize));
                prefix.check(good, pf.reads - before, if_valid(valid, 2));
            }
        }
        // prefix(diff(a)) == a and diff(prefix(a)) == a
        let back = Prefix::new(&diff_of(a)).p[1..].to_vec();
        inverse.check(back == a, 0, 0);
        inverse.check(diff_of(&Prefix::new(a).p[1..]) == a, 0, 0);
    });

    // Diff: every (l, r, v) on arrays of length 0..6, including rejected pairs, then 5,000
    // random sequences of range adds against the naive loops.
    for n in 0..=6i64 {
        for l in -1..=n + 1 {
            for r in -1..=n + 1 {
                for v in -2..=2 {
                    let valid = 0 <= l && l <= r && r <= n;
                    let mut df = Diff::new(n as usize);
                    let ok = idx(l).zip(idx(r)).is_some_and(|(l, r)| df.add(l, r, v));
                    let mut want = vec![0i64; n as usize];
                    if valid {
                        want[l as usize..r as usize]
                            .iter_mut()
                            .for_each(|x| *x += v);
                    }
                    diff.check(
                        ok == valid && df.values() == want,
                        df.writes,
                        if_valid(valid, 2),
                    );
                }
            }
        }
    }
    for _ in 0..5000 {
        let n = 1 + rng.below(20);
        let updates = 1 + rng.below(10);
        let (mut df, mut want) = (Diff::new(n), vec![0i64; n]);
        for _ in 0..updates {
            let l = rng.below(n + 1);
            let r = l + rng.below(n + 1 - l);
            let v = rng.below(11) as i64 - 5;
            df.add(l, r, v);
            want[l..r].iter_mut().for_each(|x| *x += v);
        }
        diff.check(df.values() == want, df.writes, 2 * updates);
    }

    // Prefix2D: every rectangle of 20 random 5x5 grids (reversed ones must be rejected),
    // then 2,000 random rectangles of a 30x30 grid.
    let mut rect_case =
        |g: &[Vec<i64>], t: &mut Prefix2D, r1: usize, c1: usize, r2: usize, c2: usize| {
            let valid = r1 <= r2 && c1 <= c2;
            let before = t.reads;
            let got = t.rect(r1, c1, r2, c2);
            let good =
                got.is_some() == valid && got.is_none_or(|x| x == brute_rect(g, r1, c1, r2, c2));
            rect.check(good, t.reads - before, if_valid(valid, 4));
        };
    let rand_grid = |rng: &mut Pcg, side: usize, span: usize| -> Vec<Vec<i64>> {
        (0..side)
            .map(|_| {
                (0..side)
                    .map(|_| rng.below(span) as i64 - (span / 2) as i64)
                    .collect()
            })
            .collect()
    };
    for _ in 0..20 {
        let g = rand_grid(rng, 5, 7);
        let mut t = Prefix2D::new(&g);
        for r1 in 0..=5 {
            for r2 in 0..=5 {
                for c1 in 0..=5 {
                    for c2 in 0..=5 {
                        rect_case(&g, &mut t, r1, c1, r2, c2);
                    }
                }
            }
        }
    }
    let g30 = rand_grid(rng, 30, 19);
    let mut t30 = Prefix2D::new(&g30);
    for _ in 0..2000 {
        let r1 = rng.below(31);
        let r2 = r1 + rng.below(31 - r1);
        let c1 = rng.below(31);
        let c2 = c1 + rng.below(31 - c1);
        rect_case(&g30, &mut t30, r1, c1, r2, c2);
    }

    // LongestValid: 20,000 random strings of length 1..40, at most k distinct, against the O(n^2) oracle.
    for _ in 0..20000 {
        let n = 1 + rng.below(40);
        let sigma = 1 + rng.below(5);
        let s: Vec<u8> = (0..n).map(|_| b'a' + rng.below(sigma) as u8).collect();
        let k = 1 + rng.below(3);
        let w = at_most_distinct(&s, k);
        let (longest, start, count) = brute_windows(&s, k);
        window.check(
            w.longest == longest && w.start == start && w.count == count,
            w.moves,
            2 * n,
        );
    }

    // Overflow: values near 2^59 wrap the prefix at P[16]. A range whose true sum fits
    // in i64 must still be exact (i128 is the oracle); a reversed range is rejected.
    let (mut first_neg_lo, mut first_neg_hi) = (usize::MAX, 0);
    let (mut longest_exact, mut not_fit, mut not_fit_differ, mut reversed) = (0, 0, 0, 0);
    for _ in 0..200 {
        let a: Vec<i64> = (0..20)
            .map(|_| (1i64 << 59) + rng.below(1 << 20) as i64)
            .collect();
        let mut pf = Prefix::new(&a);
        if let Some(i) = pf.p.iter().position(|&v| v < 0) {
            first_neg_lo = first_neg_lo.min(i);
            first_neg_hi = first_neg_hi.max(i);
        }
        for l in 0..=20 {
            let mut true_sum: i128 = 0;
            for r in l..=20 {
                if r > l {
                    true_sum += i128::from(a[r - 1]);
                }
                let got = pf.range(l, r);
                if let Ok(exact) = i64::try_from(true_sum) {
                    wrap.check(got == Some(exact), 0, 0);
                    longest_exact = longest_exact.max(r - l);
                } else {
                    not_fit += 1;
                    if got.is_some_and(|g| i128::from(g) != true_sum) {
                        not_fit_differ += 1;
                    }
                }
            }
            for r in 0..l {
                wrap.check(pf.range(l, r).is_none(), 0, 0);
                reversed += 1;
            }
        }
    }
    must_be(first_neg_lo != usize::MAX, "every array wrapped");

    println!(
        "2. proof against brute-force oracles (PCG(1, 2); every bound is asserted, not just printed)"
    );
    println!(
        "  {:<28} {:>8} {:>11}  {:<27} worst (observed - bound)",
        "operation", "cases", "mismatches", "bound asserted"
    );
    let (mut total, mut total_bad) = (0, 0);
    let rows = [
        &pair, &dedupe, &flag, &prefix, &diff, &inverse, &rect, &window, &wrap,
    ];
    for t in rows {
        println!(
            "  {:<28} {:>8} {:>11}  {:<27} {}",
            t.name, t.cases, t.bad, t.bound, t.hi
        );
        total += t.cases;
        total_bad += t.bad;
        must_be(t.bad == 0, &format!("{} matches its oracle", t.name));
        must_be(t.hi <= 0, &format!("{} stays within its bound", t.name));
    }
    must_be(
        dedupe.lo == 0 && flag.lo == 0 && prefix.lo == 0 && diff.lo == 0 && rect.lo == 0,
        "equality bounds are exact",
    );
    println!("  {:<28} {:>8} {:>11}", "all rows", total, total_bad);
    println!(
        "  pair_sum with no pair (n >= 1): {no_pair} cases, exactly n-1 steps in {no_pair_tight} of them"
    );
    must_be(no_pair == no_pair_tight, "no pair means exactly n-1 steps");
    let worst = if window.hi < 0 {
        format!("2n-{}", -window.hi)
    } else {
        "2n".to_string()
    };
    println!("  longest_valid: the most moves any string needed was {worst} (bound: 2n)");
    print!(
        "  overflow row: in 200 arrays of 20 values near 2^59 the first negative prefix is P[{first_neg_lo}]"
    );
    if first_neg_lo != first_neg_hi {
        print!(" to P[{first_neg_hi}]");
    }
    println!(
        ";\n  the longest range whose true sum fits is {longest_exact} elements, and all {} ranges that fit were exact",
        wrap.cases - reversed
    );
    println!(
        "  {not_fit} ranges had a true sum above i64::MAX: {not_fit_differ} came back as a different number; {reversed} reversed ranges were rejected\n"
    );
}

// ======================= 3. MEASURE =======================

const BRUTE_RANGE: usize = 1000; // range-sum queries brute force answers (the technique answers n)
const BRUTE_RECTS: usize = 100; // rectangle queries brute force answers (the technique answers 10,000)
const BRUTE_LIMIT: usize = 1_000_000_000; // skip brute force above this many predicted operations

// measure returns ns per call of f, growing the repetition count until the loop
// takes at least 5 ms. black_box keeps the optimiser from deleting the work.
fn measure(f: &dyn Fn() -> (i64, usize)) -> f64 {
    let mut reps = 1usize;
    loop {
        let start = Instant::now();
        for _ in 0..reps {
            black_box(f());
        }
        let el = start.elapsed();
        if el >= Duration::from_millis(5) || reps >= 1 << 22 {
            return el.as_nanos() as f64 / reps as f64;
        }
        reps *= 4;
    }
}

// BenchRow is one measured line. Both closures return (answer, ops); the answer is
// a checksum over the first tasks brute force answers, so the two sides can be compared.
struct BenchRow {
    task: &'static str,
    n: usize,
    brute_ns: f64, // ns per task (0 when brute force was skipped)
    fast_ns: f64,
    brute_ops: f64, // operations per task
    fast_ops: f64,
    brute_total: usize, // predicted operations of one whole brute-force call
    skipped: bool,
}

type Task<'a> = &'a dyn Fn() -> (i64, usize);

// run times brute (if it is cheap enough) against fast, interleaved, best of 3, and
// asserts that both give the same answer.
fn run(
    task: &'static str,
    n: usize,
    tasks: (usize, usize), // (brute-force tasks per call, technique tasks per call)
    predicted_brute_ops: usize,
    brute: Task,
    fast: Task,
) -> BenchRow {
    let (brute_tasks, fast_tasks) = tasks;
    let (fa, fops) = fast();
    let skipped = predicted_brute_ops > BRUTE_LIMIT;
    let mut row = BenchRow {
        task,
        n,
        brute_ns: 0.0,
        fast_ns: f64::MAX,
        brute_ops: predicted_brute_ops as f64 / brute_tasks as f64,
        fast_ops: fops as f64 / fast_tasks as f64,
        brute_total: predicted_brute_ops,
        skipped,
    };
    if !skipped {
        let (ba, bops) = brute();
        must_be(
            ba == fa,
            &format!("{task}: brute force and technique agree"),
        );
        must_be(
            bops == predicted_brute_ops,
            &format!("{task}: predicted brute-force op count is the counted one"),
        );
        row.brute_ns = f64::MAX;
    }
    for _ in 0..3 {
        if !skipped {
            row.brute_ns = row.brute_ns.min(measure(brute) / brute_tasks as f64);
        }
        row.fast_ns = row.fast_ns.min(measure(fast) / fast_tasks as f64);
    }
    row
}

impl BenchRow {
    fn print(&self) {
        let (bn, ratio, bo) = if self.skipped {
            (
                "-".to_string(),
                "-".to_string(),
                format!("{:.1} (not run)", self.brute_ops),
            )
        } else {
            (
                format!("{:.1}", self.brute_ns),
                format!("{:.1}x", self.brute_ns / self.fast_ns),
                format!("{:.1}", self.brute_ops),
            )
        };
        println!(
            "  {:<34} {:>7} {:>14} {:>14.1} {:>10} {:>14} {:>14.1}",
            self.task, self.n, bn, self.fast_ns, ratio, bo, self.fast_ops
        );
    }
}

fn measure_all(rng: &mut Pcg) {
    println!(
        "3. measured: brute force against the technique, same input, best of 3 interleaved rounds"
    );
    println!(
        "  n is the length, or the grid side for the 2-D rows; ns and ops are per task (a query, or one whole call);"
    );
    println!(
        "  the technique's ops include building its table; brute force answers the first {BRUTE_RANGE} range queries, the first {BRUTE_RECTS} rectangles,"
    );
    println!("  and is skipped when it would need more than {BRUTE_LIMIT} operations");
    println!(
        "  {:<34} {:>7} {:>14} {:>14} {:>10} {:>14} {:>14}",
        "task (per ...)", "n", "brute ns", "fast ns", "brute/fast", "brute ops", "fast ops"
    );
    let mut results = Vec::new();
    for n in [1000, 10000, 100000] {
        results.push(bench_range_sum(rng, n));
        results.push(bench_pair_sum(n));
        results.push(bench_window_max(rng, n));
        results.push(bench_count(rng, n));
    }
    for side in [100, 300, 1000] {
        results.push(bench_2d(rng, side));
    }
    for r in &results {
        r.print();
    }
    println!();
    explain(&results);
}

// explain prints conclusions that follow from the table above, computed from it.
fn explain(rows: &[BenchRow]) {
    let mut by_task: Vec<(&str, Vec<&BenchRow>)> = Vec::new();
    for r in rows {
        match by_task.iter_mut().find(|(t, _)| *t == r.task) {
            Some((_, v)) => v.push(r),
            None => by_task.push((r.task, vec![r])),
        }
    }
    let ran: Vec<&BenchRow> = rows.iter().filter(|r| !r.skipped).collect();
    let faster = ran.iter().filter(|r| r.fast_ns < r.brute_ns).count();
    println!(
        "  the technique was faster in {faster} of the {} rows where brute force ran",
        ran.len()
    );
    for (task, v) in &by_task {
        let done: Vec<&&BenchRow> = v.iter().filter(|r| !r.skipped).collect();
        if done.len() < 2 {
            continue;
        }
        let (a, b) = (done[0], done[done.len() - 1]);
        println!(
            "  {:<34} brute/fast {:.1}x at n={} -> {:.1}x at n={} (brute ops/task x{:.1}, fast ops/task x{:.2})",
            task,
            a.brute_ns / a.fast_ns,
            a.n,
            b.brute_ns / b.fast_ns,
            b.n,
            b.brute_ops / a.brute_ops,
            b.fast_ops / a.fast_ops
        );
    }
    for r in rows.iter().filter(|r| r.skipped) {
        println!(
            "  not run: {} at n={} would need {} brute-force operations",
            r.task, r.n, r.brute_total
        );
    }
}

// bench_range_sum: q = n range sums; brute force re-sums each range, the technique
// builds a prefix array once. Brute force answers the first BRUTE_RANGE of the queries.
fn bench_range_sum(rng: &mut Pcg, n: usize) -> BenchRow {
    let a = rand_ints(rng, n, n, 0, 1000);
    let (mut ls, mut rs) = (vec![0; n], vec![0; n]);
    for q in 0..n {
        ls[q] = rng.below(n);
        rs[q] = ls[q] + rng.below(n - ls[q] + 1);
    }
    let bq = n.min(BRUTE_RANGE);
    let predicted: usize = (0..bq).map(|q| rs[q] - ls[q]).sum();
    let brute = || {
        let (mut sum, mut ops) = (0, 0);
        for q in 0..bq {
            sum += brute_sum(&a, ls[q], rs[q]);
            ops += rs[q] - ls[q];
        }
        (black_box(sum), ops)
    };
    let fast = || {
        let mut pf = Prefix::new(&a);
        let mut chk = 0;
        for q in 0..n {
            let s = pf.range(ls[q], rs[q]).unwrap_or(0);
            if q < bq {
                chk += s;
            }
            black_box(s);
        }
        (chk, n + pf.reads) // n additions to build + the reads
    };
    run(
        "range sum (per query, q = n)",
        n,
        (bq, n),
        predicted,
        &brute,
        &fast,
    )
}

// bench_pair_sum: sorted even numbers and an odd target, so there is no pair.
fn bench_pair_sum(n: usize) -> BenchRow {
    let a: Vec<i64> = (0..n as i64).map(|i| 2 * i).collect();
    let target = 2 * n as i64 - 1;
    let brute = || {
        let mut ops = 0;
        for i in 0..n {
            for j in i + 1..n {
                ops += 1;
                if a[i] + a[j] == target {
                    return (1, ops);
                }
            }
        }
        (0, ops)
    };
    let fast = || {
        let (found, steps) = pair_sum(black_box(&a), target);
        (i64::from(found.is_some()), steps)
    };
    run(
        "pair sum, no pair (per call)",
        n,
        (1, 1),
        n * (n - 1) / 2,
        &brute,
        &fast,
    )
}

// bench_window_max: the best sum over every window of k = n/10 elements.
fn bench_window_max(rng: &mut Pcg, n: usize) -> BenchRow {
    let a = rand_ints(rng, n, n, 0, 1000);
    let k = n / 10;
    let brute = || {
        let (mut best, mut ops) = (0, 0);
        for s in 0..=n - k {
            let mut sum = 0;
            for &v in &a[s..s + k] {
                sum += v;
                ops += 1;
            }
            best = best.max(sum);
        }
        (black_box(best), ops)
    };
    let fast = || {
        let (mut sum, mut ops) = (0, 0);
        for &v in &a[..k] {
            sum += v;
            ops += 1;
        }
        let mut best = sum;
        for s in k..n {
            sum += a[s] - a[s - k]; // one add, one drop
            ops += 2;
            best = best.max(sum);
        }
        (black_box(best), ops)
    };
    run(
        "fixed window max sum, k = n/10",
        n,
        (1, 1),
        (n - k + 1) * k,
        &brute,
        &fast,
    )
}

// bench_count: how many subarrays of non-negative numbers have sum <= limit. Brute
// force tries every one; the window uses longest_valid and its count.
fn bench_count(rng: &mut Pcg, n: usize) -> BenchRow {
    let a = rand_ints(rng, n, n, 0, 10);
    const LIMIT: i64 = 50;
    let brute = || {
        let (mut count, mut ops) = (0, 0);
        for l in 0..n {
            let mut sum = 0;
            for &v in &a[l..] {
                sum += v;
                ops += 1;
                if sum <= LIMIT {
                    count += 1;
                }
            }
        }
        (black_box(count), ops)
    };
    let fast = || {
        let w = sum_at_most(black_box(&a), LIMIT);
        (w.count as i64, w.moves)
    };
    run(
        "count sums <= 50 (per call)",
        n,
        (1, 1),
        n * (n + 1) / 2,
        &brute,
        &fast,
    )
}

// bench_2d: q = 10,000 rectangle sums on a side x side grid. Brute force sums the
// cells and answers the first BRUTE_RECTS; the technique builds the table once.
fn bench_2d(rng: &mut Pcg, side: usize) -> BenchRow {
    let g: Vec<Vec<i64>> = (0..side)
        .map(|_| rand_ints(rng, side, side, 0, 10))
        .collect();
    const Q: usize = 10000;
    let mut qs = Vec::with_capacity(Q);
    for _ in 0..Q {
        let r1 = rng.below(side + 1);
        let c1 = rng.below(side + 1);
        let r2 = r1 + rng.below(side + 1 - r1);
        let c2 = c1 + rng.below(side + 1 - c1);
        qs.push((r1, c1, r2, c2));
    }
    let predicted: usize = qs[..BRUTE_RECTS]
        .iter()
        .map(|&(r1, c1, r2, c2)| (r2 - r1) * (c2 - c1))
        .sum();
    let brute = || {
        let (mut sum, mut ops) = (0, 0);
        for &(r1, c1, r2, c2) in &qs[..BRUTE_RECTS] {
            sum += brute_rect(&g, r1, c1, r2, c2);
            ops += (r2 - r1) * (c2 - c1);
        }
        (black_box(sum), ops)
    };
    let fast = || {
        let mut t = Prefix2D::new(&g);
        let mut chk = 0;
        for (i, &(r1, c1, r2, c2)) in qs.iter().enumerate() {
            let s = t.rect(r1, c1, r2, c2).unwrap_or(0);
            if i < BRUTE_RECTS {
                chk += s;
            }
            black_box(s);
        }
        (chk, side * side + t.reads)
    };
    run(
        "2-D rectangle sum (per query)",
        side,
        (BRUTE_RECTS, Q),
        predicted,
        &brute,
        &fast,
    )
}
```

**Rust sample output:**

```
1. the toolkit, one call each
  pair_sum([1 3 4 6 9], 12)             -> i=1 j=4 ok=true after 2 steps
  dedupe_at_most([1 1 1 2 2 3], 2)      -> [1 1 2 2 3] after 6 reads, stale tail [3]
  partition3([2 0 1 2 0 1])             -> [0 0 1 1 2 2] in 6 iterations
  Prefix([3 1 4 1 5]).range             -> [1:4] = 6, [4:1] accepted = false
  Prefix2D(1..9).rect(1, 1, 3, 3)       -> 28
  Diff(6): +5 on [1:4], +2 on [3:6]     -> [0 5 5 7 2 2] with 4 writes
  longest_valid("eceba", <= 2 distinct) -> longest 3 at 0, 10 valid windows, 8 moves

2. proof against brute-force oracles (PCG(1, 2); every bound is asserted, not just printed)
  operation                       cases  mismatches  bound asserted              worst (observed - bound)
  pair_sum                        27128           0  steps <= n-1                0
  dedupe_at_most (k 1..3)         20990           0  reads == n                  0
  partition3                       9841           0  iterations == n             0
  Prefix::range                   79824           0  2 reads per accepted query  0
  Diff::add / values               6400           0  2 writes per Add            0
  diff <-> prefix inverses         2186           0  none (equality)             0
  Prefix2D::rect                  27920           0  4 reads per accepted query  0
  longest_valid, <= k distinct    20000           0  moves <= 2n                 -1
  Prefix near 2^59 (wraps)        85200           0  exact if the true sum fits  0
  all rows                       279489           0
  pair_sum with no pair (n >= 1): 11070 cases, exactly n-1 steps in 11070 of them
  longest_valid: the most moves any string needed was 2n-1 (bound: 2n)
  overflow row: in 200 arrays of 20 values near 2^59 the first negative prefix is P[16];
  the longest range whose true sum fits is 15 elements, and all 43200 ranges that fit were exact
  3000 ranges had a true sum above i64::MAX: 3000 came back as a different number; 42000 reversed ranges were rejected

3. measured: brute force against the technique, same input, best of 3 interleaved rounds
  n is the length, or the grid side for the 2-D rows; ns and ops are per task (a query, or one whole call);
  the technique's ops include building its table; brute force answers the first 1000 range queries, the first 100 rectangles,
  and is skipped when it would need more than 1000000000 operations
  task (per ...)                           n       brute ns        fast ns brute/fast      brute ops       fast ops
  range sum (per query, q = n)          1000           13.7            1.1      12.0x          244.9            3.0
  pair sum, no pair (per call)          1000       193061.2          633.4     304.8x       499500.0          999.0
  fixed window max sum, k = n/10        1000         4310.1          459.9       9.4x        90100.0         1900.0
  count sums <= 50 (per call)           1000       140975.3         1265.8     111.4x       500500.0         1988.0
  range sum (per query, q = n)         10000          149.1            1.2     119.3x         2437.5            3.0
  pair sum, no pair (per call)         10000     18925500.0         6285.8    3010.8x     49995000.0         9999.0
  fixed window max sum, k = n/10       10000       493434.9         4591.7     107.5x      9001000.0        19000.0
  count sums <= 50 (per call)          10000     13563750.0        14345.7     945.5x     50005000.0        19993.0
  range sum (per query, q = n)        100000         1777.4            1.7    1016.5x        24261.3            3.0
  pair sum, no pair (per call)        100000              -        62929.4          - 4999950000.0 (not run)        99999.0
  fixed window max sum, k = n/10      100000     55864917.0        46049.3    1213.2x    900010000.0       190000.0
  count sums <= 50 (per call)         100000              -       371494.8          - 5000050000.0 (not run)       199990.0
  2-D rectangle sum (per query)          100           58.4            3.6      16.3x          563.3            5.0
  2-D rectangle sum (per query)          300          563.2           16.5      34.1x         5351.7           13.0
  2-D rectangle sum (per query)         1000         7883.7          158.2      49.8x        74470.8          104.0

  the technique was faster in 13 of the 13 rows where brute force ran
  range sum (per query, q = n)       brute/fast 12.0x at n=1000 -> 1016.5x at n=100000 (brute ops/task x99.1, fast ops/task x1.00)
  pair sum, no pair (per call)       brute/fast 304.8x at n=1000 -> 3010.8x at n=10000 (brute ops/task x100.1, fast ops/task x10.01)
  fixed window max sum, k = n/10     brute/fast 9.4x at n=1000 -> 1213.2x at n=100000 (brute ops/task x9989.0, fast ops/task x100.00)
  count sums <= 50 (per call)        brute/fast 111.4x at n=1000 -> 945.5x at n=10000 (brute ops/task x99.9, fast ops/task x10.06)
  2-D rectangle sum (per query)      brute/fast 16.3x at n=100 -> 49.8x at n=1000 (brute ops/task x132.2, fast ops/task x20.80)
  not run: pair sum, no pair (per call) at n=100000 would need 4999950000 brute-force operations
  not run: count sums <= 50 (per call) at n=100000 would need 5000050000 brute-force operations
```

> **Go vs Rust:** Every deterministic number matches the Go run (279,489 proof cases, 0 mismatches, the same step and operation counts); the one API change is that Go's three closures sharing counters do not compile in Rust (E0499 and E0502, compiled and read), so `longest_valid` takes the state as a `&mut S` that it lends to each callback, while `usize` makes the `-1` test coordinates unrepresentable (`usize::try_from` rejects them) and `wrapping_add`/`wrapping_sub` mark the wrap on purpose. The timings differ: Rust's brute-force sums were about 5–7x faster than Go's (range sums about 14 ns vs 70 ns per query and the fixed-window max sum about 4.4 µs vs 28–30 µs at n = 10³) while the techniques cost about the same, so its ratios are smaller (range sums about 12x / 118–120x / 800–1,020x, fixed window 9–10x / 107–108x / 1,213–1,217x, against 22–27x / 225–248x / 2,040–2,200x and 58–62x / 488–508x / 4,850–4,970x in Go), and the Rust assembly has NEON `add.2d` vector adds in the brute-force range-sum closure, a plausible mechanism that I did not check in Go's assembly. Loops that are not plain sums did not move (pair sum about 300x in both languages; the brute-force count about 141 µs in Rust and 142–158 µs in Go at n = 10³), and the window count through closures was about 3x faster in Rust (1.3–1.4 µs against 3.8–4.4 µs).

**Complexity:** `PairSum`, `DedupeAtMost`, `Partition3` and `LongestValid` are Θ(n), and the proof asserted their bounds: at most n − 1 steps (exactly n − 1 in all 11,070 no-pair cases), exactly n reads, exactly n iterations, and never more than 2n − 1 moves over 20,000 strings. `Prefix` is Θ(n) to build and exactly 2 reads per query, `Prefix2D` Θ(mn) with exactly 4 reads, and a `Diff` range add exactly 2 writes. The nine rows ran 279,489 cases with 0 mismatches against the oracles. With values near 2⁵⁹ the first negative prefix is `P[16]` in all 200 arrays; the longest range whose true sum fits is 15 elements and all 43,200 such ranges were exact, while all 3,000 ranges above MaxInt64 came back as a different number and all 42,000 reversed ranges were rejected. In the timing table the technique won all 13 rows where brute force ran, in every run (the machine was shared, so the ratios moved). brute/fast was about 22–27x, 225–248x and 2,040–2,200x for range sums at n = 10³, 10⁴ and 10⁵ (brute force sums about 245, 2,438 and 24,261 elements per query on average; the technique makes 3 operations); 295–305x and 2,600–3,020x for pair sum at 10³ and 10⁴ (10⁵ is not run: about 5·10⁹ pair checks); 58–62x, 488–508x and 4,850–4,970x for the fixed-window max sum; 32–37x and 184–250x for the subarray count; and 62–75x, 181–196x and 229–257x for 2-D rectangle sums at side 100, 300 and 1,000. For the whole-call tasks, brute-force operations grew by about ×100 per decade of n and the technique's by ×10, which is the quadratic count against the linear one; per query, brute force grew by ×10 per decade and the technique stayed at 3. The Go program is about 950 lines, over the size guide, because it is a capstone.

---

> Next lesson: [17 — Binary Trees & Traversals](../../17-binary-trees.md) — the node struct, `nil` as
> the base case, and recursion as the natural shape of the data.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
