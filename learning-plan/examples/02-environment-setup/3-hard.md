# Step 02 — Environment & Toolkit · 🔴 Hard

Examples **12–16**: the pieces of the reusable harness, then the harness itself.

Everything here is written once and copied into all 43 remaining lessons. Example 16 is the
deliverable of this whole lesson — the folder you'll start every future practice session from.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`). Most are
`#[cfg(test)]` files: one `.rs` file holding the code and its tests, where Go needed two files.

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into max.rs, then:
rustc --edition 2024 --test max.rs -o max && ./max     # run every test
./max negatives                                        # run the tests whose name contains "negatives"
```

Every example below shows the Go package and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Twins that are plain programs instead of test files (benchmarks,
fuzzing, measurements) say so under **Run it (Rust):** — build those with `rustc --edition 2024 -O`.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. A generic differential oracle

`🔴 hard` · *Generics / reusable test helper*

Lesson 01 wrote the oracle pattern by hand. Written once as a generic helper, it applies to every
algorithm in the plan without modification — the type parameters absorb the differences between
problems.

The design decisions worth copying: **`Out comparable`** so `==` is a valid verdict; **`t.Helper()`**
so failures point at the caller; a **fixed seed** so a failure reproduces exactly; and **fail fast** —
one readable counterexample beats a thousand identical ones.

**Steps:**

1. Write `Differential` once, parameterized over the input and output types.
2. Supply a generator that makes *small* inputs from a narrow value range.
3. Point it at two different problems to prove the helper is actually reusable.

**`algo.go`**

```go
package algo

// HasPair reports whether two distinct elements of xs sum to target. O(n).
func HasPair(xs []int, target int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		if _, ok := seen[target-v]; ok {
			return true
		}
		seen[v] = struct{}{}
	}
	return false
}

// CountDistinct returns how many distinct values xs holds. O(n).
func CountDistinct(xs []int) int {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		seen[v] = struct{}{}
	}
	return len(seen)
}
```

**`differential_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"testing"
)

// Differential is the oracle pattern, written once and reused forever.
//
//   - In  is whatever your function takes (often a struct of several arguments).
//   - Out must be comparable so that == is a valid verdict.
//   - gen builds one random input; keep it SMALL, so failures are readable.
//   - seed is fixed, so a failure reproduces exactly on the next run.
//
// It stops at the first disagreement: one minimal counterexample beats a
// thousand identical ones.
func Differential[In any, Out comparable](
	t *testing.T,
	trials int,
	seed uint64,
	gen func(*rand.Rand) In,
	oracle func(In) Out,
	candidate func(In) Out,
) {
	t.Helper()
	rng := rand.New(rand.NewPCG(seed, seed+1))

	for i := 0; i < trials; i++ {
		in := gen(rng)
		want := oracle(in)
		got := candidate(in)
		if got != want {
			t.Fatalf("disagreement on trial %d\n  input:     %+v\n  oracle:    %v\n  candidate: %v",
				i+1, in, want, got)
		}
	}
}

// --- use 1: two-sum ---------------------------------------------------------

type pairInput struct {
	Xs     []int
	Target int
}

func hasPairBrute(in pairInput) bool {
	for i := 0; i < len(in.Xs); i++ {
		for j := i + 1; j < len(in.Xs); j++ {
			if in.Xs[i]+in.Xs[j] == in.Target {
				return true
			}
		}
	}
	return false
}

func TestHasPairAgainstOracle(t *testing.T) {
	gen := func(rng *rand.Rand) pairInput {
		xs := make([]int, rng.IntN(7))
		for i := range xs {
			xs[i] = rng.IntN(11) - 5 // small range -> frequent collisions -> real coverage
		}
		return pairInput{Xs: xs, Target: rng.IntN(11) - 5}
	}

	Differential(t, 50_000, 42, gen,
		hasPairBrute,
		func(in pairInput) bool { return HasPair(in.Xs, in.Target) },
	)
}

// --- use 2: distinct count, same helper, different shapes --------------------

func countDistinctBrute(xs []int) int {
	n := 0
	for i, v := range xs {
		first := true
		for j := 0; j < i; j++ {
			if xs[j] == v {
				first = false
				break
			}
		}
		if first {
			n++
		}
	}
	return n
}

func TestCountDistinctAgainstOracle(t *testing.T) {
	gen := func(rng *rand.Rand) []int {
		xs := make([]int, rng.IntN(10))
		for i := range xs {
			xs[i] = rng.IntN(5)
		}
		return xs
	}

	Differential(t, 50_000, 7, gen, countDistinctBrute, CountDistinct)
}
```

**Run it:**

```bash
go test -v ./ex12
```

**Output:**

```
=== RUN   TestHasPairAgainstOracle
--- PASS: TestHasPairAgainstOracle (0.01s)
=== RUN   TestCountDistinctAgainstOracle
--- PASS: TestCountDistinctAgainstOracle (0.01s)
PASS
ok  	l02/ex12	0.623s
```

100,000 random trials across two problems, in 20 milliseconds. That's the whole argument for keeping
generated inputs tiny — you get exhaustive-feeling coverage of the interesting region for free.

**Rust:**

```rust
use std::collections::HashSet;

// ---- algo.rs: the code under test ----

/// Reports whether two distinct elements of `xs` sum to `target`. O(n).
fn has_pair(xs: &[i64], target: i64) -> bool {
    let mut seen = HashSet::with_capacity(xs.len());
    for &v in xs {
        if seen.contains(&(target - v)) {
            return true;
        }
        seen.insert(v);
    }
    false
}

/// Returns how many distinct values `xs` holds. O(n).
fn count_distinct(xs: &[i64]) -> usize {
    let seen: HashSet<i64> = xs.iter().copied().collect();
    seen.len()
}

// ---- differential_test.go -> a test module ----

#[cfg(test)]
mod tests {
    use super::*;
    use std::fmt::Debug;

    // No math/rand/v2 in std: a seeded splitmix64 stands in for rand.New(rand.NewPCG(..)).
    struct Rng(u64);
    impl Rng {
        fn new(seed: u64) -> Self {
            Rng(seed)
        }
        fn next_u64(&mut self) -> u64 {
            self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
            let mut z = self.0;
            z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
            z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
            z ^ (z >> 31)
        }
        /// like rng.IntN(n): uniform-enough in [0, n)
        fn below(&mut self, n: u64) -> u64 {
            ((self.next_u64() as u128 * n as u128) >> 64) as u64
        }
    }

    /// Differential is the oracle pattern, written once and reused forever.
    ///
    ///   - `In` is whatever your function takes (often a struct of several arguments).
    ///   - `Out: PartialEq` is what makes `!=` a valid verdict (Go's `comparable`);
    ///     `Debug` on both is how a failure gets printed (Go's %v needs no bound).
    ///   - `generate` builds one random input; keep it SMALL, so failures are readable.
    ///   - `seed` is fixed, so a failure reproduces exactly on the next run.
    ///
    /// It stops at the first disagreement: one minimal counterexample beats a
    /// thousand identical ones.
    ///
    /// `#[track_caller]` is Rust's t.Helper(): the panic is reported at the line that
    /// called `differential`, not at the `panic!` inside it. (Named `generate`
    /// because `gen` is a reserved keyword in edition 2024.)
    #[track_caller]
    fn differential<In, Out>(
        trials: u32,
        seed: u64,
        generate: impl Fn(&mut Rng) -> In,
        oracle: impl Fn(&In) -> Out,
        candidate: impl Fn(&In) -> Out,
    ) where
        In: Debug,
        Out: PartialEq + Debug,
    {
        let mut rng = Rng::new(seed);

        for i in 0..trials {
            let input = generate(&mut rng);
            let want = oracle(&input);
            let got = candidate(&input);
            if got != want {
                panic!(
                    "disagreement on trial {}\n  input:     {input:?}\n  oracle:    {want:?}\n  candidate: {got:?}",
                    i + 1
                );
            }
        }
    }

    // --- use 1: two-sum ---------------------------------------------------------

    #[derive(Debug)]
    struct PairInput {
        xs: Vec<i64>,
        target: i64,
    }

