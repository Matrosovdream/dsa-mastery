# Step 10 — Hash Tables & Sets · 🔴 Hard

Examples **12–16**: the Θ(n²) → Θ(n) rearrangement, the prefix-sum map, the attack that made every
language randomise its hash, the seven cases where a map is wrong, and the capstone.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 12 and 13 verify against brute force before they benchmark. All five report timings, sampled
on an Apple M4 with Go 1.26.3.

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

## 12. The rearrangement: two-sum and group anagrams

`🔴 hard` · *turn the inner loop into a membership question*

A whole family of problems collapses from Θ(n²) the moment you notice the inner loop is asking "is
there a `y` such that…". Rearrange it to "is `g(x)` in the set of things I have already seen" and the
loop disappears.

**Steps:**

1. Do the algebra, then find the bug that lives in the **order of two lines**.
2. Verify against brute force on inputs with duplicates.
3. Then pick a canonical key three ways, one of which overflows.

```go
package main

import (
	"fmt"
	"math/rand"
	"slices"
	"sort"
	"strings"
	"testing"
)

// A family of problems collapses from Theta(n^2) to Theta(n) the moment you
// realise the inner loop is asking a MEMBERSHIP question. The pattern is always
// the same, and it is worth being able to state:
//
//	"for each x, is there a y such that f(x, y) holds?"
//	  -> rearrange to "is g(x) in the set of things I have already seen?"
//
// The rearrangement is the whole trick. Once the question is "is this key
// present", a map answers it in Theta(1) and the nested loop disappears.

// ---- two-sum ---------------------------------------------------------------

// twoSumBrute is the honest Theta(n^2) baseline.
func twoSumBrute(nums []int, target int) (int, int, bool) {
	for i := 0; i < len(nums); i++ {
		for j := i + 1; j < len(nums); j++ {
			if nums[i]+nums[j] == target {
				return i, j, true
			}
		}
	}
	return 0, 0, false
}

// twoSum is the same problem after the rearrangement:
//
//	nums[i] + nums[j] == target
//	nums[j] == target - nums[i]     <- now it is a membership question
//
// The subtlety is WHERE the insert goes. Inserting before the lookup lets an
// element pair with itself; inserting after is correct.
func twoSum(nums []int, target int) (int, int, bool) {
	seen := make(map[int]int, len(nums))
	for j, v := range nums {
		if i, ok := seen[target-v]; ok {
			return i, j, true
		}
		seen[v] = j // AFTER the lookup
	}
	return 0, 0, false
}

// twoSumSelfPair is the bug: insert first, then look up.
func twoSumSelfPair(nums []int, target int) (int, int, bool) {
	seen := make(map[int]int, len(nums))
	for j, v := range nums {
		seen[v] = j // BEFORE -- now v can find itself
		if i, ok := seen[target-v]; ok {
			return i, j, true
		}
	}
	return 0, 0, false
}

// ---- group anagrams --------------------------------------------------------

// The insight: anagrams are not equal, but they have a canonical form that is.
// Choosing that form is the whole design decision.

func sortKey(s string) string {
	b := []byte(s)
	sort.Slice(b, func(i, j int) bool { return b[i] < b[j] })
	return string(b)
}

// countKey is the same idea without the sort: a 26-slot histogram, which is an
// ARRAY, which is comparable, which means it can be a map key directly
// (example 9). Theta(len) instead of Theta(len log len), and no allocation.
type histogram [26]int

func countKey(s string) histogram {
	var h histogram
	for i := 0; i < len(s); i++ {
		h[s[i]-'a']++
	}
	return h
}

// primeKey is the clever one, and it is wrong. Each letter gets a prime; the
// product is order-independent. It overflows.
var primes = [26]uint64{2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41,
	43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101}

func primeKey(s string) uint64 {
	p := uint64(1)
	for i := 0; i < len(s); i++ {
		p *= primes[s[i]-'a']
	}
	return p
}

func groupBySort(words []string) [][]string {
	g := map[string][]string{}
	for _, w := range words {
		g[sortKey(w)] = append(g[sortKey(w)], w)
	}
	return collect(g)
}

func groupByCount(words []string) [][]string {
	g := map[histogram][]string{}
	for _, w := range words {
		k := countKey(w)
		g[k] = append(g[k], w)
	}
	out := make([][]string, 0, len(g))
	for _, v := range g {
		slices.Sort(v)
		out = append(out, v)
	}
	slices.SortFunc(out, func(a, b []string) int { return strings.Compare(a[0], b[0]) })
	return out
}

func collect(g map[string][]string) [][]string {
	out := make([][]string, 0, len(g))
	for _, v := range g {
		slices.Sort(v)
		out = append(out, v)
	}
	slices.SortFunc(out, func(a, b []string) int { return strings.Compare(a[0], b[0]) })
	return out
}

func shorten(s string) string {
	if len(s) <= 12 {
		return s
	}
	return s[:6] + "..." + s[len(s)-3:]
}

var sinkI int
var sinkB bool
var sinkG [][]string

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("TWO-SUM: the canonical example of the rearrangement.")
	fmt.Println()
	nums := []int{2, 7, 11, 15, 3, 6}
	for _, target := range []int{9, 26, 100} {
		i, j, ok := twoSum(nums, target)
		bi, bj, bok := twoSumBrute(nums, target)
		fmt.Printf("  target %-4d map -> (%d,%d,%v)   brute -> (%d,%d,%v)\n",
			target, i, j, ok, bi, bj, bok)
	}
	fmt.Println()
	fmt.Println("      nums[i] + nums[j] == target")
	fmt.Println("      nums[j] == target - nums[i]    <- a membership question")
	fmt.Println()
	fmt.Println("  the inner loop was searching for a value. A map already knows")
	fmt.Println("  whether it holds a value. Theta(n^2) -> Theta(n).")

	fmt.Println()
	fmt.Println("and the bug that lives in the ORDER of two lines:")
	fmt.Println()
	self := []int{3, 5, 8}
	i, j, ok := twoSum(self, 6)
	bi, bj, bok := twoSumSelfPair(self, 6)
	fmt.Printf("  nums=%v target=6\n", self)
	fmt.Printf("  %-30s (%d,%d,%v)   correct: 3+3 is not a pair\n", "insert AFTER the lookup", i, j, ok)
	fmt.Printf("  %-30s (%d,%d,%v)   wrong: index 0 paired with itself\n", "insert BEFORE the lookup", bi, bj, bok)
	fmt.Println()
	fmt.Println("  both versions are three lines and differ only in their order.")
	fmt.Println("  The broken one passes every test whose target is not exactly")
	fmt.Println("  twice some element -- so test with one that is.")

	fmt.Println()
	fmt.Println("verified against brute force on 20,000 random cases:")
	fmt.Println()
	rng := rand.New(rand.NewSource(7))
	mismatch, selfBug, cases := 0, 0, 0
	for trial := 0; trial < 20_000; trial++ {
		n := 1 + rng.Intn(12)
		xs := make([]int, n)
		for i := range xs {
			xs[i] = rng.Intn(21) - 10 // small range: forces duplicates
		}
		target := rng.Intn(41) - 20
		cases++
		_, _, gotOK := twoSum(xs, target)
		_, _, wantOK := twoSumBrute(xs, target)
		if gotOK != wantOK {
			mismatch++
		}
		if _, _, bad := twoSumSelfPair(xs, target); bad != wantOK {
			selfBug++
		}
	}
	fmt.Printf("  %d cases: correct version %d mismatches, buggy version %d\n",
		cases, mismatch, selfBug)
	fmt.Println()
	fmt.Println("  the narrow value range is what finds it. Random ints from a wide")
	fmt.Println("  range almost never produce target == 2*x, so the bug survives.")
	fmt.Println("  Same discipline as lesson 09's sliding-window test.")

	fmt.Println()
	fmt.Println("the speedup, on the input where it matters (no early exit):")
	fmt.Println()
	fmt.Printf("  %10s %16s %16s %12s\n", "n", "brute ns", "map ns", "speedup")
	for _, n := range []int{100, 1000, 10_000, 50_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}
		target := -1 // never found, so both do all the work
		tb := nsPerOp(func() { _, _, sinkB = twoSumBrute(xs, target) })
		tm := nsPerOp(func() { _, _, sinkB = twoSum(xs, target) })
		fmt.Printf("  %10d %16.0f %16.0f %11.1fx\n", n, tb, tm, tb/tm)
	}
	fmt.Println()
	fmt.Println("  the ratio grows linearly with n, which is what Theta(n^2) against")
	fmt.Println("  Theta(n) looks like. Note the target that is never found: with an")
	fmt.Println("  easy target, brute force exits early and the benchmark lies.")

	fmt.Println()
	fmt.Println("GROUP ANAGRAMS: the same move, with a CANONICAL KEY.")
	fmt.Println()
	words := []string{"eat", "tea", "tan", "ate", "nat", "bat", "tab"}
	fmt.Printf("  input:  %v\n", words)
	fmt.Printf("  groups: %v\n", groupBySort(words))
	fmt.Println()
	fmt.Println("  two anagrams are not equal, so they cannot be a map key. But they")
	fmt.Println("  share a CANONICAL FORM that is, and that form becomes the key.")
	fmt.Println()
	fmt.Printf("  %-10s %-12s %v\n", "word", "sorted key", "histogram key")
	for _, w := range []string{"eat", "tea", "bat"} {
		h := countKey(w)
		fmt.Printf("  %-10s %-12q %v...\n", w, sortKey(w), h[:5])
	}

	fmt.Println()
	fmt.Println("three ways to build that key, and only two of them work:")
	fmt.Println()
	corpus := make([]string, 0, 20_000)
	letters := "abcdefghijklmnopqrstuvwxyz"
	for i := 0; i < 20_000; i++ {
		n := 3 + rng.Intn(8)
		b := make([]byte, n)
		for j := range b {
			b[j] = letters[rng.Intn(8)] // few letters -> real anagram groups
		}
		corpus = append(corpus, string(b))
	}
	tSort := nsPerOp(func() { sinkG = groupBySort(corpus) })
	tCount := nsPerOp(func() { sinkG = groupByCount(corpus) })
	gs, gc := len(groupBySort(corpus)), len(groupByCount(corpus))
	fmt.Printf("  %-34s %14.0f ns   %d groups\n", "sort each word (string key)", tSort, gs)
	fmt.Printf("  %-34s %14.0f ns   %d groups   %.1fx faster\n",
		"count letters ([26]int key)", tCount, gc, tSort/tCount)
	fmt.Println()
	fmt.Println("  the histogram key is Theta(len) instead of Theta(len log len),")
	fmt.Println("  and because [26]int is comparable it is a map key with no")
	fmt.Println("  allocation at all -- no []byte, no string conversion.")

	fmt.Println()
	fmt.Println("and the third way, which is clever and broken:")
	fmt.Println()
	fmt.Println("      assign each letter a prime; multiply. Order-independent!")
	fmt.Println()
	fmt.Printf("  primeKey(%q) = %d\n", "zzzzzzzzzz", primeKey(strings.Repeat("z", 10)))
	fmt.Printf("  primeKey(%q) = %d   <- 11 z's, and it went DOWN\n",
		"zzzzzzzzzzz", primeKey(strings.Repeat("z", 11)))
	fmt.Println()
	fmt.Println("  it wrapped. And once it can wrap, distinct words share keys --")
	fmt.Println("  here is a family of them, no search required:")
	fmt.Println()
	collisions := []string{
		strings.Repeat("a", 64),
		strings.Repeat("a", 65),
		strings.Repeat("a", 64) + "b",
		strings.Repeat("a", 70) + "xyz",
	}
	for _, w := range collisions {
		fmt.Printf("  %-18s len %-3d -> primeKey %d\n",
			fmt.Sprintf("%q", shorten(w)), len(w), primeKey(w))
	}
	fmt.Println()
	fmt.Println("  'a' is assigned the prime 2, so 64 of them is 2^64, which is 0")
	fmt.Println("  modulo 2^64. Every word above has a different letter multiset and")
	fmt.Println("  they all hash to the same key, so groupByPrime would put them in")
	fmt.Println("  one group and call them anagrams.")
	fmt.Println()
	collide := 0
	for i := 0; i < len(collisions); i++ {
		for j := i + 1; j < len(collisions); j++ {
			if primeKey(collisions[i]) == primeKey(collisions[j]) &&
				countKey(collisions[i]) != countKey(collisions[j]) {
				collide++
			}
		}
	}
	fmt.Printf("  %d of the %d pairs above are false anagrams\n",
		collide, len(collisions)*(len(collisions)-1)/2)
	fmt.Println()
	fmt.Println("  I ran this over a 20,000-word corpus first and found ZERO")
	fmt.Println("  collisions, because those words were under 11 letters and never")
	fmt.Println("  overflowed. The scheme is correct for short inputs and silently")
	fmt.Println("  wrong for long ones -- the worst possible failure mode, and one")
	fmt.Println("  that a corpus test will not find for you.")
	fmt.Println()
	fmt.Println("  prefer the boring key. [26]int is faster than sorting, allocates")
	fmt.Println("  nothing, and cannot overflow.")
	sinkI = collide
}
```

**Sample output:**

```
TWO-SUM: the canonical example of the rearrangement.

  target 9    map -> (0,1,true)   brute -> (0,1,true)
  target 26   map -> (2,3,true)   brute -> (2,3,true)
  target 100  map -> (0,0,false)   brute -> (0,0,false)

      nums[i] + nums[j] == target
      nums[j] == target - nums[i]    <- a membership question

  the inner loop was searching for a value. A map already knows
  whether it holds a value. Theta(n^2) -> Theta(n).

and the bug that lives in the ORDER of two lines:

  nums=[3 5 8] target=6
  insert AFTER the lookup        (0,0,false)   correct: 3+3 is not a pair
  insert BEFORE the lookup       (0,0,true)   wrong: index 0 paired with itself

  both versions are three lines and differ only in their order.
  The broken one passes every test whose target is not exactly
  twice some element -- so test with one that is.

verified against brute force on 20,000 random cases:

  20000 cases: correct version 0 mismatches, buggy version 1460

  the narrow value range is what finds it. Random ints from a wide
  range almost never produce target == 2*x, so the bug survives.
  Same discipline as lesson 09's sliding-window test.

the speedup, on the input where it matters (no early exit):

           n         brute ns           map ns      speedup
         100             1727             1259         1.4x
        1000           145334            11634        12.5x
       10000         13335630           148086        90.1x
       50000        331491896           967427       342.7x

  the ratio grows linearly with n, which is what Theta(n^2) against
  Theta(n) looks like. Note the target that is never found: with an
  easy target, brute force exits early and the benchmark lies.

GROUP ANAGRAMS: the same move, with a CANONICAL KEY.

  input:  [eat tea tan ate nat bat tab]
  groups: [[ate eat tea] [bat tab] [nat tan]]

  two anagrams are not equal, so they cannot be a map key. But they
  share a CANONICAL FORM that is, and that form becomes the key.

  word       sorted key   histogram key
  eat        "aet"        [1 0 0 0 1]...
  tea        "aet"        [1 0 0 0 1]...
  bat        "abt"        [1 1 0 0 0]...

three ways to build that key, and only two of them work:

  sort each word (string key)              10082286 ns   9314 groups
  count letters ([26]int key)               3329560 ns   9314 groups   3.0x faster

  the histogram key is Theta(len) instead of Theta(len log len),
  and because [26]int is comparable it is a map key with no
  allocation at all -- no []byte, no string conversion.

and the third way, which is clever and broken:

      assign each letter a prime; multiply. Order-independent!

  primeKey("zzzzzzzzzz") = 18228492172572692921
  primeKey("zzzzzzzzzzz") = 14850046132596375037   <- 11 z's, and it went DOWN

  it wrapped. And once it can wrap, distinct words share keys --
  here is a family of them, no search required:

  "aaaaaa...aaa"     len 64  -> primeKey 0
  "aaaaaa...aaa"     len 65  -> primeKey 0
  "aaaaaa...aab"     len 65  -> primeKey 0
  "aaaaaa...xyz"     len 73  -> primeKey 0

  'a' is assigned the prime 2, so 64 of them is 2^64, which is 0
  modulo 2^64. Every word above has a different letter multiset and
  they all hash to the same key, so groupByPrime would put them in
  one group and call them anagrams.

  6 of the 6 pairs above are false anagrams

  I ran this over a 20,000-word corpus first and found ZERO
  collisions, because those words were under 11 letters and never
  overflowed. The scheme is correct for short inputs and silently
  wrong for long ones -- the worst possible failure mode, and one
  that a corpus test will not find for you.

  prefer the boring key. [26]int is faster than sorting, allocates
  nothing, and cannot overflow.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::hash_map::RandomState;
use std::collections::{HashMap, HashSet};
use std::hash::{BuildHasher, BuildHasherDefault, Hash, Hasher};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// A family of problems collapses from Theta(n^2) to Theta(n) the moment you
// realise the inner loop is asking a MEMBERSHIP question. The pattern is always
// the same, and it is worth being able to state:
//
//	"for each x, is there a y such that f(x, y) holds?"
//	  -> rearrange to "is g(x) in the set of things I have already seen?"
//
// The rearrangement is the whole trick. Once the question is "is this key
// present", a map answers it in Theta(1) and the nested loop disappears.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Counting allocator: stand-in for testing.AllocsPerRun. Rust has no GC, so an
// allocation count is exact and repeatable.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)

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

/// allocation events while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> usize {
    let a0 = ALLOCS.load(Relaxed);
    black_box(f());
    ALLOCS.load(Relaxed) - a0
}

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

/// stand-in for math/rand (splitmix64; seeded, so runs are reproducible).
/// The stream is NOT Go's `rand.NewSource(7)` stream, so every count that
/// depends on random inputs differs from the Go sample (see the note).
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// ---- two-sum ---------------------------------------------------------------

// Go returns (i, j, ok); Rust has a type for "maybe a pair": Option<(usize, usize)>.

/// the honest Theta(n^2) baseline.
fn two_sum_brute(nums: &[i64], target: i64) -> Option<(usize, usize)> {
    for (i, &a) in nums.iter().enumerate() {
        // "is there a j > i with nums[j] == target - a?" is `position`; the
        // map version below answers the same question without the scan
        if let Some(off) = nums[i + 1..].iter().position(|&b| a + b == target) {
            return Some((i, i + 1 + off));
        }
    }
    None
}

// two_sum is the same problem after the rearrangement:
//
//	nums[i] + nums[j] == target
//	nums[j] == target - nums[i]     <- now it is a membership question
//
// The subtlety is WHERE the insert goes. Inserting before the lookup lets an
// element pair with itself; inserting after is correct.
fn two_sum(nums: &[i64], target: i64) -> Option<(usize, usize)> {
    let mut seen: HashMap<i64, usize> = HashMap::with_capacity(nums.len());
    for (j, &v) in nums.iter().enumerate() {
        if let Some(&i) = seen.get(&(target - v)) {
            return Some((i, j));
        }
        seen.insert(v, j); // AFTER the lookup
    }
    None
}

/// the bug: insert first, then look up.
fn two_sum_self_pair(nums: &[i64], target: i64) -> Option<(usize, usize)> {
    let mut seen: HashMap<i64, usize> = HashMap::with_capacity(nums.len());
    for (j, &v) in nums.iter().enumerate() {
        seen.insert(v, j); // BEFORE -- now v can find itself
        if let Some(&i) = seen.get(&(target - v)) {
            return Some((i, j));
        }
    }
    None
}

// ---- group anagrams --------------------------------------------------------

// The insight: anagrams are not equal, but they have a canonical form that is.
// Choosing that form is the whole design decision.

/// sorted bytes. A Vec<u8> is Hash + Eq, so it is a key as it stands; Go has to
/// convert back to a string, because a slice is not comparable there.
fn sort_key(s: &str) -> Vec<u8> {
    let mut b = s.as_bytes().to_vec();
    b.sort_unstable();
    b
}

// count_key is the same idea without the sort: a 26-slot histogram, which is an
// ARRAY, which is Hash + Eq (for any length: arrays are generic over N), which
// means it can be a map key directly (example 9). Theta(len) instead of
// Theta(len log len), and no heap allocation. u32 counts, 104 bytes (Go's
// [26]int is 208): a word would need 4 billion of one letter to overflow.
type Histogram = [u32; 26];

fn count_key(s: &str) -> Histogram {
    let mut h = [0; 26];
    for b in s.bytes() {
        h[(b - b'a') as usize] += 1; // a byte outside a..=z panics here, as in Go
    }
    h
}

// prime_key is the clever one, and it is wrong. Each letter gets a prime; the
// product is order-independent. It overflows -- and in Rust the wraparound has
// to be spelled out: a plain `*=` panics in a debug build, wraps in `-O`.
const PRIMES: [u64; 26] = [
    2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97,
    101,
];

fn prime_key(s: &str) -> u64 {
    s.bytes()
        .fold(1u64, |p, b| p.wrapping_mul(PRIMES[(b - b'a') as usize]))
}

// ---- a second hasher, to see what the key costs ----
//
// `Fast` is the same shape as example 2's finalized FNV-1a: one xor-multiply
// round per 8-byte word (an i64 key is exactly one round; bit-identical to
// example 7's `Fast` on integers), then the splitmix64 finalizer. It takes a
// word per step rather than a byte, so a 104-byte key costs 13 rounds, not 104.
// It has no random seed, so it is exactly the hash an attacker could aim at
// (example 14).
const FNV_OFFSET: u64 = 14695981039346656037;
const FNV_PRIME: u64 = 1099511628211;

#[derive(Clone, Copy)]
struct Fast(u64);
impl Default for Fast {
    fn default() -> Self {
        Fast(FNV_OFFSET)
    }
}
impl Hasher for Fast {
    fn write(&mut self, bytes: &[u8]) {
        for chunk in bytes.chunks(8) {
            let mut w = [0u8; 8];
            w[..chunk.len()].copy_from_slice(chunk);
            self.write_u64(u64::from_le_bytes(w));
        }
    }
    fn write_u64(&mut self, x: u64) {
        self.0 = (self.0 ^ x).wrapping_mul(FNV_PRIME);
    }
    fn finish(&self) -> u64 {
        let mut h = self.0;
        h ^= h >> 30;
        h = h.wrapping_mul(0xbf58476d1ce4e5b9);
        h ^= h >> 27;
        h = h.wrapping_mul(0x94d049bb133111eb);
        h ^ (h >> 31)
    }
}

/// Group `words` by `key(word)`. `S` picks the hasher: std's default is
/// `RandomState` (SipHash-1-3, a fresh random key per map).
fn group_by<'a, K: Hash + Eq, S: BuildHasher + Default>(
    words: &[&'a str],
    key: impl Fn(&str) -> K,
) -> Vec<Vec<&'a str>> {
    let mut g: HashMap<K, Vec<&str>, S> = HashMap::default();
    for &w in words {
        // the entry API computes the key once; Go's
        // `g[sortKey(w)] = append(g[sortKey(w)], w)` computes it twice
        g.entry(key(w)).or_default().push(w);
    }
    collect(g.into_values())
}

fn group_by_sort<'a>(words: &[&'a str]) -> Vec<Vec<&'a str>> {
    group_by::<_, RandomState>(words, sort_key)
}

fn group_by_count<'a>(words: &[&'a str]) -> Vec<Vec<&'a str>> {
    group_by::<_, RandomState>(words, count_key)
}

/// sort inside each group, then sort the groups by their first word, so the
/// result does not depend on the map's (random) iteration order.
fn collect<'a>(groups: impl Iterator<Item = Vec<&'a str>>) -> Vec<Vec<&'a str>> {
    let mut out: Vec<Vec<&str>> = groups.collect();
    for v in &mut out {
        v.sort_unstable();
    }
    out.sort_unstable_by(|a, b| a[0].cmp(b[0]));
    out
}

fn shorten(s: &str) -> String {
    // ASCII only: slicing by byte offset would panic inside a multi-byte char
    if s.len() <= 12 {
        s.to_string()
    } else {
        format!("{}...{}", &s[..6], &s[s.len() - 3..])
    }
}

/// `[a b c]`, the way Go's %v prints a slice of strings.
fn show_words(ws: &[&str]) -> String {
    format!("[{}]", ws.join(" "))
}

fn show_groups(g: &[Vec<&str>]) -> String {
    let inner: Vec<String> = g.iter().map(|v| show_words(v)).collect();
    format!("[{}]", inner.join(" "))
}

/// `[1 0 0 0 1]`, the way Go's %v prints a slice of ints.
fn show_nums(xs: &[u32]) -> String {
    let inner: Vec<String> = xs.iter().map(u32::to_string).collect();
    format!("[{}]", inner.join(" "))
}

fn main() {
    println!("TWO-SUM: the canonical example of the rearrangement.");
    println!();
    let nums = [2, 7, 11, 15, 3, 6];
    for target in [9, 26, 100] {
        println!(
            "  target {:<4} map -> {:?}   brute -> {:?}",
            target,
            two_sum(&nums, target),
            two_sum_brute(&nums, target)
        );
    }
    println!();
    println!("      nums[i] + nums[j] == target");
    println!("      nums[j] == target - nums[i]    <- a membership question");
    println!();
    println!("  the inner loop was searching for a value. A map already knows");
    println!("  whether it holds a value. Theta(n^2) -> Theta(n).");

    println!();
    println!("and the bug that lives in the ORDER of two lines:");
    println!();
    let selfs = [3, 5, 8];
    println!("  nums={selfs:?} target=6");
    println!(
        "  {:<30} {:?}   correct: 3+3 is not a pair",
        "insert AFTER the lookup",
        two_sum(&selfs, 6)
    );
    println!(
        "  {:<30} {:?}   wrong: index 0 paired with itself",
        "insert BEFORE the lookup",
        two_sum_self_pair(&selfs, 6)
    );
    println!();
    println!("  both versions are three lines and differ only in their order.");
    println!("  The broken one passes every test whose target is not exactly");
    println!("  twice some element -- so test with one that is.");

    println!();
    println!("verified against brute force on 20,000 random cases:");
    println!();
    let mut rng = Rng(7);
    let (mut mismatch, mut self_bug, mut cases) = (0, 0, 0);
    for _ in 0..20_000 {
        let n = 1 + rng.below(12) as usize;
        // small range: forces duplicates
        let xs: Vec<i64> = (0..n).map(|_| rng.below(21) as i64 - 10).collect();
        let target = rng.below(41) as i64 - 20;
        cases += 1;
        let want_ok = two_sum_brute(&xs, target).is_some();
        if two_sum(&xs, target).is_some() != want_ok {
            mismatch += 1;
        }
        if two_sum_self_pair(&xs, target).is_some() != want_ok {
            self_bug += 1;
        }
    }
    println!("  {cases} cases: correct version {mismatch} mismatches, buggy version {self_bug}");
    println!();
    println!("  the narrow value range is what finds it. Random ints from a wide");
    println!("  range almost never produce target == 2*x, so the bug survives.");
    println!("  Same discipline as lesson 09's sliding-window test.");

    println!();
    println!("the speedup, on the input where it matters (no early exit):");
    println!();
    println!(
        "  {:>10} {:>16} {:>16} {:>12}",
        "n", "brute ns", "map ns", "speedup"
    );
    for n in [100usize, 1000, 10_000, 50_000] {
        let xs: Vec<i64> = (0..n as i64).map(|i| i * 2).collect();
        let target = -1; // never found, so both do all the work
        let tb = ns_per_op(|| {
            black_box(two_sum_brute(black_box(&xs), black_box(target)));
        });
        let tm = ns_per_op(|| {
            black_box(two_sum(black_box(&xs), black_box(target)));
        });
        println!("  {:>10} {:>16.0} {:>16.0} {:>11.1}x", n, tb, tm, tb / tm);
    }
    println!();
    println!("  the ratio keeps growing with n, roughly in proportion to it, which is");
    println!("  what Theta(n^2) against Theta(n) looks like. Note the target that is");
    println!("  never found: with an easy target, brute force exits early and the");
    println!("  benchmark lies.");

    println!();
    println!("GROUP ANAGRAMS: the same move, with a CANONICAL KEY.");
    println!();
    let words = ["eat", "tea", "tan", "ate", "nat", "bat", "tab"];
    println!("  input:  {}", show_words(&words));
    println!("  groups: {}", show_groups(&group_by_sort(&words)));
    println!();
    println!("  two anagrams are not equal, so they cannot be a map key. But they");
    println!("  share a CANONICAL FORM that is, and that form becomes the key.");
    println!();
    println!("  {:<10} {:<12} histogram key", "word", "sorted key");
    for w in ["eat", "tea", "bat"] {
        let h = count_key(w);
        let key = format!("{:?}", String::from_utf8_lossy(&sort_key(w)));
        println!("  {:<10} {:<12} {}...", w, key, show_nums(&h[..5]));
    }

    println!();
    println!("three ways to build that key, and only two of them work:");
    println!();
    let letters = b"abcdefghijklmnopqrstuvwxyz";
    let corpus: Vec<String> = (0..20_000)
        .map(|_| {
            let n = 3 + rng.below(8) as usize;
            // few letters -> real anagram groups
            (0..n)
                .map(|_| letters[rng.below(8) as usize] as char)
                .collect()
        })
        .collect();
    let corpus: Vec<&str> = corpus.iter().map(String::as_str).collect();
    // The machine is shared and these differ by a small factor, so take the
    // minimum of five interleaved samples of each: noise only ever adds time.
    type Run<'c> = Box<dyn Fn() -> usize + 'c>;
    let rows: Vec<(&str, &str, Run)> = vec![
        (
            "sort each word (Vec<u8> key)",
            "SipHash",
            Box::new(|| group_by::<_, RandomState>(&corpus, sort_key).len()),
        ),
        (
            "count letters ([u32; 26] key)",
            "SipHash",
            Box::new(|| group_by::<_, RandomState>(&corpus, count_key).len()),
        ),
        (
            "sort each word (Vec<u8> key)",
            "Fast",
            Box::new(|| group_by::<_, BuildHasherDefault<Fast>>(&corpus, sort_key).len()),
        ),
        (
            "count letters ([u32; 26] key)",
            "Fast",
            Box::new(|| group_by::<_, BuildHasherDefault<Fast>>(&corpus, count_key).len()),
        ),
    ];
    let mut best = [f64::INFINITY; 4];
    for _ in 0..5 {
        for (b, (_, _, run)) in best.iter_mut().zip(&rows) {
            *b = b.min(ns_per_op(|| {
                black_box(run());
            }));
        }
    }
    println!("  {:<34} {:<8} {:>12}   groups", "key", "hasher", "ns");
    for (i, (label, hasher, run)) in rows.iter().enumerate() {
        let vs_sort = if i % 2 == 1 {
            format!("   {:.1}x faster than the sort key", best[i - 1] / best[i])
        } else {
            String::new()
        };
        println!(
            "  {:<34} {:<8} {:>12.0}   {}{}",
            label,
            hasher,
            best[i],
            run(),
            vs_sort
        );
    }
    let (a_sort, a_count) = (
        allocs_of(|| group_by_sort(&corpus)),
        allocs_of(|| group_by_count(&corpus)),
    );
    println!();
    println!(
        "  allocation events per call: {a_sort} for the sort key, {a_count} for the histogram key"
    );
    println!(
        "  (difference {}, for {} words: one Vec<u8> per word)",
        a_sort - a_count,
        corpus.len()
    );
    println!();
    println!("  the histogram key is Theta(len) instead of Theta(len log len),");
    println!("  and because [u32; 26] is Hash + Eq it is a map key with no heap");
    println!("  allocation at all -- no Vec<u8> to build, no string conversion.");
    println!();
    // how much does the hasher alone move each row?
    let (sort_gain, count_gain) = (best[0] / best[2], best[1] / best[3]);
    if best[1] < best[0] && best[3] < best[2] {
        println!("  the histogram key is the faster one under both hashers, but by much less");
    } else {
        println!("  the histogram key was NOT faster under both hashers in this run, and its");
        println!("  edge is in any case far smaller");
    }
    println!("  than Theta(len) against Theta(len log len) suggests: these words are 3 to");
    println!("  10 letters, log2(10) is about 3, and the key it builds is 104 bytes to");
    println!("  move and hash. Swapping SipHash for Fast speeds the sort-key version up");
    println!("  {sort_gain:.2}x and the histogram version {count_gain:.2}x");
    if count_gain > sort_gain {
        println!("  -- the long key is the one that pays for hashing.");
    } else {
        println!("  -- no more for the long key in this run.");
    }

    println!();
    println!("and the third way, which is clever and broken:");
    println!();
    println!("      assign each letter a prime; multiply. Order-independent!");
    println!();
    println!(
        "  prime_key({:?}) = {}",
        "z".repeat(10),
        prime_key(&"z".repeat(10))
    );
    println!(
        "  prime_key({:?}) = {}   <- 11 z's, and it went DOWN",
        "z".repeat(11),
        prime_key(&"z".repeat(11))
    );
    println!();
    println!("  it wrapped. And once it can wrap, distinct words share keys --");
    println!("  here is a family of them, no search required:");
    println!();
    let collisions = [
        "a".repeat(64),
        "a".repeat(65),
        "a".repeat(64) + "b",
        "a".repeat(70) + "xyz",
    ];
    for w in &collisions {
        println!(
            "  {:<18} len {:<3} -> prime_key {}",
            format!("{:?}", shorten(w)),
            w.len(),
            prime_key(w)
        );
    }
    println!();
    println!("  'a' is assigned the prime 2, so 64 of them is 2^64, which is 0");
    println!("  modulo 2^64. Every word above has a different letter multiset and");
    println!("  they all hash to the same key, so group_by_prime would put them in");
    println!("  one group and call them anagrams.");
    println!();
    let mut collide = 0;
    for (i, a) in collisions.iter().enumerate() {
        for b in &collisions[i + 1..] {
            if prime_key(a) == prime_key(b) && count_key(a) != count_key(b) {
                collide += 1;
            }
        }
    }
    println!(
        "  {} of the {} pairs above are false anagrams",
        collide,
        collisions.len() * (collisions.len() - 1) / 2
    );
    println!();
    // The Go program says it found zero collisions on its corpus. prime_key is a
    // function of the histogram, so it can only merge histograms, never split
    // them: equal counts of distinct keys means it merged none.
    let primes: HashSet<u64> = corpus.iter().map(|w| prime_key(w)).collect();
    let hists: HashSet<Histogram> = corpus.iter().map(|w| count_key(w)).collect();
    println!(
        "  on the {}-word corpus above: {} distinct prime keys, {} distinct histograms",
        corpus.len(),
        primes.len(),
        hists.len()
    );
    println!();
    println!("  I ran this over a 20,000-word corpus first and found ZERO");
    println!("  collisions, because those words were under 11 letters and never");
    println!("  overflowed. The scheme is correct for short inputs and silently");
    println!("  wrong for long ones -- the worst possible failure mode, and one");
    println!("  that a corpus test will not find for you.");
    println!();
    println!("  prefer the boring key. [u32; 26] allocates nothing per word and cannot");
    println!("  overflow; the table above says how much faster it is.");
}
```