    fn has_pair_brute(input: &PairInput) -> bool {
        let xs = &input.xs;
        (0..xs.len()).any(|i| (i + 1..xs.len()).any(|j| xs[i] + xs[j] == input.target))
    }

    #[test]
    fn has_pair_against_oracle() {
        let generate = |rng: &mut Rng| {
            let len = rng.below(7) as usize;
            // small range -> frequent collisions -> real coverage
            let xs = (0..len).map(|_| rng.below(11) as i64 - 5).collect();
            PairInput {
                xs,
                target: rng.below(11) as i64 - 5,
            }
        };

        differential(50_000, 42, generate, has_pair_brute, |input| {
            has_pair(&input.xs, input.target)
        });
    }

    // --- use 2: distinct count, same helper, different shapes --------------------

    fn count_distinct_brute(xs: &[i64]) -> usize {
        let mut n = 0;
        for (i, v) in xs.iter().enumerate() {
            if !xs[..i].contains(v) {
                n += 1;
            }
        }
        n
    }

    #[test]
    fn count_distinct_against_oracle() {
        let generate = |rng: &mut Rng| {
            let len = rng.below(10) as usize;
            (0..len).map(|_| rng.below(5) as i64).collect::<Vec<_>>()
        };

        // The closures only adapt `&Vec<i64>` to `&[i64]`; no other glue is needed.
        differential(
            50_000,
            7,
            generate,
            |xs| count_distinct_brute(xs),
            |xs| count_distinct(xs),
        );
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test algo.rs -o algo && ./algo
```

**Rust sample output:**

```

running 2 tests
test tests::has_pair_against_oracle ... ok
test tests::count_distinct_against_oracle ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

> **Go vs Rust:** Go's `Out comparable` becomes `Out: PartialEq + Debug`: `PartialEq` is what `!=` needs, and `Debug` is needed because Rust prints nothing by reflection. `#[track_caller]` plays the part of `t.Helper()`; I checked it with a deliberately wrong candidate, and the panic was reported at the `differential(...)` call in the test, not at the `panic!` inside the helper. `Vec<i64>` is already `PartialEq`, so the use-2 closures only adapt `&Vec<i64>` to `&[i64]`; the inputs come from a splitmix64 generator, not Go's PCG, so they are different random inputs (both runs pass).

**Takeaway:** one generic helper, every algorithm. Note the second call passes a plain `[]int` as `In` while the first passes a struct — the type parameters absorb the difference.

---

## 13. One property is never enough

`🔴 hard` · *Property-based invariants*

Everyone's first instinct for testing a sort is "check the output is sorted". Here is a function that
satisfies that property **on every input** and is catastrophically wrong: it's insertion sort with the
final write deleted — a one-line omission that survives code review.

It still shifts larger elements rightward, so the result is still non-decreasing. But the element it
was supposed to insert is never written back; it gets overwritten by a copy of its neighbour. Data
disappears, silently, behind a perfectly sorted-looking result.

The **permutation** property — same values, same counts — catches it immediately. Every sort needs both.

**Steps:**

1. Write both properties: `isSorted` and `isPermutation`.
2. Verify the real sort satisfies both across 20,000 random inputs.
3. Verify `BadSort` satisfies the weak one across 20,000 inputs — and *assert* that it violates the strong one, so the demonstration is itself a passing test.

**`sort.go`**

```go
package algo

// InsertionSort sorts xs in place.
func InsertionSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		xs[j+1] = v // drop this one line and you get BadSort
	}
}

// BadSort is InsertionSort with the final write missing -- a one-line omission,
// the kind that survives code review.
//
// It still shifts the larger elements right, so the result is still
// non-decreasing on every input. But the element it was supposed to insert is
// never written back: it is overwritten by a copy of its neighbour. The sort
// silently destroys data while looking perfectly sorted.
func BadSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		// missing: xs[j+1] = v
	}
}
```

**`sort_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"slices"
	"testing"
)

// --- the two properties every sort must satisfy -----------------------------

// Property 1: the output is non-decreasing. The obvious check -- and on its own,
// worthless.
func isSorted(xs []int) bool {
	for i := 1; i < len(xs); i++ {
		if xs[i] < xs[i-1] {
			return false
		}
	}
	return true
}

// Property 2: the output is a permutation of the input -- same values, same
// counts. This is the property that says "you didn't invent or lose data".
func isPermutation(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}
	count := make(map[int]int, len(a))
	for _, v := range a {
		count[v]++
	}
	for _, v := range b {
		count[v]--
		if count[v] < 0 {
			return false
		}
	}
	return true
}

func randomSlice(rng *rand.Rand) []int {
	xs := make([]int, rng.IntN(12))
	for i := range xs {
		xs[i] = rng.IntN(6) // small range: duplicates are the interesting case
	}
	return xs
}

// --- the real sort passes both ----------------------------------------------

func TestInsertionSortProperties(t *testing.T) {
	rng := rand.New(rand.NewPCG(1, 2))
	for i := 0; i < 20_000; i++ {
		in := randomSlice(rng)
		out := slices.Clone(in)
		InsertionSort(out)

		if !isSorted(out) {
			t.Fatalf("not sorted: %v -> %v", in, out)
		}
		if !isPermutation(in, out) {
			t.Fatalf("not a permutation: %v -> %v", in, out)
		}
	}
}

// --- and here is why one property is never enough ---------------------------

// BadSort satisfies "is sorted" on every input. A test suite that checks only
// this property is green.
func TestBadSortSatisfiesTheWeakProperty(t *testing.T) {
	rng := rand.New(rand.NewPCG(3, 4))
	for i := 0; i < 20_000; i++ {
		out := randomSlice(rng)
		BadSort(out)
		if !isSorted(out) {
			t.Fatalf("BadSort produced unsorted output: %v", out)
		}
	}
}

// The permutation property rejects it -- and here we assert that it does, so
// the demonstration itself is a passing test.
func TestBadSortViolatesThePermutationProperty(t *testing.T) {
	in := []int{3, 1, 3} // any input with a duplicate
	out := slices.Clone(in)
	BadSort(out)

	if !isSorted(out) {
		t.Fatalf("expected BadSort output to look sorted, got %v", out)
	}
	if isPermutation(in, out) {
		t.Fatalf("expected BadSort to lose data, but %v is a permutation of %v", out, in)
	}
	t.Logf("BadSort(%v) = %v -- perfectly sorted, and the 1 is simply gone", in, out)
}
```

**Run it:**

```bash
go test -v ./ex13
```

**Output:**

```
=== RUN   TestInsertionSortProperties
--- PASS: TestInsertionSortProperties (0.01s)
=== RUN   TestBadSortSatisfiesTheWeakProperty
--- PASS: TestBadSortSatisfiesTheWeakProperty (0.00s)
=== RUN   TestBadSortViolatesThePermutationProperty
    sort_test.go:95: BadSort([3 1 3]) = [3 3 3] -- perfectly sorted, and the 1 is simply gone
--- PASS: TestBadSortViolatesThePermutationProperty (0.00s)
PASS
ok  	l02/ex13	0.369s
```

`BadSort([3 1 3])` returns `[3 3 3]`. Sorted? Yes. A sort? No — the 1 is gone. **20,000 random inputs
could not tell the difference using `isSorted` alone.**

**Rust:**

```rust
// ---- sort.rs: the code under test ----

/// Sorts `xs` in place.
fn insertion_sort(xs: &mut [i64]) {
    for i in 1..xs.len() {
        let v = xs[i];
        let mut j = i;
        while j > 0 && xs[j - 1] > v {
            xs[j] = xs[j - 1];
            j -= 1;
        }
        xs[j] = v; // drop this one line and you get bad_sort
    }
}

/// `insertion_sort` with the final write missing -- a one-line omission, the kind
/// that survives code review.
///
/// It still shifts the larger elements right, so the result is still
/// non-decreasing on every input. But the element it was supposed to insert is
/// never written back: it is overwritten by a copy of its neighbour. The sort
/// silently destroys data while looking perfectly sorted.
///
/// Rust compiles it without a murmur: no warning, no borrow-checker objection.
/// Memory safety is not correctness.
fn bad_sort(xs: &mut [i64]) {
    for i in 1..xs.len() {
        let v = xs[i];
        let mut j = i;
        while j > 0 && xs[j - 1] > v {
            xs[j] = xs[j - 1];
            j -= 1;
        }
        // missing: xs[j] = v;
    }
}

// ---- sort_test.go -> a test module ----

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;
    use std::io::Write;

    // go test -v prints t.Logf lines; libtest swallows the println! of a PASSING
    // test unless you pass --show-output. Writing to the real stdout skips that
    // capture, so the line below is visible with a plain `./sort`.
    fn log(line: &str) {
        let _ = writeln!(std::io::stdout(), "{line}");
    }

    // No math/rand/v2 in std: a seeded splitmix64 stands in for rand.New(rand.NewPCG(..)).
    struct Rng(u64);
    impl Rng {
        fn new(seed: u64) -> Self {
            Rng(seed)
        }
        fn next_u64(&mut self) -> u64 {
            self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
            let mut z = self.0;
            z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
            z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
            z ^ (z >> 31)
        }
        /// like rng.IntN(n): uniform-enough in [0, n)
        fn below(&mut self, n: u64) -> u64 {
            ((self.next_u64() as u128 * n as u128) >> 64) as u64
        }
    }

    // --- the two properties every sort must satisfy -----------------------------

    // Property 1: the output is non-decreasing. The obvious check -- and on its own,
    // worthless. (std has `slice::is_sorted` now; written out so the property is
    // visible.)
    fn is_sorted(xs: &[i64]) -> bool {
        xs.windows(2).all(|w| w[0] <= w[1])
    }

    // Property 2: the output is a permutation of the input -- same values, same
    // counts. This is the property that says "you didn't invent or lose data".
    fn is_permutation(a: &[i64], b: &[i64]) -> bool {
        if a.len() != b.len() {
            return false;
        }
        let mut count: HashMap<i64, i64> = HashMap::with_capacity(a.len());
        for &v in a {
            *count.entry(v).or_insert(0) += 1;
        }
        for &v in b {
            let c = count.entry(v).or_insert(0);
            *c -= 1;
            if *c < 0 {
                return false;
            }
        }
        true
    }

    fn random_slice(rng: &mut Rng) -> Vec<i64> {
        let len = rng.below(12) as usize;
        // small range: duplicates are the interesting case
        (0..len).map(|_| rng.below(6) as i64).collect()
    }

    // --- the real sort passes both ----------------------------------------------

    #[test]
    fn insertion_sort_properties() {
        let mut rng = Rng::new(1);
        for _ in 0..20_000 {
            let input = random_slice(&mut rng);
            let mut out = input.clone();
            insertion_sort(&mut out);

            assert!(is_sorted(&out), "not sorted: {input:?} -> {out:?}");
            assert!(
                is_permutation(&input, &out),
                "not a permutation: {input:?} -> {out:?}"
            );
        }
    }

    // --- and here is why one property is never enough ---------------------------

    // bad_sort satisfies "is sorted" on every input. A test suite that checks only
    // this property is green.
    #[test]
    fn bad_sort_satisfies_the_weak_property() {
        let mut rng = Rng::new(3);
        for _ in 0..20_000 {
            let mut out = random_slice(&mut rng);
            bad_sort(&mut out);
            assert!(
                is_sorted(&out),
                "bad_sort produced unsorted output: {out:?}"
            );
        }
    }

    // The permutation property rejects it -- and here we assert that it does, so
    // the demonstration itself is a passing test.
    #[test]
    fn bad_sort_violates_the_permutation_property() {
        let input = vec![3, 1, 3]; // any input with a duplicate
        let mut out = input.clone();
        bad_sort(&mut out);

        assert!(
            is_sorted(&out),
            "expected bad_sort output to look sorted, got {out:?}"
        );
        assert!(
            !is_permutation(&input, &out),
            "expected bad_sort to lose data, but {out:?} is a permutation of {input:?}"
        );
        log(&format!(
            "bad_sort({input:?}) = {out:?} -- perfectly sorted, and the 1 is simply gone"
        ));
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test sort.rs -o sort && ./sort
```

**Rust sample output:**

```

running 3 tests
bad_sort([3, 1, 3]) = [3, 3, 3] -- perfectly sorted, and the 1 is simply gone
test tests::bad_sort_violates_the_permutation_property ... ok
test tests::bad_sort_satisfies_the_weak_property ... ok
test tests::insertion_sort_properties ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** Rust compiles `bad_sort` without a warning: the borrow checker guarantees memory safety, not that the algorithm keeps your data. libtest hides the `println!` of a passing test (Go's `t.Logf` shows under `-v`), so the `BadSort([3 1 3])` line is written straight to stdout; the std way to see captured output is `./sort --show-output`. The result matches Go: `[3, 3, 3]` is sorted and not a permutation, and `HashMap<i64, i64>` plays the part of `map[int]int`.

**Takeaway:** a property test is only as good as the property. For any rearrangement, always assert the multiset is preserved — that's the property that says "you didn't invent or lose data".

---

## 14. Where did the time go

`🔴 hard` · *pprof*

Profiling takes thirty seconds and routinely contradicts your guess. Here a function does one O(n)
pass followed by one O(n²) pass; the profile settles which one matters without any argument.

**Steps:**

1. Run the benchmark with `-cpuprofile`, keeping the test binary with `-o`.
2. Open the profile with `go tool pprof -top`.
3. Read the `flat` column: time spent *in* that function; `cum` includes everything it called.

**`pairs.go`**

```go
package algo

// CountPairs counts index pairs (i<j) whose values sum to target.
// Deliberately quadratic -- this is the function the profiler should point at.
func CountPairs(xs []int, target int) int {
	n := 0
	for i := 0; i < len(xs); i++ {
		for j := i + 1; j < len(xs); j++ {
			if xs[i]+xs[j] == target {
				n++
			}
		}
	}
	return n
}

// normalize is a cheap helper called once per element. It is here to show what
// a *small* contributor looks like in a profile, next to the real hot spot.
func normalize(xs []int, mod int) []int {
	out := make([]int, len(xs))
	for i, v := range xs {
		out[i] = ((v % mod) + mod) % mod
	}
	return out
}

// Analyze is the entry point: one O(n) pass, then one O(n^2) pass.
func Analyze(xs []int, target, mod int) int {
	return CountPairs(normalize(xs, mod), target)
}
```

**`pairs_test.go`**

```go
package algo

import (
	"math/rand/v2"
	"testing"
)

func benchInput(n int) []int {
	rng := rand.New(rand.NewPCG(5, 6))
	xs := make([]int, n)
	for i := range xs {
		xs[i] = rng.IntN(1000) - 500
	}
	return xs
}

func BenchmarkAnalyze(b *testing.B) {
	xs := benchInput(4000)
	b.ReportAllocs()
	for b.Loop() {
		Analyze(xs, 100, 97)
	}
}
```

**Run it:**

```bash
go test -bench=Analyze -run='^$' -cpuprofile=cpu.out -o ex14.test ./ex14
go tool pprof -top -nodecount=8 ex14.test cpu.out
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex14
cpu: Apple M4
BenchmarkAnalyze-10    	     537	   1992599 ns/op	   32798 B/op	       1 allocs/op
PASS
ok  	l02/ex14	1.587s
```

```
File: ex14.test
Type: cpu
Duration: 1.21s, Total samples = 940ms (77.63%)
Showing nodes accounting for 940ms, 100% of 940ms total
Showing top 8 nodes out of 28
      flat  flat%   sum%        cum   cum%
     760ms 80.85% 80.85%      820ms 87.23%  l02/ex14.CountPairs (inline)
      80ms  8.51% 89.36%       80ms  8.51%  runtime.madvise
      60ms  6.38% 95.74%       60ms  6.38%  runtime.asyncPreempt
      10ms  1.06% 96.81%       30ms  3.19%  l02/ex14.normalize (inline)
      10ms  1.06% 97.87%       10ms  1.06%  runtime.(*mspan).init
      10ms  1.06% 98.94%       10ms  1.06%  runtime.kevent
      10ms  1.06%   100%       10ms  1.06%  runtime.memclrNoHeapPointers
         0     0%   100%      850ms 90.43%  l02/ex14.Analyze
```

`CountPairs` is **80.85%** flat; `normalize` is **1.06%**. Optimizing `normalize` to infinite speed
would buy 1%. Other views worth knowing: `-list=CountPairs` annotates the source line by line,
`-http=:8080` opens the flame graph, and `-memprofile` does the same for allocations.

**Rust:**

```rust
// There is no `go tool pprof` bundled with Rust, and no profiler is installed
// here, so this is the std-only version of the same question: time each phase
// ALONE and compare. (The real tools are `cargo flamegraph`, `samply`, `perf`
// and Instruments -- none run here.)
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- pairs.rs: the code under test ----

/// Counts index pairs (i<j) whose values sum to `target`.
/// Deliberately quadratic -- this is the function the profiler should point at.
///
/// `#[inline(never)]` here and on `normalize` keeps them separate functions: with
/// inlining the optimizer would fold both into `analyze`, and neither a profiler
/// frame nor this program's per-phase timing could tell them apart.
#[inline(never)]
fn count_pairs(xs: &[i64], target: i64) -> usize {
    let mut n = 0;
    for (i, &a) in xs.iter().enumerate() {
        for &b in &xs[i + 1..] {
            if a + b == target {
                n += 1;
            }
        }
    }
    n
}

/// A cheap helper called once per element. It is here to show what a *small*
/// contributor looks like next to the real hot spot.
#[inline(never)]
fn normalize(xs: &[i64], modulus: i64) -> Vec<i64> {
    xs.iter().map(|&v| v.rem_euclid(modulus)).collect()
}

/// The entry point: one O(n) pass, then one O(n^2) pass.
fn analyze(xs: &[i64], target: i64, modulus: i64) -> usize {
    count_pairs(&normalize(xs, modulus), target)
}

// ---- the measuring kit: std-only stand-ins for -benchmem and b.Loop ----

struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes requested, summed over all events

unsafe impl GlobalAlloc for Counting {
    // `unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every method only
    // counts and then forwards to the system allocator unchanged.
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

/// Calibrates like b.Loop: grows the iteration count until a run takes ~200 ms.
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

// ---- pairs_test.go: the benchmark ----

// splitmix64, seeded: stands in for rand.New(rand.NewPCG(5, 6)).
fn bench_input(n: usize) -> Vec<i64> {
    let mut state: u64 = 5;
    (0..n)
        .map(|_| {
            state = state.wrapping_add(0x9e3779b97f4a7c15);
            let mut z = state;
            z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
            z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
            z ^= z >> 31;
            (z % 1000) as i64 - 500
        })
        .collect()
}

fn main() {
    let xs = bench_input(4000);

    // BenchmarkAnalyze: ns/op, B/op, allocs/op
    let (allocs, bytes, pairs) = allocs_of(|| analyze(black_box(&xs), 100, 97));
    let total = ns_per_op(|| {
        black_box(analyze(black_box(&xs), black_box(100), black_box(97)));
    });
    println!("Analyze(n=4000) found {pairs} pairs");
    println!("BenchmarkAnalyze  {total:>12.0} ns/op  {bytes:>8} B/op  {allocs:>3} allocs/op");

    // The profile, by hand: each phase timed alone, on the same data.
    let normalized = normalize(&xs, 97);
    let t_norm = ns_per_op(|| {
        black_box(normalize(black_box(&xs), black_box(97)));
    });
    let t_pairs = ns_per_op(|| {
        black_box(count_pairs(black_box(&normalized), black_box(100)));
    });

    let sum = t_norm + t_pairs;
    println!();
    println!("{:<12} {:>12} {:>8}", "function", "ns/op", "share");
    println!(
        "{:<12} {:>12.0} {:>7.2}%",
        "count_pairs",
        t_pairs,
        100.0 * t_pairs / sum
    );
    println!(
        "{:<12} {:>12.0} {:>7.2}%",
        "normalize",
        t_norm,
        100.0 * t_norm / sum
    );
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O pairs.rs -o pairs && ./pairs
```

**Rust sample output:**

```
Analyze(n=4000) found 78564 pairs
BenchmarkAnalyze        767001 ns/op     32000 B/op    1 allocs/op

function            ns/op    share
count_pairs        767641   99.73%
normalize            2106    0.27%
```

> **Go vs Rust:** Rust bundles no profiler and none is installed here (`cargo flamegraph`, `samply`, `perf` and Instruments are the usual tools), so the table is a hand-made profile: each phase timed alone on the same data, and its shares are of those two times, not of all samples as in `pprof` (hence no `runtime.*` rows). It lands on the same verdict, `count_pairs` about 99.7% against `normalize` about 0.3% in each of three runs, and `B/op` is exactly `4000 x 8 = 32000` in 1 allocation, where Go reported 32798. As a one-off cross-check outside this program I ran macOS `sample` on a build that loops `analyze` for 2.5 s: 1506 samples in `count_pairs`, 8 in `normalize`.

**Takeaway:** `flat` is where the CPU actually is. Profile first — the answer is often not where you'd bet, and it is never worth optimizing a 1% line.

---

## 15. An allocation budget that fails the build

`🔴 hard` · *testing.AllocsPerRun as a unit test*

A benchmark reports allocations; nobody notices when the number creeps up. A **test** with a budget
fails CI the moment someone regresses it — no human required to read a number and judge.

Allocation counts are deterministic (they follow the growth policy, not the clock), which is what makes
this viable as a hard assertion rather than a flaky one.

**Steps:**

1. Assign the result to a package-level sink so nothing is optimized away.
2. Measure with `testing.AllocsPerRun`, compare against a budget, and `t.Logf` the actual number either way.
3. Set each budget to the value you **measured**, not one you hoped for.

**`index.go`**

```go
package algo

// Dedupe returns the distinct values of xs in first-seen order.
// The capacity is reserved up front: one allocation for the result, whatever n is.
func Dedupe(xs []int) []int {
	seen := make(map[int]struct{}, len(xs))
	out := make([]int, 0, len(xs))
	for _, v := range xs {
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		out = append(out, v)
	}
	return out
}

// SumEven adds the even elements of xs. It allocates nothing at all, and a test
// should hold it to that.
func SumEven(xs []int) int {
	total := 0
	for _, v := range xs {
		if v%2 == 0 {
			total += v
		}
	}
	return total
}
```

**`index_test.go`**

```go
package algo

import "testing"

var (
	sinkSlice []int
	sinkInt   int
)

func input(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i % 500 // guarantees duplicates
	}
	return xs
}

// An allocation budget is a unit test, not a benchmark: it fails the build when
// someone regresses it, and it needs no human to read a number and judge.
//
// Set each budget to the value you MEASURED, not to a round number you hoped for.
func TestAllocationBudget(t *testing.T) {
	xs := input(1000)

	tests := []struct {
		name   string
		budget float64
		fn     func()
	}{
		{"SumEven allocates nothing", 0, func() { sinkInt = SumEven(xs) }},
		{"Dedupe: 1 slice + the map's buckets", 6, func() { sinkSlice = Dedupe(xs) }},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := testing.AllocsPerRun(100, tt.fn)
			if got > tt.budget {
				t.Errorf("allocs/op = %.0f, over the budget of %.0f", got, tt.budget)
			}
			t.Logf("allocs/op = %.0f (budget %.0f)", got, tt.budget)
		})
	}
}
```

**Run it:**

```bash
go test -v ./ex15
```

**Output:**

```
=== RUN   TestAllocationBudget
=== RUN   TestAllocationBudget/SumEven_allocates_nothing
    index_test.go:40: allocs/op = 0 (budget 0)
=== RUN   TestAllocationBudget/Dedupe:_1_slice_+_the_map's_buckets
    index_test.go:40: allocs/op = 6 (budget 6)
--- PASS: TestAllocationBudget (0.00s)
    --- PASS: TestAllocationBudget/SumEven_allocates_nothing (0.00s)
    --- PASS: TestAllocationBudget/Dedupe:_1_slice_+_the_map's_buckets (0.00s)
PASS
ok  	l02/ex15	0.373s
```

**What a regression looks like.** Writing this example I guessed a budget of 5 for `Dedupe` before
measuring. Here is what the same test printed:

```
=== RUN   TestAllocationBudget/Dedupe:_1_slice_+_the_map
    index_test.go:38: allocs/op = 6, over the budget of 5
    index_test.go:40: allocs/op = 6 (budget 5)
--- FAIL: TestAllocationBudget (0.00s)
    --- PASS: TestAllocationBudget/SumEven_allocates_nothing (0.00s)
    --- FAIL: TestAllocationBudget/Dedupe:_1_slice_+_the_map (0.00s)
FAIL
```

Six, not five: one for the result slice and five for the map's internal growth. **Measure, then write
the number down** — the same rule as everywhere else in this plan.

**Rust:**

```rust
use std::collections::HashSet;

// ---- index.rs: the code under test ----

/// Returns the distinct values of `xs` in first-seen order.
/// The capacity is reserved up front for both the set and the result.
fn dedupe(xs: &[i64]) -> Vec<i64> {
    let mut seen = HashSet::with_capacity(xs.len());
    let mut out = Vec::with_capacity(xs.len());
    for &v in xs {
        if seen.insert(v) {
            out.push(v);
        }
    }
    out
}

/// Adds the even elements of `xs`. It allocates nothing at all, and a test
/// should hold it to that.
fn sum_even(xs: &[i64]) -> i64 {
    xs.iter().filter(|&&v| v % 2 == 0).sum()
}

// ---- index_test.go -> a test module ----

#[cfg(test)]
mod tests {
    use super::*;
    use std::alloc::{GlobalAlloc, Layout, System};
    use std::cell::Cell;
    use std::hint::black_box;
    use std::io::Write;