**Rust sample output:**

```
TWO-SUM: the canonical example of the rearrangement.

  target 9    map -> Some((0, 1))   brute -> Some((0, 1))
  target 26   map -> Some((2, 3))   brute -> Some((2, 3))
  target 100  map -> None   brute -> None

      nums[i] + nums[j] == target
      nums[j] == target - nums[i]    <- a membership question

  the inner loop was searching for a value. A map already knows
  whether it holds a value. Theta(n^2) -> Theta(n).

and the bug that lives in the ORDER of two lines:

  nums=[3, 5, 8] target=6
  insert AFTER the lookup        None   correct: 3+3 is not a pair
  insert BEFORE the lookup       Some((0, 0))   wrong: index 0 paired with itself

  both versions are three lines and differ only in their order.
  The broken one passes every test whose target is not exactly
  twice some element -- so test with one that is.

verified against brute force on 20,000 random cases:

  20000 cases: correct version 0 mismatches, buggy version 1499

  the narrow value range is what finds it. Random ints from a wide
  range almost never produce target == 2*x, so the bug survives.
  Same discipline as lesson 09's sliding-window test.

the speedup, on the input where it matters (no early exit):

           n         brute ns           map ns      speedup
         100             2049             2141         1.0x
        1000           181438            20597         8.8x
       10000         16705240           200756        83.2x
       50000        419004250          2258172       185.6x

  the ratio keeps growing with n, roughly in proportion to it, which is
  what Theta(n^2) against Theta(n) looks like. Note the target that is
  never found: with an easy target, brute force exits early and the
  benchmark lies.

GROUP ANAGRAMS: the same move, with a CANONICAL KEY.

  input:  [eat tea tan ate nat bat tab]
  groups: [[ate eat tea] [bat tab] [nat tan]]

  two anagrams are not equal, so they cannot be a map key. But they
  share a CANONICAL FORM that is, and that form becomes the key.

  word       sorted key   histogram key
  eat        "aet"        [1 0 0 0 1]...
  tea        "aet"        [1 0 0 0 1]...
  bat        "abt"        [1 1 0 0 0]...

three ways to build that key, and only two of them work:

  key                                hasher             ns   groups
  sort each word (Vec<u8> key)       SipHash       4553393   9275
  count letters ([u32; 26] key)      SipHash       3723339   9275   1.2x faster than the sort key
  sort each word (Vec<u8> key)       Fast          4049221   9275
  count letters ([u32; 26] key)      Fast          2903830   9275   1.4x faster than the sort key

  allocation events per call: 30433 for the sort key, 10433 for the histogram key
  (difference 20000, for 20000 words: one Vec<u8> per word)

  the histogram key is Theta(len) instead of Theta(len log len),
  and because [u32; 26] is Hash + Eq it is a map key with no heap
  allocation at all -- no Vec<u8> to build, no string conversion.

  the histogram key is the faster one under both hashers, but by much less
  than Theta(len) against Theta(len log len) suggests: these words are 3 to
  10 letters, log2(10) is about 3, and the key it builds is 104 bytes to
  move and hash. Swapping SipHash for Fast speeds the sort-key version up
  1.12x and the histogram version 1.28x
  -- the long key is the one that pays for hashing.

and the third way, which is clever and broken:

      assign each letter a prime; multiply. Order-independent!

  prime_key("zzzzzzzzzz") = 18228492172572692921
  prime_key("zzzzzzzzzzz") = 14850046132596375037   <- 11 z's, and it went DOWN

  it wrapped. And once it can wrap, distinct words share keys --
  here is a family of them, no search required:

  "aaaaaa...aaa"     len 64  -> prime_key 0
  "aaaaaa...aaa"     len 65  -> prime_key 0
  "aaaaaa...aab"     len 65  -> prime_key 0
  "aaaaaa...xyz"     len 73  -> prime_key 0

  'a' is assigned the prime 2, so 64 of them is 2^64, which is 0
  modulo 2^64. Every word above has a different letter multiset and
  they all hash to the same key, so group_by_prime would put them in
  one group and call them anagrams.

  6 of the 6 pairs above are false anagrams

  on the 20000-word corpus above: 9275 distinct prime keys, 9275 distinct histograms

  I ran this over a 20,000-word corpus first and found ZERO
  collisions, because those words were under 11 letters and never
  overflowed. The scheme is correct for short inputs and silently
  wrong for long ones -- the worst possible failure mode, and one
  that a corpus test will not find for you.

  prefer the boring key. [u32; 26] allocates nothing per word and cannot
  overflow; the table above says how much faster it is.
```

> **Go vs Rust:** Fed to a copy of the Go program, this file's splitmix64 gives identical counts (1,499 buggy cases, 9,275 groups, 6 of 6 false pairs, the same wrapped prime keys); the Go sample's 1,460 and 9,314 differ only because `rand.NewSource(7)` is a different stream. The Go prose's "3.0x faster" for the histogram key does not reproduce: it is 1.1-1.3x under std's SipHash and 1.3-1.5x under the `Fast` hasher over 8 runs, and part of Go's gap is `g[sortKey(w)] = append(g[sortKey(w)], w)` sorting every word twice (with the key computed once, the Go program was 1.4-2.2x in 4 runs here; allocations per call: 175,730 twice, 95,730 once, 25,005 for the histogram, against Rust's 30,433 and 10,433 with `entry`, which computes the key once). Two-sum at n=50,000 is 349-400x faster with the map (Go sample: 343x); `Option<(usize, usize)>` replaces Go's `(i, j, ok)`, the wraparound in `prime_key` is an explicit `wrapping_mul` (a plain `*` panics in a debug build and wraps under `-O`), and `[u32; 26]` is `Hash + Eq` at any length, at half the size of Go's `[26]int`.

**Complexity:** Θ(n²) → Θ(n), measured at **343×** by n=50,000 · inserting *before* the lookup lets an
element pair with itself and fails **1,460 of 20,000** random cases · a `[26]int` histogram key beats
sorting by **2.5×** and cannot overflow, unlike the prime-product key — where 64 a's give **0**

---

## 13. Longest consecutive sequence, and the prefix-sum map

`🔴 hard` · *one where the map wins, and one where it loses*

Two more Θ(n) rewrites. One replaces a **sort** and the aggregate argument is subtle; the other
replaces a **nested loop** and holds prefix sums. Only one of them is actually faster.

**Steps:**

1. Write the set version, then delete the one-line guard and watch it go quadratic.
2. Race it against `slices.Sort` — and lose.
3. Then do subarray-sum-equals-k, including the line everyone forgets.

```go
package main

import (
	"fmt"
	"math/rand"
	"slices"
	"testing"
)

// Two more problems that collapse to Theta(n) with a map, and each one teaches
// something the other does not:
//
//	LONGEST CONSECUTIVE SEQUENCE   the map replaces a SORT, and the Theta(n)
//	                               argument is an aggregate argument, not an
//	                               obvious one
//	SUBARRAY SUM EQUALS K          the map holds PREFIX SUMS, which is the
//	                               single most transferable trick in this file

// ---- longest consecutive sequence ------------------------------------------

// bySort is the obvious answer and it is Theta(n log n).
func bySort(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	s := slices.Clone(nums)
	slices.Sort(s)
	best, cur := 1, 1
	for i := 1; i < len(s); i++ {
		switch {
		case s[i] == s[i-1]:
		case s[i] == s[i-1]+1:
			cur++
			best = max(best, cur)
		default:
			cur = 1
		}
	}
	return best
}

// bySet is Theta(n), and the reason is the `if` on the third line of the loop.
func bySet(nums []int) int {
	set := make(map[int]struct{}, len(nums))
	for _, v := range nums {
		set[v] = struct{}{}
	}
	best := 0
	for v := range set {
		if _, ok := set[v-1]; ok {
			continue // v is not the START of a run -- skip it entirely
		}
		n := 1
		for {
			if _, ok := set[v+n]; !ok {
				break
			}
			n++
		}
		best = max(best, n)
	}
	return best
}

// bySetNoGuard drops that one line. It is still CORRECT, and it is quadratic.
func bySetNoGuard(nums []int, steps *int) int {
	set := make(map[int]struct{}, len(nums))
	for _, v := range nums {
		set[v] = struct{}{}
	}
	best := 0
	for v := range set {
		n := 1
		for {
			*steps++
			if _, ok := set[v+n]; !ok {
				break
			}
			n++
		}
		best = max(best, n)
	}
	return best
}

func bySetSteps(nums []int, steps *int) int {
	set := make(map[int]struct{}, len(nums))
	for _, v := range nums {
		set[v] = struct{}{}
	}
	best := 0
	for v := range set {
		if _, ok := set[v-1]; ok {
			continue
		}
		n := 1
		for {
			*steps++
			if _, ok := set[v+n]; !ok {
				break
			}
			n++
		}
		best = max(best, n)
	}
	return best
}

// ---- subarray sum equals k -------------------------------------------------

// The rearrangement, and it is worth writing out:
//
//	sum(i..j) == k
//	prefix[j] - prefix[i-1] == k
//	prefix[i-1] == prefix[j] - k        <- a membership question again
//
// So: walk once, keep a running prefix sum, and ask how many earlier prefixes
// equal (current - k). The map holds COUNTS, not positions, because several
// earlier prefixes can have the same value.
func subarraySum(nums []int, k int) int {
	counts := map[int]int{0: 1} // the empty prefix -- the line everyone forgets
	sum, total := 0, 0
	for _, v := range nums {
		sum += v
		total += counts[sum-k]
		counts[sum]++
	}
	return total
}

// subarraySumNoBase omits the {0: 1} seed.
func subarraySumNoBase(nums []int, k int) int {
	counts := map[int]int{}
	sum, total := 0, 0
	for _, v := range nums {
		sum += v
		total += counts[sum-k]
		counts[sum]++
	}
	return total
}

func subarraySumBrute(nums []int, k int) int {
	total := 0
	for i := range nums {
		sum := 0
		for j := i; j < len(nums); j++ {
			sum += nums[j]
			if sum == k {
				total++
			}
		}
	}
	return total
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("LONGEST CONSECUTIVE SEQUENCE: how long is the longest run of")
	fmt.Println("consecutive integers, in an unsorted slice with no duplicates rule?")
	fmt.Println()
	ex := []int{100, 4, 200, 1, 3, 2}
	fmt.Printf("  %v -> %d   (1,2,3,4)\n", ex, bySet(ex))
	fmt.Println()
	fmt.Println("  sorting answers it in Theta(n log n). A set answers it in")
	fmt.Println("  Theta(n), and the argument is not obvious:")
	fmt.Println()
	fmt.Println("      for v := range set {")
	fmt.Println("          if set has v-1 { continue }     // <- the whole trick")
	fmt.Println("          walk v, v+1, v+2, ... while present")
	fmt.Println("      }")
	fmt.Println()
	fmt.Println("  the guard means the inner walk only ever starts at the FIRST")
	fmt.Println("  element of a run. Every element is therefore visited by exactly")
	fmt.Println("  one walk, so the total inner work is Theta(n) no matter how the")
	fmt.Println("  runs are shaped. That is the aggregate argument from lessons 08")
	fmt.Println("  and 09, in a third disguise.")

	fmt.Println()
	fmt.Println("drop the guard and it is still correct, and quadratic:")
	fmt.Println()
	fmt.Printf("  %10s %16s %18s %12s\n", "n", "steps with guard", "steps without", "ratio")
	for _, n := range []int{1000, 4000, 16_000, 64_000} {
		run := make([]int, n)
		for i := range run {
			run[i] = i // ONE long run: the worst case for the unguarded version
		}
		var withG, without int
		a := bySetSteps(run, &withG)
		b := bySetNoGuard(run, &without)
		if a != b || a != n {
			panic("answers differ")
		}
		fmt.Printf("  %10d %16d %18d %11.1fx\n", n, withG, without, float64(without)/float64(withG))
	}
	fmt.Println()
	fmt.Println("  n QUADRUPLES down that table and so does the ratio -- 500, 2000,")
	fmt.Println("  8000, 32000. A ratio that grows linearly in n is Theta(n^2)")
	fmt.Println("  against Theta(n), and here the unguarded version does exactly")
	fmt.Println("  n(n+1)/2 steps against n.")
	fmt.Println()
	fmt.Println("  both functions return the same answer on every input. The guard")
	fmt.Println("  is not about correctness at all -- only about not re-walking a")
	fmt.Println("  run once per element of it.")

	fmt.Println()
	fmt.Println("against the sort, on the clock:")
	fmt.Println()
	rng := rand.New(rand.NewSource(11))
	fmt.Printf("  %10s %16s %16s %12s\n", "n", "sort ns", "set ns", "set/sort")
	for _, n := range []int{1000, 10_000, 100_000, 1_000_000} {
		xs := rng.Perm(n)
		ts := nsPerOp(func() { sinkI = bySort(xs) })
		tm := nsPerOp(func() { sinkI = bySet(xs) })
		fmt.Printf("  %10d %16.0f %16.0f %11.2fx\n", n, ts, tm, tm/ts)
	}
	fmt.Println()
	fmt.Println("  the Theta(n) algorithm LOSES at 1,000, 10,000 and 1,000,000 --")
	fmt.Println("  and WINS at 100,000. That is not noise; it reproduces at 0.58 to")
	fmt.Println("  0.61 across runs.")
	fmt.Println()
	fmt.Println("  sorting a permutation of 0..n-1 is Theta(n log n) over contiguous")
	fmt.Println("  memory with a branch-predictable comparison -- close to the")
	fmt.Println("  fastest thing a CPU does. The set version pays n hashes, n")
	fmt.Println("  inserts and n random-access lookups, and the log factor it saves")
	fmt.Println("  is worth only about 20 at a million elements.")
	fmt.Println()
	fmt.Println("  so the map wins in a BAND, not above a threshold. It needs n big")
	fmt.Println("  enough for log n to matter and small enough that the table still")
	fmt.Println("  sits in cache -- and 100,000 int keys is also one of the densest")
	fmt.Println("  points on example 6's bytes/key curve, which helps twice.")
	fmt.Println()
	fmt.Println("  this is lesson 03's warning in its sharpest form: 'I found the")
	fmt.Println("  Theta(n) solution' is not the end of the conversation, and the")
	fmt.Println("  crossover is not even guaranteed to be a single point.")

	fmt.Println()
	fmt.Println("SUBARRAY SUM EQUALS K: the prefix-sum map, which is the one to")
	fmt.Println("remember.")
	fmt.Println()
	nums := []int{1, 2, 3, -3, 1, 1, 1}
	for _, k := range []int{3, 2, 0} {
		fmt.Printf("  %v k=%-3d -> %d subarrays (brute: %d)\n",
			nums, k, subarraySum(nums, k), subarraySumBrute(nums, k))
	}
	fmt.Println()
	fmt.Println("      sum(i..j) == k")
	fmt.Println("      prefix[j] - prefix[i-1] == k")
	fmt.Println("      prefix[i-1] == prefix[j] - k     <- membership, again")
	fmt.Println()
	fmt.Println("  the map holds COUNTS of prefix sums seen so far, not positions,")
	fmt.Println("  because several prefixes can share a value -- which is exactly")
	fmt.Println("  what happens as soon as the input contains a zero or a negative.")

	fmt.Println()
	fmt.Println("and the one line everyone forgets:")
	fmt.Println()
	fmt.Println("      counts := map[int]int{0: 1}")
	fmt.Println()
	mism, base := 0, 0
	for trial := 0; trial < 20_000; trial++ {
		n := 1 + rng.Intn(10)
		xs := make([]int, n)
		for i := range xs {
			xs[i] = rng.Intn(7) - 3
		}
		k := rng.Intn(9) - 4
		want := subarraySumBrute(xs, k)
		if subarraySum(xs, k) != want {
			mism++
		}
		if subarraySumNoBase(xs, k) != want {
			base++
		}
	}
	fmt.Printf("  20,000 random cases vs brute force:\n")
	fmt.Printf("  %-34s %d mismatches\n", "with counts[0] = 1", mism)
	fmt.Printf("  %-34s %d mismatches\n", "without it", base)
	fmt.Println()
	fmt.Println("  the seed represents the EMPTY PREFIX -- it is what lets a")
	fmt.Println("  subarray starting at index 0 be counted. Without it you miss")
	fmt.Println("  exactly the subarrays that begin at the start, which is a large")
	fmt.Println("  minority of inputs and none of the ones you check by hand.")

	fmt.Println()
	fmt.Println("the speedup over the quadratic version:")
	fmt.Println()
	fmt.Printf("  %10s %16s %16s %12s\n", "n", "brute ns", "prefix map ns", "speedup")
	for _, n := range []int{1000, 10_000, 100_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = rng.Intn(21) - 10
		}
		tb := nsPerOp(func() { sinkI = subarraySumBrute(xs, 7) })
		tm := nsPerOp(func() { sinkI = subarraySum(xs, 7) })
		fmt.Printf("  %10d %16.0f %16.0f %11.0fx\n", n, tb, tm, tb/tm)
	}
	fmt.Println()
	fmt.Println("  here the map DOES win, and by a growing margin -- because the")
	fmt.Println("  alternative is genuinely Theta(n^2), not Theta(n log n). Compare")
	fmt.Println("  that with the previous problem, where the alternative was a sort")
	fmt.Println("  and the map lost.")
	fmt.Println()
	fmt.Println("  the rule that falls out: a map beats a NESTED LOOP reliably. It")
	fmt.Println("  beats a SORT only when n is large and the constant works out, and")
	fmt.Println("  you have to measure to know.")
	fmt.Println()
	fmt.Println("  the prefix-sum-in-a-map pattern generalises far past this")
	fmt.Println("  problem: subarray sums divisible by k (store sum mod k), longest")
	fmt.Println("  subarray with equal 0s and 1s (map -1/+1 and store the running")
	fmt.Println("  balance), and most of lesson 16.")
}
```

**Sample output:**

```
LONGEST CONSECUTIVE SEQUENCE: how long is the longest run of
consecutive integers, in an unsorted slice with no duplicates rule?

  [100 4 200 1 3 2] -> 4   (1,2,3,4)

  sorting answers it in Theta(n log n). A set answers it in
  Theta(n), and the argument is not obvious:

      for v := range set {
          if set has v-1 { continue }     // <- the whole trick
          walk v, v+1, v+2, ... while present
      }

  the guard means the inner walk only ever starts at the FIRST
  element of a run. Every element is therefore visited by exactly
  one walk, so the total inner work is Theta(n) no matter how the
  runs are shaped. That is the aggregate argument from lessons 08
  and 09, in a third disguise.

drop the guard and it is still correct, and quadratic:

           n steps with guard      steps without        ratio
        1000             1000             500500       500.5x
        4000             4000            8002000      2000.5x
       16000            16000          128008000      8000.5x
       64000            64000         2048032000     32000.5x

  n QUADRUPLES down that table and so does the ratio -- 500, 2000,
  8000, 32000. A ratio that grows linearly in n is Theta(n^2)
  against Theta(n), and here the unguarded version does exactly
  n(n+1)/2 steps against n.

  both functions return the same answer on every input. The guard
  is not about correctness at all -- only about not re-walking a
  run once per element of it.

against the sort, on the clock:

           n          sort ns           set ns     set/sort
        1000             9912            19672        1.98x
       10000           140700           228583        1.62x
      100000          4814193          2720815        0.57x
     1000000         59109892         86587048        1.46x

  the Theta(n) algorithm LOSES at 1,000, 10,000 and 1,000,000 --
  and WINS at 100,000. That is not noise; it reproduces at 0.58 to
  0.61 across runs.

  sorting a permutation of 0..n-1 is Theta(n log n) over contiguous
  memory with a branch-predictable comparison -- close to the
  fastest thing a CPU does. The set version pays n hashes, n
  inserts and n random-access lookups, and the log factor it saves
  is worth only about 20 at a million elements.

  so the map wins in a BAND, not above a threshold. It needs n big
  enough for log n to matter and small enough that the table still
  sits in cache -- and 100,000 int keys is also one of the densest
  points on example 6's bytes/key curve, which helps twice.

  this is lesson 03's warning in its sharpest form: 'I found the
  Theta(n) solution' is not the end of the conversation, and the
  crossover is not even guaranteed to be a single point.

SUBARRAY SUM EQUALS K: the prefix-sum map, which is the one to
remember.

  [1 2 3 -3 1 1 1] k=3   -> 6 subarrays (brute: 6)
  [1 2 3 -3 1 1 1] k=2   -> 5 subarrays (brute: 5)
  [1 2 3 -3 1 1 1] k=0   -> 2 subarrays (brute: 2)

      sum(i..j) == k
      prefix[j] - prefix[i-1] == k
      prefix[i-1] == prefix[j] - k     <- membership, again

  the map holds COUNTS of prefix sums seen so far, not positions,
  because several prefixes can share a value -- which is exactly
  what happens as soon as the input contains a zero or a negative.

and the one line everyone forgets:

      counts := map[int]int{0: 1}

  20,000 random cases vs brute force:
  with counts[0] = 1                 0 mismatches
  without it                         6769 mismatches

  the seed represents the EMPTY PREFIX -- it is what lets a
  subarray starting at index 0 be counted. Without it you miss
  exactly the subarrays that begin at the start, which is a large
  minority of inputs and none of the ones you check by hand.

the speedup over the quadratic version:

           n         brute ns    prefix map ns      speedup
        1000           134705            16291           8x
       10000         12764677           154698          83x
      100000       1335619125          1782248         749x

  here the map DOES win, and by a growing margin -- because the
  alternative is genuinely Theta(n^2), not Theta(n log n). Compare
  that with the previous problem, where the alternative was a sort
  and the map lost.

  the rule that falls out: a map beats a NESTED LOOP reliably. It
  beats a SORT only when n is large and the constant works out, and
  you have to measure to know.

  the prefix-sum-in-a-map pattern generalises far past this
  problem: subarray sums divisible by k (store sum mod k), longest
  subarray with equal 0s and 1s (map -1/+1 and store the running
  balance), and most of lesson 16.
```

**Rust:**

```rust
use std::collections::hash_map::RandomState;
use std::collections::{HashMap, HashSet};
use std::fmt::Display;
use std::hash::{BuildHasher, BuildHasherDefault, Hasher};
use std::hint::black_box;
use std::time::{Duration, Instant};

// Two more problems that collapse to Theta(n) with a map, and each one teaches
// something the other does not:
//
//	LONGEST CONSECUTIVE SEQUENCE   the map replaces a SORT, and the Theta(n)
//	                               argument is an aggregate argument, not an
//	                               obvious one
//	SUBARRAY SUM EQUALS K          the map holds PREFIX SUMS, which is the
//	                               single most transferable trick in this file

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

/// stand-in for math/rand (splitmix64; seeded, so runs are reproducible).
/// The stream is NOT Go's `rand.NewSource(11)` stream, so every count that
/// depends on random inputs differs from the Go sample (see the note).
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
    /// like rand.Perm(n): a random permutation of 0..n
    fn perm(&mut self, n: usize) -> Vec<i64> {
        let mut p: Vec<i64> = (0..n as i64).collect();
        for i in (1..n).rev() {
            p.swap(i, self.below(i as u64 + 1) as usize);
        }
        p
    }
}

// ---- a second hasher, so the timing comparison is fair ----
//
// std's default is SipHash-1-3 with a random key per set: built to resist
// collision attacks (example 14), and not built to be the fastest hash there
// is. `Fast` is the same shape as example 2's finalized FNV-1a: one xor-multiply
// round per 8-byte word (an i64 key is exactly one round; identical to example
// 7's `Fast` on integers), then the splitmix64 finalizer. It has no random
// seed, so it is exactly the hash an attacker could aim at.
const FNV_OFFSET: u64 = 14695981039346656037;
const FNV_PRIME: u64 = 1099511628211;

#[derive(Clone, Copy)]
struct Fast(u64);
impl Default for Fast {
    fn default() -> Self {
        Fast(FNV_OFFSET)
    }
}
impl Hasher for Fast {
    fn write(&mut self, bytes: &[u8]) {
        for chunk in bytes.chunks(8) {
            let mut w = [0u8; 8];
            w[..chunk.len()].copy_from_slice(chunk);
            self.write_u64(u64::from_le_bytes(w));
        }
    }
    fn write_u64(&mut self, x: u64) {
        self.0 = (self.0 ^ x).wrapping_mul(FNV_PRIME);
    }
    fn finish(&self) -> u64 {
        let mut h = self.0;
        h ^= h >> 30;
        h = h.wrapping_mul(0xbf58476d1ce4e5b9);
        h ^= h >> 27;
        h = h.wrapping_mul(0x94d049bb133111eb);
        h ^ (h >> 31)
    }
}

// ---- longest consecutive sequence ------------------------------------------

/// by_sort is the obvious answer and it is Theta(n log n).
/// (Go's slices.Sort is an unstable pattern-defeating quicksort; Rust's
/// `sort_unstable` is the unstable counterpart -- a different algorithm, but
/// the right one to compare, since the stable `sort` does extra work for order.)
fn by_sort(nums: &[i64]) -> usize {
    if nums.is_empty() {
        return 0;
    }
    let mut s = nums.to_vec();
    s.sort_unstable();
    let (mut best, mut cur) = (1, 1);
    for w in s.windows(2) {
        let (a, b) = (w[0], w[1]);
        if b == a {
            // a duplicate: neither extends nor breaks the run
        } else if a.checked_add(1) == Some(b) {
            cur += 1;
            best = best.max(cur);
        } else {
            cur = 1;
        }
    }
    best
}

/// by_set is Theta(n), and the reason is the `if` on the third line of the loop.
fn by_set<S: BuildHasher + Default>(nums: &[i64]) -> usize {
    let mut set: HashSet<i64, S> = HashSet::with_capacity_and_hasher(nums.len(), S::default());
    set.extend(nums);
    let mut best = 0;
    for &v in &set {
        // checked_sub: Go's `v-1` would wrap at the smallest int; here it is None
        if v.checked_sub(1).is_some_and(|p| set.contains(&p)) {
            continue; // v is not the START of a run -- skip it entirely
        }
        // walk v, v+1, v+2, ... while present (a range, so no overflow past MAX)
        let n = (v..=i64::MAX).take_while(|x| set.contains(x)).count();
        best = best.max(n);
    }
    best
}

// The two step-counting versions below use the `Fast` hasher: the count of steps
// does not depend on the hash, and the unguarded walk makes two BILLION lookups
// at n = 64,000, which SipHash would take about three times as long to answer.

/// by_set_no_guard drops that one line. It is still CORRECT, and it is quadratic.
/// `steps` counts every lookup made while walking.
fn by_set_no_guard(nums: &[i64], steps: &mut u64) -> usize {
    let set: HashSet<i64, BuildHasherDefault<Fast>> = nums.iter().copied().collect();
    let mut best = 0;
    for &v in &set {
        let mut n = 1;
        loop {
            *steps += 1;
            if !set.contains(&(v + n)) {
                break;
            }
            n += 1;
        }
        best = best.max(n as usize);
    }
    best
}

fn by_set_steps(nums: &[i64], steps: &mut u64) -> usize {
    let set: HashSet<i64, BuildHasherDefault<Fast>> = nums.iter().copied().collect();
    let mut best = 0;
    for &v in &set {
        if set.contains(&(v - 1)) {
            continue;
        }
        let mut n = 1;
        loop {
            *steps += 1;
            if !set.contains(&(v + n)) {
                break;
            }
            n += 1;
        }
        best = best.max(n as usize);
    }
    best
}

// ---- subarray sum equals k -------------------------------------------------

// The rearrangement, and it is worth writing out:
//
//	sum(i..j) == k
//	prefix[j] - prefix[i-1] == k
//	prefix[i-1] == prefix[j] - k        <- a membership question again
//
// So: walk once, keep a running prefix sum, and ask how many earlier prefixes
// equal (current - k). The map holds COUNTS, not positions, because several
// earlier prefixes can have the same value.
fn subarray_sum(nums: &[i64], k: i64) -> usize {
    let mut counts: HashMap<i64, usize> = HashMap::from([(0, 1)]); // the empty prefix -- the line everyone forgets
    let (mut sum, mut total) = (0, 0);
    for &v in nums {
        sum += v;
        // Go's `counts[sum-k]` is 0 for a missing key; Rust's `[]` would panic,
        // so say what absent means
        total += counts.get(&(sum - k)).copied().unwrap_or(0);
        *counts.entry(sum).or_insert(0) += 1;
    }
    total
}

/// subarray_sum_no_base omits the `(0, 1)` seed.
fn subarray_sum_no_base(nums: &[i64], k: i64) -> usize {
    let mut counts: HashMap<i64, usize> = HashMap::new();
    let (mut sum, mut total) = (0, 0);
    for &v in nums {
        sum += v;
        total += counts.get(&(sum - k)).copied().unwrap_or(0);
        *counts.entry(sum).or_insert(0) += 1;
    }
    total
}

fn subarray_sum_brute(nums: &[i64], k: i64) -> usize {
    let mut total = 0;
    for start in 0..nums.len() {
        let mut sum = 0;
        for &v in &nums[start..] {
            sum += v;
            if sum == k {
                total += 1;
            }
        }
    }
    total
}

/// `[1 2 3]`, the way Go's %v prints a slice.
fn show<T: Display>(xs: &[T]) -> String {
    let inner: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", inner.join(" "))
}

fn main() {
    println!("LONGEST CONSECUTIVE SEQUENCE: how long is the longest run of");
    println!("consecutive integers, in an unsorted slice with no duplicates rule?");
    println!();
    let ex = [100, 4, 200, 1, 3, 2];
    println!(
        "  {} -> {}   (1,2,3,4)",
        show(&ex),
        by_set::<RandomState>(&ex)
    );
    println!();
    println!("  sorting answers it in Theta(n log n). A set answers it in");
    println!("  Theta(n), and the argument is not obvious:");
    println!();
    println!("      for &v in &set {{");
    println!("          if set has v-1 {{ continue }}     // <- the whole trick");
    println!("          walk v, v+1, v+2, ... while present");
    println!("      }}");
    println!();
    println!("  the guard means the inner walk only ever starts at the FIRST");
    println!("  element of a run. Every element is therefore visited by exactly");
    println!("  one walk, so the total inner work is Theta(n) no matter how the");
    println!("  runs are shaped. That is the aggregate argument from lessons 08");
    println!("  and 09, in a third disguise.");

    println!();
    println!("drop the guard and it is still correct, and quadratic:");
    println!();
    println!(
        "  {:>10} {:>16} {:>18} {:>12}",
        "n", "steps with guard", "steps without", "ratio"
    );
    for n in [1000usize, 4000, 16_000, 64_000] {
        // ONE long run: the worst case for the unguarded version
        let run: Vec<i64> = (0..n as i64).collect();
        let (mut with_g, mut without) = (0u64, 0u64);
        let a = by_set_steps(&run, &mut with_g);
        let b = by_set_no_guard(&run, &mut without);
        assert!(a == b && a == n, "answers differ");
        println!(
            "  {:>10} {:>16} {:>18} {:>11.1}x",
            n,
            with_g,
            without,
            without as f64 / with_g as f64
        );
    }
    println!();
    println!("  n QUADRUPLES down that table and so does the ratio -- 500, 2000,");
    println!("  8000, 32000. A ratio that grows linearly in n is Theta(n^2)");
    println!("  against Theta(n), and here the unguarded version does exactly");
    println!("  n(n+1)/2 steps against n.");
    println!();
    println!("  both functions return the same answer on every input. The guard");
    println!("  is not about correctness at all -- only about not re-walking a");
    println!("  run once per element of it.");

    println!();
    println!("against the sort, on the clock (std's SipHash set, and the `Fast` one):");
    println!();
    let mut rng = Rng(11);
    println!(
        "  {:>10} {:>14} {:>14} {:>14} {:>10} {:>10}",
        "n", "sort ns", "set sip ns", "set fast ns", "sip/sort", "fast/sort"
    );
    // (n, sort, sip, fast) per row
    let mut table: Vec<(usize, f64, f64, f64)> = Vec::new();
    for n in [1000usize, 10_000, 100_000, 1_000_000] {
        let xs = rng.perm(n);
        // The machine is shared and the three differ by a small factor, so take
        // the minimum of three interleaved samples of each: noise only adds time.
        let (mut ts, mut tp, mut tf) = (f64::INFINITY, f64::INFINITY, f64::INFINITY);
        for _ in 0..3 {
            ts = ts.min(ns_per_op(|| {
                black_box(by_sort(black_box(&xs)));
            }));
            tp = tp.min(ns_per_op(|| {
                black_box(by_set::<RandomState>(black_box(&xs)));
            }));
            tf = tf.min(ns_per_op(|| {
                black_box(by_set::<BuildHasherDefault<Fast>>(black_box(&xs)));
            }));
        }
        println!(
            "  {:>10} {:>14.0} {:>14.0} {:>14.0} {:>9.2}x {:>9.2}x",
            n,
            ts,
            tp,
            tf,
            tp / ts,
            tf / ts
        );
        table.push((n, ts, tp, tf));
    }
    let beats = |pick: fn(&(usize, f64, f64, f64)) -> f64| -> String {
        let ns: Vec<usize> = table
            .iter()
            .filter(|r| pick(r) < r.1)
            .map(|r| r.0)
            .collect();
        if ns.is_empty() {
            "no row".to_string()
        } else {
            format!("n in {}", show(&ns))
        }
    };
    println!();
    println!("  the SipHash set beats the sort at:   {}", beats(|r| r.2));
    println!("  the Fast-hash set beats the sort at: {}", beats(|r| r.3));
    let range = |f: fn(&(usize, f64, f64, f64)) -> f64| {
        table
            .iter()
            .map(f)
            .fold((f64::INFINITY, 0.0f64), |(lo, hi), x| {
                (lo.min(x), hi.max(x))
            })
    };
    let (sip_lo, sip_hi) = range(|r| r.2 / r.3);
    let (fast_lo, fast_hi) = range(|r| r.3 / r.1);
    println!();
    println!(
        "  the SipHash set costs {sip_lo:.1}x to {sip_hi:.1}x the Fast-hash set on these rows,"
    );
    println!(
        "  and the Fast-hash set takes {fast_lo:.2}x to {fast_hi:.2}x the sort's time: the hash"
    );
    println!("  function is most of the difference between those two sets.");
    println!();
    println!("  sorting a permutation of 0..n-1 is Theta(n log n) over contiguous");
    println!("  memory; the set version pays n hashes, n inserts and 2n lookups,");
    println!("  and the log factor it saves is worth only about 20 at a million");
    println!("  elements. Per element, from the rows above:");
    println!();
    println!(
        "  {:>10} {:>14} {:>14} {:>14}",
        "n", "sort ns/elem", "sip ns/elem", "fast ns/elem"
    );
    for &(n, ts, tp, tf) in &table {
        let n = n as f64;
        println!(
            "  {:>10} {:>14.1} {:>14.1} {:>14.1}",
            n,
            ts / n,
            tp / n,
            tf / n
        );
    }
    println!();
    let (first, last) = (table[0], table[table.len() - 1]);
    if last.1 / last.0 as f64 > first.1 / first.0 as f64
        && last.2 / last.0 as f64 > first.2 / first.0 as f64
        && last.3 / last.0 as f64 > first.3 / first.0 as f64
    {
        println!("  no column is flat: every one costs more per element at the last row than");
        println!("  at the first. For the set that is plausibly cache misses once the table");
        println!("  outgrows the cache (not measured here), and it means Theta(n) operations");
        println!("  is not Theta(n) time.");
    }
    println!();
    println!("  this is lesson 03's warning in its sharpest form: 'I found the");
    println!("  Theta(n) solution' is not the end of the conversation, and which of two");
    println!("  close contenders wins at a given n is for the clock to say.");
    println!();
    println!("SUBARRAY SUM EQUALS K: the prefix-sum map, which is the one to");
    println!("remember.");
    println!();
    let nums = [1, 2, 3, -3, 1, 1, 1];
    for k in [3, 2, 0] {
        println!(
            "  {} k={:<3} -> {} subarrays (brute: {})",
            show(&nums),
            k,
            subarray_sum(&nums, k),
            subarray_sum_brute(&nums, k)
        );
    }
    println!();
    println!("      sum(i..j) == k");
    println!("      prefix[j] - prefix[i-1] == k");
    println!("      prefix[i-1] == prefix[j] - k     <- membership, again");
    println!();
    println!("  the map holds COUNTS of prefix sums seen so far, not positions,");
    println!("  because several prefixes can share a value -- which is exactly");
    println!("  what happens as soon as the input contains a zero or a negative.");

    println!();
    println!("and the one line everyone forgets:");
    println!();
    println!("      let mut counts = HashMap::from([(0, 1)]);");
    println!();
    let (mut mism, mut base) = (0, 0);
    for _ in 0..20_000 {
        let n = 1 + rng.below(10) as usize;
        let xs: Vec<i64> = (0..n).map(|_| rng.below(7) as i64 - 3).collect();
        let k = rng.below(9) as i64 - 4;
        let want = subarray_sum_brute(&xs, k);
        if subarray_sum(&xs, k) != want {
            mism += 1;
        }
        if subarray_sum_no_base(&xs, k) != want {
            base += 1;
        }
    }
    println!("  20,000 random cases vs brute force:");
    println!("  {:<34} {} mismatches", "with counts[0] = 1", mism);
    println!("  {:<34} {} mismatches", "without it", base);
    println!();
    println!("  the seed represents the EMPTY PREFIX -- it is what lets a");
    println!("  subarray starting at index 0 be counted. Without it you miss");
    println!("  exactly the subarrays that begin at the start, which is a large");
    println!("  minority of inputs and none of the ones you check by hand.");

    println!();
    println!("the speedup over the quadratic version:");
    println!();
    println!(
        "  {:>10} {:>16} {:>16} {:>12}",
        "n", "brute ns", "prefix map ns", "speedup"
    );
    for n in [1000usize, 10_000, 100_000] {
        let xs: Vec<i64> = (0..n).map(|_| rng.below(21) as i64 - 10).collect();
        let tb = ns_per_op(|| {
            black_box(subarray_sum_brute(black_box(&xs), black_box(7)));
        });
        let tm = ns_per_op(|| {
            black_box(subarray_sum(black_box(&xs), black_box(7)));
        });
        println!("  {:>10} {:>16.0} {:>16.0} {:>11.0}x", n, tb, tm, tb / tm);
    }
    println!();
    println!("  here the map DOES win, and by a growing margin -- because the");
    println!("  alternative is genuinely Theta(n^2), not Theta(n log n). Compare");
    println!("  that with the previous problem, where the alternative was a sort");
    if table.iter().all(|r| r.2 > r.1) {
        println!("  and the set with std's hasher lost at every size.");
    } else {
        println!("  and the set with std's hasher did not lose everywhere in this run.");
    }
    println!();
    println!("  the rule that falls out: a map beats a NESTED LOOP reliably. It");
    println!("  beats a SORT only when n is large and the constant works out, and");
    println!("  you have to measure to know.");
    println!();
    println!("  the prefix-sum-in-a-map pattern generalises far past this");
    println!("  problem: subarray sums divisible by k (store sum mod k), longest");
    println!("  subarray with equal 0s and 1s (map -1/+1 and store the running");
    println!("  balance), and most of lesson 16.");
}
```

**Rust sample output:**

```
LONGEST CONSECUTIVE SEQUENCE: how long is the longest run of
consecutive integers, in an unsorted slice with no duplicates rule?

  [100 4 200 1 3 2] -> 4   (1,2,3,4)

  sorting answers it in Theta(n log n). A set answers it in
  Theta(n), and the argument is not obvious:

      for &v in &set {
          if set has v-1 { continue }     // <- the whole trick
          walk v, v+1, v+2, ... while present
      }

  the guard means the inner walk only ever starts at the FIRST
  element of a run. Every element is therefore visited by exactly
  one walk, so the total inner work is Theta(n) no matter how the
  runs are shaped. That is the aggregate argument from lessons 08
  and 09, in a third disguise.

drop the guard and it is still correct, and quadratic:

           n steps with guard      steps without        ratio
        1000             1000             500500       500.5x
        4000             4000            8002000      2000.5x
       16000            16000          128008000      8000.5x
       64000            64000         2048032000     32000.5x

  n QUADRUPLES down that table and so does the ratio -- 500, 2000,
  8000, 32000. A ratio that grows linearly in n is Theta(n^2)
  against Theta(n), and here the unguarded version does exactly
  n(n+1)/2 steps against n.

  both functions return the same answer on every input. The guard
  is not about correctness at all -- only about not re-walking a
  run once per element of it.

against the sort, on the clock (std's SipHash set, and the `Fast` one):

           n        sort ns     set sip ns    set fast ns   sip/sort  fast/sort
        1000           5731          16080           5817      2.81x      1.02x
       10000          71226         167485          63407      2.35x      0.89x
      100000         875104        2073693         868736      2.37x      0.99x
     1000000       12197178       50186156       14841991      4.11x      1.22x

  the SipHash set beats the sort at:   no row
  the Fast-hash set beats the sort at: n in [10000 100000]

  the SipHash set costs 2.4x to 3.4x the Fast-hash set on these rows,
  and the Fast-hash set takes 0.89x to 1.22x the sort's time: the hash
  function is most of the difference between those two sets.

  sorting a permutation of 0..n-1 is Theta(n log n) over contiguous
  memory; the set version pays n hashes, n inserts and 2n lookups,
  and the log factor it saves is worth only about 20 at a million
  elements. Per element, from the rows above:

           n   sort ns/elem    sip ns/elem   fast ns/elem
        1000            5.7           16.1            5.8
       10000            7.1           16.7            6.3
      100000            8.8           20.7            8.7
     1000000           12.2           50.2           14.8

  no column is flat: every one costs more per element at the last row than
  at the first. For the set that is plausibly cache misses once the table
  outgrows the cache (not measured here), and it means Theta(n) operations
  is not Theta(n) time.

  this is lesson 03's warning in its sharpest form: 'I found the
  Theta(n) solution' is not the end of the conversation, and which of two
  close contenders wins at a given n is for the clock to say.

SUBARRAY SUM EQUALS K: the prefix-sum map, which is the one to
remember.

  [1 2 3 -3 1 1 1] k=3   -> 6 subarrays (brute: 6)
  [1 2 3 -3 1 1 1] k=2   -> 5 subarrays (brute: 5)
  [1 2 3 -3 1 1 1] k=0   -> 2 subarrays (brute: 2)

      sum(i..j) == k
      prefix[j] - prefix[i-1] == k
      prefix[i-1] == prefix[j] - k     <- membership, again

  the map holds COUNTS of prefix sums seen so far, not positions,
  because several prefixes can share a value -- which is exactly
  what happens as soon as the input contains a zero or a negative.

and the one line everyone forgets:

      let mut counts = HashMap::from([(0, 1)]);

  20,000 random cases vs brute force:
  with counts[0] = 1                 0 mismatches
  without it                         6708 mismatches

  the seed represents the EMPTY PREFIX -- it is what lets a
  subarray starting at index 0 be counted. Without it you miss
  exactly the subarrays that begin at the start, which is a large
  minority of inputs and none of the ones you check by hand.

the speedup over the quadratic version:

           n         brute ns    prefix map ns      speedup
        1000           137404            17886           8x
       10000         12991625           150457          86x
      100000       1353189292          1245305        1087x

  here the map DOES win, and by a growing margin -- because the
  alternative is genuinely Theta(n^2), not Theta(n log n). Compare
  that with the previous problem, where the alternative was a sort
  and the set with std's hasher lost at every size.

  the rule that falls out: a map beats a NESTED LOOP reliably. It
  beats a SORT only when n is large and the constant works out, and
  you have to measure to know.

  the prefix-sum-in-a-map pattern generalises far past this
  problem: subarray sums divisible by k (store sum mod k), longest
  subarray with equal 0s and 1s (map -1/+1 and store the running
  balance), and most of lesson 16.
```

> **Go vs Rust:** Every count matches Go: the step table (500,500 to 2,048,032,000 steps, ratios 500.5 to 32,000.5) and the three subarray answers are identical, and a copy of the Go program fed this file's splitmix64 prints the same 6,708 mismatches without the seed (the Go sample's 6,769 is another stream, `rand.NewSource(11)`). The Go prose's claim that the set "WINS at 100,000 ... reproduces at 0.58 to 0.61" does not hold in Rust: std's SipHash set was slower than the sort at every size in 7 of 7 runs (2.2-3.3x), and with the `Fast` hasher the two are within 0.88-1.36x of each other, the set ahead in some rows by at most 12%, so there is no band where it clearly wins. Rust's `sort_unstable` is also far faster than Go's (11.6-12.5 ms at n=1,000,000 against 59-70 ms in Go) and so is its set (SipHash 25.8-40.1 ms, `Fast` 10.5-16.1 ms, against 87-153 ms), and the prefix map beats the nested loop by 8-13x, 83-95x and 578-1,213x (Go sample: 8x, 83x, 749x). The unguarded walk at n=64,000 is 2 billion lookups, so the two step-counting versions use `Fast` (the counts do not depend on the hash) to keep the program near 20 s; Go's `counts[sum-k]` zero default becomes `get(..).copied().unwrap_or(0)`, because indexing a missing key panics, and `v-1` / `v+n` become `checked_sub` and a `(v..=i64::MAX)` range where Go would silently wrap.

**Complexity:** the guard makes it Θ(n) instead of Θ(n²) — the step ratio grows linearly, **500 →
32,000×** · but the Θ(n) set **loses to the Θ(n log n) sort** at 1,000, 10,000 and 1,000,000 and only
wins at 100,000 · `counts[0] = 1` is not optional: **6,769 of 20,000** cases wrong without it

---

## 14. hashDoS: the attack that made everyone randomise

`🔴 hard` · *a worst case someone else gets to choose*

If your hash is deterministic and public — and if it is in your source, it is public — anyone can
compute a set of keys that all collide, POST them as JSON keys, and turn one request into a quadratic
amount of server work.

**Steps:**

1. Generate the colliding key set. It takes milliseconds.
2. Feed it to a table with a fixed hash.
3. Then add a seed, and feed the same keys to Go's map.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// Every hash table has a worst case, and example 3 measured it: when every key
// lands in one bucket, Theta(1) becomes Theta(n) and the whole table is a
// linked list.
//
// The uncomfortable part is that an attacker can ARRANGE that. If your hash
// function is deterministic and public -- and if it is in your source, it is
// public -- then anyone can compute a set of keys that all collide, POST them
// as form fields or JSON keys, and turn one request into a quadratic amount of
// server work.
//
// That is hashDoS. It was disclosed against PHP, Python, Ruby, Java and Node in
// 2011, and it is why every one of those languages now randomises its hash.
// This example is the attack and the defence, both measured.

// ---- a deterministic, public hash -- i.e. the vulnerable kind ---------------

type Fixed struct {
	buckets [][]string
	n       int
}

func NewFixed(size int) *Fixed { return &Fixed{buckets: make([][]string, size)} }

// weakHash is deliberately simple, but the attack does not depend on the hash
// being weak. It depends on the hash being KNOWN.
func weakHash(s string) uint64 {
	h := uint64(5381)
	for i := 0; i < len(s); i++ {
		h = h*33 + uint64(s[i])
	}
	return h
}

func (f *Fixed) Put(k string) {
	i := weakHash(k) % uint64(len(f.buckets))
	for _, e := range f.buckets[i] {
		if e == k {
			return
		}
	}
	f.buckets[i] = append(f.buckets[i], k)
	f.n++
}

func (f *Fixed) Get(k string) bool {
	i := weakHash(k) % uint64(len(f.buckets))
	for _, e := range f.buckets[i] {
		if e == k {
			return true
		}
	}
	return false
}

func (f *Fixed) longest() int {
	m := 0
	for _, b := range f.buckets {
		if len(b) > m {
			m = len(b)
		}
	}
	return m
}

// ---- the same table with a per-instance random seed -------------------------

type Seeded struct {
	buckets [][]string
	seed    uint64
	n       int
}

func NewSeeded(size int) *Seeded {
	return &Seeded{buckets: make([][]string, size), seed: rand.Uint64()}
}

// seededHash mixes a secret the attacker cannot know. Note the seed goes in
// FIRST, as the initial state -- appending it at the end would not help,
// because the attacker's collisions would still be collisions.
func seededHash(seed uint64, s string) uint64 {
	h := seed
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= 1099511628211
	}
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 31
	return h
}

func (f *Seeded) Put(k string) {
	i := seededHash(f.seed, k) % uint64(len(f.buckets))
	for _, e := range f.buckets[i] {
		if e == k {
			return
		}
	}
	f.buckets[i] = append(f.buckets[i], k)
	f.n++
}

func (f *Seeded) Get(k string) bool {
	i := seededHash(f.seed, k) % uint64(len(f.buckets))
	for _, e := range f.buckets[i] {
		if e == k {
			return true
		}
	}
	return false
}

func (f *Seeded) longest() int {
	m := 0
	for _, b := range f.buckets {
		if len(b) > m {
			m = len(b)
		}
	}
	return m
}

// ---- generating the attack --------------------------------------------------

// collidingKeys finds n distinct strings that all hash to the same bucket.
// This is brute force and it takes milliseconds, which is the point: the
// attacker's cost to generate is trivial compared to the server's cost to
// process.
func collidingKeys(n, buckets int) []string {
	target := weakHash("aaaa") % uint64(buckets)
	out := make([]string, 0, n)
	buf := make([]byte, 8)
	for i := 0; len(out) < n; i++ {
		v := i
		for j := range buf {
			buf[j] = byte('a' + v%26)
			v /= 26
		}
		s := string(buf)
		if weakHash(s)%uint64(buckets) == target {
			out = append(out, s)
		}
	}
	return out
}

func normalKeys(n int) []string {
	out := make([]string, n)
	for i := range out {
		out[i] = fmt.Sprintf("user_%d_session", i)
	}
	return out
}

var sinkB bool
var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	const buckets = 4096
	const n = 8000

	fmt.Println("the attack: find keys that all hash to the same bucket.")
	fmt.Println()
	attack := collidingKeys(n, buckets)
	fmt.Printf("  generated %d colliding keys, e.g. %v\n", len(attack), attack[:4])
	fmt.Printf("  weakHash %% %d for each of those: ", buckets)
	for _, k := range attack[:4] {
		fmt.Printf("%d ", weakHash(k)%buckets)
	}
	fmt.Println()
	fmt.Println()
	fmt.Println("  that took milliseconds to compute, and it is a one-time cost for")
	fmt.Println("  the attacker. Every request afterwards is free to send.")

	fmt.Println()
	fmt.Println("what those keys do to a table with a fixed, public hash:")
	fmt.Println()
	normal := normalKeys(n)
	fixedNormal := NewFixed(buckets)
	for _, k := range normal {
		fixedNormal.Put(k)
	}
	fixedAttack := NewFixed(buckets)
	for _, k := range attack {
		fixedAttack.Put(k)
	}
	fmt.Printf("  %-30s %8d keys, longest bucket %d\n",
		"normal input", fixedNormal.n, fixedNormal.longest())
	fmt.Printf("  %-30s %8d keys, longest bucket %d\n",
		"attacker input", fixedAttack.n, fixedAttack.longest())

	tNormal := nsPerOp(func() {
		f := NewFixed(buckets)
		for _, k := range normal {
			f.Put(k)
		}
		sinkI = f.n
	})
	tAttack := nsPerOp(func() {
		f := NewFixed(buckets)
		for _, k := range attack {
			f.Put(k)
		}
		sinkI = f.n
	})
	fmt.Println()
	fmt.Printf("  %-30s %14.0f ns to insert %d keys\n", "normal input", tNormal, n)
	fmt.Printf("  %-30s %14.0f ns   %.0fx slower\n", "attacker input", tAttack, tAttack/tNormal)
	fmt.Println()
	fmt.Println("  same code, same number of keys, same table size -- and the")
	fmt.Println("  longest bucket went from 8 to 8000. One request became 200x more")
	fmt.Println("  expensive, with no memory blow-up and nothing in the logs.")
	fmt.Println()
	fmt.Println("  scale that: an HTTP server parsing a POST body into a map, or a")
	fmt.Println("  JSON decoder building map[string]any, will do this on any input")
	fmt.Println("  it is handed. That is the 2011 disclosure, exactly.")

	fmt.Println()
	fmt.Println("the defence is one word: SEED.")
	fmt.Println()
	seededAttack := NewSeeded(buckets)
	for _, k := range attack {
		seededAttack.Put(k)
	}
	fmt.Printf("  %-30s %8d keys, longest bucket %d\n",
		"attacker input, seeded", seededAttack.n, seededAttack.longest())
	tSeeded := nsPerOp(func() {
		f := NewSeeded(buckets)
		for _, k := range attack {
			f.Put(k)
		}
		sinkI = f.n
	})
	fmt.Printf("  %-30s %14.0f ns   %.0fx faster than the attack\n",
		"", tSeeded, tAttack/tSeeded)
	fmt.Println()
	fmt.Println("  the keys still collide with EACH OTHER under weakHash. They do")
	fmt.Println("  not collide under seededHash, because the attacker did not know")
	fmt.Println("  the seed when they generated them -- and could not have, since it")
	fmt.Println("  is chosen at run time.")

	fmt.Println()
	fmt.Println("Go's map does this for you, and has since 1.0:")
	fmt.Println()
	goNormal := nsPerOp(func() {
		m := make(map[string]struct{}, n)
		for _, k := range normal {
			m[k] = struct{}{}
		}
		sinkI = len(m)
	})
	goAttack := nsPerOp(func() {
		m := make(map[string]struct{}, n)
		for _, k := range attack {
			m[k] = struct{}{}
		}
		sinkI = len(m)
	})
	fmt.Printf("  %-30s %14.0f ns\n", "map, normal input", goNormal)
	fmt.Printf("  %-30s %14.0f ns   %.2fx\n", "map, attacker input", goAttack, goAttack/goNormal)
	fmt.Println()
	fmt.Println("  no difference worth the name. The runtime picks a random seed per")
	fmt.Println("  process and mixes it into every hash, so a collision set computed")
	fmt.Println("  against one Go process is worthless against another -- and")
	fmt.Println("  worthless against the same process after a restart.")
	fmt.Println()
	fmt.Println("  on amd64/arm64 with AES support Go goes further and uses AESENC")
	fmt.Println("  instructions for string hashing, which makes finding collisions")
	fmt.Println("  a cryptographic problem rather than an arithmetic one.")

	fmt.Println()
	fmt.Println("what this means for code you write:")
	fmt.Println()
	fmt.Printf("  %-40s %s\n", "map[K]V with untrusted keys", "safe -- seeded for you")
	fmt.Printf("  %-40s %s\n", "your own hash table, fixed hash", "VULNERABLE")
	fmt.Printf("  %-40s %s\n", "hash/fnv, hash/crc32 as a table hash", "VULNERABLE -- no seed")
	fmt.Printf("  %-40s %s\n", "maphash.Hash", "safe -- seeded, and that is its job")
	fmt.Println()
	fmt.Println("  hash/fnv is a fine checksum and a bad table hash for untrusted")
	fmt.Println("  input, because it takes no seed. If you must build your own")
	fmt.Println("  table, use hash/maphash -- it exists precisely for this, and it")
	fmt.Println("  exposes the same seeded hash the runtime uses.")
	fmt.Println()
	fmt.Println("  and the general principle, which outlives this example: a data")
	fmt.Println("  structure whose worst case is much worse than its average case")
	fmt.Println("  has a security property, not just a performance one. Ask who")
	fmt.Println("  chooses the input.")
	sinkB = true
}
```

**Sample output:**

```
the attack: find keys that all hash to the same bucket.

  generated 8000 colliding keys, e.g. [evdaaaaa inhaaaaa mflaaaaa twtaaaaa]
  weakHash % 4096 for each of those: 2505 2505 2505 2505 

  that took milliseconds to compute, and it is a one-time cost for
  the attacker. Every request afterwards is free to send.

what those keys do to a table with a fixed, public hash:

  normal input                       8000 keys, longest bucket 8
  attacker input                     8000 keys, longest bucket 8000

  normal input                           277920 ns to insert 8000 keys
  attacker input                       50515555 ns   182x slower

  same code, same number of keys, same table size -- and the
  longest bucket went from 8 to 8000. One request became 200x more
  expensive, with no memory blow-up and nothing in the logs.

  scale that: an HTTP server parsing a POST body into a map, or a
  JSON decoder building map[string]any, will do this on any input
  it is handed. That is the 2011 disclosure, exactly.

the defence is one word: SEED.

  attacker input, seeded             8000 keys, longest bucket 10
                                         351377 ns   144x faster than the attack

  the keys still collide with EACH OTHER under weakHash. They do
  not collide under seededHash, because the attacker did not know
  the seed when they generated them -- and could not have, since it
  is chosen at run time.

Go's map does this for you, and has since 1.0:

  map, normal input                      101481 ns
  map, attacker input                    109397 ns   1.08x

  no difference worth the name. The runtime picks a random seed per
  process and mixes it into every hash, so a collision set computed
  against one Go process is worthless against another -- and
  worthless against the same process after a restart.

  on amd64/arm64 with AES support Go goes further and uses AESENC
  instructions for string hashing, which makes finding collisions
  a cryptographic problem rather than an arithmetic one.

what this means for code you write:

  map[K]V with untrusted keys              safe -- seeded for you
  your own hash table, fixed hash          VULNERABLE
  hash/fnv, hash/crc32 as a table hash     VULNERABLE -- no seed
  maphash.Hash                             safe -- seeded, and that is its job

  hash/fnv is a fine checksum and a bad table hash for untrusted
  input, because it takes no seed. If you must build your own
  table, use hash/maphash -- it exists precisely for this, and it
  exposes the same seeded hash the runtime uses.

  and the general principle, which outlives this example: a data
  structure whose worst case is much worse than its average case
  has a security property, not just a performance one. Ask who
  chooses the input.
```

**Rust:**

```rust
use std::cell::Cell;
use std::collections::{BTreeMap, HashSet};
use std::hash::{BuildHasher, BuildHasherDefault, DefaultHasher, Hash, Hasher, RandomState};
use std::hint::black_box;
use std::time::{Duration, Instant};

// Every hash table has a worst case, and example 3 measured it: when every key
// lands in one bucket, Theta(1) becomes Theta(n) and the whole table is a
// linked list.
//
// The uncomfortable part is that an attacker can ARRANGE that. If your hash
// function is deterministic and public -- and if it is in your source, it is
// public -- then anyone can compute a set of keys that all collide, POST them
// as form fields or JSON keys, and turn one request into a quadratic amount of
// server work.
//
// That is hashDoS. It was disclosed against PHP, Python, Ruby, Java and Node in
// 2011, and it is why every one of those languages now randomises its hash.
// This example is the attack and the defence, both measured -- on this
// example's own tables first, then on Rust's std HashSet, which is where the
// Rust twin has more to say than the Go one: std's table is only as safe as the
// hasher it is given.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

// splitmix64: stand-in for math/rand. Seeded, so the seed sweep below is
// reproducible. (The seeds the tables below use for the headline run are NOT
// from this generator; see random_seed.)
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
}

/// a per-run random u64, std only: Rust has no math/rand in std, but every
/// `RandomState` is keyed from the operating system's random source, so hashing
/// a constant with a fresh one gives 64 unpredictable bits. This is the
/// `rand.Uint64()` of Go's NewSeeded.
fn random_seed() -> u64 {
    RandomState::new().hash_one(0u64)
}