    // ---- allocation counter: stand-in for testing.AllocsPerRun ----
    //
    // Counts per THREAD, not globally: libtest runs the tests in parallel, and a
    // global counter would also count whatever the other tests allocate while
    // this one is being measured. (Go's AllocsPerRun avoids the same problem by
    // pinning GOMAXPROCS to 1 while it measures.) A `const` thread-local with no
    // destructor is safe to touch from inside an allocator.
    struct Counting;
    thread_local! {
        static ALLOCS: Cell<usize> = const { Cell::new(0) };
    }

    unsafe impl GlobalAlloc for Counting {
        // `unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every method only
        // counts and then forwards to the system allocator unchanged.
        unsafe fn alloc(&self, l: Layout) -> *mut u8 {
            ALLOCS.with(|c| c.set(c.get() + 1));
            unsafe { System.alloc(l) }
        }
        unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
            unsafe { System.dealloc(p, l) }
        }
        unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
            ALLOCS.with(|c| c.set(c.get() + 1));
            unsafe { System.realloc(p, l, new) }
        }
    }
    #[global_allocator]
    static GLOBAL: Counting = Counting;

    /// Allocation events (alloc + realloc) on this thread while running `f`.
    fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
        let before = ALLOCS.with(Cell::get);
        let r = f();
        (ALLOCS.with(Cell::get) - before, r)
    }

    /// Like testing.AllocsPerRun: one warm-up call, then the average over `runs` calls.
    fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> usize {
        f(); // warm-up
        let (total, ()) = allocs_of(|| {
            for _ in 0..runs {
                f();
            }
        });
        total / runs
    }

    // go test -v prints t.Logf lines; libtest swallows the println! of a PASSING
    // test unless you pass --show-output. Writing to the real stdout skips that
    // capture, so the numbers are visible with a plain `./budget`.
    fn log(line: &str) {
        let _ = writeln!(std::io::stdout(), "{line}");
    }

    fn input(n: i64) -> Vec<i64> {
        (0..n).map(|i| i % 500).collect() // guarantees duplicates
    }

    /// Measures `f` and compares it with `budget`. Logs the actual number either way.
    fn check_budget(name: &str, budget: usize, f: impl FnMut()) -> Result<(), String> {
        let got = allocs_per_run(100, f);
        log(&format!("{name}: allocs/op = {got} (budget {budget})"));
        if got > budget {
            return Err(format!(
                "{name}: allocs/op = {got}, over the budget of {budget}"
            ));
        }
        Ok(())
    }

    // An allocation budget is a unit test, not a benchmark: it fails the build when
    // someone regresses it, and it needs no human to read a number and judge.
    //
    // Set each budget to the value you MEASURED, not to a round number you hoped for.
    #[test]
    fn allocation_budget() {
        let xs = input(1000);

        // The closures end in black_box: the Rust counterpart of Go's package-level
        // sink, so the optimizer cannot delete the call being measured.
        let sum_even_alloc_nothing = || {
            black_box(sum_even(black_box(&xs)));
        };
        let dedupe_alloc = || {
            black_box(dedupe(black_box(&xs)));
        };
        let tests: [(&str, usize, &dyn Fn()); 2] = [
            ("SumEven allocates nothing", 0, &sum_even_alloc_nothing),
            ("Dedupe: 1 Vec + 1 hash table", 2, &dedupe_alloc),
        ];

        // Like t.Errorf in a subtest: keep going, report every violation at the end.
        let mut failures = Vec::new();
        for (name, budget, f) in tests {
            if let Err(msg) = check_budget(name, budget, f) {
                failures.push(msg);
            }
        }
        assert!(failures.is_empty(), "{}", failures.join("\n"));
    }

    // What a regression looks like: the same Dedupe without the two reservations.
    // The budget check must reject it -- asserted here with #[should_panic], so the
    // demonstration is itself a passing test.
    #[test]
    #[should_panic(expected = "over the budget of 2")]
    fn regression_trips_the_budget() {
        let xs = input(1000);
        let dedupe_without_reserve = || {
            let mut seen = HashSet::new();
            let mut out = Vec::new();
            for &v in black_box(&xs) {
                if seen.insert(v) {
                    out.push(v);
                }
            }
            black_box(out);
        };
        if let Err(msg) = check_budget("Dedupe without reserve", 2, dedupe_without_reserve) {
            panic!("{msg}");
        }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test budget.rs -o budget && ./budget
```

**Rust sample output:**

```

running 2 tests
SumEven allocates nothing: allocs/op = 0 (budget 0)
Dedupe: 1 Vec + 1 hash table: allocs/op = 2 (budget 2)
test tests::allocation_budget ... ok
Dedupe without reserve: allocs/op = 17 (budget 2)
test tests::regression_trips_the_budget - should panic ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** There is no `testing.AllocsPerRun`; the Rust way is a counting `#[global_allocator]` and `allocs_of` (a packaged version is the `dhat` crate, not used here). It counts per thread because libtest runs tests in parallel, and the counts are exact: `Dedupe` is 2 allocations (the `Vec` and the hash table) where Go's took 6, and it stayed `2` in every run. Instead of quoting a failing run, `regression_trips_the_budget` asserts with `#[should_panic]` that the same function without the two `with_capacity` calls (17 allocations here) is rejected, so the demonstration is itself a passing test.

**Takeaway:** turn any number you care about into an assertion. Allocation counts are deterministic enough to be a hard gate, unlike timings.

---

## 16. The harness

`🔴 hard` · *The deliverable*

This is what lesson 02 exists to produce: a practice folder you can copy into every remaining lesson.
Four files, three of which never change.

| File | Changes per lesson? | What it is |
|------|--------------------|------------|
| `reverse.go` | ✅ replaced | whatever the lesson asked you to implement |
| `reverse_test.go` | ✅ rewritten | the lesson's own tests — table, invariants, oracle |
| `harness_test.go` | ❌ copied as-is | `Differential`, generators, `IsPermutation`, `EdgeCases`, `CheckInPlace` |
| `bench_test.go` | 🔸 lightly edited | empty control + the size sweep |
| `check.sh` | ❌ copied as-is | the five gates, one command |

The test file shows the **four layers** every lesson should have: table → invariants → oracle → a
named property. Each catches things the others don't.

**Steps:**

1. Copy the four harness files into `practice/NN-topic/`.
2. Replace the implementation file and rewrite the test file for the lesson's structure.
3. Run `./check.sh` until it prints "all gates passed", then tick the lesson in PROGRESS.md.

**`reverse.go`** — the part that changes

```go
package algo

// This file stands in for "whatever the lesson asked you to implement".
// Everything else in this folder is the harness, and does not change.

// Reverse reverses xs in place. O(n) time, O(1) space.
func Reverse(xs []int) {
	for i, j := 0, len(xs)-1; i < j; i, j = i+1, j-1 {
		xs[i], xs[j] = xs[j], xs[i]
	}
}

// Rotate shifts xs left by k positions, in place, using the triple-reversal
// trick. O(n) time, O(1) space.
func Rotate(xs []int, k int) {
	n := len(xs)
	if n == 0 {
		return
	}
	k = ((k % n) + n) % n // normalize, including negative k
	Reverse(xs[:k])
	Reverse(xs[k:])
	Reverse(xs)
}
```

**`harness_test.go`** — copy this into every lesson, unchanged

```go
package algo

import (
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// The harness. Copy this file into every practice/NN-topic/ folder unchanged.
// ============================================================================

// Differential runs candidate against a slow-but-obviously-correct oracle on
// `trials` generated inputs, and stops at the first disagreement.
func Differential[In any, Out comparable](
	t *testing.T,
	trials int,
	seed uint64,
	gen func(*rand.Rand) In,
	oracle func(In) Out,
	candidate func(In) Out,
) {
	t.Helper()
	rng := rand.New(rand.NewPCG(seed, seed+1))

	for i := 0; i < trials; i++ {
		in := gen(rng)
		want := oracle(in)
		got := candidate(in)
		if got != want {
			t.Fatalf("disagreement on trial %d\n  input:     %+v\n  oracle:    %v\n  candidate: %v",
				i+1, in, want, got)
		}
	}
}

// RandomSlice generates a small slice with a small value range -- small enough
// that a failure is readable, narrow enough that duplicates actually occur.
func RandomSlice(rng *rand.Rand, maxLen, maxVal int) []int {
	xs := make([]int, rng.IntN(maxLen+1))
	for i := range xs {
		xs[i] = rng.IntN(maxVal)
	}
	return xs
}

// IsPermutation reports whether a and b hold the same values with the same
// counts. The property that catches a "sort" which loses data.
func IsPermutation(a, b []int) bool {
	if len(a) != len(b) {
		return false
	}
	count := make(map[int]int, len(a))
	for _, v := range a {
		count[v]++
	}
	for _, v := range b {
		count[v]--
		if count[v] < 0 {
			return false
		}
	}
	return true
}

// EdgeCases returns the inputs that break sequence algorithms, in the order
// they should be tried. Start every table with these.
func EdgeCases() []struct {
	Name string
	Xs   []int
} {
	return []struct {
		Name string
		Xs   []int
	}{
		{"nil", nil},
		{"empty", []int{}},
		{"single", []int{1}},
		{"two", []int{1, 2}},
		{"all equal", []int{7, 7, 7}},
		{"duplicates", []int{1, 2, 1, 2}},
		{"sorted", []int{1, 2, 3, 4, 5}},
		{"reverse sorted", []int{5, 4, 3, 2, 1}},
		{"negatives", []int{-2, -1, 0, 1}},
	}
}

// CheckInPlace runs fn over every edge case and asserts the invariant that
// holds for any in-place rearrangement: the multiset is unchanged.
func CheckInPlace(t *testing.T, name string, fn func([]int)) {
	t.Helper()
	for _, tc := range EdgeCases() {
		t.Run(name+"/"+tc.Name, func(t *testing.T) {
			original := slices.Clone(tc.Xs)
			got := slices.Clone(tc.Xs)
			fn(got)

			if len(got) != len(original) {
				t.Fatalf("length changed: %d -> %d", len(original), len(got))
			}
			if !IsPermutation(original, got) {
				t.Errorf("not a permutation: %v -> %v", original, got)
			}
		})
	}
}
```

**`reverse_test.go`** — the four layers, rewritten each lesson

```go
package algo

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// The lesson's own tests. This is the only file you write per lesson.
// ============================================================================

// 1. The table: specific inputs, specific expected outputs.
func TestReverse(t *testing.T) {
	tests := []struct {
		name string
		xs   []int
		want []int
	}{
		{"nil", nil, nil},
		{"empty", []int{}, nil},
		{"single", []int{1}, []int{1}},
		{"even length", []int{1, 2, 3, 4}, []int{4, 3, 2, 1}},
		{"odd length", []int{1, 2, 3}, []int{3, 2, 1}},
		{"duplicates", []int{1, 1, 2}, []int{2, 1, 1}},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := slices.Clone(tt.xs)
			Reverse(got)
			if !slices.Equal(got, tt.want) {
				t.Errorf("Reverse(%v) = %v, want %v", tt.xs, got, tt.want)
			}
		})
	}
}

func TestRotate(t *testing.T) {
	tests := []struct {
		name string
		xs   []int
		k    int
		want []int
	}{
		{"k=0", []int{1, 2, 3}, 0, []int{1, 2, 3}},
		{"k=1", []int{1, 2, 3}, 1, []int{2, 3, 1}},
		{"k=len", []int{1, 2, 3}, 3, []int{1, 2, 3}},
		{"k>len", []int{1, 2, 3}, 4, []int{2, 3, 1}},
		{"k negative", []int{1, 2, 3}, -1, []int{3, 1, 2}},
		{"empty", nil, 5, nil},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := slices.Clone(tt.xs)
			Rotate(got, tt.k)
			if !slices.Equal(got, tt.want) {
				t.Errorf("Rotate(%v, %d) = %v, want %v", tt.xs, tt.k, got, tt.want)
			}
		})
	}
}

// 2. The invariants: what must hold for EVERY input, checked over the edge cases.
func TestInPlaceInvariants(t *testing.T) {
	CheckInPlace(t, "Reverse", Reverse)
	CheckInPlace(t, "Rotate", func(xs []int) { Rotate(xs, 3) })
}

// 3. The oracle: agreement with a slow reference, over random input.
func TestReverseAgainstOracle(t *testing.T) {
	// The oracle: allocate a new slice and fill it backwards. Obviously correct.
	oracle := func(xs []int) string {
		out := make([]int, len(xs))
		for i, v := range xs {
			out[len(xs)-1-i] = v
		}
		return fmt.Sprint(out)
	}
	candidate := func(xs []int) string {
		out := slices.Clone(xs)
		Reverse(out)
		return fmt.Sprint(out)
	}

	Differential(t, 50_000, 1,
		func(rng *rand.Rand) []int { return RandomSlice(rng, 10, 6) },
		oracle, candidate)
}

// 4. Reverse is its own inverse -- a property worth stating outright.
func TestReverseTwiceIsIdentity(t *testing.T) {
	rng := rand.New(rand.NewPCG(2, 3))
	for i := 0; i < 20_000; i++ {
		original := RandomSlice(rng, 12, 8)
		got := slices.Clone(original)
		Reverse(got)
		Reverse(got)
		if !slices.Equal(got, original) {
			t.Fatalf("reversing twice changed %v into %v", original, got)
		}
	}
}
```

> Note the oracle returns a `string`: `Differential` needs a `comparable` output, and slices aren't
> comparable. Formatting with `fmt.Sprint` is the cheap, general way to compare non-comparable
> results — fine at 50,000 tiny inputs.

**`bench_test.go`** — control + sweep

```go
package algo

import (
	"fmt"
	"testing"
)

// ============================================================================
// The benchmark skeleton. Copy this file too; change only the sweep body.
// ============================================================================

// Always present, always first. No benchmark below this floor means anything.
func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// The sweep: the same operation across growing n, so the GROWTH is visible.
// Read the ns/op column, not any single row.
func BenchmarkReverse(b *testing.B) {
	for _, n := range []int{16, 256, 4096, 65536} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				Reverse(xs)
			}
		})
	}
}
```

**`check.sh`** — the five gates in one command

```bash
#!/usr/bin/env bash
# The gates. Run this before ticking a lesson done in PROGRESS.md.
# Usage: ./check.sh        (from inside a practice/NN-topic/ folder)

set -uo pipefail
fail=0

step() { printf '\n\033[1m== %s ==\033[0m\n' "$1"; }
ok()   { printf 'ok\n'; }

step "gofmt"
unformatted=$(gofmt -l .)
if [ -n "$unformatted" ]; then
	echo "needs formatting:"
	echo "$unformatted"
	fail=1
else
	ok
fi

step "go vet"
go vet ./... && ok || fail=1

step "go test -race"
go test -race ./... || fail=1

step "coverage"
go test -cover ./... || fail=1

step "benchmarks"
go test -bench=. -benchmem -run='^$' ./... || fail=1

printf '\n'
if [ "$fail" -ne 0 ]; then
	printf '\033[31mFAILED\033[0m -- fix the above before ticking this lesson done\n'
	exit 1
fi
printf '\033[32mall gates passed\033[0m\n'
```

**Run it:**

```bash
chmod +x check.sh
./check.sh
```

**Sample output** (colour codes stripped; timings vary):

```
== gofmt ==
ok

== go vet ==
ok

== go test -race ==
ok  	l02/ex16	1.614s

== coverage ==
ok  	l02/ex16	0.355s	coverage: 100.0% of statements

== benchmarks ==
goos: darwin
goarch: arm64
pkg: l02/ex16
cpu: Apple M4
BenchmarkEmptyControl-10    	638505613	         1.651 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=16-10    	385413799	         3.135 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=256-10   	24295677	        50.04 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=4096-10  	 1638211	       733.4 ns/op	       0 B/op	       0 allocs/op
BenchmarkReverse/n=65536-10 	   99662	     12152 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	l02/ex16	6.196s

all gates passed
```

Read the sweep: 16× the input costs 16× the time, all the way up, with **zero allocations** — exactly
what an in-place O(n) algorithm should look like. And the control at 1.651 ns confirms the n=16 row
(3.135 ns) is real work rather than harness noise.

**Rust:**

```rust
// ============================================================================
// plain rustc takes ONE file, so the folder's files are inline modules below.
// In a Cargo crate each section is a file of its own, with the same roles:
//   reverse.rs (changes)  harness (copied as-is)  tests (rewritten)  bench (edited)
// ============================================================================

// ---- reverse.rs -- the part that changes ----------------------------------

// This file stands in for "whatever the lesson asked you to implement".
// Everything else in this folder is the harness, and does not change.

/// Reverses `xs` in place. O(n) time, O(1) space.
/// (std has `<[T]>::reverse`; it is written out because it is the lesson.)
fn reverse(xs: &mut [i64]) {
    let n = xs.len();
    // Go's two-index loop `i < j` would underflow a usize at n == 0; n / 2 swaps do not.
    for i in 0..n / 2 {
        xs.swap(i, n - 1 - i);
    }
}

/// Shifts `xs` left by `k` positions, in place, using the triple-reversal
/// trick. O(n) time, O(1) space.
fn rotate(xs: &mut [i64], k: i64) {
    let n = xs.len();
    if n == 0 {
        return;
    }
    let k = k.rem_euclid(n as i64) as usize; // normalize, including negative k
    reverse(&mut xs[..k]);
    reverse(&mut xs[k..]);
    reverse(xs);
}

// ---- harness_test.go -- copy this into every lesson, unchanged -------------

#[cfg(test)]
mod harness {
    use std::collections::HashMap;
    use std::fmt::Debug;

    // No math/rand/v2 in std: a seeded splitmix64 stands in for rand.New(rand.NewPCG(..)).
    pub struct Rng(u64);
    impl Rng {
        pub fn new(seed: u64) -> Self {
            Rng(seed)
        }
        fn next_u64(&mut self) -> u64 {
            self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
            let mut z = self.0;
            z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
            z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
            z ^ (z >> 31)
        }
        /// like rng.IntN(n): uniform-enough in [0, n)
        pub fn below(&mut self, n: u64) -> u64 {
            ((self.next_u64() as u128 * n as u128) >> 64) as u64
        }
    }

    /// Runs `candidate` against a slow-but-obviously-correct `oracle` on `trials`
    /// generated inputs, and stops at the first disagreement.
    /// (`#[track_caller]` is t.Helper(); `Out: PartialEq + Debug` is `comparable`.)
    #[track_caller]
    pub fn differential<In, Out>(
        trials: u32,
        seed: u64,
        generate: impl Fn(&mut Rng) -> In,
        oracle: impl Fn(&In) -> Out,
        candidate: impl Fn(&In) -> Out,
    ) where
        In: Debug,
        Out: PartialEq + Debug,
    {
        let mut rng = Rng::new(seed);

        for i in 0..trials {
            let input = generate(&mut rng);
            let want = oracle(&input);
            let got = candidate(&input);
            if got != want {
                panic!(
                    "disagreement on trial {}\n  input:     {input:?}\n  oracle:    {want:?}\n  candidate: {got:?}",
                    i + 1
                );
            }
        }
    }

    /// Generates a small slice with a small value range -- small enough that a
    /// failure is readable, narrow enough that duplicates actually occur.
    pub fn random_slice(rng: &mut Rng, max_len: u64, max_val: u64) -> Vec<i64> {
        let len = rng.below(max_len + 1);
        (0..len).map(|_| rng.below(max_val) as i64).collect()
    }

    /// Reports whether `a` and `b` hold the same values with the same counts.
    /// The property that catches a "sort" which loses data.
    pub fn is_permutation(a: &[i64], b: &[i64]) -> bool {
        if a.len() != b.len() {
            return false;
        }
        let mut count: HashMap<i64, i64> = HashMap::with_capacity(a.len());
        for &v in a {
            *count.entry(v).or_insert(0) += 1;
        }
        for &v in b {
            let c = count.entry(v).or_insert(0);
            *c -= 1;
            if *c < 0 {
                return false;
            }
        }
        true
    }

    /// The inputs that break sequence algorithms, in the order they should be
    /// tried. Start every table with these. (Rust has no nil slice, so Go's "nil"
    /// and "empty" rows are one row here.)
    pub const EDGE_CASES: [(&str, &[i64]); 8] = [
        ("empty", &[]),
        ("single", &[1]),
        ("two", &[1, 2]),
        ("all equal", &[7, 7, 7]),
        ("duplicates", &[1, 2, 1, 2]),
        ("sorted", &[1, 2, 3, 4, 5]),
        ("reverse sorted", &[5, 4, 3, 2, 1]),
        ("negatives", &[-2, -1, 0, 1]),
    ];

    /// Runs `f` over every edge case and asserts the invariant that holds for any
    /// in-place rearrangement: the multiset is unchanged. (No t.Run in libtest, so
    /// the failing case is named in the message; Go's length check is gone too,
    /// because a `&mut [i64]` cannot change length.)
    #[track_caller]
    pub fn check_in_place(name: &str, f: impl Fn(&mut [i64])) {
        for (case, xs) in EDGE_CASES {
            let original = xs.to_vec();
            let mut got = xs.to_vec();
            f(&mut got);

            assert!(
                is_permutation(&original, &got),
                "{name}/{case}: not a permutation: {original:?} -> {got:?}"
            );
        }
    }
}