/// `[a b c]`, the way Go's %v prints a slice ({:?} would give `["a", "b"]`)
fn show(xs: &[String]) -> String {
    format!("[{}]", xs.join(" "))
}

// ---- two hashes: a deterministic, public one -- i.e. the vulnerable kind ----

// weak_hash (djb2) is deliberately simple, but the attack does not depend on
// the hash being weak. It depends on the hash being KNOWN. Go wraps silently on
// overflow; here the wraparound is written out.
fn weak_hash(s: &str) -> u64 {
    let mut h: u64 = 5381;
    for &b in s.as_bytes() {
        h = h.wrapping_mul(33).wrapping_add(b as u64);
    }
    h
}

// seeded_hash mixes a secret the attacker cannot know. Note the seed goes in
// FIRST, as the initial state -- appending it at the end would not help,
// because the attacker's collisions would still be collisions. Same arithmetic
// as Go's seededHash (FNV-1a round, then ONE splitmix64 mixing round).
fn seeded_hash(seed: u64, s: &str) -> u64 {
    let mut h = seed;
    for &b in s.as_bytes() {
        h ^= b as u64;
        h = h.wrapping_mul(1099511628211);
    }
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58476d1ce4e5b9);
    h ^= h >> 31;
    h
}

/// the closure form of seeded_hash: a hash function with the seed baked in.
fn seeded(seed: u64) -> impl Fn(&str) -> u64 {
    move |s| seeded_hash(seed, s)
}

// ---- the table: ONE type, parameterised by its hash ----
//
// Go writes the table twice (Fixed with a fixed hash, Seeded with a seed field)
// and the two copies differ in one line. In Rust the hash is a type parameter,
// so the two tables below are literally the same code with a different `H`:
// that is the example's point -- nothing about the TABLE changed between the
// vulnerable version and the safe one.
//
// Keys are `&'a str` borrows, as in example 4: the table never copies a key.
struct Table<'a, H: Fn(&str) -> u64> {
    buckets: Vec<Vec<&'a str>>,
    hash: H,
    n: usize,
    // string comparisons made so far. A Cell so that `get(&self)` can count too.
    cmps: Cell<u64>,
}

impl<'a, H: Fn(&str) -> u64> Table<'a, H> {
    fn new(size: usize, hash: H) -> Self {
        Table {
            buckets: vec![Vec::new(); size],
            hash,
            n: 0,
            cmps: Cell::new(0),
        }
    }

    fn index(&self, k: &str) -> usize {
        ((self.hash)(k) % self.buckets.len() as u64) as usize
    }

    fn put(&mut self, k: &'a str) {
        let i = self.index(k);
        for &e in &self.buckets[i] {
            self.cmps.set(self.cmps.get() + 1);
            if e == k {
                return;
            }
        }
        self.buckets[i].push(k);
        self.n += 1;
    }

    fn get(&self, k: &str) -> bool {
        for &e in &self.buckets[self.index(k)] {
            self.cmps.set(self.cmps.get() + 1);
            if e == k {
                return true;
            }
        }
        false
    }

    fn longest(&self) -> usize {
        self.buckets.iter().map(Vec::len).max().unwrap_or(0)
    }
}

// ---- generating the attack ----

// colliding_keys finds n distinct strings that all hash to the same bucket.
// This is brute force: it tries every 8-letter string in order and keeps the
// ones that land in the bucket of "aaaa". Go's version is hard-wired to
// weakHash; here the hash is a parameter, because the second half of this
// example attacks std's hashers with the same loop. Returns the keys and the
// number of candidates tried.
fn colliding_keys(n: usize, buckets: usize, hash: impl Fn(&str) -> u64) -> (Vec<String>, usize) {
    let target = hash("aaaa") % buckets as u64;
    let mut out = Vec::with_capacity(n);
    let mut buf = [0u8; 8];
    let mut tried = 0;
    while out.len() < n {
        let mut v = tried;
        for b in &mut buf {
            *b = b'a' + (v % 26) as u8;
            v /= 26;
        }
        let s = std::str::from_utf8(&buf).expect("ASCII letters");
        if hash(s) % buckets as u64 == target {
            out.push(s.to_string());
        }
        tried += 1;
    }
    (out, tried)
}

// djb2 has something better than brute force: a COMPLETE 64-bit collision that
// works from any starting state. "Ez" and "FY" both leave the state at
// h*33*33 + 2399 (69*33 + 122 = 70*33 + 89 = 2399), so any concatenation of
// 13 blocks, each "Ez" or "FY", is one of 8192 different keys with ONE hash.
// (Java's String.hashCode has the same flaw in "Aa"/"BB".) No searching.
fn block_keys(n: usize) -> Vec<String> {
    assert!(n <= 1 << 13, "13 blocks give 8192 keys");
    (0..n)
        .map(|i| {
            (0..13)
                .map(|j| if (i >> j) & 1 == 1 { "FY" } else { "Ez" })
                .collect()
        })
        .collect()
}

fn normal_keys(n: usize) -> Vec<String> {
    (0..n).map(|i| format!("user_{i}_session")).collect()
}

// ---- Rust's std table, with a hasher the attacker can know ----

// Djb2 as a `Hasher`, so that std's own HashSet can be given the vulnerable
// hash. `BuildHasherDefault<Djb2>` is a hasher factory with NO seed: every
// HashSet built from it hashes identically, on every run.
struct Djb2(u64);
impl Default for Djb2 {
    fn default() -> Self {
        Djb2(5381)
    }
}
impl Hasher for Djb2 {
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 = self.0.wrapping_mul(33).wrapping_add(b as u64);
        }
    }
    fn finish(&self) -> u64 {
        self.0
    }
}
// Note: `str`'s Hash impl feeds the bytes and then ONE extra 0xff byte. The
// extra step is the same for every key, so it cannot separate keys that
// collide, and the keys above stay colliding.

// A key whose `==` counts its own calls. std's table calls `==` once for every
// stored key whose hash tag matches, so this counts how much equality work an
// insert sequence really did -- a deterministic number, unlike a stopwatch.
// (The count is a property of hashbrown's probing in THIS std version.)
thread_local! {
    static EQS: Cell<u64> = const { Cell::new(0) };
}
struct Counted<'a>(&'a str);
impl PartialEq for Counted<'_> {
    fn eq(&self, other: &Self) -> bool {
        EQS.with(|c| c.set(c.get() + 1));
        self.0 == other.0
    }
}
impl Eq for Counted<'_> {}
impl Hash for Counted<'_> {
    // hash exactly as the &str would, so the table lays out the same.
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.0.hash(state)
    }
}

/// how many `==` calls inserting `keys` into a pre-sized HashSet<_, S> makes
fn eq_calls<S: BuildHasher + Default>(keys: &[String]) -> u64 {
    EQS.with(|c| c.set(0));
    let mut set: HashSet<Counted, S> = HashSet::with_capacity_and_hasher(keys.len(), S::default());
    for k in keys {
        set.insert(Counted(k));
    }
    black_box(&set);
    EQS.with(Cell::get)
}

/// ns to build a pre-sized HashSet<&str, S> from `keys` (the Go benchmark's
/// `make(map[string]struct{}, n)` plus n inserts)
fn time_set<S: BuildHasher + Default>(keys: &[String]) -> f64 {
    ns_per_op(|| {
        let mut set: HashSet<&str, S> = HashSet::with_capacity_and_hasher(keys.len(), S::default());
        for k in black_box(keys) {
            set.insert(k.as_str());
        }
        black_box(set.len());
    })
}

/// What one pre-sized HashSet<_, S> makes of `keys`: (ns to build it, `==`
/// calls, distinct 7-bit tags, distinct starting slots). hashbrown keeps the top
/// 7 bits of a hash as a one-byte tag in a control array, uses the LOW bits to
/// pick the slot where probing starts, and only calls `==` on keys whose tags
/// match (hashbrown 0.17.1 inside this std; an implementation detail). So many
/// distinct tags and many distinct starting slots is what a healthy hash looks like.
fn profile<S: BuildHasher + Default>(keys: &[String]) -> (f64, u64, usize, usize) {
    let hasher = S::default();
    // the table has the smallest power-of-two number of slots above 8/7 of the
    // capacity asked for
    let slots = (keys.len() * 8 / 7).next_power_of_two() as u64;
    let hashes = keys.iter().map(|k| hasher.hash_one(k.as_str()));
    let tags: HashSet<u64> = hashes.clone().map(|h| h >> 57).collect();
    let starts: HashSet<u64> = hashes.map(|h| h & (slots - 1)).collect();
    (
        time_set::<S>(keys),
        eq_calls::<S>(keys),
        tags.len(),
        starts.len(),
    )
}

fn print_profile_header(keys_label: &str) {
    println!(
        "  {:<22} {:<18} {:>13} {:>9} {:>11} {:>5} {:>6}",
        "hasher", keys_label, "ns", "x normal", "== calls", "tags", "starts"
    );
}

fn print_profile(hasher: &str, keys: &str, p: (f64, u64, usize, usize), base_ns: f64) {
    println!(
        "  {:<22} {:<18} {:>13.0} {:>8.1}x {:>11} {:>5} {:>6}",
        hasher,
        keys,
        p.0,
        p.0 / base_ns,
        p.1,
        p.2,
        p.3
    );
}

type PublicDjb2 = BuildHasherDefault<Djb2>;
// DefaultHasher is std's SipHash-1-3. `DefaultHasher::new()` -- and so
// BuildHasherDefault<DefaultHasher> -- uses FIXED keys; only RandomState
// supplies random ones.
type FixedSip = BuildHasherDefault<DefaultHasher>;

fn main() {
    const BUCKETS: usize = 4096;
    const N: usize = 8000;

    println!("the attack: find keys that all hash to the same bucket.");
    println!();
    let t0 = Instant::now();
    let (attack, tried) = colliding_keys(N, BUCKETS, weak_hash);
    let gen_ms = t0.elapsed().as_secs_f64() * 1000.0;
    println!(
        "  generated {} colliding keys, e.g. {}",
        attack.len(),
        show(&attack[..4])
    );
    print!("  weak_hash % {BUCKETS} for each of those: ");
    for k in &attack[..4] {
        print!("{} ", weak_hash(k) % BUCKETS as u64);
    }
    println!();
    println!();
    // Go's text says this takes "milliseconds". Brute force tries every key in
    // order, so it needs about BUCKETS candidates per colliding key: the count
    // is exact, the time is a stopwatch reading.
    println!("  brute force tried {tried} candidate strings ({gen_ms:.0} ms on this run): about");
    println!(
        "  {} candidates per key found, because one candidate in {BUCKETS} lands in the",
        tried / N
    );
    println!("  right bucket. A one-time cost for the attacker; every request after");
    println!("  it is free to send.");
    println!();
    let t0 = Instant::now();
    let blocks = block_keys(N);
    let block_us = t0.elapsed().as_secs_f64() * 1e6;
    println!("  but djb2 does not even need the search. Concatenating 13 blocks, each");
    println!("  \"Ez\" or \"FY\", builds keys that collide on ALL 64 bits:");
    println!();
    for k in &blocks[..4] {
        println!("    {k}  weak_hash = {:#018x}", weak_hash(k));
    }
    let distinct: HashSet<u64> = blocks.iter().map(|k| weak_hash(k)).collect();
    println!();
    println!(
        "  {} keys, {} distinct hash value(s), built in {block_us:.0} microseconds.",
        blocks.len(),
        distinct.len()
    );

    println!();
    println!("what those keys do to a table with a fixed, public hash:");
    println!();
    let normal = normal_keys(N);
    let mut fixed_normal = Table::new(BUCKETS, weak_hash);
    for k in &normal {
        fixed_normal.put(k);
    }
    let mut fixed_attack = Table::new(BUCKETS, weak_hash);
    for k in &attack {
        fixed_attack.put(k);
    }
    println!(
        "  {:<30} {:>8} keys, longest bucket {}",
        "normal input",
        fixed_normal.n,
        fixed_normal.longest()
    );
    println!(
        "  {:<30} {:>8} keys, longest bucket {}",
        "attacker input",
        fixed_attack.n,
        fixed_attack.longest()
    );
    // The Go program shows the damage as a stopwatch reading; the same damage
    // is also a COUNT: string comparisons made by the N puts, then by N gets.
    println!();
    println!("  string comparisons (counted, not timed):");
    for (name, t) in [
        ("normal input", &fixed_normal),
        ("attacker input", &fixed_attack),
    ] {
        let after_puts = t.cmps.get();
        t.cmps.set(0);
        let keys = if name == "normal input" {
            &normal
        } else {
            &attack
        };
        assert!(keys.iter().all(|k| t.get(k)), "every key must be found");
        println!(
            "  {:<30} {:>12} to put {} keys, {:>12} to get them all back ({:.1} per get)",
            name,
            after_puts,
            N,
            t.cmps.get(),
            t.cmps.get() as f64 / N as f64
        );
    }

    let t_normal = ns_per_op(|| {
        let mut f = Table::new(BUCKETS, weak_hash);
        for k in black_box(&normal) {
            f.put(k);
        }
        black_box(f.n);
    });
    let t_attack = ns_per_op(|| {
        let mut f = Table::new(BUCKETS, weak_hash);
        for k in black_box(&attack) {
            f.put(k);
        }
        black_box(f.n);
    });
    println!();
    println!(
        "  {:<30} {:>14.0} ns to insert {} keys",
        "normal input", t_normal, N
    );
    println!(
        "  {:<30} {:>14.0} ns   {:.0}x slower",
        "attacker input",
        t_attack,
        t_attack / t_normal
    );
    println!();
    println!("  same code, same number of keys, same table size -- and the");
    println!(
        "  longest bucket went from {} to {}. One request became {:.0}x more",
        fixed_normal.longest(),
        fixed_attack.longest(),
        t_attack / t_normal
    );
    println!("  expensive, with no memory blow-up and nothing in the logs.");
    println!();
    println!("  scale that: an HTTP server parsing a POST body into a map, or a");
    println!("  JSON decoder building a map of strings, will do this on any input");
    println!("  it is handed. That is the 2011 disclosure, exactly.");

    println!();
    println!("the defence is one word: SEED.");
    println!();
    let mut seeded_attack = Table::new(BUCKETS, seeded(random_seed()));
    for k in &attack {
        seeded_attack.put(k);
    }
    println!(
        "  {:<30} {:>8} keys, longest bucket {}",
        "attacker input, seeded",
        seeded_attack.n,
        seeded_attack.longest()
    );
    let t_seeded = ns_per_op(|| {
        // a fresh random seed per table, as Go's NewSeeded does
        let mut f = Table::new(BUCKETS, seeded(random_seed()));
        for k in black_box(&attack) {
            f.put(k);
        }
        black_box(f.n);
    });
    println!(
        "  {:<30} {:>14.0} ns   {:.0}x faster than the attack",
        "",
        t_seeded,
        t_attack / t_seeded
    );
    println!();
    println!("  the keys still collide with EACH OTHER under weak_hash. They do");
    println!("  not collide under seeded_hash, because the attacker did not know");
    println!("  the seed when they generated them -- and could not have, since it");
    println!("  is chosen at run time.");

    // One seed is one sample, and the longest bucket depends on the seed.
    // 1000 seeds from a fixed generator, so this table is reproducible: the
    // attacker's keys against the seeded hash, and ordinary keys against it.
    println!();
    println!("one seed is one sample. The longest bucket over 1000 fixed seeds:");
    println!();
    let mut rng = Rng(14);
    let mut hist: BTreeMap<usize, (u32, u32)> = BTreeMap::new();
    for _ in 0..1000 {
        let seed = rng.next_u64();
        let mut a = Table::new(BUCKETS, seeded(seed));
        let mut b = Table::new(BUCKETS, seeded(seed));
        for k in &attack {
            a.put(k);
        }
        for k in &normal {
            b.put(k);
        }
        hist.entry(a.longest()).or_default().0 += 1;
        hist.entry(b.longest()).or_default().1 += 1;
    }
    println!(
        "  {:>14} {:>22} {:>20}",
        "longest bucket", "seeds, attacker keys", "seeds, normal keys"
    );
    for (len, (a, b)) in &hist {
        println!("  {:>14} {:>22} {:>20}", len, a, b);
    }
    println!();
    println!("  under a seed the attacker keys behave like any other keys: the same");
    println!("  few values, nowhere near 8000.");

    println!();
    println!("Rust's HashSet does this for you, by default:");
    println!();
    let std_normal = time_set::<RandomState>(&normal);
    let std_attack = time_set::<RandomState>(&attack);
    println!("  {:<30} {:>14.0} ns", "HashSet, normal input", std_normal);
    println!(
        "  {:<30} {:>14.0} ns   {:.2}x",
        "HashSet, attacker input",
        std_attack,
        std_attack / std_normal
    );
    println!();
    println!("  The default hasher is `RandomState`: SipHash-1-3 keyed with 128 random");
    println!("  bits that std draws from the operating system once per thread (each");
    println!("  `RandomState::new()` then bumps one half of the key by one, so no two");
    println!("  sets share a key). A collision set computed against one set is worthless");
    println!("  against another set, and against the same program after a restart:");
    println!();
    let fixed = |s: &str| {
        let mut h = DefaultHasher::new();
        s.hash(&mut h);
        h.finish()
    };
    let probe = "evdaaaaa";
    let r: Vec<String> = (0..3)
        .map(|_| format!("{:#018x}", RandomState::new().hash_one(probe)))
        .collect();
    println!("  RandomState::new() x3, hash of {probe:?}:");
    for h in &r {
        println!("    {h}");
    }
    println!("  DefaultHasher::new() x3, hash of {probe:?}:");
    for _ in 0..3 {
        println!("    {:#018x}", fixed(probe));
    }
    println!();
    println!("  The three RandomStates disagree with each other and will disagree again");
    println!("  on the next run. `DefaultHasher::new()` has FIXED keys (0, 0): one value,");
    println!("  every time, in every program built with this std. The algorithm is");
    println!("  documented as unspecified and may change between releases, so never store");
    println!("  the value -- and it is not a defence, as the tables below show.");

    // The Go section ends here. What Go cannot show: std's table is not what
    // protects you. Give the very same HashSet an unseeded hasher and the
    // attack comes back.
    println!();
    println!("the Swiss table is not the defence. The hasher is. The same std HashSet,");
    println!("{N} keys, three different key sets, with the seeded default hasher and with");
    println!("a public unseeded one (djb2 behind `BuildHasherDefault`):");
    println!();
    print_profile_header("keys");
    let sets: [(&str, &[String]); 3] = [
        ("normal", &normal),
        ("mod 4096 (above)", &attack),
        ("full (Ez/FY)", &blocks),
    ];
    let sip: Vec<_> = sets
        .iter()
        .map(|(_, k)| profile::<RandomState>(k))
        .collect();
    let djb: Vec<_> = sets.iter().map(|(_, k)| profile::<PublicDjb2>(k)).collect();
    for (label, rows) in [("RandomState (seeded)", &sip), ("djb2, no seed", &djb)] {
        for ((name, _), p) in sets.iter().zip(rows) {
            print_profile(label, name, *p, rows[0].0);
        }
    }
    println!();
    println!("  `== calls` counts how many times the table compared two keys for equality");
    println!("  while building the set; `tags` is how many different 7-bit tags the keys");
    println!("  carry (at most 128); `starts` is how many different slots probing starts");
    println!("  from (at most 16384). Keys that collide completely cost the unseeded");
    println!(
        "  table {} comparisons -- {} x {} / 2, the very number the chained table",
        djb[2].1,
        N,
        N - 1
    );
    println!(
        "  made above -- and the seeded table {}. Keys that collide only in",
        sip[2].1
    );
    println!(
        "  the low 12 bits cost the unseeded table {} as well: the 8-letter keys' djb2",
        djb[1].1
    );
    println!(
        "  hashes are small numbers, so their top bits barely vary ({} distinct tag(s) among",
        djb[1].2
    );
    println!("  {N} keys), the tag cannot tell the keys apart, and every key in the probe");
    println!("  window has to be compared.");

    // And SipHash itself is only as good as its key. DefaultHasher::new() uses
    // the key (0, 0), which is public, so an attacker can search for keys that
    // share the low bits of a SipHash output -- the bits std uses to pick the
    // starting slot. 4000 keys keep the search to a fraction of a second.
    println!();
    println!("and SipHash itself is only as good as its key. With the key public");
    println!("(`DefaultHasher::new()`, key 0), the brute-force search from the top of this");
    println!("example works on it too, aimed at the low 13 bits that a 4000-key table");
    println!("(8192 slots) uses to pick where to start looking:");
    println!();
    let t0 = Instant::now();
    let (flood, tried) = colliding_keys(4000, 8192, fixed);
    let flood_ms = t0.elapsed().as_secs_f64() * 1000.0;
    println!(
        "  found {} keys after {tried} candidates ({flood_ms:.0} ms on this run), e.g. {}",
        flood.len(),
        show(&flood[..3])
    );
    println!();
    let small = &normal[..4000];
    let rows = [
        (
            "RandomState (seeded)",
            "normal",
            profile::<RandomState>(small),
        ),
        (
            "RandomState (seeded)",
            "aimed at key 0",
            profile::<RandomState>(&flood),
        ),
        ("DefaultHasher, key 0", "normal", profile::<FixedSip>(small)),
        (
            "DefaultHasher, key 0",
            "aimed at key 0",
            profile::<FixedSip>(&flood),
        ),
    ];
    print_profile_header("4000 keys");
    for (i, (label, name, p)) in rows.iter().enumerate() {
        print_profile(label, name, *p, rows[i / 2 * 2].2.0);
    }
    println!();
    println!(
        "  every aimed key starts probing from the same slot ({} of 8192), yet only",
        rows[3].2.3
    );
    println!(
        "  {} comparisons were needed, because the tags still differ ({} of them).",
        rows[3].2.1, rows[3].2.2
    );
    println!("  So this attack is weaker than the one on djb2 -- but it is a slowdown the");
    println!("  seeded hasher does not have, and it comes entirely from the key being known.");
    println!("  (That the extra time goes into walking the probe sequence is the");
    println!("  plausible mechanism; it is not measured here.)");
    println!();

    println!("and the tables from the top of this example, with the seed supplied by");
    println!("std instead of by hand -- `RandomState::hash_one` is Rust's `maphash`:");
    println!();
    let rs = RandomState::new();
    let mut with_std = Table::new(BUCKETS, move |s| rs.hash_one(s));
    for k in &attack {
        with_std.put(k);
    }
    let t_std = ns_per_op(|| {
        let rs = RandomState::new();
        let mut f = Table::new(BUCKETS, move |s| rs.hash_one(s));
        for k in black_box(&attack) {
            f.put(k);
        }
        black_box(f.n);
    });
    println!(
        "  {:<30} {:>8} keys, longest bucket {}, {:>10.0} ns",
        "attacker input, hash_one",
        with_std.n,
        with_std.longest(),
        t_std
    );

    println!();
    println!("what this means for code you write:");
    println!();
    for (what, verdict) in [
        (
            "HashMap<K, V> / HashSet<T> (RandomState)",
            "safe -- seeded for you",
        ),
        ("your own hash table, fixed hash", "VULNERABLE"),
        (
            "HashMap<K, V, BuildHasherDefault<H>>",
            "VULNERABLE -- no seed (measured: djb2)",
        ),
        (
            "BuildHasherDefault<DefaultHasher>",
            "VULNERABLE -- key 0 is public (measured)",
        ),
        (
            "your own table, RandomState::hash_one",
            "safe -- seeded, and that is its job",
        ),
    ] {
        println!("  {what:<44} {verdict}");
    }
    println!();
    println!("  A hasher that takes no seed is a fine checksum and a bad table hash for");
    println!("  untrusted input. Reach for `BuildHasherDefault` only when the keys are");
    println!("  yours (the `Fast` hasher of example 7 is exactly that: faster, unseeded,");
    println!("  and for trusted keys only). If you must build your own table, seed it");
    println!("  from `RandomState`.");
    println!();
    println!("  and the general principle, which outlives this example: a data");
    println!("  structure whose worst case is much worse than its average case");
    println!("  has a security property, not just a performance one. Ask who");
    println!("  chooses the input.");
}
```

**Rust sample output:**

```
the attack: find keys that all hash to the same bucket.

  generated 8000 colliding keys, e.g. [evdaaaaa inhaaaaa mflaaaaa twtaaaaa]
  weak_hash % 4096 for each of those: 2505 2505 2505 2505 

  brute force tried 32729586 candidate strings (231 ms on this run): about
  4091 candidates per key found, because one candidate in 4096 lands in the
  right bucket. A one-time cost for the attacker; every request after
  it is free to send.

  but djb2 does not even need the search. Concatenating 13 blocks, each
  "Ez" or "FY", builds keys that collide on ALL 64 bits:

    EzEzEzEzEzEzEzEzEzEzEzEzEz  weak_hash = 0xb1c6690c7c62e798
    FYEzEzEzEzEzEzEzEzEzEzEzEz  weak_hash = 0xb1c6690c7c62e798
    EzFYEzEzEzEzEzEzEzEzEzEzEz  weak_hash = 0xb1c6690c7c62e798
    FYFYEzEzEzEzEzEzEzEzEzEzEz  weak_hash = 0xb1c6690c7c62e798

  8000 keys, 1 distinct hash value(s), built in 509 microseconds.

what those keys do to a table with a fixed, public hash:

  normal input                       8000 keys, longest bucket 8
  attacker input                     8000 keys, longest bucket 8000

  string comparisons (counted, not timed):
  normal input                          12331 to put 8000 keys,        20331 to get them all back (2.5 per get)
  attacker input                     31996000 to put 8000 keys,     32004000 to get them all back (4000.5 per get)

  normal input                           248101 ns to insert 8000 keys
  attacker input                       48517658 ns   196x slower

  same code, same number of keys, same table size -- and the
  longest bucket went from 8 to 8000. One request became 196x more
  expensive, with no memory blow-up and nothing in the logs.

  scale that: an HTTP server parsing a POST body into a map, or a
  JSON decoder building a map of strings, will do this on any input
  it is handed. That is the 2011 disclosure, exactly.

the defence is one word: SEED.

  attacker input, seeded             8000 keys, longest bucket 9
                                         293957 ns   165x faster than the attack

  the keys still collide with EACH OTHER under weak_hash. They do
  not collide under seeded_hash, because the attacker did not know
  the seed when they generated them -- and could not have, since it
  is chosen at run time.

one seed is one sample. The longest bucket over 1000 fixed seeds:

  longest bucket   seeds, attacker keys   seeds, normal keys
               7                     26                   15
               8                    433                  407
               9                    412                  431
              10                    104                  118
              11                     23                   28
              12                      2                    1

  under a seed the attacker keys behave like any other keys: the same
  few values, nowhere near 8000.

Rust's HashSet does this for you, by default:

  HashSet, normal input                   87369 ns
  HashSet, attacker input                 76192 ns   0.87x

  The default hasher is `RandomState`: SipHash-1-3 keyed with 128 random
  bits that std draws from the operating system once per thread (each
  `RandomState::new()` then bumps one half of the key by one, so no two
  sets share a key). A collision set computed against one set is worthless
  against another set, and against the same program after a restart:

  RandomState::new() x3, hash of "evdaaaaa":
    0x08e3bf4576a377d0
    0x7c7580f18aadf763
    0xe9690dec6331a608
  DefaultHasher::new() x3, hash of "evdaaaaa":
    0x9440fcedf89c15af
    0x9440fcedf89c15af
    0x9440fcedf89c15af

  The three RandomStates disagree with each other and will disagree again
  on the next run. `DefaultHasher::new()` has FIXED keys (0, 0): one value,
  every time, in every program built with this std. The algorithm is
  documented as unspecified and may change between releases, so never store
  the value -- and it is not a defence, as the tables below show.

the Swiss table is not the defence. The hasher is. The same std HashSet,
8000 keys, three different key sets, with the seeded default hasher and with
a public unseeded one (djb2 behind `BuildHasherDefault`):

  hasher                 keys                          ns  x normal    == calls  tags starts
  RandomState (seeded)   normal                     86886      1.0x         132   128   6338
  RandomState (seeded)   mod 4096 (above)           76791      0.9x         116   128   6350
  RandomState (seeded)   full (Ez/FY)               94898      1.1x         129   128   6325
  djb2, no seed          normal                     61720      1.0x           0    35   5292
  djb2, no seed          mod 4096 (above)        13479336    218.4x     7996043     1      4
  djb2, no seed          full (Ez/FY)            48151992    780.2x    31996000     1      1

  `== calls` counts how many times the table compared two keys for equality
  while building the set; `tags` is how many different 7-bit tags the keys
  carry (at most 128); `starts` is how many different slots probing starts
  from (at most 16384). Keys that collide completely cost the unseeded
  table 31996000 comparisons -- 8000 x 7999 / 2, the very number the chained table
  made above -- and the seeded table 129. Keys that collide only in
  the low 12 bits cost the unseeded table 7996043 as well: the 8-letter keys' djb2
  hashes are small numbers, so their top bits barely vary (1 distinct tag(s) among
  8000 keys), the tag cannot tell the keys apart, and every key in the probe
  window has to be compared.

and SipHash itself is only as good as its key. With the key public
(`DefaultHasher::new()`, key 0), the brute-force search from the top of this
example works on it too, aimed at the low 13 bits that a 4000-key table
(8192 slots) uses to pick where to start looking:

  found 4000 keys after 32501487 candidates (313 ms on this run), e.g. [xvhaaaaa ktnaaaaa tvpaaaaa]

  hasher                 4000 keys                     ns  x normal    == calls  tags starts
  RandomState (seeded)   normal                     42550      1.0x          62   128   3185
  RandomState (seeded)   aimed at key 0             37646      0.9x          54   128   3165
  DefaultHasher, key 0   normal                     42777      1.0x          64   128   3159
  DefaultHasher, key 0   aimed at key 0            938896     21.9x       62586   128      1

  every aimed key starts probing from the same slot (1 of 8192), yet only
  62586 comparisons were needed, because the tags still differ (128 of them).
  So this attack is weaker than the one on djb2 -- but it is a slowdown the
  seeded hasher does not have, and it comes entirely from the key being known.
  (That the extra time goes into walking the probe sequence is the
  plausible mechanism; it is not measured here.)

and the tables from the top of this example, with the seed supplied by
std instead of by hand -- `RandomState::hash_one` is Rust's `maphash`:

  attacker input, hash_one           8000 keys, longest bucket 8,     332696 ns

what this means for code you write:

  HashMap<K, V> / HashSet<T> (RandomState)     safe -- seeded for you
  your own hash table, fixed hash              VULNERABLE
  HashMap<K, V, BuildHasherDefault<H>>         VULNERABLE -- no seed (measured: djb2)
  BuildHasherDefault<DefaultHasher>            VULNERABLE -- key 0 is public (measured)
  your own table, RandomState::hash_one        safe -- seeded, and that is its job

  A hasher that takes no seed is a fine checksum and a bad table hash for
  untrusted input. Reach for `BuildHasherDefault` only when the keys are
  yours (the `Fast` hasher of example 7 is exactly that: faster, unseeded,
  and for trusted keys only). If you must build your own table, seed it
  from `RandomState`.

  and the general principle, which outlives this example: a data
  structure whose worst case is much worse than its average case
  has a security property, not just a performance one. Ask who
  chooses the input.
```

> **Go vs Rust:** Every deterministic row is identical to Go: the same 8,000 keys (`evdaaaaa inhaaaaa mflaaaaa twtaaaaa`), bucket 2505, longest bucket 8 against 8,000, and the attack is 161-196x slower over five Rust runs (Go: 182x in this file's sample, 131x in my re-run). Go's "that took milliseconds to compute" is generous for the brute force: it tries 32,729,586 candidates and takes 245-292 ms here (455-757 ms in three runs of a copy of Go's `collidingKeys`), whereas djb2's complete 64-bit collisions (`Ez`/`FY` blocks) build 8,000 keys in about half a millisecond. The Go text's "9" and the sample's "10" are two draws of one random seed: over 1,000 fixed seeds the longest bucket is 7-12 for the attacker keys and for ordinary keys alike. Rust's std `HashSet` with its default `RandomState` (SipHash-1-3, random key per instance) shows no slowdown on the attacker keys (0.83-0.95x; Go's map: 1.05-1.08x), but the Swiss table is not what protects it: the same `HashSet` with an unseeded djb2 behind `BuildHasherDefault` is 220-237x slower on keys that collide mod 4096 (7,996,043 `==` calls: those 8-letter hashes all carry the same top 7 bits, so hashbrown's one-byte tag cannot tell the keys apart) and 790-850x slower on full collisions (31,996,000 calls, exactly 8000x7999/2), and even `DefaultHasher::new()` (fixed keys 0,0) is 18-22x slower on 4,000 keys brute-forced against its low 13 bits. `DefaultHasher::new()` gives one value in every run (deterministic for this std, documented as unspecified), `RandomState::new()` a new one each time, and `RandomState::hash_one` is Rust's `maphash`. One generic `Table<H>` replaces Go's two copies of the table; the wraparound in both hashes is an explicit `wrapping_mul`/`wrapping_add`.

**Complexity:** 8,000 attacker keys took the longest bucket from **8 to 8,000** and insertion **182×
slower** · a per-instance seed brings the longest bucket back to **9** · Go's map shows **1.08×**,
because the runtime seeds every hash — `hash/fnv` takes no seed and `hash/maphash` exists for this

---

## 15. Seven times a map is the wrong answer

`🔴 hard` · *"use a map" is the most over-applied advice in this repo*

Small `n`, dense integer keys, ordered output, memory, GC, concurrency, reproducibility. Every one of
these comes up in real code, and two of the measurements here contradicted what I was about to write.

**Steps:**

1. Find the small-`n` crossover for string keys and for int keys — they are not the same.
2. Compare `map[int]Config` against `map[int]*Config` on bytes *and* on GC.
3. Then price `sync.RWMutex` and `sync.Map` with no contention at all.

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
	"sync"
	"testing"
	"time"
)

// Nine examples of a map winning. Here is the other side, because "use a map"
// is the most over-applied advice in this repo and every one of the cases below
// comes up in real code.
//
//	1. small n                     a slice is faster and simpler
//	2. dense integer keys          an array IS a perfect hash table
//	3. ordered output              the map cannot answer the question
//	4. memory pressure             3-4x the raw data (example 1)
//	5. GC pressure                 pointer keys or values are traced forever
//	6. concurrent access           a map is not safe, and the fixes are not free
//	7. reproducible output         iteration order is randomised on purpose

var sinkI int
var sinkB bool

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func liveKB(build func() any) float64 {
	runtime.GC()
	runtime.GC()
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	keep := build()
	runtime.GC()
	runtime.GC()
	runtime.ReadMemStats(&b)
	runtime.KeepAlive(keep)
	return float64(b.HeapAlloc-a.HeapAlloc) / 1024
}

// gcCost measures the WALL TIME of a forced collection with the structure
// live, plus the object count. MemStats.PauseTotalNs measures only stop-the-
// world time, and Go's collector is concurrent -- so it reports ~0 whatever you
// do, which is how my first version of this example "proved" that a map of
// pointers costs the GC nothing.
func gcCost(build func() any) (objects uint64, msPerCycle float64) {
	runtime.GC()
	runtime.GC()
	keep := build()
	const cycles = 20
	start := time.Now()
	for i := 0; i < cycles; i++ {
		runtime.GC()
	}
	el := time.Since(start)
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	runtime.KeepAlive(keep)
	return ms.HeapObjects, float64(el.Microseconds()) / 1000 / cycles
}

type Config struct {
	Name  string
	Value int
}

func main() {
	fmt.Println("1. SMALL n -- but the crossover is lower than the folklore says:")
	fmt.Println()
	fmt.Printf("  %6s %12s %10s %10s %10s %10s %10s\n",
		"n", "str slice", "str map", "ratio", "int slice", "int map", "ratio")
	for _, n := range []int{2, 4, 8, 16, 32, 64} {
		skeys := make([]string, n)
		ikeys := make([]int, n)
		for i := range skeys {
			skeys[i] = fmt.Sprintf("field_%d", i)
			ikeys[i] = i * 2654435761
		}
		sm := make(map[string]int, n)
		im := make(map[int]int, n)
		for i := range skeys {
			sm[skeys[i]] = i
			im[ikeys[i]] = i
		}
		sProbe, iProbe := skeys[n-1], ikeys[n-1] // the slice's worst case
		tss := nsPerOp(func() {
			for i, k := range skeys {
				if k == sProbe {
					sinkI = i
					break
				}
			}
		})
		tsm := nsPerOp(func() { _, sinkB = sm[sProbe] })
		tis := nsPerOp(func() {
			for i, k := range ikeys {
				if k == iProbe {
					sinkI = i
					break
				}
			}
		})
		tim := nsPerOp(func() { _, sinkB = im[iProbe] })
		fmt.Printf("  %6d %12.2f %10.2f %9.2fx %10.2f %10.2f %9.2fx\n",
			n, tss, tsm, tss/tsm, tis, tim, tis/tim)
	}
	fmt.Println()
	fmt.Println("  I expected the slice to win up to n=30 or so. The ratios cross")
	fmt.Println("  1.0 between n=2 and n=4 for STRING keys, and between n=8 and")
	fmt.Println("  n=16 for INT keys -- and the gap between those is the point.")
	fmt.Println()
	fmt.Println("  comparing two strings means comparing their bytes; comparing two")
	fmt.Println("  ints is one instruction. Meanwhile hashing a short string costs")
	fmt.Println("  about what hashing an int does. So the slice's advantage is eaten")
	fmt.Println("  by the comparison, not by the hash.")
	fmt.Println()
	fmt.Println("  lesson 01 measured the int crossover at n=16 and it is in the")
	fmt.Println("  same place here. The rule to carry away is not a number -- it is")
	fmt.Println("  that the number depends on how expensive `==` is for your key.")
	fmt.Println()
	fmt.Println("  (this is the slice's worst case: a full scan to the last element.")
	fmt.Println("  If lookups are usually hits near the front, it does better.)")

	fmt.Println("2. DENSE INTEGER KEYS -- an array is already a perfect hash table:")
	fmt.Println()
	const dense = 1_000_000
	tMapDense := nsPerOp(func() {
		m := make(map[int]int, dense)
		for i := 0; i < dense; i++ {
			m[i] = i * 2
		}
		sinkI = len(m)
	})
	tSliceDense := nsPerOp(func() {
		s := make([]int, dense)
		for i := 0; i < dense; i++ {
			s[i] = i * 2
		}
		sinkI = len(s)
	})
	mapKB := liveKB(func() any {
		m := make(map[int]int, dense)
		for i := 0; i < dense; i++ {
			m[i] = i * 2
		}
		return m
	})
	sliceKB := liveKB(func() any {
		s := make([]int, dense)
		for i := 0; i < dense; i++ {
			s[i] = i * 2
		}
		return s
	})
	fmt.Printf("  %-30s %14.0f ns build, %8.0f KB\n", "map[int]int, keys 0..999999", tMapDense, mapKB)
	fmt.Printf("  %-30s %14.0f ns build, %8.0f KB\n", "[]int of the same", tSliceDense, sliceKB)
	fmt.Printf("  %-30s %13.0fx faster, %7.1fx smaller\n", "",
		tMapDense/tSliceDense, mapKB/sliceKB)
	fmt.Println()
	fmt.Println("  a slice index IS a hash function: perfect, collision-free, one")
	fmt.Println("  instruction. Whenever the keys are integers in a known, mostly-")
	fmt.Println("  dense range, an array is not an optimisation of a map -- it is")
	fmt.Println("  the correct data structure and the map is the workaround.")
	fmt.Println()
	fmt.Println("  the same applies to byte and rune keys ([256]T), small enums, and")
	fmt.Println("  anything you can number. Lesson 14's counting sort is this idea.")

	fmt.Println()
	fmt.Println("3. ORDERED OUTPUT -- the map cannot answer the question at all:")
	fmt.Println()
	fmt.Printf("  %-34s %-16s %s\n", "question", "sorted slice", "map")
	for _, r := range [][3]string{
		{"is x present?", "Theta(log n)", "Theta(1)"},
		{"what is the smallest key?", "Theta(1)", "Theta(n)"},
		{"the smallest key above x?", "Theta(log n)", "Theta(n)"},
		{"all keys between a and b?", "Theta(log n + k)", "Theta(n)"},
		{"the median key?", "Theta(1)", "Theta(n log n)"},
		{"iterate in order", "Theta(n)", "Theta(n log n)"},
	} {
		fmt.Printf("  %-34s %-16s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  a map answers exactly one of those well. If your workload has")
	fmt.Println("  more than one row in it, you want a tree (lessons 17-19) or a")
	fmt.Println("  sorted slice -- and lesson 11's table measured the cost.")

	fmt.Println()
	fmt.Println("4 and 5. MEMORY AND GC -- and a result I had to fix twice:")
	fmt.Println()
	const n = 300_000
	valObjs, valMS := gcCost(func() any {
		m := make(map[int]Config, n)
		for i := 0; i < n; i++ {
			m[i] = Config{Name: "x", Value: i}
		}
		return m
	})
	ptrObjs, ptrMS := gcCost(func() any {
		m := make(map[int]*Config, n)
		for i := 0; i < n; i++ {
			m[i] = &Config{Name: "x", Value: i}
		}
		return m
	})
	valKB := liveKB(func() any {
		m := make(map[int]Config, n)
		for i := 0; i < n; i++ {
			m[i] = Config{Name: "x", Value: i}
		}
		return m
	})
	ptrKB := liveKB(func() any {
		m := make(map[int]*Config, n)
		for i := 0; i < n; i++ {
			m[i] = &Config{Name: "x", Value: i}
		}
		return m
	})
	fmt.Printf("  %-24s %10s %14s %18s\n", "", "KB", "heap objects", "ms per GC cycle")
	fmt.Printf("  %-24s %10.0f %14d %18.3f\n", "map[int]Config", valKB, valObjs, valMS)
	fmt.Printf("  %-24s %10.0f %14d %18.3f\n", "map[int]*Config", ptrKB, ptrObjs, ptrMS)
	fmt.Println()
	fmt.Printf("  the pointer map is SMALLER -- %.0f KB against %.0f -- which is the\n", ptrKB, valKB)
	fmt.Println("  opposite of the advice I was about to write. Config is 24 bytes,")
	fmt.Println("  so storing it by value makes every SLOT 32 bytes wide, and a map")
	fmt.Println("  allocates slots for its capacity, not its length. Replacing the")
	fmt.Println("  value with an 8-byte pointer shrinks every slot including the")
	fmt.Println("  empty ones, and that saves more than the 300,000 little")
	fmt.Println("  allocations cost.")
	fmt.Println()
	fmt.Printf("  and now the column that matters: %d heap objects against %d,\n", ptrObjs, valObjs)
	fmt.Printf("  and %.1fx the GC time on every cycle, forever, whether or not the\n", ptrMS/valMS)
	fmt.Println("  map is being used.")
	fmt.Println()
	fmt.Println("  so the guidance survives but the reason changes: store small")
	fmt.Println("  values BY VALUE not to save bytes -- it may cost bytes -- but to")
	fmt.Println("  keep them out of the collector's reach.")
	fmt.Println()
	fmt.Println("  lesson 04 measured this as 59x for []*T over []T, lesson 07 as")
	fmt.Println("  149,997 heap objects for a list, and example 11 as 31x GC time")
	fmt.Println("  for chaining. Fourth structure, same finding.")

	fmt.Println("6. CONCURRENCY -- a map is not safe, and the runtime says so:")
	fmt.Println()
	fmt.Println("      fatal error: concurrent map writes")
	fmt.Println()
	fmt.Println("  that is a FATAL ERROR, not a panic. `recover` does not catch it")
	fmt.Println("  and the process dies -- the same policy as lesson 05's stack")
	fmt.Println("  overflow, and for the same reason: the runtime cannot know what")
	fmt.Println("  state the map is in.")
	fmt.Println()
	const conc = 200_000
	keys := make([]int, conc)
	for i := range keys {
		keys[i] = i * 2654435761
	}
	plain := make(map[int]int, conc)
	for _, k := range keys {
		plain[k] = 1
	}
	var mu sync.RWMutex
	var sm sync.Map
	for _, k := range keys {
		sm.Store(k, 1)
	}
	tPlain := nsPerOp(func() {
		for _, k := range keys {
			_, sinkB = plain[k]
		}
	}) / conc
	tRW := nsPerOp(func() {
		for _, k := range keys {
			mu.RLock()
			_, sinkB = plain[k]
			mu.RUnlock()
		}
	}) / conc
	tSync := nsPerOp(func() {
		for _, k := range keys {
			_, sinkB = sm.Load(k)
		}
	}) / conc
	fmt.Printf("  %-30s %10.2f ns/read\n", "plain map (unsafe)", tPlain)
	fmt.Printf("  %-30s %10.2f ns/read   %.2fx\n", "sync.RWMutex + map", tRW, tRW/tPlain)
	fmt.Printf("  %-30s %10.2f ns/read   %.2fx\n", "sync.Map", tSync, tSync/tPlain)
	fmt.Println()
	fmt.Println("  single-goroutine, uncontended -- so this is the FLOOR, the price")
	fmt.Println("  before any contention at all. sync.Map is not a faster map; it is")
	fmt.Println("  a map optimised for a specific shape (write-once, read-many, or")
	fmt.Println("  disjoint key sets per goroutine) and it is slower than an RWMutex")
	fmt.Println("  everywhere else. Its own documentation says so.")
	fmt.Println()
	fmt.Println("  the usual right answer is a plain map behind a mutex, or -- much")
	fmt.Println("  better -- SHARDING: N maps each with its own lock, selected by")
	fmt.Println("  hash. Lesson 38 builds that.")

	fmt.Println()
	fmt.Println("7. REPRODUCIBLE OUTPUT -- the randomisation is not optional:")
	fmt.Println()
	m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4}
	for i := 0; i < 3; i++ {
		out := make([]string, 0, len(m))
		for k := range m {
			out = append(out, k)
		}
		fmt.Printf("  range order: %v\n", out)
	}
	sortedOut := slices.Sorted(func(yield func(string) bool) {
		for k := range m {
			if !yield(k) {
				return
			}
		}
	})
	fmt.Printf("  sorted:      %v   <- what you actually wanted\n", sortedOut)
	fmt.Println()
	fmt.Println("  anything that produces a file, a hash, a signature, a golden test")
	fmt.Println("  or a diff must sort. This is the single most common source of")
	fmt.Println("  flaky tests in Go, and it is a feature -- the alternative is code")
	fmt.Println("  that depends on an order the runtime never promised.")

	fmt.Println()
	fmt.Println("the summary, which is the useful part of this example:")
	fmt.Println()
	fmt.Printf("  %-34s %s\n", "n < ~16 and keys are ints", "use a slice")
	fmt.Printf("  %-34s %s\n", "integer keys in a dense range", "use an array")
	fmt.Printf("  %-34s %s\n", "you need order or ranges", "use a sorted slice or a tree")
	fmt.Printf("  %-34s %s\n", "values are small structs", "store by VALUE (for the GC)")
	fmt.Printf("  %-34s %s\n", "goroutines share it", "mutex, or shard it")
	fmt.Printf("  %-34s %s\n", "output must be stable", "sort the keys, every time")
	fmt.Printf("  %-34s %s\n", "anything else", "use a map -- it is very good")
}
```

**Sample output:**