// ---- reverse_test.go -- the four layers, rewritten each lesson ------------

// The lesson's own tests. This is the only test module you write per lesson.

#[cfg(test)]
mod tests {
    use super::harness::*;
    use super::*;

    // 1. The table: specific inputs, specific expected outputs.
    #[test]
    fn reverse_table() {
        let tests: [(&str, &[i64], &[i64]); 5] = [
            ("empty", &[], &[]),
            ("single", &[1], &[1]),
            ("even length", &[1, 2, 3, 4], &[4, 3, 2, 1]),
            ("odd length", &[1, 2, 3], &[3, 2, 1]),
            ("duplicates", &[1, 1, 2], &[2, 1, 1]),
        ];

        for (name, xs, want) in tests {
            let mut got = xs.to_vec();
            reverse(&mut got);
            assert_eq!(got, want, "{name}: reverse({xs:?})");
        }
    }

    #[test]
    fn rotate_table() {
        let tests: [(&str, &[i64], i64, &[i64]); 6] = [
            ("k=0", &[1, 2, 3], 0, &[1, 2, 3]),
            ("k=1", &[1, 2, 3], 1, &[2, 3, 1]),
            ("k=len", &[1, 2, 3], 3, &[1, 2, 3]),
            ("k>len", &[1, 2, 3], 4, &[2, 3, 1]),
            ("k negative", &[1, 2, 3], -1, &[3, 1, 2]),
            ("empty", &[], 5, &[]),
        ];

        for (name, xs, k, want) in tests {
            let mut got = xs.to_vec();
            rotate(&mut got, k);
            assert_eq!(got, want, "{name}: rotate({xs:?}, {k})");
        }
    }