```
1. SMALL n -- but the crossover is lower than the folklore says:

       n    str slice    str map      ratio  int slice    int map      ratio
       2         3.55       5.24      0.68x       1.79       2.04      0.88x
       4         7.24       7.56      0.96x       1.86       2.53      0.74x
       8        13.41       7.27      1.84x       2.63       3.67      0.72x
      16        12.81       4.95      2.59x       5.45       3.42      1.59x
      32        32.26       4.90      6.58x       9.61       3.39      2.83x
      64        81.43       4.88     16.67x      17.68       3.52      5.03x

  I expected the slice to win up to n=30 or so. The ratios cross
  1.0 between n=2 and n=4 for STRING keys, and between n=8 and
  n=16 for INT keys -- and the gap between those is the point.

  comparing two strings means comparing their bytes; comparing two
  ints is one instruction. Meanwhile hashing a short string costs
  about what hashing an int does. So the slice's advantage is eaten
  by the comparison, not by the hash.

  lesson 01 measured the int crossover at n=16 and it is in the
  same place here. The rule to carry away is not a number -- it is
  that the number depends on how expensive `==` is for your key.

  (this is the slice's worst case: a full scan to the last element.
  If lookups are usually hits near the front, it does better.)
2. DENSE INTEGER KEYS -- an array is already a perfect hash table:

  map[int]int, keys 0..999999          35873293 ns build,    36946 KB
  []int of the same                      357424 ns build,     7816 KB
                                           100x faster,     4.7x smaller

  a slice index IS a hash function: perfect, collision-free, one
  instruction. Whenever the keys are integers in a known, mostly-
  dense range, an array is not an optimisation of a map -- it is
  the correct data structure and the map is the workaround.

  the same applies to byte and rune keys ([256]T), small enums, and
  anything you can number. Lesson 14's counting sort is this idea.

3. ORDERED OUTPUT -- the map cannot answer the question at all:

  question                           sorted slice     map
  is x present?                      Theta(log n)     Theta(1)
  what is the smallest key?          Theta(1)         Theta(n)
  the smallest key above x?          Theta(log n)     Theta(n)
  all keys between a and b?          Theta(log n + k) Theta(n)
  the median key?                    Theta(1)         Theta(n log n)
  iterate in order                   Theta(n)         Theta(n log n)

  a map answers exactly one of those well. If your workload has
  more than one row in it, you want a tree (lessons 17-19) or a
  sorted slice -- and lesson 11's table measured the cost.

4 and 5. MEMORY AND GC -- and a result I had to fix twice:

                                   KB   heap objects    ms per GC cycle
  map[int]Config                20501           1355              0.531
  map[int]*Config               16268         301357              2.127

  the pointer map is SMALLER -- 16268 KB against 20501 -- which is the
  opposite of the advice I was about to write. Config is 24 bytes,
  so storing it by value makes every SLOT 32 bytes wide, and a map
  allocates slots for its capacity, not its length. Replacing the
  value with an 8-byte pointer shrinks every slot including the
  empty ones, and that saves more than the 300,000 little
  allocations cost.

  and now the column that matters: 301357 heap objects against 1355,
  and 4.0x the GC time on every cycle, forever, whether or not the
  map is being used.

  so the guidance survives but the reason changes: store small
  values BY VALUE not to save bytes -- it may cost bytes -- but to
  keep them out of the collector's reach.

  lesson 04 measured this as 59x for []*T over []T, lesson 07 as
  149,997 heap objects for a list, and example 11 as 31x GC time
  for chaining. Fourth structure, same finding.
6. CONCURRENCY -- a map is not safe, and the runtime says so:

      fatal error: concurrent map writes

  that is a FATAL ERROR, not a panic. `recover` does not catch it
  and the process dies -- the same policy as lesson 05's stack
  overflow, and for the same reason: the runtime cannot know what
  state the map is in.

  plain map (unsafe)                   5.54 ns/read
  sync.RWMutex + map                   6.62 ns/read   1.20x
  sync.Map                            17.76 ns/read   3.21x

  single-goroutine, uncontended -- so this is the FLOOR, the price
  before any contention at all. sync.Map is not a faster map; it is
  a map optimised for a specific shape (write-once, read-many, or
  disjoint key sets per goroutine) and it is slower than an RWMutex
  everywhere else. Its own documentation says so.

  the usual right answer is a plain map behind a mutex, or -- much
  better -- SHARDING: N maps each with its own lock, selected by
  hash. Lesson 38 builds that.

7. REPRODUCIBLE OUTPUT -- the randomisation is not optional:

  range order: [d a b c]
  range order: [d a b c]
  range order: [a b c d]
  sorted:      [a b c d]   <- what you actually wanted

  anything that produces a file, a hash, a signature, a golden test
  or a diff must sort. This is the single most common source of
  flaky tests in Go, and it is a feature -- the alternative is code
  that depends on an order the runtime never promised.

the summary, which is the useful part of this example:

  n < ~16 and keys are ints          use a slice
  integer keys in a dense range      use an array
  you need order or ranges           use a sorted slice or a tree
  values are small structs           store by VALUE (for the GC)
  goroutines share it                mutex, or shard it
  output must be stable              sort the keys, every time
  anything else                      use a map -- it is very good
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::collections::{BTreeMap, HashMap};
use std::hash::{BuildHasher, BuildHasherDefault, Hash, Hasher, RandomState};
use std::hint::black_box;
use std::mem::size_of;
use std::ops::Bound::{Excluded, Unbounded};
use std::process::Command;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::sync::{Mutex, RwLock};
use std::thread;
use std::time::{Duration, Instant};

// Nine examples of a map winning. Here is the other side, because "use a map"
// is the most over-applied advice in this repo and every one of the cases below
// comes up in real code.
//
//	1. small n                     a slice is faster and simpler
//	2. dense integer keys          an array IS a perfect hash table
//	3. ordered output              the map cannot answer the question
//	4. memory pressure             3-4x the raw data (example 1)
//	5. GC pressure                 pointer keys or values are traced forever
//	6. concurrent access           a map is not safe, and the fixes are not free
//	7. reproducible output         iteration order is randomised on purpose
//
// Rust has no GC, so case 5 becomes the question Rust DOES pay for -- one heap
// allocation per boxed value, and what that costs to build, walk and drop.
// Case 6 is a compile error here rather than a run-time crash.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Counting allocator: stand-in for runtime.MemStats (HeapAlloc, HeapObjects).
// Rust has no GC, so "how many bytes / how many objects" is an exact number and
// there is no `runtime.GC()` twice before every reading. It counts the bytes
// REQUESTED; what malloc rounds them up to is not visible from here.
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live
static LIVE_ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocations currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        LIVE.fetch_add(l.size(), Relaxed);
        LIVE_ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        LIVE_ALLOCS.fetch_sub(1, Relaxed);
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

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

// splitmix64 + Fisher-Yates: stand-in for math/rand (seeded, so reproducible).
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    fn shuffle<T>(&mut self, xs: &mut [T]) {
        for i in (1..xs.len()).rev() {
            let j = ((self.next_u64() as u128 * (i as u128 + 1)) >> 64) as usize;
            xs.swap(i, j);
        }
    }
}

/// (KB still live after `build`, allocations still live, the built value):
/// liveKB and HeapObjects of the Go program in one call.
fn footprint<T>(build: impl FnOnce() -> T) -> (f64, usize, T) {
    let (b0, a0) = (LIVE.load(Relaxed) as i64, LIVE_ALLOCS.load(Relaxed) as i64);
    let kept = build();
    let kb = (LIVE.load(Relaxed) as i64 - b0) as f64 / 1024.0;
    let allocs = (LIVE_ALLOCS.load(Relaxed) as i64 - a0) as usize;
    (kb, allocs, kept)
}

fn median(mut xs: Vec<f64>) -> f64 {
    xs.sort_by(f64::total_cmp);
    xs[xs.len() / 2]
}

/// `[a b c]`, the way Go's %v prints a slice ({:?} would give `["a", "b"]`)
fn show(xs: &[&str]) -> String {
    format!("[{}]", xs.join(" "))
}

// ---- a second hasher, so the timing comparisons are fair ----
//
// std's default is SipHash-1-3 with a random key per map: built to resist
// collision attacks (example 14), not to be the fastest hash there is. `Fast` is
// the finalized FNV-1a of example 2 (the same one example 7 uses): FNV-1a over
// the bytes of a string, ONE xor-multiply round over the whole word for an
// integer, then the splitmix64 finalizer. It has no random seed, so it is
// exactly the hash example 14 shows an attacker can aim at: use it for keys you
// trust, as here.
const FNV_OFFSET: u64 = 14695981039346656037;
const FNV_PRIME: u64 = 1099511628211;

struct Fast(u64);
impl Default for Fast {
    fn default() -> Self {
        Fast(FNV_OFFSET)
    }
}
impl Hasher for Fast {
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 ^= b as u64;
            self.0 = self.0.wrapping_mul(FNV_PRIME);
        }
    }
    // i64 keys arrive here: write_i64 forwards to write_u64.
    fn write_u64(&mut self, x: u64) {
        self.0 = (self.0 ^ x).wrapping_mul(FNV_PRIME);
    }
    fn finish(&self) -> u64 {
        let mut h = self.0;
        h ^= h >> 30;
        h = h.wrapping_mul(0xbf58476d1ce4e5b9);
        h ^= h >> 27;
        h = h.wrapping_mul(0x94d049bb133111eb);
        h ^ (h >> 31)
    }
}
type FastState = BuildHasherDefault<Fast>;

// ---- 1. small n ----

/// one row of the Go table: ns per lookup of the LAST key (the slice's worst
/// case, a full scan) for (string slice, string map, int slice, int map).
fn small_row<S: BuildHasher + Default>(n: usize) -> (f64, f64, f64, f64) {
    let skeys: Vec<String> = (0..n).map(|i| format!("field_{i}")).collect();
    let ikeys: Vec<i64> = (0..n as i64).map(|i| i * 2654435761).collect();
    let mut sm: HashMap<String, usize, S> = HashMap::with_capacity_and_hasher(n, S::default());
    let mut im: HashMap<i64, usize, S> = HashMap::with_capacity_and_hasher(n, S::default());
    for i in 0..n {
        sm.insert(skeys[i].clone(), i);
        im.insert(ikeys[i], i);
    }
    let (sprobe, iprobe) = (skeys[n - 1].as_str(), ikeys[n - 1]);
    // a scan returns the index it found the key at, as Go's sinkI does;
    // black_box keeps the optimiser from hoisting or deleting the loops.
    let tss = ns_per_op(|| {
        let p = black_box(sprobe);
        black_box(black_box(&skeys).iter().position(|k| k == p));
    });
    let tsm = ns_per_op(|| {
        black_box(sm.get(black_box(sprobe)));
    });
    let tis = ns_per_op(|| {
        let p = black_box(iprobe);
        black_box(black_box(&ikeys).iter().position(|&k| k == p));
    });
    let tim = ns_per_op(|| {
        black_box(im.get(&black_box(iprobe)));
    });
    (tss, tsm, tis, tim)
}

/// print the Go table for one hasher; returns the first n at which the map beat
/// the slice, for strings and for ints (None: never in this range).
fn small_table<S: BuildHasher + Default>(sizes: &[usize]) -> (Option<usize>, Option<usize>) {
    println!(
        "  {:>6} {:>12} {:>10} {:>10} {:>10} {:>10} {:>10}",
        "n", "str slice", "str map", "ratio", "int slice", "int map", "ratio"
    );
    let (mut s_cross, mut i_cross) = (None, None);
    for &n in sizes {
        let (tss, tsm, tis, tim) = small_row::<S>(n);
        println!(
            "  {:>6} {:>12.2} {:>10.2} {:>9.2}x {:>10.2} {:>10.2} {:>9.2}x",
            n,
            tss,
            tsm,
            tss / tsm,
            tis,
            tim,
            tis / tim
        );
        if tss > tsm && s_cross.is_none() {
            s_cross = Some(n);
        }
        if tis > tim && i_cross.is_none() {
            i_cross = Some(n);
        }
    }
    (s_cross, i_cross)
}

fn cross(c: Option<usize>) -> String {
    c.map_or("never".to_string(), |n| format!("n={n}"))
}

// ---- 3. ordered output ----

// A key whose comparisons are counted, so that "how much work did that
// question take" is a number rather than a stopwatch. `Ord` is what a sorted
// slice and a BTreeMap use; `Eq` is what a HashMap uses to confirm a hit. WORK
// also counts the entries a scan steps over (see `visit`). Together: "keys
// examined".
thread_local! {
    static WORK: Cell<u64> = const { Cell::new(0) };
}
fn visit() {
    WORK.with(|w| w.set(w.get() + 1));
}
/// run `f`, return its result and how many keys it examined
fn examined<R>(f: impl FnOnce() -> R) -> (R, u64) {
    WORK.with(|w| w.set(0));
    let r = f();
    (r, WORK.with(Cell::get))
}

#[derive(Clone, Copy)]
struct Key(u64);
impl PartialEq for Key {
    fn eq(&self, o: &Key) -> bool {
        visit();
        self.0 == o.0
    }
}
impl Eq for Key {}
impl PartialOrd for Key {
    fn partial_cmp(&self, o: &Key) -> Option<std::cmp::Ordering> {
        Some(self.cmp(o))
    }
}
impl Ord for Key {
    fn cmp(&self, o: &Key) -> std::cmp::Ordering {
        visit();
        self.0.cmp(&o.0)
    }
}
impl Hash for Key {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.0.hash(state)
    }
}

// ---- 4 and 5. memory and the per-value allocation ----

struct Config {
    name: &'static str,
    value: i64,
}

struct MapRow {
    kb: f64,
    allocs: usize,
    build_ms: f64,
    scan_ms: f64,
    drop_ms: f64,
}

/// what one map of `n` configs costs: bytes and allocations (exact), and the
/// median of 5 runs of build / walk-every-value / drop (milliseconds).
fn measure_map<V>(n: usize, make: impl Fn(i64) -> V, value_of: impl Fn(&V) -> i64) -> MapRow {
    let build = || {
        let mut m: HashMap<i64, V> = HashMap::with_capacity(n);
        for i in 0..n as i64 {
            m.insert(i, make(i));
        }
        m
    };
    let (kb, allocs, m) = footprint(build);
    drop(m);
    let (mut b, mut s, mut d) = (Vec::new(), Vec::new(), Vec::new());
    for _ in 0..5 {
        let t = Instant::now();
        let m = build();
        b.push(t.elapsed().as_secs_f64() * 1000.0);
        let t = Instant::now();
        black_box(m.values().map(&value_of).sum::<i64>());
        s.push(t.elapsed().as_secs_f64() * 1000.0);
        let t = Instant::now();
        drop(black_box(m));
        d.push(t.elapsed().as_secs_f64() * 1000.0);
    }
    MapRow {
        kb,
        allocs,
        build_ms: median(b),
        scan_ms: median(s),
        drop_ms: median(d),
    }
}

// ---- 6. concurrency ----

/// 16 maps, each behind its own lock, picked by a cheap multiplicative hash.
struct Sharded<M> {
    shards: Vec<M>,
}
fn shard_of(k: i64) -> usize {
    ((k as u64).wrapping_mul(0x9e3779b97f4a7c15) >> 60) as usize
}

/// resident set size of this process in KB, as `ps` reports it
fn rss_kb() -> Option<usize> {
    let out = Command::new("ps")
        .args(["-o", "rss=", "-p", &std::process::id().to_string()])
        .output()
        .ok()?;
    String::from_utf8_lossy(&out.stdout).trim().parse().ok()
}

/// the child half of the resident-memory measurement: build ONE map in a fresh
/// process and print how much the resident set grew. (A fresh process, because
/// inside the one that already built and freed a dozen maps the allocator would
/// hand back memory it still holds, and the delta would measure that instead.)
fn rss_child(boxed: bool, n: usize) {
    let before = rss_kb();
    let sum: i64 = if boxed {
        let mut m: HashMap<i64, Box<Config>> = HashMap::with_capacity(n);
        for i in 0..n as i64 {
            m.insert(
                i,
                Box::new(Config {
                    name: "x",
                    value: i,
                }),
            );
        }
        let after = rss_kb();
        println!("{}", after.zip(before).map_or(0, |(a, b)| a - b));
        m.values().map(|c| c.value + c.name.len() as i64).sum()
    } else {
        let mut m: HashMap<i64, Config> = HashMap::with_capacity(n);
        for i in 0..n as i64 {
            m.insert(
                i,
                Config {
                    name: "x",
                    value: i,
                },
            );
        }
        let after = rss_kb();
        println!("{}", after.zip(before).map_or(0, |(a, b)| a - b));
        m.values().map(|c| c.value + c.name.len() as i64).sum()
    };
    black_box(sum);
}

/// run this same program as a child to measure one map's resident growth (KB)
fn resident_of(mode: &str) -> Option<usize> {
    let mut runs: Vec<usize> = (0..3)
        .filter_map(|_| {
            let out = Command::new(std::env::current_exe().ok()?)
                .arg(mode)
                .output()
                .ok()?;
            String::from_utf8_lossy(&out.stdout).trim().parse().ok()
        })
        .filter(|&kb| kb > 0)
        .collect();
    runs.sort_unstable();
    runs.get(runs.len() / 2).copied()
}

fn main() {
    // `ex15 value` / `ex15 box`: the child runs of the resident-memory check in 4 and 5
    if let Some(mode) = std::env::args().nth(1) {
        rss_child(mode == "box", 300_000);
        return;
    }
    println!("1. SMALL n -- where does the map start to win?");
    println!();
    println!("  std's default hasher (SipHash-1-3, random key per map):");
    println!();
    let sizes = [2, 4, 8, 16, 32, 64];
    let (sip_s, sip_i) = small_table::<RandomState>(&sizes);
    println!();
    println!("  the same lookups with the `Fast` hasher (finalized FNV-1a, no seed):");
    println!();
    let (fast_s, fast_i) = small_table::<FastState>(&sizes);
    println!();
    println!("  first n at which the map beats the scan, string keys / int keys:");
    println!(
        "    SipHash-1-3  {} / {}      Fast  {} / {}",
        cross(sip_s),
        cross(sip_i),
        cross(fast_s),
        cross(fast_i)
    );
    println!();
    println!("  so the crossover is not one number. It depends on the key (what `==` and");
    println!("  the hash cost for it) and on the hasher: SipHash is the dearer lookup, so");
    println!("  the scan holds out longer against it. (Run it again: the Fast crossovers");
    println!("  move by one step between runs, because the ratios at n=2..4 are near 1.0.)");
    println!();
    println!("  the pieces, one operation each (ns, including about the same loop and");
    println!("  black_box overhead for every row, which the first row shows alone):");
    println!();
    let a = "field_63".to_string();
    let b = a.clone();
    let (x, y) = (63 * 2654435761i64, 63 * 2654435761i64);
    let sip = RandomState::new();
    let fast = FastState::default();
    let t_nothing = ns_per_op(|| {
        black_box(1u64);
    });
    let t_hs_sip = ns_per_op(|| {
        black_box(sip.hash_one(black_box(a.as_str())));
    });
    let t_hs_fast = ns_per_op(|| {
        black_box(fast.hash_one(black_box(a.as_str())));
    });
    let t_hi_sip = ns_per_op(|| {
        black_box(sip.hash_one(black_box(x)));
    });
    let t_hi_fast = ns_per_op(|| {
        black_box(fast.hash_one(black_box(x)));
    });
    let t_eq_s = ns_per_op(|| {
        black_box(black_box(&a) == black_box(&b));
    });
    let t_eq_i = ns_per_op(|| {
        black_box(black_box(x) == black_box(y));
    });
    println!(
        "  {:<34} {:>8.2}",
        "nothing (black_box of a constant)", t_nothing
    );
    println!(
        "  {:<34} {:>8.2}   {:>8.2}   (SipHash, Fast)",
        "hash an 8-byte string", t_hs_sip, t_hs_fast
    );
    println!(
        "  {:<34} {:>8.2}   {:>8.2}",
        "hash an i64", t_hi_sip, t_hi_fast
    );
    println!("  {:<34} {:>8.2}", "== on two equal 8-byte Strings", t_eq_s);
    println!("  {:<34} {:>8.2}", "== on two i64", t_eq_i);
    println!();
    println!(
        "  a string hash costs {:.1}x an int hash under SipHash and {:.1}x under Fast",
        t_hs_sip / t_hi_sip,
        t_hs_fast / t_hi_fast
    );
    println!("  (Fast hashes an i64 with one multiply and a string byte by byte, as written).");
    println!();
    // The string scan is not monotonic in n above: the keys field_0..field_9
    // are 7 bytes, field_10.. are 8. Strings of different length differ at the
    // length check, so a scan over mixed lengths skips the byte compare for
    // some keys. Same keys padded to one width test whether that is the cause.
    let scan = |keys: &[String]| {
        let p = keys[keys.len() - 1].as_str();
        ns_per_op(|| {
            black_box(black_box(keys).iter().position(|k| k == black_box(p)));
        })
    };
    let plain = |n: usize| (0..n).map(|i| format!("field_{i}")).collect::<Vec<_>>();
    let padded = |n: usize| (0..n).map(|i| format!("field_{i:02}")).collect::<Vec<_>>();
    println!(
        "  string scan, ns per key examined, keys field_N (7-8 bytes) against field_NN (8 bytes):"
    );
    for n in [8, 16, 32] {
        println!(
            "    n={n:<3} field_N {:.2}    field_NN {:.2}",
            scan(&plain(n)) / n as f64,
            scan(&padded(n)) / n as f64
        );
    }
    println!();
    println!("  fixed-width keys cost about the same per key at every n; mixed-width keys");
    println!("  do not, so the dip in the string-scan columns at n=16 is about the KEYS");
    println!("  (field_0..field_9 are 7 bytes, field_10.. are 8), not about scanning.");
    println!("  Plausible mechanism, not measured separately: strings of unequal length");
    println!("  differ at the length check, before any byte is compared.");
    println!();
    println!("  (this is the slice's worst case: a full scan to the last element.");
    println!("  If lookups are usually hits near the front, it does better.)");

    println!();
    println!("2. DENSE INTEGER KEYS -- an array is already a perfect hash table:");
    println!();
    const DENSE: usize = 1_000_000;
    let build_map = || {
        let mut m: HashMap<i64, i64> = HashMap::with_capacity(DENSE);
        for i in 0..DENSE as i64 {
            m.insert(i, i * 2);
        }
        m
    };
    let build_fast = || {
        let mut m: HashMap<i64, i64, FastState> =
            HashMap::with_capacity_and_hasher(DENSE, FastState::default());
        for i in 0..DENSE as i64 {
            m.insert(i, i * 2);
        }
        m
    };
    let build_vec = || (0..DENSE as i64).map(|i| i * 2).collect::<Vec<i64>>();
    let t_map = ns_per_op(|| {
        let m = build_map();
        black_box(&m);
    });
    let t_fast = ns_per_op(|| {
        let m = build_fast();
        black_box(&m);
    });
    let t_vec = ns_per_op(|| {
        let v = build_vec();
        black_box(&v);
    });
    let (kb_map, _, m1) = footprint(build_map);
    let (kb_fast, _, m2) = footprint(build_fast);
    let (kb_vec, _, v) = footprint(build_vec);
    black_box((&m1, &m2, &v));
    drop((m1, m2, v));
    println!(
        "  {:<34} {:>14.0} ns build, {:>8.0} KB",
        "HashMap<i64, i64>, keys 0..999999", t_map, kb_map
    );
    println!(
        "  {:<34} {:>14.0} ns build, {:>8.0} KB",
        "  the same with Fast", t_fast, kb_fast
    );
    println!(
        "  {:<34} {:>14.0} ns build, {:>8.0} KB",
        "Vec<i64> of the same", t_vec, kb_vec
    );
    println!(
        "  {:<34} {:>13.0}x faster, {:>7.1}x smaller   (SipHash map)",
        "",
        t_map / t_vec,
        kb_map / kb_vec
    );
    println!(
        "  {:<34} {:>13.0}x faster, {:>7.1}x smaller   (Fast map)",
        "",
        t_fast / t_vec,
        kb_fast / kb_vec
    );
    println!();
    println!("  a slice index IS a hash function: perfect, collision-free, one");
    println!("  instruction. Whenever the keys are integers in a known, mostly-");
    println!("  dense range, an array is not an optimisation of a map -- it is");
    println!("  the correct data structure and the map is the workaround.");
    println!();
    println!("  the same applies to byte keys ([T; 256]), small enums, and");
    println!("  anything you can number. Lesson 14's counting sort is this idea.");

    println!();
    println!("3. ORDERED OUTPUT -- the map cannot answer the question at all:");
    println!();
    println!("  {:<34} {:<16} map", "question", "sorted slice");
    for r in [
        ["is x present?", "Theta(log n)", "Theta(1)"],
        ["what is the smallest key?", "Theta(1)", "Theta(n)"],
        ["the smallest key above x?", "Theta(log n)", "Theta(n)"],
        ["all keys between a and b?", "Theta(log n + k)", "Theta(n)"],
        ["the median key?", "Theta(1)", "Theta(n log n)"],
        ["iterate in order", "Theta(n)", "Theta(n log n)"],
    ] {
        println!("  {:<34} {:<16} {}", r[0], r[1], r[2]);
    }
    println!();
    println!("  a map answers exactly one of those well. If your workload has");
    println!("  more than one row in it, you want a tree (lessons 17-19) or a");
    println!("  sorted slice -- and lesson 11's table measured the cost.");
    println!();
    println!("  Rust's std has the tree: BTreeMap. Here are those six questions asked of a");
    println!("  sorted Vec, a BTreeMap and a HashMap holding the same 100,000 keys (the");
    println!("  even numbers 0..199998, inserted in shuffled order), x = 77778 (a key) and");
    println!("  the range 50000..=50998. The cost is counted, not timed: key comparisons");
    println!("  plus entries stepped over by a scan -- the same unit for every cell.");
    println!();
    const N3: u64 = 100_000;
    let mut order: Vec<u64> = (0..N3).map(|i| i * 2).collect();
    Rng(15).shuffle(&mut order);
    let hm: HashMap<Key, ()> = order.iter().map(|&k| (Key(k), ())).collect();
    let bt: BTreeMap<Key, ()> = order.iter().map(|&k| (Key(k), ())).collect();
    let mut sv: Vec<u64> = order.clone();
    sv.sort_unstable();
    let (x, lo, hi) = (77_778u64, 50_000u64, 50_998u64);
    // every row: (answer, work for the sorted Vec, for the BTreeMap, for the HashMap)
    type Row3 = (&'static str, String, [u64; 3]);
    let mut rows: Vec<Row3> = Vec::new();

    let (a1, w1) = examined(|| {
        sv.binary_search_by(|k| {
            visit();
            k.cmp(&x)
        })
        .is_ok()
    });
    let (a2, w2) = examined(|| bt.contains_key(&Key(x)));
    let (a3, w3) = examined(|| hm.contains_key(&Key(x)));
    assert!(a1 && a2 && a3);
    rows.push(("is x present?", a1.to_string(), [w1, w2, w3]));

    let (a1, w1) = examined(|| sv.first().copied());
    let (a2, w2) = examined(|| bt.first_key_value().map(|(k, _)| k.0));
    let (a3, w3) = examined(|| {
        hm.keys()
            .map(|k| {
                visit();
                k.0
            })
            .min()
    });
    assert!(a1 == a2 && a2 == a3);
    rows.push((
        "what is the smallest key?",
        format!("{}", a1.unwrap()),
        [w1, w2, w3],
    ));

    let (a1, w1) = examined(|| {
        let i = sv.partition_point(|&k| {
            visit();
            k <= x
        });
        sv.get(i).copied()
    });
    let (a2, w2) = examined(|| {
        bt.range((Excluded(Key(x)), Unbounded))
            .next()
            .map(|(k, _)| k.0)
    });
    let (a3, w3) = examined(|| {
        hm.keys()
            .map(|k| {
                visit();
                k.0
            })
            .filter(|&k| k > x)
            .min()
    });
    assert!(a1 == a2 && a2 == a3);
    rows.push((
        "the smallest key above x?",
        format!("{}", a1.unwrap()),
        [w1, w2, w3],
    ));

    let (a1, w1) = examined(|| {
        let from = sv.partition_point(|&k| {
            visit();
            k < lo
        });
        let to = sv.partition_point(|&k| {
            visit();
            k <= hi
        });
        // reading the k answers is part of the cost: Theta(log n + k)
        sv[from..to].iter().inspect(|_| visit()).count()
    });
    let (a2, w2) = examined(|| bt.range(Key(lo)..=Key(hi)).inspect(|_| visit()).count());
    let (a3, w3) = examined(|| {
        hm.keys()
            .map(|k| {
                visit();
                k.0
            })
            .filter(|&k| (lo..=hi).contains(&k))
            .count()
    });
    assert!(a1 == a2 && a2 == a3);
    rows.push((
        "all keys between a and b?",
        format!("{a1} keys"),
        [w1, w2, w3],
    ));

    let (a1, w1) = examined(|| sv[sv.len() / 2]);
    let (a2, w2) = examined(|| {
        bt.keys()
            .inspect(|_| visit())
            .nth(bt.len() / 2)
            .map(|k| k.0)
            .unwrap()
    });
    let (a3, w3) = examined(|| {
        let mut all: Vec<u64> = hm
            .keys()
            .map(|k| {
                visit();
                k.0
            })
            .collect();
        all.sort_unstable_by(|a, b| {
            visit();
            a.cmp(b)
        });
        all[all.len() / 2]
    });
    assert!(a1 == a2 && a2 == a3);
    rows.push(("the median key?", format!("{a1}"), [w1, w2, w3]));

    let (a1, w1) = examined(|| sv.iter().inspect(|_| visit()).count());
    let (a2, w2) = examined(|| bt.keys().inspect(|_| visit()).count());
    let (a3, w3) = examined(|| {
        let mut all: Vec<u64> = hm
            .keys()
            .map(|k| {
                visit();
                k.0
            })
            .collect();
        all.sort_unstable_by(|a, b| {
            visit();
            a.cmp(b)
        });
        all.len()
    });
    assert!(a1 == a2 && a2 == a3);
    rows.push(("iterate in order", format!("{a1} keys"), [w1, w2, w3]));

    println!(
        "  {:<28} {:>13} {:>12} {:>12} {:>12}",
        "question", "answer", "sorted Vec", "BTreeMap", "HashMap"
    );
    for (q, ans, w) in &rows {
        println!(
            "  {:<28} {:>13} {:>12} {:>12} {:>12}",
            q, ans, w[0], w[1], w[2]
        );
    }
    println!();
    println!("  all three structures give the same answer to every question (asserted),");
    println!("  and the HashMap and Vec costs are the Theta table's (at this one n).");
    println!("  The BTreeMap is the one to notice: it is of the Vec's order on every row");
    println!("  but the median, where it has to step over half the keys -- std's BTreeMap has no");
    println!("  \"k-th key\" operation, so the Vec's O(1) index is still its advantage.");

    println!();
    println!("4 and 5. MEMORY AND THE PER-VALUE ALLOCATION -- Rust has no GC to pay,");
    println!("so the question is what a pointer to every value costs instead:");
    println!();
    const N: usize = 300_000;
    let by_value = measure_map(
        N,
        |i| Config {
            name: "x",
            value: i,
        },
        |c| c.value + c.name.len() as i64,
    );
    let boxed = measure_map(
        N,
        |i| {
            Box::new(Config {
                name: "x",
                value: i,
            })
        },
        |c| c.value + c.name.len() as i64,
    );
    println!(
        "  {:<26} {:>9} {:>13} {:>10} {:>10} {:>10}",
        "", "KB", "allocations", "build ms", "walk ms", "drop ms"
    );
    println!(
        "  {:<26} {:>9.0} {:>13} {:>10.2} {:>10.2} {:>10.2}",
        "HashMap<i64, Config>",
        by_value.kb,
        by_value.allocs,
        by_value.build_ms,
        by_value.scan_ms,
        by_value.drop_ms
    );
    println!(
        "  {:<26} {:>9.0} {:>13} {:>10.2} {:>10.2} {:>10.2}",
        "HashMap<i64, Box<Config>>",
        boxed.kb,
        boxed.allocs,
        boxed.build_ms,
        boxed.scan_ms,
        boxed.drop_ms
    );
    println!();
    let probe: HashMap<i64, Config> = HashMap::with_capacity(N);
    println!(
        "  Config is {} bytes (a &str is 16, the i64 is 8), so a by-value slot is",
        size_of::<Config>()
    );
    println!(
        "  {} bytes and a boxed slot is {} (plus one control byte per slot). A table",
        size_of::<(i64, Config)>(),
        size_of::<(i64, Box<Config>)>()
    );
    println!(
        "  asked to hold {} entries has room for {} -- a map allocates slots",
        N,
        probe.capacity()
    );
    println!("  for its capacity, not its length, so every empty slot is paid for too.");
    println!();
    println!(
        "  in bytes REQUESTED from the allocator (the KB column) the boxed map is {:.0}% {}:",
        100.0 * (boxed.kb - by_value.kb).abs() / by_value.kb,
        if boxed.kb < by_value.kb {
            "smaller"
        } else {
            "larger"
        }
    );
    println!(
        "  {:.0} KB against {:.0}. But the allocator adds its own overhead to every",
        boxed.kb, by_value.kb
    );
    println!("  small allocation and the counting allocator cannot see it, so here is what a");
    println!("  fresh process that builds only that map really grows by (resident set, `ps`):");
    println!();
    match (resident_of("value"), resident_of("box")) {
        (Some(v), Some(b)) => {
            println!("  {:<26} {:>9} KB resident", "HashMap<i64, Config>", v);
            println!(
                "  {:<26} {:>9} KB resident   {:+.0}%",
                "HashMap<i64, Box<Config>>",
                b,
                100.0 * (b as f64 - v as f64) / v as f64
            );
            println!();
            println!(
                "  resident minus requested: {:.0} KB for the by-value map and {:.0} KB for the",
                v as f64 - by_value.kb,
                b as f64 - boxed.kb
            );
            println!(
                "  boxed one, which is {:.1} bytes per box that nothing in the program asked for.",
                (b as f64 - boxed.kb - (v as f64 - by_value.kb)) * 1024.0 / N as f64
            );
        }
        _ => println!("  (could not run `ps` or the child process here)"),
    }
    println!();
    println!("  the column that matters in Go is heap objects and GC time per cycle. Rust");
    println!(
        "  has {} allocations against {} and no collector, so what they cost here is",
        boxed.allocs, by_value.allocs
    );
    println!("  the time to make them (build), to chase them (walk) and to free them one");
    println!("  by one (drop).");
    println!();
    println!("6. CONCURRENCY -- a map is not safe, and the compiler says so:");
    println!();
    println!("  Go finds the bug when it happens, and kills the process:");
    println!("      fatal error: concurrent map writes");
    println!("  Rust refuses to build the program. Two scoped threads that both insert");
    println!("  into one `HashMap` (rustc 1.98.1):");
    println!("      error[E0499]: cannot borrow `m` as mutable more than once at a time");
    println!("  and the version with `Rc<RefCell<HashMap>>` moved into `thread::spawn`:");
    println!("      error[E0277]: `Rc<RefCell<HashMap<i32, i32>>>` cannot be sent between");
    println!("      threads safely");
    println!("  There is no run-time check and nothing to recover from: the data race");
    println!("  cannot be written in safe Rust. What is left is the price of the locks");
    println!("  that make sharing legal.");
    println!();
    const CONC: usize = 200_000;
    let keys: Vec<i64> = (0..CONC as i64).map(|i| i * 2654435761).collect();
    let plain: HashMap<i64, i64> = keys.iter().map(|&k| (k, 1)).collect();
    let rw = RwLock::new(plain.clone());
    let mx = Mutex::new(plain.clone());
    let sharded_rw = Sharded {
        shards: (0..16).map(|_| RwLock::new(HashMap::new())).collect(),
    };
    for &k in &keys {
        sharded_rw.shards[shard_of(k)].write().unwrap().insert(k, 1);
    }
    let t_plain = ns_per_op(|| {
        for k in black_box(&keys) {
            black_box(plain.get(k));
        }
    }) / CONC as f64;
    let t_rw = ns_per_op(|| {
        for k in black_box(&keys) {
            black_box(rw.read().unwrap().get(k));
        }
    }) / CONC as f64;
    let t_mx = ns_per_op(|| {
        for k in black_box(&keys) {
            black_box(mx.lock().unwrap().get(k));
        }
    }) / CONC as f64;
    let t_sh = ns_per_op(|| {
        for k in black_box(&keys) {
            black_box(sharded_rw.shards[shard_of(*k)].read().unwrap().get(k));
        }
    }) / CONC as f64;
    println!(
        "  {:<34} {:>8.2} ns/read",
        "plain HashMap (one thread only)", t_plain
    );
    println!(
        "  {:<34} {:>8.2} ns/read   {:.2}x",
        "RwLock<HashMap>, read lock",
        t_rw,
        t_rw / t_plain
    );
    println!(
        "  {:<34} {:>8.2} ns/read   {:.2}x",
        "Mutex<HashMap>",
        t_mx,
        t_mx / t_plain
    );
    println!(
        "  {:<34} {:>8.2} ns/read   {:.2}x",
        "16 x RwLock<HashMap>, sharded",
        t_sh,
        t_sh / t_plain
    );
    let sizes: Vec<usize> = sharded_rw
        .shards
        .iter()
        .map(|s| s.read().unwrap().len())
        .collect();
    println!();
    println!("  single thread, uncontended -- so this is the FLOOR, the price before any");
    println!(
        "  contention at all. (The 16 shards hold {}..{} keys each.) std has no",
        sizes.iter().min().unwrap(),
        sizes.iter().max().unwrap()
    );
    println!("  counterpart to Go's sync.Map, so there is no third row to price.");
    println!();

    // What the compiler allows instead of the crash: shared writes, correctly,
    // and a measurement of what they cost when threads really do collide.
    // 1000 hot keys, so every thread keeps hitting the same entries.
    const THREADS: usize = 4;
    const ADDS: usize = 400_000;
    const RUNS: usize = 5;
    let all = (THREADS * ADDS) as u64;
    let wall = |threads: usize, work: &(dyn Fn() + Sync)| {
        let t = Instant::now();
        thread::scope(|s| {
            for _ in 0..threads {
                s.spawn(work);
            }
        });
        t.elapsed().as_secs_f64() * 1000.0
    };
    // (median ms, min ms, max ms, every run's total) over RUNS fresh maps
    let run = |one_run: &dyn Fn() -> (f64, u64)| {
        let rs: Vec<(f64, u64)> = (0..RUNS).map(|_| one_run()).collect();
        let ms: Vec<f64> = rs.iter().map(|r| r.0).collect();
        let lo = ms.iter().copied().fold(f64::MAX, f64::min);
        let hi = ms.iter().copied().fold(0.0, f64::max);
        (median(ms), lo, hi, rs.iter().all(|r| r.1 == all))
    };
    let serial = run(&|| {
        let t = Instant::now();
        let mut m = HashMap::<u32, u64>::new();
        for i in 0..THREADS * ADDS {
            *m.entry((i % 1000) as u32).or_default() += 1;
        }
        let ms = t.elapsed().as_secs_f64() * 1000.0;
        (ms, m.values().sum())
    });
    let one_lock = run(&|| {
        let counts = Mutex::new(HashMap::<u32, u64>::new());
        let ms = wall(THREADS, &|| {
            for i in 0..ADDS {
                *counts.lock().unwrap().entry((i % 1000) as u32).or_default() += 1;
            }
        });
        (ms, counts.lock().unwrap().values().sum())
    });
    let sharded = run(&|| {
        let shards = Sharded {
            shards: (0..16)
                .map(|_| Mutex::new(HashMap::<u32, u64>::new()))
                .collect::<Vec<_>>(),
        };
        let ms = wall(THREADS, &|| {
            for i in 0..ADDS {
                let k = (i % 1000) as u32;
                *shards.shards[shard_of(k as i64)]
                    .lock()
                    .unwrap()
                    .entry(k)
                    .or_default() += 1;
            }
        });
        (
            ms,
            shards
                .shards
                .iter()
                .map(|s| s.lock().unwrap().values().sum::<u64>())
                .sum(),
        )
    });
    println!("  {all} increments over 1000 hot keys, median of {RUNS} runs (min - max), ms:");
    println!();
    for (name, (med, lo, hi, _)) in [
        ("1 thread, plain HashMap, no lock", serial),
        ("4 threads, one Mutex<HashMap>", one_lock),
        ("4 threads, 16 x Mutex<HashMap> shards", sharded),
    ] {
        println!("  {name:<40} {med:>8.1}   ({lo:.1} - {hi:.1})");
    }
    println!();
    println!(
        "  every run of every variant counted exactly {all}: {}. No lost update, no",
        serial.3 && one_lock.3 && sharded.3
    );
    println!("  fatal error -- the lock makes the sharing legal; it does not make it fast.");
    println!(
        "  one lock took {:.1}x the serial time and 16 shards took {:.1}x; sharding {},",
        one_lock.0 / serial.0,
        sharded.0 / serial.0,
        if sharded.0 < one_lock.0 {
            "helped"
        } else {
            "did not help on this run"
        }
    );
    println!(
        "  and four threads on hot keys {} one thread with no lock.",
        if sharded.0 > serial.0 && one_lock.0 > serial.0 {
            "still lose to"
        } else {
            "beat"
        }
    );
    println!();
    println!("  the usual right answer is a plain map behind a mutex, or -- much");
    println!("  better -- SHARDING: N maps each with its own lock, selected by");
    println!("  hash. Lesson 38 builds that.");

    println!();
    println!("7. REPRODUCIBLE OUTPUT -- the randomisation is not optional:");
    println!();
    let build = || -> HashMap<&'static str, i32> {
        [("a", 1), ("b", 2), ("c", 3), ("d", 4)]
            .into_iter()
            .collect()
    };
    let m = build();
    for _ in 0..3 {
        let order: Vec<&str> = m.keys().copied().collect();
        println!("  one map, iterated again:   {}", show(&order));
    }
    for _ in 0..4 {
        let order: Vec<&str> = build().keys().copied().collect();
        println!("  a new map, same inserts:   {}", show(&order));
    }
    let mut sorted: Vec<&str> = m.keys().copied().collect();
    sorted.sort_unstable();
    println!(
        "  sorted:                    {}   <- what you actually wanted",
        show(&sorted)
    );
    println!();
    println!("  Go randomises where every `range` over a map starts; Rust fixes the order");
    println!("  when the map is built: iterating one map twice gives the same order, because");
    println!("  the map's random key does not change. A new map gets a new key, so the order");
    println!("  changes from map to map and from run to run.");
    let build_fixed = || -> HashMap<&'static str, i32, FastState> {
        let mut m = HashMap::with_hasher(FastState::default());
        for (k, v) in [("a", 1), ("b", 2), ("c", 3), ("d", 4)] {
            m.insert(k, v);
        }
        m
    };
    println!();
    for _ in 0..3 {
        let order: Vec<&str> = build_fixed().keys().copied().collect();
        println!("  unseeded `Fast` hasher:    {}", show(&order));
    }
    println!();
    println!("  with an unseeded hasher the order is the same for every map and every");
    println!("  run. That is reproducible only as an accident of the hash, and it is the");
    println!("  same property that made the table attackable in example 14.");
    println!();
    println!("  anything that produces a file, a hash, a signature, a golden test");
    println!("  or a diff must sort. This is the single most common source of");
    println!("  flaky tests, and it is a feature -- the alternative is code");
    println!("  that depends on an order the standard library never promised.");

    println!();
    println!("the summary, which is the useful part of this example:");
    println!();
    for (when, then) in [
        (
            "n small, keys cheap to compare",
            "use a slice (see 1 for where)",
        ),
        ("integer keys in a dense range", "use a Vec or an array"),
        ("you need order or ranges", "use a sorted Vec or a BTreeMap"),
        (
            "values are small structs",
            "store by VALUE: no allocation each",
        ),
        ("threads share it", "mutex, or shard it"),
        ("output must be stable", "sort the keys, every time"),
        ("anything else", "use a HashMap -- it is very good"),
    ] {
        println!("  {when:<34} {then}");
    }
}
```

**Rust sample output:**