    // 2. The invariants: what must hold for EVERY input, checked over the edge cases.
    #[test]
    fn in_place_invariants() {
        check_in_place("reverse", reverse);
        check_in_place("rotate", |xs| rotate(xs, 3));
    }

    // 3. The oracle: agreement with a slow reference, over random input.
    #[test]
    fn reverse_against_oracle() {
        // The oracle: collect the elements backwards into a new Vec. Obviously
        // correct. `Vec<i64>` is already `PartialEq`, so unlike Go (where slices
        // are not comparable and the outputs went through fmt.Sprint) it can be
        // compared as it is.
        let oracle = |xs: &Vec<i64>| xs.iter().rev().copied().collect::<Vec<_>>();
        let candidate = |xs: &Vec<i64>| {
            let mut out = xs.clone();
            reverse(&mut out);
            out
        };

        differential(50_000, 1, |rng| random_slice(rng, 10, 6), oracle, candidate);
    }

    // 4. Reverse is its own inverse -- a property worth stating outright.
    #[test]
    fn reverse_twice_is_identity() {
        let mut rng = Rng::new(2);
        for _ in 0..20_000 {
            let original = random_slice(&mut rng, 12, 8);
            let mut got = original.clone();
            reverse(&mut got);
            reverse(&mut got);
            assert_eq!(got, original, "reversing twice changed {original:?}");
        }
    }
}

// ---- bench_test.go -- control + sweep: copy this too; change only the sweep body ----

// Stable Rust has no `testing.B` (`#[bench]` is nightly-only; the real tools are
// criterion and divan under `cargo bench`). So this is std-only: a counting
// allocator gives B/op and allocs/op, a calibrating loop gives ns/op, and with
// no Cargo to give benchmarks their own command, the sweep runs as a test.
#[cfg(test)]
mod bench {
    use super::*;
    use std::alloc::{GlobalAlloc, Layout, System};
    use std::cell::Cell;
    use std::hint::black_box;
    use std::io::Write;
    use std::time::{Duration, Instant};

    // Counts per THREAD: libtest runs the tests in parallel, and a global counter
    // would also count what the other tests allocate during a measurement.
    struct Counting;
    thread_local! {
        static ALLOCS: Cell<usize> = const { Cell::new(0) }; // allocation events (alloc + realloc)
        static BYTES: Cell<usize> = const { Cell::new(0) }; // bytes requested, summed over all events
    }

    fn count(size: usize) {
        ALLOCS.with(|c| c.set(c.get() + 1));
        BYTES.with(|c| c.set(c.get() + size));
    }

    unsafe impl GlobalAlloc for Counting {
        // `unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every method only
        // counts and then forwards to the system allocator unchanged.
        unsafe fn alloc(&self, l: Layout) -> *mut u8 {
            count(l.size());
            unsafe { System.alloc(l) }
        }
        unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
            unsafe { System.dealloc(p, l) }
        }
        unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
            count(new);
            unsafe { System.realloc(p, l, new) }
        }
    }
    #[global_allocator]
    static GLOBAL: Counting = Counting;

    /// (allocation events, bytes requested) on this thread while running `f`.
    fn allocs_of(f: impl FnOnce()) -> (usize, usize) {
        let (a0, b0) = (ALLOCS.with(Cell::get), BYTES.with(Cell::get));
        f();
        (ALLOCS.with(Cell::get) - a0, BYTES.with(Cell::get) - b0)
    }

    /// Calibrates like b.Loop: grows the iteration count until a run takes ~200 ms.
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

    // libtest swallows the println! of a PASSING test unless you pass
    // --show-output; writing to the real stdout skips that capture.
    fn row(name: &str, ns: f64, (allocs, bytes): (usize, usize)) {
        let _ = writeln!(
            std::io::stdout(),
            "{name:<24} {ns:>10.3} ns/op {bytes:>8} B/op {allocs:>3} allocs/op"
        );
    }

    #[test]
    fn benchmarks() {
        // Always present, always first. No benchmark below this floor means
        // anything. black_box keeps the optimizer from deleting the empty loop.
        let ns = ns_per_op(|| black_box(()));
        row("EmptyControl", ns, allocs_of(|| black_box(())));

        // The sweep: the same operation across growing n, so the GROWTH is visible.
        // Read the ns/op column, not any single row.
        for n in [16, 256, 4096, 65536] {
            let mut xs: Vec<i64> = (0..n).collect();
            let ns = ns_per_op(|| reverse(black_box(&mut xs)));
            row(
                &format!("Reverse/n={n}"),
                ns,
                allocs_of(|| reverse(black_box(&mut xs))),
            );
        }
    }
}
```

**Run it (Rust):**

```bash
rustfmt --edition 2024 --check reverse.rs                                 # gate 1: gofmt -> rustfmt (silent when clean)
clippy-driver --edition 2024 -O --test -D warnings reverse.rs -o reverse  # gate 2: go vet -> clippy (silent when clean)
rustc --edition 2024 -O --test reverse.rs -o reverse && ./reverse         # gates 3 + 5: the tests, then the benchmark sweep
# not run here: -race (a data race does not compile in safe Rust), cargo llvm-cov (coverage; not installed)
```

**Rust sample output:**

```