```
1. SMALL n -- where does the map start to win?

  std's default hasher (SipHash-1-3, random key per map):

       n    str slice    str map      ratio  int slice    int map      ratio
       2         6.07       8.88      0.68x       1.12       9.50      0.12x
       4        12.18       8.88      1.37x       1.54       5.65      0.27x
       8        24.41       8.92      2.74x       2.54       5.81      0.44x
      16        13.73       7.65      1.79x       5.11       5.61      0.91x
      32        42.22       7.79      5.42x       9.36       5.70      1.64x
      64        90.78       7.64     11.88x      17.39       5.65      3.08x

  the same lookups with the `Fast` hasher (finalized FNV-1a, no seed):

       n    str slice    str map      ratio  int slice    int map      ratio
       2         5.85       5.81      1.01x       1.01       1.40      0.72x
       4        12.53       5.81      2.16x       1.51       1.38      1.09x
       8        22.79       6.13      3.71x       2.59       1.38      1.88x
      16        14.02       5.40      2.59x       5.16       1.38      3.75x
      32        42.18       5.36      7.87x       9.32       1.39      6.69x
      64        90.35       5.36     16.86x      17.45       1.42     12.31x

  first n at which the map beats the scan, string keys / int keys:
    SipHash-1-3  n=4 / n=32      Fast  n=2 / n=4

  so the crossover is not one number. It depends on the key (what `==` and
  the hash cost for it) and on the hasher: SipHash is the dearer lookup, so
  the scan holds out longer against it. (Run it again: the Fast crossovers
  move by one step between runs, because the ratios at n=2..4 are near 1.0.)

  the pieces, one operation each (ns, including about the same loop and
  black_box overhead for every row, which the first row shows alone):

  nothing (black_box of a constant)      0.25
  hash an 8-byte string                  4.84       2.57   (SipHash, Fast)
  hash an i64                            5.03       0.44
  == on two equal 8-byte Strings         1.77
  == on two i64                          0.51

  a string hash costs 1.0x an int hash under SipHash and 5.8x under Fast
  (Fast hashes an i64 with one multiply and a string byte by byte, as written).

  string scan, ns per key examined, keys field_N (7-8 bytes) against field_NN (8 bytes):
    n=8   field_N 2.89    field_NN 1.72
    n=16  field_N 0.82    field_NN 1.60
    n=32  field_N 1.31    field_NN 1.71

  fixed-width keys cost about the same per key at every n; mixed-width keys
  do not, so the dip in the string-scan columns at n=16 is about the KEYS
  (field_0..field_9 are 7 bytes, field_10.. are 8), not about scanning.
  Plausible mechanism, not measured separately: strings of unequal length
  differ at the length check, before any byte is compared.

  (this is the slice's worst case: a full scan to the last element.
  If lookups are usually hits near the front, it does better.)

2. DENSE INTEGER KEYS -- an array is already a perfect hash table:

  HashMap<i64, i64>, keys 0..999999        11159319 ns build,    34816 KB
    the same with Fast                      5711862 ns build,    34816 KB
  Vec<i64> of the same                       352326 ns build,     7812 KB
                                                32x faster,     4.5x smaller   (SipHash map)
                                                16x faster,     4.5x smaller   (Fast map)

  a slice index IS a hash function: perfect, collision-free, one
  instruction. Whenever the keys are integers in a known, mostly-
  dense range, an array is not an optimisation of a map -- it is
  the correct data structure and the map is the workaround.

  the same applies to byte keys ([T; 256]), small enums, and
  anything you can number. Lesson 14's counting sort is this idea.

3. ORDERED OUTPUT -- the map cannot answer the question at all:

  question                           sorted slice     map
  is x present?                      Theta(log n)     Theta(1)
  what is the smallest key?          Theta(1)         Theta(n)
  the smallest key above x?          Theta(log n)     Theta(n)
  all keys between a and b?          Theta(log n + k) Theta(n)
  the median key?                    Theta(1)         Theta(n log n)
  iterate in order                   Theta(n)         Theta(n log n)

  a map answers exactly one of those well. If your workload has
  more than one row in it, you want a tree (lessons 17-19) or a
  sorted slice -- and lesson 11's table measured the cost.

  Rust's std has the tree: BTreeMap. Here are those six questions asked of a
  sorted Vec, a BTreeMap and a HashMap holding the same 100,000 keys (the
  even numbers 0..199998, inserted in shuffled order), x = 77778 (a key) and
  the range 50000..=50998. The cost is counted, not timed: key comparisons
  plus entries stepped over by a scan -- the same unit for every cell.

  question                            answer   sorted Vec     BTreeMap      HashMap
  is x present?                         true           18           31            1
  what is the smallest key?                0            0            0       100000
  the smallest key above x?            77780           18           31       100000
  all keys between a and b?         500 keys          536          533       100000
  the median key?                     100000            0        50001      1820509
  iterate in order               100000 keys       100000       100000      1820509

  all three structures give the same answer to every question (asserted),
  and the HashMap and Vec costs are the Theta table's (at this one n).
  The BTreeMap is the one to notice: it is of the Vec's order on every row
  but the median, where it has to step over half the keys -- std's BTreeMap has no
  "k-th key" operation, so the Vec's O(1) index is still its advantage.

4 and 5. MEMORY AND THE PER-VALUE ALLOCATION -- Rust has no GC to pay,
so the question is what a pointer to every value costs instead:

                                    KB   allocations   build ms    walk ms    drop ms
  HashMap<i64, Config>           16896             1       4.24       0.56       0.08
  HashMap<i64, Box<Config>>      15735        300001      10.55       1.06      14.36

  Config is 24 bytes (a &str is 16, the i64 is 8), so a by-value slot is
  32 bytes and a boxed slot is 16 (plus one control byte per slot). A table
  asked to hold 300000 entries has room for 458752 -- a map allocates slots
  for its capacity, not its length, so every empty slot is paid for too.

  in bytes REQUESTED from the allocator (the KB column) the boxed map is 7% smaller:
  15735 KB against 16896. But the allocator adds its own overhead to every
  small allocation and the counting allocator cannot see it, so here is what a
  fresh process that builds only that map really grows by (resident set, `ps`):

  HashMap<i64, Config>           17072 KB resident
  HashMap<i64, Box<Config>>      18528 KB resident   +9%

  resident minus requested: 176 KB for the by-value map and 2793 KB for the
  boxed one, which is 8.9 bytes per box that nothing in the program asked for.

  the column that matters in Go is heap objects and GC time per cycle. Rust
  has 300001 allocations against 1 and no collector, so what they cost here is
  the time to make them (build), to chase them (walk) and to free them one
  by one (drop).

6. CONCURRENCY -- a map is not safe, and the compiler says so:

  Go finds the bug when it happens, and kills the process:
      fatal error: concurrent map writes
  Rust refuses to build the program. Two scoped threads that both insert
  into one `HashMap` (rustc 1.98.1):
      error[E0499]: cannot borrow `m` as mutable more than once at a time
  and the version with `Rc<RefCell<HashMap>>` moved into `thread::spawn`:
      error[E0277]: `Rc<RefCell<HashMap<i32, i32>>>` cannot be sent between
      threads safely
  There is no run-time check and nothing to recover from: the data race
  cannot be written in safe Rust. What is left is the price of the locks
  that make sharing legal.

  plain HashMap (one thread only)        9.25 ns/read
  RwLock<HashMap>, read lock            10.03 ns/read   1.08x
  Mutex<HashMap>                        13.31 ns/read   1.44x
  16 x RwLock<HashMap>, sharded         11.51 ns/read   1.24x

  single thread, uncontended -- so this is the FLOOR, the price before any
  contention at all. (The 16 shards hold 12497..12505 keys each.) std has no
  counterpart to Go's sync.Map, so there is no third row to price.

  1600000 increments over 1000 hot keys, median of 5 runs (min - max), ms:

  1 thread, plain HashMap, no lock             13.0   (10.2 - 13.2)
  4 threads, one Mutex<HashMap>                70.9   (68.9 - 73.4)
  4 threads, 16 x Mutex<HashMap> shards        39.7   (39.3 - 40.6)

  every run of every variant counted exactly 1600000: true. No lost update, no
  fatal error -- the lock makes the sharing legal; it does not make it fast.
  one lock took 5.5x the serial time and 16 shards took 3.1x; sharding helped,
  and four threads on hot keys still lose to one thread with no lock.

  the usual right answer is a plain map behind a mutex, or -- much
  better -- SHARDING: N maps each with its own lock, selected by
  hash. Lesson 38 builds that.

7. REPRODUCIBLE OUTPUT -- the randomisation is not optional:

  one map, iterated again:   [b a d c]
  one map, iterated again:   [b a d c]
  one map, iterated again:   [b a d c]
  a new map, same inserts:   [c b d a]
  a new map, same inserts:   [c a d b]
  a new map, same inserts:   [b a c d]
  a new map, same inserts:   [a b d c]
  sorted:                    [a b c d]   <- what you actually wanted

  Go randomises where every `range` over a map starts; Rust fixes the order
  when the map is built: iterating one map twice gives the same order, because
  the map's random key does not change. A new map gets a new key, so the order
  changes from map to map and from run to run.

  unseeded `Fast` hasher:    [b a c d]
  unseeded `Fast` hasher:    [b a c d]
  unseeded `Fast` hasher:    [b a c d]

  with an unseeded hasher the order is the same for every map and every
  run. That is reproducible only as an accident of the hash, and it is the
  same property that made the table attackable in example 14.

  anything that produces a file, a hash, a signature, a golden test
  or a diff must sort. This is the single most common source of
  flaky tests, and it is a feature -- the alternative is code
  that depends on an order the standard library never promised.

the summary, which is the useful part of this example:

  n small, keys cheap to compare     use a slice (see 1 for where)
  integer keys in a dense range      use a Vec or an array
  you need order or ranges           use a sorted Vec or a BTreeMap
  values are small structs           store by VALUE: no allocation each
  threads share it                   mutex, or shard it
  output must be stable              sort the keys, every time
  anything else                      use a HashMap -- it is very good
```

> **Go vs Rust:** Rust contradicts Go's headline in 4 and 5: the `Box<Config>` map is smaller only in bytes REQUESTED (15,735 KB against 16,896 KB, 7% less, which is Go's finding); in the resident memory of a fresh process it is LARGER (18,336-18,416 KB against 17,008-17,104 KB, +7-8% over six runs), because malloc adds about 8-9 bytes to every 24-byte box, which the counting allocator cannot see. Go's GC column has no Rust counterpart; what boxing costs here is 300,001 allocations against 1, a build 2.5-3.9x slower, a walk 1.7-2.3x slower and a drop of 14-17 ms against 0.06-0.14 ms (roughly 100-250x). Go's small-n crossovers do not carry over exactly: with SipHash strings cross at n=4 (Go: between 2 and 4) but ints at n=32 (Go: between 8 and 16), and with the unseeded `Fast` hasher both cross at n=2-8, moving by a step between runs; hashing a short string costs 1.0-1.1x an int under SipHash (Go's claim holds) but 5.4-6.5x under `Fast`, and the dip at n=16 in the string column (Go's table has it too) is the key lengths (`field_9` is 7 bytes, `field_10` is 8), not the scan. A dense `Vec<i64>` builds 24-40x faster than the map with SipHash and 12-18x with `Fast` (Go: 50-100x), in 4.5x less memory (Go: 4.7x). Go's "fatal error: concurrent map writes" is a compile error here (E0499 and E0277, quoted in the output, so there is nothing for `go run -race` to find), `sync.Map` has no std equivalent, and uncontended reads cost 1.15-1.24x (`RwLock`) and 1.48-1.73x (`Mutex`) of a plain map, against Go's 1.06-1.20x for `RWMutex`; four threads on 1,000 hot keys took 66-70 ms behind one `Mutex` and 38-41 ms across 16 shards, against 8-14 ms for one thread with no lock (in one run on a loaded machine the shards lost, 52 ms to 40 ms). The ordered-output table is Go's, extended with counted costs for a sorted `Vec`, a `BTreeMap` and a `HashMap`; `BTreeMap` is of the `Vec`'s order on every row but the median, which std offers no way to index. Iterating one `HashMap` twice gives the same order (Go reshuffles per `range`), and an unseeded hasher gives the same order in every map and run.

**Complexity:** the slice/map crossover is **between 2 and 4 for string keys** and **between 8 and 16
for ints** — it depends on how expensive `==` is · `map[int]*Config` is **smaller in bytes** and costs
**301,357 heap objects against 1,355** and ~4× the GC time · a dense `[]int` beats `map[int]int` by
**100×** and 4.7× the memory

---

## 16. Capstone: `HashMap[K,V]`, built, proved, measured

`🔴 hard` · *the three-pass method, one more time*

Every design decision in this type came from a measurement in this lesson. The proof has three layers,
and the measurement ends with an honest verdict.

**Steps:**

1. **Build it** — open addressing, power-of-two, seeded, tombstones counted against the load factor.
2. **Prove it** — 300,000 random ops against Go's own map, invariants after every one, plus the
   adversarial key set from example 14.
3. **Measure it** — against the map it is imitating, and decide whether it was worth writing.

```go
package main

import (
	"fmt"
	"hash/maphash"
	"math/rand"
	"runtime"
	"testing"
)

// The capstone, and the three-pass method one more time:
//
//	BUILD IT    HashMap[K,V] -- open addressing, power-of-two capacity,
//	            seeded hashing, tombstones, and a load factor that counts
//	            them
//	PROVE IT    a model oracle against Go's own map over 300,000 random
//	            operations, invariants after every one, and an adversarial
//	            key set that would kill a fixed-hash table
//	MEASURE IT  against the map it is imitating -- and then decide honestly
//	            whether it was worth writing
//
// Everything here is a finding from examples 1-15, assembled.

// ---- BUILD IT ---------------------------------------------------------------

type slotState uint8

const (
	free slotState = iota // never used -- a probe may stop here
	used
	tomb // was used; a probe must NOT stop here (example 4)
)

// maxLoad counts used + tombstones. Counting only `used` is the bug that makes
// a churned table stop terminating (examples 4 and 5).
const maxLoad = 0.75

type HashMap[K comparable, V any] struct {
	keys   []K
	vals   []V
	states []slotState
	used   int
	tombs  int

	seed maphash.Seed // per-map, so an attacker cannot precompute (example 14)

	// instrumentation, so the tests can assert on behaviour rather than guess
	rehashes int
	probes   int
}

func New[K comparable, V any](hint int) *HashMap[K, V] {
	size := 8
	for float64(hint) > maxLoad*float64(size) {
		size <<= 1
	}
	return &HashMap[K, V]{
		keys:   make([]K, size),
		vals:   make([]V, size),
		states: make([]slotState, size),
		seed:   maphash.MakeSeed(),
	}
}

func (m *HashMap[K, V]) mask() uint64 { return uint64(len(m.keys) - 1) }

// hash uses maphash, which takes a SEED. hash/fnv does not, and example 14 is
// the reason that matters.
func (m *HashMap[K, V]) hash(k K) uint64 { return maphash.Comparable(m.seed, k) }

func (m *HashMap[K, V]) Len() int { return m.used }
func (m *HashMap[K, V]) Cap() int { return len(m.keys) }

func (m *HashMap[K, V]) Get(k K) (V, bool) {
	var zero V
	i := m.hash(k) & m.mask()
	for {
		m.probes++
		switch m.states[i] {
		case free:
			return zero, false
		case used:
			if m.keys[i] == k {
				return m.vals[i], true
			}
		}
		i = (i + 1) & m.mask()
	}
}

func (m *HashMap[K, V]) Put(k K, v V) {
	if float64(m.used+m.tombs+1) > maxLoad*float64(len(m.keys)) {
		m.rehash()
	}
	m.put(k, v)
}

func (m *HashMap[K, V]) put(k K, v V) {
	i := m.hash(k) & m.mask()
	firstTomb := -1
	for {
		switch m.states[i] {
		case free:
			if firstTomb >= 0 {
				i, m.tombs = uint64(firstTomb), m.tombs-1
			}
			m.keys[i], m.vals[i], m.states[i] = k, v, used
			m.used++
			return
		case tomb:
			if firstTomb < 0 {
				firstTomb = int(i)
			}
		case used:
			if m.keys[i] == k {
				m.vals[i] = v
				return
			}
		}
		i = (i + 1) & m.mask()
	}
}

func (m *HashMap[K, V]) Delete(k K) bool {
	i := m.hash(k) & m.mask()
	for {
		switch m.states[i] {
		case free:
			return false
		case used:
			if m.keys[i] == k {
				var zk K
				var zv V
				m.states[i] = tomb
				m.keys[i], m.vals[i] = zk, zv // release pointers (example 5, l09)
				m.used--
				m.tombs++
				return true
			}
		}
		i = (i + 1) & m.mask()
	}
}

// rehash grows only when the LIVE count justifies it. A table that is mostly
// tombstones rebuilds at the same size (example 5).
func (m *HashMap[K, V]) rehash() {
	size := len(m.keys)
	if float64(m.used+1) > maxLoad*float64(size)/2 {
		size *= 2
	}
	ok, ov, os := m.keys, m.vals, m.states
	m.keys = make([]K, size)
	m.vals = make([]V, size)
	m.states = make([]slotState, size)
	m.used, m.tombs = 0, 0
	for i, s := range os {
		if s == used {
			m.put(ok[i], ov[i])
		}
	}
	m.rehashes++
}

func (m *HashMap[K, V]) All(yield func(K, V) bool) {
	for i, s := range m.states {
		if s == used && !yield(m.keys[i], m.vals[i]) {
			return
		}
	}
}

// ---- PROVE IT ---------------------------------------------------------------

// invariants must hold after EVERY operation.
func (m *HashMap[K, V]) invariants() error {
	if n := len(m.keys); n == 0 || n&(n-1) != 0 {
		return fmt.Errorf("capacity %d is not a power of two", n)
	}
	if len(m.vals) != len(m.keys) || len(m.states) != len(m.keys) {
		return fmt.Errorf("parallel arrays out of step")
	}
	live, tombs := 0, 0
	for _, s := range m.states {
		switch s {
		case used:
			live++
		case tomb:
			tombs++
		}
	}
	if live != m.used {
		return fmt.Errorf("used=%d but %d slots are occupied", m.used, live)
	}
	if tombs != m.tombs {
		return fmt.Errorf("tombs=%d but %d slots are tombstones", m.tombs, tombs)
	}
	if float64(m.used+m.tombs) > maxLoad*float64(len(m.keys)) {
		return fmt.Errorf("load %.3f exceeds %.3f",
			float64(m.used+m.tombs)/float64(len(m.keys)), maxLoad)
	}
	return nil
}

var sinkI int
var sinkB bool

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func liveKB(build func() any) float64 {
	runtime.GC()
	runtime.GC()
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	keep := build()
	runtime.GC()
	runtime.GC()
	runtime.ReadMemStats(&b)
	runtime.KeepAlive(keep)
	return float64(b.HeapAlloc-a.HeapAlloc) / 1024
}

func weakHash(s string) uint64 {
	h := uint64(5381)
	for i := 0; i < len(s); i++ {
		h = h*33 + uint64(s[i])
	}
	return h
}

func collidingKeys(n, buckets int) []string {
	target := weakHash("aaaa") % uint64(buckets)
	out := make([]string, 0, n)
	buf := make([]byte, 8)
	for i := 0; len(out) < n; i++ {
		v := i
		for j := range buf {
			buf[j] = byte('a' + v%26)
			v /= 26
		}
		if s := string(buf); weakHash(s)%uint64(buckets) == target {
			out = append(out, s)
		}
	}
	return out
}

func main() {
	fmt.Println("PASS 1 -- BUILD IT")
	fmt.Println()
	m := New[string, int](4)
	for i, k := range []string{"go", "rust", "zig", "c"} {
		m.Put(k, i)
	}
	m.Delete("rust")
	m.Put("odin", 9)
	got := map[string]int{}
	for k, v := range m.All {
		got[k] = v
	}
	fmt.Printf("  Len=%d Cap=%d rehashes=%d tombstones=%d\n", m.Len(), m.Cap(), m.rehashes, m.tombs)
	fmt.Printf("  contents: %v\n", got)
	v, ok := m.Get("zig")
	_, gone := m.Get("rust")
	fmt.Printf("  Get(\"zig\")=%d,%v   Get(\"rust\") present=%v\n", v, ok, gone)

	fmt.Println()
	fmt.Println("PASS 2 -- PROVE IT")
	fmt.Println()

	// 2a. model oracle: every operation mirrored against Go's own map
	rng := rand.New(rand.NewSource(21))
	hm := New[int, int](4)
	model := map[int]int{}
	const ops = 300_000
	mismatch, invFail := 0, 0
	for i := 0; i < ops; i++ {
		k := rng.Intn(20_000) // narrow range: forces real collisions and updates
		switch rng.Intn(10) {
		case 0, 1, 2, 3, 4, 5: // Put
			v := rng.Int()
			hm.Put(k, v)
			model[k] = v
		case 6, 7: // Delete
			a := hm.Delete(k)
			_, b := model[k]
			delete(model, k)
			if a != b {
				mismatch++
			}
		default: // Get
			a, aok := hm.Get(k)
			b, bok := model[k]
			if aok != bok || (aok && a != b) {
				mismatch++
			}
		}
		if hm.Len() != len(model) {
			mismatch++
		}
		if err := hm.invariants(); err != nil {
			invFail++
		}
	}
	// full agreement at the end, both directions
	for k, want := range model {
		if v, ok := hm.Get(k); !ok || v != want {
			mismatch++
		}
	}
	for k, v := range hm.All {
		if want, ok := model[k]; !ok || want != v {
			mismatch++
		}
	}
	fmt.Printf("  %-52s %d\n",
		fmt.Sprintf("%d random ops vs Go's map -- mismatches:", ops), mismatch)
	fmt.Printf("  %-52s %d\n", "invariant failures (checked after EVERY op):", invFail)
	fmt.Printf("  %-52s %d entries, %d rehashes\n", "final state:", hm.Len(), hm.rehashes)

	// 2b. the churn case: tombstones must not accumulate without bound
	churn := New[int, int](1024)
	for i := 0; i < 500; i++ {
		churn.Put(i, i)
	}
	worstLoad := 0.0
	for round := 0; round < 2000; round++ {
		churn.Put(1_000_000+round, round)
		churn.Delete(1_000_000 + round)
		l := float64(churn.used+churn.tombs) / float64(len(churn.keys))
		if l > worstLoad {
			worstLoad = l
		}
	}
	fmt.Printf("  %-52s %.3f (limit %.2f)\n",
		"worst load factor over 2000 put/delete cycles:", worstLoad, maxLoad)

	// 2c. the adversarial set from example 14
	attack := collidingKeys(4000, 4096)
	adv := New[string, int](4000)
	for i, k := range attack {
		adv.Put(k, i)
	}
	adv.probes = 0
	for _, k := range attack {
		adv.Get(k)
	}
	fmt.Printf("  %-52s %.2f\n",
		"probes/lookup on 4000 hashDoS keys:", float64(adv.probes)/float64(len(attack)))

	fmt.Println()
	fmt.Println("  three layers, and each catches what the others cannot: the model")
	fmt.Println("  finds wrong ANSWERS, the invariants find corrupt STATE the model")
	fmt.Println("  has not surfaced yet, and the adversarial set reaches a case that")
	fmt.Println("  random testing never will.")

	fmt.Println()
	fmt.Println("PASS 3 -- MEASURE IT")
	fmt.Println()
	const n = 200_000
	ikeys := make([]int, n)
	absent := make([]int, n)
	for i := range ikeys {
		ikeys[i] = i*2654435761 + 1
		absent[i] = -(i*2654435761 + 1)
	}

	mine := New[int, int](n)
	theirs := make(map[int]int, n)
	for i, k := range ikeys {
		mine.Put(k, i)
		theirs[k] = i
	}

	fmt.Printf("  %-22s %14s %14s %10s\n", "operation", "HashMap ns", "Go map ns", "ratio")
	ratios := map[string]float64{}
	row := func(name string, a, b func()) {
		ta := nsPerOp(a) / n
		tb := nsPerOp(b) / n
		ratios[name] = ta / tb
		fmt.Printf("  %-22s %14.2f %14.2f %9.2fx\n", name, ta, tb, ta/tb)
	}
	row("build", func() {
		mm := New[int, int](n)
		for i, k := range ikeys {
			mm.Put(k, i)
		}
		sinkI = mm.Len()
	}, func() {
		mm := make(map[int]int, n)
		for i, k := range ikeys {
			mm[k] = i
		}
		sinkI = len(mm)
	})
	row("lookup, hit", func() {
		for _, k := range ikeys {
			_, sinkB = mine.Get(k)
		}
	}, func() {
		for _, k := range ikeys {
			_, sinkB = theirs[k]
		}
	})
	row("lookup, miss", func() {
		for _, k := range absent {
			_, sinkB = mine.Get(k)
		}
	}, func() {
		for _, k := range absent {
			_, sinkB = theirs[k]
		}
	})
	row("iterate", func() {
		for _, v := range mine.All {
			sinkI = v
		}
	}, func() {
		for _, v := range theirs {
			sinkI = v
		}
	})

	mineKB := liveKB(func() any {
		mm := New[int, int](n)
		for i, k := range ikeys {
			mm.Put(k, i)
		}
		return mm
	})
	theirsKB := liveKB(func() any {
		mm := make(map[int]int, n)
		for i, k := range ikeys {
			mm[k] = i
		}
		return mm
	})
	fmt.Printf("  %-22s %13.0fK %13.0fK %9.2fx\n", "memory", mineKB, theirsKB, mineKB/theirsKB)

	fmt.Println()
	fmt.Printf("  it holds its own on BUILD (%.2fx) and is FASTER on MISS (%.2fx),\n",
		ratios["build"], ratios["lookup, miss"])
	fmt.Println("  which I did not expect. At 0.75 load my table has plenty of free")
	fmt.Println("  slots, so a miss stops at the first one -- example 6's finding,")
	fmt.Println("  seen from the other side.")
	fmt.Println()
	fmt.Println("  it loses the ones that matter:")
	fmt.Println()
	fmt.Printf("    HIT     %.2fx   Go resolves eight slots in one instruction\n", ratios["lookup, hit"])
	fmt.Println("                    against the control word (example 6); mine")
	fmt.Println("                    probes one slot at a time")
	fmt.Printf("    ITERATE %.2fx   I scan every slot, including the empty ones\n", ratios["iterate"])
	fmt.Printf("    MEMORY  %.2fx   separate full-width key and value arrays,\n", mineKB/theirsKB)
	fmt.Println("                    at a lower load factor")
	fmt.Println()
	fmt.Println("  and none of that counts the things I did not implement: string")
	fmt.Println("  and int fast paths, AES-accelerated hashing, incremental growth,")
	fmt.Println("  or the extendible-hashing directory that keeps a huge map's")
	fmt.Println("  rehashes bounded.")
	fmt.Println()
	fmt.Println("  so: do not ship this. The point of building it was never to beat")
	fmt.Println("  the runtime -- it was to be able to say WHY the built-in map is")
	fmt.Println("  shaped the way it is, and every design decision above came from a")
	fmt.Println("  measurement in this lesson.")

	fmt.Println()
	fmt.Println("THE CHECKLIST, which is the deliverable:")
	fmt.Println()
	for _, line := range []string{
		"capacity is a power of two             (& instead of %)",
		"the load factor counts TOMBSTONES      (or a churned table never terminates)",
		"Delete writes a tombstone, not free    (or it deletes its neighbours too)",
		"Delete zeroes the slot                 (or K and V's pointers never die)",
		"rehash may keep the SAME size          (or a tombstoned table doubles forever)",
		"the hash is SEEDED                     (or example 14 is your outage)",
		"tests use a NARROW key range           (or collisions never happen)",
		"tests check invariants every operation (the model alone is not enough)",
	} {
		fmt.Printf("  [x] %s\n", line)
	}
	fmt.Println()
	fmt.Println("  eight lines. Six of them exist because a measurement in this")
	fmt.Println("  lesson contradicted the sentence I had already written, and the")
	fmt.Println("  other two because Go's runtime source says so.")

	fmt.Println()
	fmt.Println("and the one-line summary of lesson 10:")
	fmt.Println()
	fmt.Println("      write map[K]V, and know exactly what you are getting.")
}
```

**Sample output:**