running 6 tests
test tests::reverse_table ... ok
test tests::rotate_table ... ok
test tests::in_place_invariants ... ok
test tests::reverse_twice_is_identity ... ok
test tests::reverse_against_oracle ... ok
EmptyControl                  0.253 ns/op        0 B/op   0 allocs/op
Reverse/n=16                  3.563 ns/op        0 B/op   0 allocs/op
Reverse/n=256                41.216 ns/op        0 B/op   0 allocs/op
Reverse/n=4096              565.072 ns/op        0 B/op   0 allocs/op
Reverse/n=65536           10902.348 ns/op        0 B/op   0 allocs/op
test bench::benchmarks ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.41s
```

> **Go vs Rust:** Plain `rustc` takes one file, so the folder's files are inline modules (separate files in a Cargo crate), and the benchmark sweep runs inside the test run because stable has no `testing.B`; `-O` matters, so a debug build would make the sweep meaningless. `cargo llvm-cov` is not installed, so there is no coverage line above; as a one-off check, a separate `-C instrument-coverage` build read with Apple's `llvm-cov` showed every line of `reverse` and `rotate` executed, and the only unexecuted lines in the file were failure branches of the harness itself (`differential`'s `panic!`, two `return false` in `is_permutation`). The sweep reports 0 allocations at every size and about 11x to 20x more time per 16x of input over five runs (not a clean 16x at every step), and because `Vec<i64>` is already `PartialEq` the oracle compares vectors directly, with no `fmt.Sprint` workaround.

**Takeaway:** this folder is the starting point for every lesson from 03 to 45. Copy it, replace the implementation, keep the gates green.

---

> That's the toolkit. Next: [03 — Complexity Analysis](../../03-complexity-analysis.md), where you'll
> use the sweep from example 8 to confirm the notation's claims empirically.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