```
PASS 1 -- BUILD IT

  Len=4 Cap=8 rehashes=0 tombstones=1
  contents: map[c:3 go:0 odin:9 zig:2]
  Get("zig")=2,true   Get("rust") present=false

PASS 2 -- PROVE IT

  300000 random ops vs Go's map -- mismatches:         0
  invariant failures (checked after EVERY op):         0
  final state:                                         15037 entries, 12 rehashes
  worst load factor over 2000 put/delete cycles:       0.750 (limit 0.75)
  probes/lookup on 4000 hashDoS keys:                  1.48

  three layers, and each catches what the others cannot: the model
  finds wrong ANSWERS, the invariants find corrupt STATE the model
  has not surfaced yet, and the adversarial set reaches a case that
  random testing never will.

PASS 3 -- MEASURE IT

  operation                  HashMap ns      Go map ns      ratio
  build                           11.66          12.01      0.97x
  lookup, hit                      8.80           5.79      1.52x
  lookup, miss                    11.94          14.52      0.82x
  iterate                          6.11           4.20      1.45x
  memory                          8704K          3050K      2.85x

  it holds its own on BUILD (0.97x) and is FASTER on MISS (0.82x),
  which I did not expect. At 0.75 load my table has plenty of free
  slots, so a miss stops at the first one -- example 6's finding,
  seen from the other side.

  it loses the ones that matter:

    HIT     1.52x   Go resolves eight slots in one instruction
                    against the control word (example 6); mine
                    probes one slot at a time
    ITERATE 1.45x   I scan every slot, including the empty ones
    MEMORY  2.85x   separate full-width key and value arrays,
                    at a lower load factor

  and none of that counts the things I did not implement: string
  and int fast paths, AES-accelerated hashing, incremental growth,
  or the extendible-hashing directory that keeps a huge map's
  rehashes bounded.

  so: do not ship this. The point of building it was never to beat
  the runtime -- it was to be able to say WHY the built-in map is
  shaped the way it is, and every design decision above came from a
  measurement in this lesson.

THE CHECKLIST, which is the deliverable:

  [x] capacity is a power of two             (& instead of %)
  [x] the load factor counts TOMBSTONES      (or a churned table never terminates)
  [x] Delete writes a tombstone, not free    (or it deletes its neighbours too)
  [x] Delete zeroes the slot                 (or K and V's pointers never die)
  [x] rehash may keep the SAME size          (or a tombstoned table doubles forever)
  [x] the hash is SEEDED                     (or example 14 is your outage)
  [x] tests use a NARROW key range           (or collisions never happen)
  [x] tests check invariants every operation (the model alone is not enough)

  eight lines. Six of them exist because a measurement in this
  lesson contradicted the sentence I had already written, and the
  other two because Go's runtime source says so.

and the one-line summary of lesson 10:

      write map[K]V, and know exactly what you are getting.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::collections::hash_map::RandomState;
use std::collections::{BTreeMap, HashMap as StdMap};
use std::hash::{BuildHasher, BuildHasherDefault, Hash, Hasher};
use std::hint::black_box;
use std::mem::{replace, size_of, take};
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// The capstone, and the three-pass method one more time:
//
//	BUILD IT    HashMap<K, V, S> -- open addressing, power-of-two capacity,
//	            seeded hashing, tombstones, and a load factor that counts
//	            them
//	PROVE IT    a model oracle against std's own HashMap over 300,000 random
//	            operations, invariants after every one, and an adversarial
//	            key set that would kill a fixed-hash table
//	MEASURE IT  against the map it is imitating -- and then decide honestly
//	            whether it was worth writing
//
// Everything here is a finding from examples 1-15, assembled.
//
// Names: this file's `HashMap` is the one being built; std's is imported as
// `StdMap` and is "the map it is imitating" (a hashbrown Swiss table with
// SipHash-1-3 by default).

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Counting allocator: stand-in for runtime.MemStats. Rust has no GC, so "how
// many bytes does this structure hold" is an exact, repeatable number -- there
// is no `runtime.GC()` twice before every reading.
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

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

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

/// KB of heap that `build`'s result keeps alive (liveKB in the Go program).
fn live_kb<T>(build: impl FnOnce() -> T) -> f64 {
    let before = LIVE.load(Relaxed) as i64;
    let kept = build();
    let kb = (LIVE.load(Relaxed) as i64 - before) as f64 / 1024.0;
    black_box(kept);
    kb
}

/// stand-in for math/rand (splitmix64, seeded): the model-oracle run is
/// reproducible, but its digits are not the ones Go's generator would give.
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// ---- two more hashers, so the comparisons say exactly what they compare ----
//
// The capstone's own hash is std's `RandomState`: SipHash-1-3 under a random
// key chosen when the map is created. That is the Rust spelling of Go's
// `maphash.MakeSeed()` + `maphash.Comparable`, and like Go's it is what makes
// the map safe against example 14. Two hashers with NO seed:
//
//   `Fast`   example 7's finalized FNV-1a: cheap, well mixed, attackable.
//            Used to take the hash function's cost out of the benchmark.
//   `Djb2`   the deliberately weak hash of example 14 (h*33 + byte), used only
//            to show what the adversarial key set does to a table WITHOUT a seed.
const FNV_OFFSET: u64 = 14695981039346656037;
const FNV_PRIME: u64 = 1099511628211;

#[derive(Clone, Copy)]
struct Fast(u64);
impl Default for Fast {
    fn default() -> Self {
        Fast(FNV_OFFSET)
    }
}
impl Hasher for Fast {
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 ^= b as u64;
            self.0 = self.0.wrapping_mul(FNV_PRIME);
        }
    }
    // i64 keys arrive here: write_i64 forwards to write_u64.
    fn write_u64(&mut self, x: u64) {
        self.0 = (self.0 ^ x).wrapping_mul(FNV_PRIME);
    }
    fn finish(&self) -> u64 {
        let mut h = self.0;
        h ^= h >> 30;
        h = h.wrapping_mul(0xbf58476d1ce4e5b9);
        h ^= h >> 27;
        h = h.wrapping_mul(0x94d049bb133111eb);
        h ^ (h >> 31)
    }
}
type FastBuild = BuildHasherDefault<Fast>;

#[derive(Clone, Copy)]
struct Djb2(u64);
impl Default for Djb2 {
    fn default() -> Self {
        Djb2(5381)
    }
}
impl Hasher for Djb2 {
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 = self.0.wrapping_mul(33).wrapping_add(b as u64);
        }
    }
    fn finish(&self) -> u64 {
        self.0
    }
}
type Djb2Build = BuildHasherDefault<Djb2>;

// ---- BUILD IT ---------------------------------------------------------------

#[derive(Clone, Copy, PartialEq)]
enum State {
    Free, // never used -- a probe may stop here
    Used,
    Tomb, // was used; a probe must NOT stop here (example 4)
}

// maxLoad counts used + tombstones. Counting only `used` is the bug that makes
// a churned table stop terminating (examples 4 and 5).
const MAX_LOAD: f64 = 0.75;

// Three parallel arrays, as in Go and as in example 5: 8 + 8 + 1 = 17 bytes
// per slot for an i64 -> i64 map, which is also what a hashbrown bucket costs
// (a 16-byte entry plus one control byte), so the memory comparison below is
// like for like. One `Vec` of an enum would be tidier (example 4) and 24 bytes
// per slot, because the tag pads out to the entry's alignment.
//
// The price of parallel arrays is that a free slot still holds SOME key and
// SOME value. Go zero-fills with `make`; Rust has no zero value for a generic
// type, so K and V must be `Default` -- and "Delete zeroes the slot" becomes
// a real ownership matter: overwriting a slot DROPS what was in it.
//
// `S` is the hasher, std's own parameter. The default, `RandomState`, is the
// per-map seed.
struct HashMap<K, V, S = RandomState> {
    keys: Vec<K>,
    vals: Vec<V>,
    states: Vec<State>,
    used: usize,
    tombs: usize,

    hasher: S, // per-map, so an attacker cannot precompute (example 14)

    // instrumentation, so the tests can assert on behaviour rather than guess
    rehashes: usize,
    probes: Cell<usize>, // a Cell so that `get(&self)` can still count
}

fn zeroed<T: Default>(n: usize) -> Vec<T> {
    std::iter::repeat_with(T::default).take(n).collect()
}

impl<K: Hash + Eq + Default, V: Default> HashMap<K, V> {
    fn new(hint: usize) -> Self {
        Self::with_hasher(hint, RandomState::new())
    }
}

impl<K: Hash + Eq + Default, V: Default, S: BuildHasher> HashMap<K, V, S> {
    fn with_hasher(hint: usize, hasher: S) -> Self {
        let mut size = 8;
        while hint as f64 > MAX_LOAD * size as f64 {
            size <<= 1;
        }
        HashMap {
            keys: zeroed(size),
            vals: zeroed(size),
            states: vec![State::Free; size],
            used: 0,
            tombs: 0,
            hasher,
            rehashes: 0,
            probes: Cell::new(0),
        }
    }

    fn mask(&self) -> usize {
        self.keys.len() - 1
    }

    // `hash_one` is Go's `maphash.Comparable(seed, k)`: it hashes any `Hash`
    // value under this map's own seed. (Go's own hash/fnv has no seed, and
    // example 14 is why that matters.)
    fn home(&self, k: &K) -> usize {
        self.hasher.hash_one(k) as usize & self.mask()
    }

    fn len(&self) -> usize {
        self.used
    }

    fn cap(&self) -> usize {
        self.keys.len()
    }

    fn get(&self, k: &K) -> Option<&V> {
        let mut i = self.home(k);
        loop {
            self.probes.set(self.probes.get() + 1);
            match self.states[i] {
                State::Free => return None,
                State::Used if self.keys[i] == *k => return Some(&self.vals[i]),
                State::Used | State::Tomb => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    fn put(&mut self, k: K, v: V) {
        if (self.used + self.tombs + 1) as f64 > MAX_LOAD * self.keys.len() as f64 {
            self.rehash();
        }
        self.place(k, v);
    }

    fn place(&mut self, k: K, v: V) {
        let mut i = self.home(&k);
        let mut first_tomb: Option<usize> = None; // Go: -1 means "none yet"
        loop {
            match self.states[i] {
                State::Free => {
                    let at = match first_tomb {
                        Some(t) => {
                            self.tombs -= 1;
                            t
                        }
                        None => i,
                    };
                    self.keys[at] = k;
                    self.vals[at] = v;
                    self.states[at] = State::Used;
                    self.used += 1;
                    return;
                }
                State::Tomb => {
                    first_tomb.get_or_insert(i);
                }
                State::Used if self.keys[i] == k => {
                    self.vals[i] = v;
                    return;
                }
                State::Used => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    // Returns the removed value (Go returns a bool; `Option<V>` says more and
    // is what std's `remove` returns).
    fn delete(&mut self, k: &K) -> Option<V> {
        let mut i = self.home(k);
        loop {
            match self.states[i] {
                State::Free => return None,
                State::Used if self.keys[i] == *k => {
                    self.states[i] = State::Tomb;
                    // Go: `m.keys[i], m.vals[i] = zk, zv` -- release pointers
                    // (example 5, l09). Here the same two lines move the old
                    // value out (`take`) and DROP the old key (the assignment).
                    self.keys[i] = K::default();
                    let v = take(&mut self.vals[i]);
                    self.used -= 1;
                    self.tombs += 1;
                    return Some(v);
                }
                State::Used | State::Tomb => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    // rehash grows only when the LIVE count justifies it. A table that is mostly
    // tombstones rebuilds at the same size (example 5).
    fn rehash(&mut self) {
        let mut size = self.keys.len();
        if (self.used + 1) as f64 > MAX_LOAD * size as f64 / 2.0 {
            size *= 2;
        }
        // Go copies three slice headers; here the old arrays are moved out and
        // their entries MOVED into the new ones (no clone).
        let old_k = replace(&mut self.keys, zeroed(size));
        let old_v = replace(&mut self.vals, zeroed(size));
        let old_s = replace(&mut self.states, vec![State::Free; size]);
        self.used = 0;
        self.tombs = 0;
        for ((k, v), s) in old_k.into_iter().zip(old_v).zip(old_s) {
            if s == State::Used {
                self.place(k, v);
            }
        }
        self.rehashes += 1;
    }

    // Go's `All` is a range-over-func iterator; this is the Rust spelling, and
    // stopping early is just dropping it.
    fn iter(&self) -> impl Iterator<Item = (&K, &V)> {
        self.keys
            .iter()
            .zip(&self.vals)
            .zip(&self.states)
            .filter(|(_, s)| **s == State::Used)
            .map(|((k, v), _)| (k, v))
    }

    // ---- PROVE IT ----------------------------------------------------------

    // invariants must hold after EVERY operation.
    fn invariants(&self) -> Result<(), String> {
        let n = self.keys.len();
        if !n.is_power_of_two() {
            return Err(format!("capacity {n} is not a power of two"));
        }
        if self.vals.len() != n || self.states.len() != n {
            return Err("parallel arrays out of step".to_string());
        }
        let live = self.states.iter().filter(|&&s| s == State::Used).count();
        let tombs = self.states.iter().filter(|&&s| s == State::Tomb).count();
        if live != self.used {
            return Err(format!("used={} but {live} slots are occupied", self.used));
        }
        if tombs != self.tombs {
            return Err(format!(
                "tombs={} but {tombs} slots are tombstones",
                self.tombs
            ));
        }
        if (self.used + self.tombs) as f64 > MAX_LOAD * n as f64 {
            return Err(format!(
                "load {:.3} exceeds {MAX_LOAD:.3}",
                (self.used + self.tombs) as f64 / n as f64
            ));
        }
        Ok(())
    }
}

/// ns per element of one `n`-element pass.
fn ns_each(n: usize, f: impl FnMut()) -> f64 {
    ns_per_op(f) / n as f64
}

fn weak_hash(s: &[u8]) -> u64 {
    let mut h = Djb2::default();
    h.write(s);
    h.finish()
}

// collidingKeys finds n distinct strings that all hash to the same bucket
// under example 14's weak hash.
fn colliding_keys(n: usize, buckets: u64) -> Vec<String> {
    let target = weak_hash(b"aaaa") % buckets;
    let mut out = Vec::with_capacity(n);
    let mut buf = [0u8; 8];
    let mut i = 0usize;
    while out.len() < n {
        let mut v = i;
        for b in &mut buf {
            *b = b'a' + (v % 26) as u8;
            v /= 26;
        }
        if weak_hash(&buf) % buckets == target {
            out.push(String::from_utf8(buf.to_vec()).expect("ascii"));
        }
        i += 1;
    }
    out
}

// (row name, mine ns/element, std ns/element)
type Row = (&'static str, f64, f64);

fn row<'a>(rows: &'a [Row], name: &str) -> &'a Row {
    rows.iter().find(|r| r.0 == name).expect("row exists")
}

// One benchmark table, generic over the hasher so that BOTH maps use the same
// one.
fn compare<S: BuildHasher + Default>(ikeys: &[i64], absent: &[i64]) -> Vec<Row> {
    let n = ikeys.len();
    let mut mine = HashMap::<i64, i64, S>::with_hasher(n, S::default());
    let mut theirs = StdMap::<i64, i64, S>::with_capacity_and_hasher(n, S::default());
    for (i, &k) in ikeys.iter().enumerate() {
        mine.put(k, i as i64);
        theirs.insert(k, i as i64);
    }
    vec![
        (
            "build",
            ns_each(n, || {
                let mut m = HashMap::<i64, i64, S>::with_hasher(n, S::default());
                for (i, &k) in ikeys.iter().enumerate() {
                    m.put(k, i as i64);
                }
                black_box(m.len());
            }),
            ns_each(n, || {
                let mut m = StdMap::<i64, i64, S>::with_capacity_and_hasher(n, S::default());
                for (i, &k) in ikeys.iter().enumerate() {
                    m.insert(k, i as i64);
                }
                black_box(m.len());
            }),
        ),
        (
            "lookup, hit",
            ns_each(n, || {
                for k in ikeys {
                    black_box(mine.get(k));
                }
            }),
            ns_each(n, || {
                for k in ikeys {
                    black_box(theirs.get(k));
                }
            }),
        ),
        (
            "lookup, miss",
            ns_each(n, || {
                for k in absent {
                    black_box(mine.get(k));
                }
            }),
            ns_each(n, || {
                for k in absent {
                    black_box(theirs.get(k));
                }
            }),
        ),
        (
            "iterate",
            ns_each(n, || {
                for (_, v) in mine.iter() {
                    black_box(v);
                }
            }),
            ns_each(n, || {
                for v in theirs.values() {
                    black_box(v);
                }
            }),
        ),
    ]
}

// The verdicts below are computed from the numbers, so they stay true
// whichever way a noisy run falls. ratio = mine / std.
fn class(r: f64) -> &'static str {
    if r >= 1.15 {
        "slower"
    } else if r <= 1.0 / 1.15 {
        "faster"
    } else {
        "level"
    }
}

fn verdict(with_sip: f64, with_fast: f64) -> String {
    let (a, b) = (class(with_sip), class(with_fast));
    let summary = if a == b {
        format!("{a} under both hashers")
    } else {
        format!("{a} with SipHash, {b} with Fast")
    };
    format!("{with_sip:.2}x with SipHash, {with_fast:.2}x with Fast: {summary}")
}

fn print_rows(rows: &[Row]) {
    println!(
        "  {:<22} {:>14} {:>14} {:>10}",
        "operation", "HashMap ns", "std map ns", "ratio"
    );
    for &(name, a, b) in rows {
        println!("  {:<22} {:>14.2} {:>14.2} {:>9.2}x", name, a, b, a / b);
    }
}

fn main() {
    println!("PASS 1 -- BUILD IT");
    println!();
    let mut m = HashMap::<&str, i32>::new(4);
    for (i, k) in ["go", "rust", "zig", "c"].into_iter().enumerate() {
        m.put(k, i as i32);
    }
    m.delete(&"rust");
    m.put("odin", 9);
    let got: BTreeMap<&str, i32> = m.iter().map(|(&k, &v)| (k, v)).collect();
    let shown: Vec<String> = got.iter().map(|(k, v)| format!("{k}:{v}")).collect();
    println!(
        "  Len={} Cap={} rehashes={} tombstones={}",
        m.len(),
        m.cap(),
        m.rehashes,
        m.tombs
    );
    println!("  contents: map[{}]", shown.join(" "));
    println!(
        "  Get(\"zig\")={:?}   Get(\"rust\")={:?}",
        m.get(&"zig"),
        m.get(&"rust")
    );

    println!();
    println!("PASS 2 -- PROVE IT");
    println!();

    // 2a. model oracle: every operation mirrored against std's own map
    let mut rng = Rng(21);
    let mut hm = HashMap::<i64, i64>::new(4);
    let mut model = StdMap::<i64, i64>::new();
    const OPS: usize = 300_000;
    let (mut mismatch, mut inv_fail) = (0, 0);
    for _ in 0..OPS {
        let k = rng.below(20_000) as i64; // narrow range: forces real collisions and updates
        match rng.below(10) {
            0..=5 => {
                // Put
                let v = (rng.next_u64() >> 1) as i64;
                hm.put(k, v);
                model.insert(k, v);
            }
            6 | 7 => {
                // Delete
                if hm.delete(&k) != model.remove(&k) {
                    mismatch += 1;
                }
            }
            _ => {
                // Get
                if hm.get(&k) != model.get(&k) {
                    mismatch += 1;
                }
            }
        }
        if hm.len() != model.len() {
            mismatch += 1;
        }
        if hm.invariants().is_err() {
            inv_fail += 1;
        }
    }
    // full agreement at the end, both directions
    for (k, want) in &model {
        if hm.get(k) != Some(want) {
            mismatch += 1;
        }
    }
    for (k, v) in hm.iter() {
        if model.get(k) != Some(v) {
            mismatch += 1;
        }
    }
    println!(
        "  {:<52} {}",
        format!("{OPS} random ops vs std's map -- mismatches:"),
        mismatch
    );
    println!(
        "  {:<52} {}",
        "invariant failures (checked after EVERY op):", inv_fail
    );
    println!(
        "  {:<52} {} entries, {} rehashes",
        "final state:",
        hm.len(),
        hm.rehashes
    );

    // 2b. the churn case: tombstones must not accumulate without bound
    let mut churn = HashMap::<i64, i64>::new(1024);
    for i in 0..500 {
        churn.put(i, i);
    }
    let mut worst_load = 0.0f64;
    for round in 0..2000 {
        churn.put(1_000_000 + round, round);
        churn.delete(&(1_000_000 + round));
        let l = (churn.used + churn.tombs) as f64 / churn.keys.len() as f64;
        worst_load = worst_load.max(l);
    }
    println!(
        "  {:<52} {:.3} (limit {:.2})",
        "worst load factor over 2000 put/delete cycles:", worst_load, MAX_LOAD
    );

    // 2c. the adversarial set from example 14
    let attack = colliding_keys(4000, 4096);
    let mut adv = HashMap::<String, usize>::new(4000);
    for (i, k) in attack.iter().enumerate() {
        adv.put(k.clone(), i);
    }
    adv.probes.set(0);
    for k in &attack {
        adv.get(k);
    }
    let seeded_probes = adv.probes.get() as f64 / attack.len() as f64;
    println!(
        "  {:<52} {:.2}",
        "probes/lookup on 4000 hashDoS keys:", seeded_probes
    );

    // Not in the Go program: the same keys and the same table with the seed
    // taken away -- an unseeded Djb2, the hash those keys were built against.
    let mut fixed = HashMap::<String, usize, Djb2Build>::with_hasher(4000, Djb2Build::default());
    for (i, k) in attack.iter().enumerate() {
        fixed.put(k.clone(), i);
    }
    fixed.probes.set(0);
    for k in &attack {
        fixed.get(k);
    }
    let fixed_probes = fixed.probes.get() as f64 / attack.len() as f64;
    println!(
        "  {:<52} {:.2}",
        "  same keys, same table, UNSEEDED Djb2 hash:", fixed_probes
    );

    // Not in the Go program: Go's checklist says "Delete zeroes the slot" so
    // that K and V's pointers can die. Rust has no GC to be fooled, but the
    // same slip is possible: a slot that is only MARKED deleted would still own
    // its key and value. An `Rc` makes the ownership visible: `delete` returns
    // the value and the caller drops it at once, so the count must fall.
    let token = Rc::new(());
    let mut owned = HashMap::<String, Rc<()>>::new(4);
    owned.put("a".to_string(), Rc::clone(&token));
    let while_stored = Rc::strong_count(&token);
    owned.delete(&"a".to_string());
    let after_delete = Rc::strong_count(&token);
    println!(
        "  {:<52} {} -> {}",
        "  Rc owners of a value: stored, after Delete:", while_stored, after_delete
    );

    println!();
    println!("  three layers, and each catches what the others cannot: the model");
    println!("  finds wrong ANSWERS, the invariants find corrupt STATE the model");
    println!("  has not surfaced yet, and the adversarial set reaches a case that");
    println!("  random testing never will.");
    println!();
    println!(
        "  the seed is what the adversarial rows are about: {seeded_probes:.2} probes per lookup with a"
    );
    println!(
        "  per-map random key, {fixed_probes:.2} without one ({:.0}x). The keys were chosen against Djb2;",
        fixed_probes / seeded_probes
    );
    println!("  under a seed they cannot know, they behave like any other keys.");

    println!();
    println!("PASS 3 -- MEASURE IT");
    println!();
    const N: usize = 200_000;
    let ikeys: Vec<i64> = (0..N as i64).map(|i| i * 2654435761 + 1).collect();
    let absent: Vec<i64> = ikeys.iter().map(|&k| -k).collect();

    println!("  both maps hash with std's RandomState (SipHash-1-3, random key per map):");
    println!();
    let sip = compare::<RandomState>(&ikeys, &absent);
    print_rows(&sip);

    let mine_kb = live_kb(|| {
        let mut mm = HashMap::<i64, i64>::new(N);
        for (i, &k) in ikeys.iter().enumerate() {
            mm.put(k, i as i64);
        }
        mm
    });
    let (theirs_kb, theirs_cap) = {
        let mut cap = 0;
        let kb = live_kb(|| {
            let mut mm = StdMap::<i64, i64>::with_capacity(N);
            for (i, &k) in ikeys.iter().enumerate() {
                mm.insert(k, i as i64);
            }
            cap = mm.capacity();
            mm
        });
        (kb, cap)
    };
    println!(
        "  {:<22} {:>13.0}K {:>13.0}K {:>9.2}x",
        "memory",
        mine_kb,
        theirs_kb,
        mine_kb / theirs_kb
    );

    // Not in the Go program: the same rows with a cheap hash on BOTH sides, so
    // that SipHash's own cost is not the largest line in the table; and at a
    // size where both tables sit in cache, to separate per-probe work from
    // memory traffic.
    println!();
    println!("  the same rows with the cheap unseeded `Fast` hash in BOTH maps:");
    println!();
    let fast = compare::<FastBuild>(&ikeys, &absent);
    print_rows(&fast);
    const SMALL: usize = 2_000;
    println!();
    println!("  and with `Fast` at {SMALL} entries, where both tables fit in cache:");
    println!();
    let small = compare::<FastBuild>(&ikeys[..SMALL], &absent[..SMALL]);
    print_rows(&small);

    // Not in the Go program: what the big-table numbers are made of.
    let mut probe_map = HashMap::<i64, i64>::new(N);
    for (i, &k) in ikeys.iter().enumerate() {
        probe_map.put(k, i as i64);
    }
    probe_map.probes.set(0);
    for k in &ikeys {
        probe_map.get(k);
    }
    let hit_probes = probe_map.probes.get() as f64 / N as f64;
    probe_map.probes.set(0);
    for k in &absent {
        probe_map.get(k);
    }
    let miss_probes = probe_map.probes.get() as f64 / N as f64;
    let slot_bytes = 2 * size_of::<i64>() + size_of::<State>();
    let slots = probe_map.cap();
    let theirs_buckets = (theirs_kb * 1024.0) as usize / (size_of::<(i64, i64)>() + 1);
    println!();
    println!(
        "  mine: {slots} slots for {N} entries = load {:.2}, {slot_bytes} B per slot;",
        N as f64 / slots as f64
    );
    println!("        {hit_probes:.2} probes per hit, {miss_probes:.2} per miss");
    println!(
        "  std:  ~{theirs_buckets} buckets (live bytes / 17: a 16-byte entry + 1 control byte) = load {:.2},",
        N as f64 / theirs_buckets as f64
    );
    println!("        capacity() = {theirs_cap}");

    let ratio = |rows: &[Row], name: &str| {
        let &(_, a, b) = row(rows, name);
        a / b
    };
    println!();
    println!("  reading the rows (ratio = mine / std, above 1 means mine is slower;");
    println!("  within 15% either way counts as level):");
    println!();
    for (label, name) in [
        ("BUILD  ", "build"),
        ("HIT    ", "lookup, hit"),
        ("MISS   ", "lookup, miss"),
        ("ITERATE", "iterate"),
    ] {
        println!(
            "    {label} {}",
            verdict(ratio(&sip, name), ratio(&fast, name))
        );
    }
    println!(
        "    MEMORY  {:.2}x: {slot_bytes} B per slot in both, {slots} slots against ~{theirs_buckets} buckets",
        mine_kb / theirs_kb
    );
    println!();
    println!(
        "  the load factor of this benchmark is {:.2}, not 0.75: New rounds {N} entries up to {slots}",
        N as f64 / slots as f64
    );
    println!("  slots, and 0.75 is only the ceiling that triggers a rehash. So a miss walks");
    println!(
        "  {miss_probes:.2} slots on average and a hit {hit_probes:.2}, and the 2x in memory is the slot count alone."
    );
    println!();
    let (hit_big, hit_small) = (ratio(&fast, "lookup, hit"), ratio(&small, "lookup, hit"));
    println!(
        "  HIT with the cheap hash: {hit_big:.2}x at {N} entries, {hit_small:.2}x at {SMALL}."
    );
    println!(
        "  {}",
        if hit_big < 1.15 {
            "No gap to explain at the large size in this run."
        } else if hit_small < 1.15 {
            "The gap is gone when both tables are in cache: it tracks table size, so memory traffic (not isolated further)."
        } else if hit_small < hit_big {
            "The gap shrinks but does not close when both tables are in cache: some memory traffic, some per-probe work (split not measured)."
        } else {
            "The gap does not shrink when both tables are in cache, so it is not a cache effect."
        }
    );
    let (_, it_mine, it_std) = *row(&sip, "iterate");
    println!(
        "  ITERATE: mine visits all {slots} slots, std ~{theirs_buckets} buckets: {:.1} ns per slot against {:.1} ns per bucket.",
        it_mine * N as f64 / slots as f64,
        it_std * N as f64 / theirs_buckets as f64
    );

    println!();
    println!("  and none of that counts what std has and mine lacks. From hashbrown 0.17.1, the");
    println!("  source vendored with std (not measured here, except capacity()): a group probe of");
    println!("  8 control tags at a time (NEON on aarch64; 16 with SSE2); a 7/8 limit before");
    println!(
        "  growing (capacity() = {theirs_cap} of ~{theirs_buckets} buckets above); and an in-place rehash instead of"
    );
    println!("  growth when at most half of the capacity is live, the same idea as mine. What");
    println!("  Go's text lists beyond that has no counterpart: std's default hash is SipHash-1-3");
    println!("  (no AES), the table is one flat array (no directory), and growth moves every");
    println!("  entry in one call (resize_inner), as rehash above does.");

    println!();
    println!("  so: do not ship this. The point of building it was never to beat");
    println!("  the runtime -- it was to be able to say WHY the built-in map is");
    println!("  shaped the way it is, and every design decision above came from a");
    println!("  measurement in this lesson.");

    println!();
    println!("THE CHECKLIST, which is the deliverable:");
    println!();
    for line in [
        "capacity is a power of two             (& instead of %)",
        "the load factor counts TOMBSTONES      (or a churned table never terminates)",
        "Delete writes a tombstone, not free    (or it deletes its neighbours too)",
        "Delete overwrites the slot             (or K and V are never dropped: the Rc count)",
        "rehash may keep the SAME size          (or a tombstoned table doubles forever)",
        "the hash is SEEDED                     (or example 14 is your outage; RandomState does it)",
        "tests use a NARROW key range           (or collisions never happen)",
        "tests check invariants every operation (the model alone is not enough)",
    ] {
        println!("  [x] {line}");
    }
    println!();
    println!("and the one-line summary of lesson 10:");
    println!();
    println!("      write HashMap<K, V>, and know exactly what you are getting.");
}
```

**Rust sample output:**

```
PASS 1 -- BUILD IT

  Len=4 Cap=8 rehashes=0 tombstones=1
  contents: map[c:3 go:0 odin:9 zig:2]
  Get("zig")=Some(2)   Get("rust")=None

PASS 2 -- PROVE IT

  300000 random ops vs std's map -- mismatches:        0
  invariant failures (checked after EVERY op):         0
  final state:                                         14915 entries, 12 rehashes
  worst load factor over 2000 put/delete cycles:       0.750 (limit 0.75)
  probes/lookup on 4000 hashDoS keys:                  1.49
    same keys, same table, UNSEEDED Djb2 hash:         1000.50
    Rc owners of a value: stored, after Delete:        2 -> 1

  three layers, and each catches what the others cannot: the model
  finds wrong ANSWERS, the invariants find corrupt STATE the model
  has not surfaced yet, and the adversarial set reaches a case that
  random testing never will.

  the seed is what the adversarial rows are about: 1.49 probes per lookup with a
  per-map random key, 1000.50 without one (670x). The keys were chosen against Djb2;
  under a seed they cannot know, they behave like any other keys.

PASS 3 -- MEASURE IT

  both maps hash with std's RandomState (SipHash-1-3, random key per map):

  operation                  HashMap ns     std map ns      ratio
  build                           15.02          11.25      1.34x
  lookup, hit                     15.30           8.83      1.73x
  lookup, miss                    13.90          14.31      0.97x
  iterate                          4.99           1.29      3.86x
  memory                          8704K          4352K      2.00x

  the same rows with the cheap unseeded `Fast` hash in BOTH maps:

  operation                  HashMap ns     std map ns      ratio
  build                            8.37           5.03      1.67x
  lookup, hit                      4.93           2.76      1.78x
  lookup, miss                     7.70           7.31      1.05x
  iterate                          6.63           1.91      3.48x

  and with `Fast` at 2000 entries, where both tables fit in cache:

  operation                  HashMap ns     std map ns      ratio
  build                            4.28           2.94      1.46x
  lookup, hit                      2.10           1.72      1.22x
  lookup, miss                     1.86           1.65      1.12x
  iterate                          1.12           0.56      2.02x

  mine: 524288 slots for 200000 entries = load 0.38, 17 B per slot;
        1.31 probes per hit, 1.82 per miss
  std:  ~262144 buckets (live bytes / 17: a 16-byte entry + 1 control byte) = load 0.76,
        capacity() = 229376

  reading the rows (ratio = mine / std, above 1 means mine is slower;
  within 15% either way counts as level):

    BUILD   1.34x with SipHash, 1.67x with Fast: slower under both hashers
    HIT     1.73x with SipHash, 1.78x with Fast: slower under both hashers
    MISS    0.97x with SipHash, 1.05x with Fast: level under both hashers
    ITERATE 3.86x with SipHash, 3.48x with Fast: slower under both hashers
    MEMORY  2.00x: 17 B per slot in both, 524288 slots against ~262144 buckets

  the load factor of this benchmark is 0.38, not 0.75: New rounds 200000 entries up to 524288
  slots, and 0.75 is only the ceiling that triggers a rehash. So a miss walks
  1.82 slots on average and a hit 1.31, and the 2x in memory is the slot count alone.

  HIT with the cheap hash: 1.78x at 200000 entries, 1.22x at 2000.
  The gap shrinks but does not close when both tables are in cache: some memory traffic, some per-probe work (split not measured).
  ITERATE: mine visits all 524288 slots, std ~262144 buckets: 1.9 ns per slot against 1.0 ns per bucket.

  and none of that counts what std has and mine lacks. From hashbrown 0.17.1, the
  source vendored with std (not measured here, except capacity()): a group probe of
  8 control tags at a time (NEON on aarch64; 16 with SSE2); a 7/8 limit before
  growing (capacity() = 229376 of ~262144 buckets above); and an in-place rehash instead of
  growth when at most half of the capacity is live, the same idea as mine. What
  Go's text lists beyond that has no counterpart: std's default hash is SipHash-1-3
  (no AES), the table is one flat array (no directory), and growth moves every
  entry in one call (resize_inner), as rehash above does.

  so: do not ship this. The point of building it was never to beat
  the runtime -- it was to be able to say WHY the built-in map is
  shaped the way it is, and every design decision above came from a
  measurement in this lesson.

THE CHECKLIST, which is the deliverable:

  [x] capacity is a power of two             (& instead of %)
  [x] the load factor counts TOMBSTONES      (or a churned table never terminates)
  [x] Delete writes a tombstone, not free    (or it deletes its neighbours too)
  [x] Delete overwrites the slot             (or K and V are never dropped: the Rc count)
  [x] rehash may keep the SAME size          (or a tombstoned table doubles forever)
  [x] the hash is SEEDED                     (or example 14 is your outage; RandomState does it)
  [x] tests use a NARROW key range           (or collisions never happen)
  [x] tests check invariants every operation (the model alone is not enough)

and the one-line summary of lesson 10:

      write HashMap<K, V>, and know exactly what you are getting.
```

> **Go vs Rust:** Same hash on both sides (std's `RandomState`, SipHash-1-3; a second table gives both maps the cheap `Fast` hash). Over some fifteen runs BUILD is slower (1.3-1.6x with SipHash; Go 0.97x), MISS is level rather than faster (0.9-1.0x in quiet runs, up to 1.4x under load; Go 0.82x), ITERATE is 3.6-5.1x slower (Go 1.45x), and HIT is slower as in Go (1.4-1.9x; Go 1.52x), a gap that shrinks to about 1.05-1.25x with `Fast` at 2,000 entries. MEMORY is exactly 2.00x, not 2.85x: both layouts cost 17 B per slot (a `Vec` of enums would cost 24), so the ratio is the slot count alone (524,288 against ~262,144), and the "at 0.75 load" in Go's text is 0.38 in both languages.
> Go's `make` zero-fill becomes `K: Default, V: Default`, `maphash.MakeSeed` becomes `RandomState` plus `hash_one`, and "Delete zeroes the slot" becomes `take` plus an overwrite, whose effect an `Rc` count shows (2 -> 1). A Go twin and this program, both switched to the same fixed FNV hash and the same splitmix64 stream, printed identical counts (14,915 entries, 12 rehashes, 0.750, 1.43 probes per lookup), so the sample differs from Go's only through the RNG and the per-map seed; an unseeded Djb2 table on the same 4,000 keys needs ~1,000 probes per lookup against ~1.5.

**Complexity:** all operations Θ(1) amortized · **0 mismatches and 0 invariant failures over 300,000
operations**, load factor pinned at 0.750 through 2,000 churn cycles, **1.48 probes/lookup on 4,000
hashDoS keys** · it beats Go's map on build and miss, loses **1.5× on hit** and **2.85× on memory** —
and you should still not ship it

---

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Lesson: [10-hash-tables.md](../../10-hash-tables.md)
