# Step 02 — Environment & Toolkit · 🟢 Easy

Examples **1–6**: the test file, the table, the comparison trap, the checklist, coverage, benchmarks.

> ⚠️ **These examples are `go test` packages, not `go run` programs.** Each one is a folder with two
> or three files. Create the folder, put the files in it, and run the command shown.

```bash
mkdir -p /tmp/dsa-t/ex01 && cd /tmp/dsa-t   # once: go mod init scratch
# put the files in ex01/, then:
go test -v ./ex01
```

Test output includes a wall-clock time on the `ok` line (`ok l02/ex01 0.546s`) — yours will differ.
Everything above that line is deterministic.

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

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. Your first algorithm test

`🟢 easy` · *Test mechanics*

The whole contract: a file ending in `_test.go`, in the same package as the code, holding functions
named `TestXxx(t *testing.T)`. No framework, no assertions library, no annotations. Note the
signature under test — an algorithm that can fail (`Max` of nothing) says so with a `bool` rather
than panicking or inventing a zero.

**Steps:**

1. Put the code and the test in the same package, in two files.
2. Use `t.Fatal` when nothing after it would make sense, `t.Errorf` when you want the rest to run.
3. Run with `-v` to see each test named.

**`max.go`**

```go
package algo

// Max returns the largest element of xs. The bool reports whether xs had any
// elements at all -- an algorithm that can fail must say so in its signature.
func Max(xs []int) (int, bool) {
	if len(xs) == 0 {
		return 0, false
	}
	best := xs[0]
	for _, v := range xs[1:] {
		if v > best {
			best = v
		}
	}
	return best, true
}
```

**`max_test.go`**

```go
package algo

import "testing"

// A test file lives beside the code, ends in _test.go, and shares the package.
// Every test is func TestXxx(t *testing.T).

func TestMax(t *testing.T) {
	got, ok := Max([]int{3, 9, 4})
	if !ok {
		// Fatal stops this test now: the value below would be meaningless.
		t.Fatal("Max([3 9 4]) returned ok=false for a non-empty slice")
	}
	if got != 9 {
		// Errorf records the failure but keeps going.
		t.Errorf("Max([3 9 4]) = %d, want 9", got)
	}
}

func TestMaxEmpty(t *testing.T) {
	if _, ok := Max(nil); ok {
		t.Error("Max(nil) reported ok=true, want false")
	}
}
```

**Run it:**

```bash
go test -v ./ex01
```

**Output:**

```
=== RUN   TestMax
--- PASS: TestMax (0.00s)
=== RUN   TestMaxEmpty
--- PASS: TestMaxEmpty (0.00s)
PASS
ok  	l02/ex01	0.546s
```

**Rust:**

```rust
// Rust has no separate test file: unit tests sit in the SAME file as the code,
// inside a `#[cfg(test)] mod tests` that exists only in a test build
// (`rustc --test`, or `cargo test`). Every test is a plain `#[test] fn` --
// no framework, no assertions library.

/// Returns the largest element of `xs`. `None` means `xs` had no elements at
/// all -- an algorithm that can fail must say so in its signature, and in Rust
/// the type does it: there is no zero to confuse with a real answer.
fn max(xs: &[i64]) -> Option<i64> {
    let (&first, rest) = xs.split_first()?;
    let mut best = first;
    for &v in rest {
        if v > best {
            best = v;
        }
    }
    Some(best)
}

#[cfg(test)]
mod tests {
    // The test module is a child of the module under test, so `super::*`
    // reaches everything in it -- private items included.
    use super::*;

    #[test]
    fn max_of_three() {
        // expect() is Go's t.Fatal: it stops this test now, because the value
        // below would be meaningless.
        let got = max(&[3, 9, 4]).expect("max([3, 9, 4]) returned None for a non-empty slice");
        // Rust has no t.Errorf: a failed assert_eq! also ends the test.
        assert_eq!(got, 9, "max([3, 9, 4])");
    }

    #[test]
    fn max_of_nothing() {
        assert_eq!(
            max(&[]),
            None,
            "max([]) must report that there is no maximum"
        );
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test max.rs -o max && ./max
```

**Rust sample output:**

```

running 2 tests
test tests::max_of_three ... ok
test tests::max_of_nothing ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** Rust keeps unit tests in the same file, in a `#[cfg(test)] mod tests` that is a child of the code it tests, so it sees private items just as the Go test shares the package. `Option<i64>` replaces the `(int, bool)` pair, and std has no `t.Errorf`: a failed assertion panics and ends that test. libtest prints its own summary line with a timing, so the output is a sample.

**Takeaway:** tests are ordinary Go code in the same package — which is why they can reach the unexported fields your data structures will keep their invariants in.

---

## 2. The table-driven form

`🟢 easy` · *Table + subtests*

The default shape for every test in this plan. One slice of cases, one loop, one `t.Run` per case.
Three payoffs: adding a case is adding a line, each case reports its own name on failure, and you can
re-run exactly one case with `-run` while debugging.

**Steps:**

1. Declare an anonymous struct slice with a `name` field and the inputs/outputs.
2. Loop, calling `t.Run(tt.name, ...)`.
3. Re-run a single case with `-run 'TestMax/negatives'` — note that spaces in names become underscores.

**`max.go`**

```go
package algo

// Max returns the largest element of xs, and false if xs is empty.
func Max(xs []int) (int, bool) {
	if len(xs) == 0 {
		return 0, false
	}
	best := xs[0]
	for _, v := range xs[1:] {
		if v > best {
			best = v
		}
	}
	return best, true
}
```

**`max_test.go`**

```go
package algo

import "testing"

// The table-driven form: one slice of cases, one loop, one t.Run per case.
// Adding a case is adding a line -- which is the whole point.
func TestMax(t *testing.T) {
	tests := []struct {
		name   string
		xs     []int
		want   int
		wantOK bool
	}{
		{"empty", nil, 0, false},
		{"single", []int{7}, 7, true},
		{"ascending", []int{1, 2, 3}, 3, true},
		{"descending", []int{3, 2, 1}, 3, true},
		{"max in middle", []int{1, 9, 2}, 9, true},
		{"duplicates", []int{4, 4, 4}, 4, true},
		{"negatives", []int{-5, -2, -9}, -2, true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, ok := Max(tt.xs)
			if ok != tt.wantOK {
				t.Fatalf("Max(%v) ok = %v, want %v", tt.xs, ok, tt.wantOK)
			}
			if ok && got != tt.want {
				t.Errorf("Max(%v) = %d, want %d", tt.xs, got, tt.want)
			}
		})
	}
}
```

**Run it:**

```bash
go test -v ./ex02
```

**Output:**

```
=== RUN   TestMax
=== RUN   TestMax/empty
=== RUN   TestMax/single
=== RUN   TestMax/ascending
=== RUN   TestMax/descending
=== RUN   TestMax/max_in_middle
=== RUN   TestMax/duplicates
=== RUN   TestMax/negatives
--- PASS: TestMax (0.00s)
    --- PASS: TestMax/empty (0.00s)
    --- PASS: TestMax/single (0.00s)
    --- PASS: TestMax/ascending (0.00s)
    --- PASS: TestMax/descending (0.00s)
    --- PASS: TestMax/max_in_middle (0.00s)
    --- PASS: TestMax/duplicates (0.00s)
    --- PASS: TestMax/negatives (0.00s)
PASS
ok  	l02/ex02	0.289s
```

**Then run one case only:**

```bash
go test -v -run 'TestMax/negatives' ./ex02
```

```
=== RUN   TestMax
=== RUN   TestMax/negatives
--- PASS: TestMax (0.00s)
    --- PASS: TestMax/negatives (0.00s)
PASS
ok  	l02/ex02	0.135s
```

**Rust:**

```rust
/// Returns the largest element of `xs`, and `None` if `xs` is empty.
fn max(xs: &[i64]) -> Option<i64> {
    let (&first, rest) = xs.split_first()?;
    let mut best = first;
    for &v in rest {
        if v > best {
            best = v;
        }
    }
    Some(best)
}

#[cfg(test)]
mod tests {
    use super::*;

    // libtest has no t.Run subtests, so "one named test per case" is built with
    // a small macro: every row of the table below expands into its own
    // `#[test] fn`. Adding a case is adding a line -- which is the whole point --
    // and each case reports its own name and can be re-run alone by filtering.
    macro_rules! max_cases {
        ($($name:ident: $xs:expr => $want:expr,)*) => {
            $(
                #[test]
                fn $name() {
                    let xs: &[i64] = &$xs;
                    assert_eq!(max(xs), $want, "max({xs:?})");
                }
            )*
        };
    }

    max_cases! {
        empty: [] => None,
        single: [7] => Some(7),
        ascending: [1, 2, 3] => Some(3),
        descending: [3, 2, 1] => Some(3),
        max_in_middle: [1, 9, 2] => Some(9),
        duplicates: [4, 4, 4] => Some(4),
        negatives: [-5, -2, -9] => Some(-2),
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test max.rs -o max && ./max
```

**Rust sample output:**

```

running 7 tests
test tests::empty ... ok
test tests::ascending ... ok
test tests::max_in_middle ... ok
test tests::descending ... ok
test tests::duplicates ... ok
test tests::negatives ... ok
test tests::single ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** libtest has no `t.Run` subtests, so a small `macro_rules!` expands each row of the table into its own `#[test]`; that is what makes a case individually named and runnable: `./max negatives` runs only that test (the summary then reads `1 passed; ... 6 filtered out`). The seven tests run in parallel, so their lines can appear in any order; a plain array-plus-loop table (used in examples 3 and 4) is shorter but cannot be filtered per case.

**Takeaway:** `-run 'TestName/case_name'` is the debugging loop — one case, one second, no noise.

---

## 3. nil vs empty, and how to compare slices

`🟢 easy` · *slices.Equal vs reflect.DeepEqual*

The trap that will bite you in half the lessons ahead. `var out []int` is **nil** until the first
`append`, so a filter that matches nothing returns nil, not `[]int{}`. Whether your test passes then
depends entirely on which comparison you reached for — and the popular one gets it wrong.

**Steps:**

1. Confirm the "no matches" return really is nil, and that its length is 0.
2. Check it against `[]int{}` with `slices.Equal` (contents) and `reflect.DeepEqual` (representation).
3. Note that the table below never has to decide between `nil` and `[]int{}` — `slices.Equal` doesn't care.

**`filter.go`**

```go
package algo

// Evens returns the even numbers in xs, in order.
//
// Note what it returns when nothing matches: a nil slice, because `var out []int`
// is nil until the first append. That is idiomatic Go -- and it is the single
// most common source of confusing test failures in this whole plan.
func Evens(xs []int) []int {
	var out []int
	for _, v := range xs {
		if v%2 == 0 {
			out = append(out, v)
		}
	}
	return out
}
```

**`filter_test.go`**

```go
package algo

import (
	"reflect"
	"slices"
	"testing"
)

// You cannot compare slices with == in Go. There are two real choices, and they
// disagree about exactly one thing: whether nil and empty are the same value.

func TestNilVersusEmpty(t *testing.T) {
	got := Evens([]int{1, 3, 5}) // nothing matches

	if got != nil {
		t.Errorf("expected a nil slice, got %#v", got)
	}
	if len(got) != 0 {
		t.Errorf("expected length 0, got %d", len(got))
	}

	// slices.Equal compares CONTENTS: nil and empty are both "no elements".
	if !slices.Equal(got, []int{}) {
		t.Error("slices.Equal(nil, []int{}) = false, want true")
	}

	// reflect.DeepEqual compares REPRESENTATION: nil != empty.
	if reflect.DeepEqual(got, []int{}) {
		t.Error("reflect.DeepEqual(nil, []int{}) = true, want false")
	}
}

// So: use slices.Equal for algorithm output, unless nil-ness is part of the
// contract you are actually testing.
func TestEvens(t *testing.T) {
	tests := []struct {
		name string
		xs   []int
		want []int
	}{
		{"nil input", nil, nil},
		{"no matches", []int{1, 3}, nil},
		{"all match", []int{2, 4}, []int{2, 4}},
		{"mixed", []int{1, 2, 3, 4}, []int{2, 4}},
		{"negatives", []int{-2, -3}, []int{-2}},
		{"zero is even", []int{0, 1}, []int{0}},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// slices.Equal treats want=nil and want=[]int{} identically, so the
			// table above does not have to care which one it wrote.
			if got := Evens(tt.xs); !slices.Equal(got, tt.want) {
				t.Errorf("Evens(%v) = %v, want %v", tt.xs, got, tt.want)
			}
		})
	}
}
```

**Run it:**

```bash
go test -v ./ex03
```

**Output:**

```
=== RUN   TestNilVersusEmpty
--- PASS: TestNilVersusEmpty (0.00s)
=== RUN   TestEvens
=== RUN   TestEvens/nil_input
=== RUN   TestEvens/no_matches
=== RUN   TestEvens/all_match
=== RUN   TestEvens/mixed
=== RUN   TestEvens/negatives
=== RUN   TestEvens/zero_is_even
--- PASS: TestEvens (0.00s)
    --- PASS: TestEvens/nil_input (0.00s)
    --- PASS: TestEvens/no_matches (0.00s)
    --- PASS: TestEvens/all_match (0.00s)
    --- PASS: TestEvens/mixed (0.00s)
    --- PASS: TestEvens/negatives (0.00s)
    --- PASS: TestEvens/zero_is_even (0.00s)
PASS
ok  	l02/ex03	0.285s
```

**Rust:**

```rust
/// Returns the even numbers in `xs`, in order.
///
/// Note what it returns when nothing matches: an empty Vec that has never
/// allocated (capacity 0). Rust has no nil slice, so this is as close as it
/// gets to Go's "nil until the first append" -- and, unlike Go, it is not a
/// trap, because nothing in the language can tell it apart from any other
/// empty Vec.
fn evens(xs: &[i64]) -> Vec<i64> {
    xs.iter().copied().filter(|v| v % 2 == 0).collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    // In Rust `==` already works on Vec and slices, and it compares CONTENTS --
    // that is Go's slices.Equal. There is no reflect.DeepEqual counterpart
    // that looks at representation, so nil-versus-empty cannot be asserted
    // here; the only difference left to observe is capacity.
    #[test]
    fn empty_is_empty() {
        let got = evens(&[1, 3, 5]); // nothing matches

        assert!(got.is_empty());
        assert_eq!(
            got.capacity(),
            0,
            "no match: nothing was ever pushed, so nothing was allocated"
        );

        // Same contents -> equal, whatever the capacity.
        assert_eq!(got, Vec::<i64>::new());
        assert_eq!(got, Vec::<i64>::with_capacity(16));

        // If "no answer" must differ from "an empty answer", say so in the
        // type: Option<Vec<_>> keeps them apart, and `==` then agrees.
        let no_answer: Option<Vec<i64>> = None;
        let empty_answer: Option<Vec<i64>> = Some(Vec::new());
        assert_ne!(no_answer, empty_answer);
    }

    // So: compare with `==` (assert_eq!) for algorithm output.
    #[test]
    fn evens_table() {
        // (name, input, want). The slices are `&[i64]`, so a case never has to
        // decide between "nil" and "empty": `&[]` is the only way to write it.
        let tests: &[(&str, &[i64], &[i64])] = &[
            ("empty input", &[], &[]),
            ("no matches", &[1, 3], &[]),
            ("all match", &[2, 4], &[2, 4]),
            ("mixed", &[1, 2, 3, 4], &[2, 4]),
            ("negatives", &[-2, -3], &[-2]),
            ("zero is even", &[0, 1], &[0]),
        ];

        for &(name, xs, want) in tests {
            assert_eq!(evens(xs), want, "{name}: evens({xs:?})");
        }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test filter.rs -o filter && ./filter
```

**Rust sample output:**

```

running 2 tests
test tests::evens_table ... ok
test tests::empty_is_empty ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** Rust has no nil slice, so the trap does not exist: `==` on `Vec`/slices compares contents like `slices.Equal`, and nothing plays the role of `reflect.DeepEqual`. The nearest thing to Go's "nil until the first append" is a `Vec` with capacity 0 (asserted above); if "no answer" must differ from "empty answer", the type to use is `Option<Vec<T>>`.

**Takeaway:** `slices.Equal` for contents. Reach for `reflect.DeepEqual` only when nil-vs-empty is the thing you mean to assert — and then say so in a comment.

---

## 4. The edge-case checklist

`🟢 easy` · *What to test before the interesting case*

Most algorithm bugs are not in the clever part. They're at the boundary: the empty input, the
single-element input, the two-element input, the all-duplicates input. This is the checklist every
sequence algorithm in this plan starts from, grouped so you can see what each group probes.

**Steps:**

1. Group the table by what it stresses: the empty/tiny boundary, degenerate content, ordering, value range.
2. Include `math.MaxInt` and `math.MinInt` — overflow bugs hide nowhere else.
3. `t.Fatalf` on the `ok` mismatch so a bad case doesn't cascade into confusing follow-up errors.

**`minmax.go`**

```go
package algo

// MinMax returns the smallest and largest elements of xs in a single pass.
func MinMax(xs []int) (min, max int, ok bool) {
	if len(xs) == 0 {
		return 0, 0, false
	}
	min, max = xs[0], xs[0]
	for _, v := range xs[1:] {
		if v < min {
			min = v
		}
		if v > max {
			max = v
		}
	}
	return min, max, true
}
```

**`minmax_test.go`**

```go
package algo

import (
	"math"
	"testing"
)

// The checklist. Every sequence algorithm in this plan gets these cases, in this
// order, before any "interesting" case is written. Most algorithm bugs live in
// the first four rows -- not in the logic you were concentrating on.
func TestMinMax(t *testing.T) {
	tests := []struct {
		name    string
		xs      []int
		wantMin int
		wantMax int
		wantOK  bool
	}{
		// --- the empty/tiny boundary ---
		{"nil", nil, 0, 0, false},
		{"empty", []int{}, 0, 0, false},
		{"single", []int{5}, 5, 5, true},
		{"two ascending", []int{1, 2}, 1, 2, true},
		{"two descending", []int{2, 1}, 1, 2, true},

		// --- degenerate content ---
		{"all equal", []int{3, 3, 3}, 3, 3, true},
		{"duplicates", []int{2, 5, 2, 5}, 2, 5, true},

		// --- ordering ---
		{"already sorted", []int{1, 2, 3, 4}, 1, 4, true},
		{"reverse sorted", []int{4, 3, 2, 1}, 1, 4, true},
		{"min at end", []int{5, 6, 0}, 0, 6, true},
		{"max at end", []int{5, 1, 9}, 1, 9, true},

		// --- value range ---
		{"negatives", []int{-3, -1, -7}, -7, -1, true},
		{"crosses zero", []int{-1, 0, 1}, -1, 1, true},
		{"extremes", []int{math.MaxInt, math.MinInt}, math.MinInt, math.MaxInt, true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			min, max, ok := MinMax(tt.xs)
			if ok != tt.wantOK {
				t.Fatalf("MinMax(%v) ok = %v, want %v", tt.xs, ok, tt.wantOK)
			}
			if !ok {
				return
			}
			if min != tt.wantMin || max != tt.wantMax {
				t.Errorf("MinMax(%v) = (%d, %d), want (%d, %d)", tt.xs, min, max, tt.wantMin, tt.wantMax)
			}
		})
	}
}
```

**Run it:**

```bash
go test ./ex04
```

**Output:**

```
ok  	l02/ex04	0.308s
```

**Rust:**

```rust
/// Returns the smallest and largest elements of `xs` in a single pass.
fn min_max(xs: &[i64]) -> Option<(i64, i64)> {
    let (&first, rest) = xs.split_first()?;
    let (mut min, mut max) = (first, first);
    for &v in rest {
        if v < min {
            min = v;
        }
        if v > max {
            max = v;
        }
    }
    Some((min, max))
}

#[cfg(test)]
mod tests {
    use super::*;

    // The checklist. Every sequence algorithm in this plan gets these cases, in
    // this order, before any "interesting" case is written. Most algorithm bugs
    // live in the first four rows -- not in the logic you were concentrating on.
    #[test]
    fn min_max_checklist() {
        // (name, input, want). Rust has no nil slice, so Go's "nil" and "empty"
        // rows are one row here.
        type Want = Option<(i64, i64)>;
        let tests: &[(&str, &[i64], Want)] = &[
            // --- the empty/tiny boundary ---
            ("empty", &[], None),
            ("single", &[5], Some((5, 5))),
            ("two ascending", &[1, 2], Some((1, 2))),
            ("two descending", &[2, 1], Some((1, 2))),
            // --- degenerate content ---
            ("all equal", &[3, 3, 3], Some((3, 3))),
            ("duplicates", &[2, 5, 2, 5], Some((2, 5))),
            // --- ordering ---
            ("already sorted", &[1, 2, 3, 4], Some((1, 4))),
            ("reverse sorted", &[4, 3, 2, 1], Some((1, 4))),
            ("min at end", &[5, 6, 0], Some((0, 6))),
            ("max at end", &[5, 1, 9], Some((1, 9))),
            // --- value range ---
            ("negatives", &[-3, -1, -7], Some((-7, -1))),
            ("crosses zero", &[-1, 0, 1], Some((-1, 1))),
            (
                "extremes",
                &[i64::MAX, i64::MIN],
                Some((i64::MIN, i64::MAX)),
            ),
        ];

        // One assert_eq! covers both "was there an answer" and "which one", so
        // a missing answer cannot cascade into a confusing second failure.
        for &(name, xs, want) in tests {
            assert_eq!(min_max(xs), want, "{name}: min_max({xs:?})");
        }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test minmax.rs -o minmax && ./minmax
```

**Rust sample output:**

```

running 1 test
test tests::min_max_checklist ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** `Option<(i64, i64)>` replaces `(min, max, ok)`, so the "was there an answer" check and the value check are one `assert_eq!`, and Go's `nil` and `empty` rows collapse into one because Rust has no nil slice (13 rows, not 14). This is one test looping over the table, so the first failing row ends it, where Go's subtests each report separately; and a binary built without `-O`, as in the command above, has overflow checks on, so an overflow bug at the `extremes` row panics instead of wrapping silently.

**Takeaway:** the checklist is: **nil · empty · single · two · all-equal · duplicates · sorted · reverse-sorted · negatives · extremes**. Write it before you write the interesting cases.

---

## 5. 100% coverage, still wrong

`🟢 easy` · *go test -cover*

This is the example to remember. Below is a binary search with the most common binary-search bug
there is, and a table of four sensible-looking cases. Every case passes. `go test -cover` reports
**100.0% of statements**. The function is still broken — it cannot find the first or last element of
a slice, because the loop exits before checking a range that has narrowed to one.

An 8-line oracle over a tiny domain catches it instantly.

**Steps:**

1. Read `search.go` and try to spot the bug before reading on.
2. Run the table with `-cover` — note 100.0%, all green.
3. Run the oracle test and read the two counterexamples it prints.

**`search.go`**

```go
package algo

// Search returns the index of target in the sorted slice xs, or -1.
//
// This implementation has a bug. It is the single most common binary-search bug
// there is, and the tests in search_test.go do not catch it -- while reporting
// 100% statement coverage.
func Search(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo < hi { // <-- the bug lives here
		mid := (lo + hi) / 2
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
```

**`search_test.go`**

```go
package algo

import "testing"

// A perfectly reasonable-looking table. Every branch of Search runs at least
// once, so `go test -cover` reports 100.0% -- and every case passes.
func TestSearchTable(t *testing.T) {
	xs := []int{1, 3, 5, 7}

	tests := []struct {
		name   string
		target int
		want   int
	}{
		{"hits mid immediately", 3, 1},
		{"found after moving lo", 5, 2},
		{"absent above range", 9, -1},
		{"absent below range", 0, -1},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Search(xs, tt.target); got != tt.want {
				t.Errorf("Search(%v, %d) = %d, want %d", xs, tt.target, got, tt.want)
			}
		})
	}
}
```

**`oracle_test.go`**

```go
package algo

import "testing"

// linearSearch is the oracle: too slow to ship, too simple to be wrong.
func linearSearch(xs []int, target int) int {
	for i, v := range xs {
		if v == target {
			return i
		}
	}
	return -1
}

// Exhaustive over a tiny domain -- no randomness needed. Every element of the
// slice, plus every value that isn't in it.
func TestSearchAgainstOracle(t *testing.T) {
	xs := []int{1, 3, 5, 7}

	for target := 0; target <= 8; target++ {
		want := linearSearch(xs, target)
		if got := Search(xs, target); got != want {
			t.Errorf("Search(%v, %d) = %d, oracle says %d", xs, target, got, want)
		}
	}
}
```

**Run the table with coverage:**

```bash
go test -cover -run 'TestSearchTable' ./ex05
```

```
ok  	l02/ex05	0.344s	coverage: 100.0% of statements
```

**Now run the oracle:**

```bash
go test -run 'TestSearchAgainstOracle' ./ex05
```

```
--- FAIL: TestSearchAgainstOracle (0.00s)
    oracle_test.go:23: Search([1 3 5 7], 1) = -1, oracle says 0
    oracle_test.go:23: Search([1 3 5 7], 7) = -1, oracle says 3
FAIL
FAIL	l02/ex05	0.283s
FAIL
```

The bug: `for lo < hi` exits when the range narrows to exactly one element, without ever comparing it.
So the first and last elements — the ones a binary search reaches last — are never found. The fix is
`for lo <= hi`. The four table cases all happened to terminate before that point.

**Rust:**

```rust
use std::cmp::Ordering;

/// Returns the index of `target` in the sorted slice `xs`, or `None`.
///
/// This implementation has a bug. It is the single most common binary-search
/// bug there is, and the tests in `search_table` do not catch it -- while
/// a coverage tool would report every statement as run.
fn search(xs: &[i64], target: i64) -> Option<usize> {
    // isize, not usize: `hi = mid - 1` reaches -1 for a target below xs[0],
    // and a usize would underflow there.
    let (mut lo, mut hi) = (0isize, xs.len() as isize - 1);
    while lo < hi {
        // <-- the bug lives here
        let mid = (lo + hi) / 2;
        let at = mid as usize;
        match xs[at].cmp(&target) {
            Ordering::Equal => return Some(at),
            Ordering::Less => lo = mid + 1,
            Ordering::Greater => hi = mid - 1,
        }
    }
    None
}

#[cfg(test)]
mod tests {
    use super::*;

    // A perfectly reasonable-looking table. Every branch of `search` runs at
    // least once, so statement coverage would be 100% -- and every case passes.
    #[test]
    fn search_table() {
        let xs = [1, 3, 5, 7];
        // (name, target, want)
        let tests = [
            ("hits mid immediately", 3, Some(1)),
            ("found after moving lo", 5, Some(2)),
            ("absent above range", 9, None),
            ("absent below range", 0, None),
        ];

        for (name, target, want) in tests {
            assert_eq!(
                search(&xs, target),
                want,
                "{name}: search({xs:?}, {target})"
            );
        }
    }

    // linear_search is the oracle: too slow to ship, too simple to be wrong.
    fn linear_search(xs: &[i64], target: i64) -> Option<usize> {
        xs.iter().position(|&v| v == target)
    }

    // Exhaustive over a tiny domain -- no randomness needed. Every element of
    // the slice, plus every value that isn't in it.
    //
    // In the Go version this test FAILS, and that is the point. Here it is
    // marked #[should_panic] with the exact counterexamples, so the file can be
    // shown running green; delete the attribute and it fails like the Go one.
    #[test]
    #[should_panic(expected = "search([1, 3, 5, 7], 1) = None, oracle says Some(0)\n\
                    search([1, 3, 5, 7], 7) = None, oracle says Some(3)")]
    fn search_against_oracle() {
        let xs = [1, 3, 5, 7];

        let mut failures = Vec::new();
        for target in 0..=8 {
            let (got, want) = (search(&xs, target), linear_search(&xs, target));
            if got != want {
                failures.push(format!(
                    "search({xs:?}, {target}) = {got:?}, oracle says {want:?}"
                ));
            }
        }
        assert!(failures.is_empty(), "{}", failures.join("\n"));
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 --test search.rs -o search && ./search
```

**Rust sample output:**

```

running 2 tests
test tests::search_table ... ok
test tests::search_against_oracle - should panic ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> **Go vs Rust:** Both Go commands become one test binary: `search_against_oracle` is marked `#[should_panic]` with both counterexamples spelled out so the file runs green; without the attribute it fails exactly like the Go run (exit code 101), and with `lo <= hi` it passes. `go test -cover` has no std counterpart; `cargo llvm-cov` is the usual tool and is not installed here, so no coverage figure appears above (a separate `-C instrument-coverage` build read with Apple's `llvm-cov` showed every line of `search` executed by the table test, which is the 100% the Go output reports). "Not found" is `None` instead of `-1`, hence `None`/`Some(0)` in the counterexamples.

**Takeaway:** coverage answers "which lines ran", never "were the assertions right". A 4-case table can be 100% covered and 0% correct.

---

## 6. Benchmarks in a test file

`🟢 easy` · *go test -bench -benchmem*

Benchmarks live in `_test.go` files too, as `func BenchmarkXxx(b *testing.B)`. Two flags matter from
day one: `-run='^$'` (match no test, so only benchmarks run) and `-benchmem` (add the allocation
columns). `b.Loop` is the Go 1.24+ loop form — it calibrates the iteration count and keeps the call
from being optimized away.

**Steps:**

1. Write two benchmarks: one that allocates nothing, one that allocates a lot.
2. Run with `-bench=. -benchmem -run='^$'`.
3. Read all five columns of the output.

**`sum.go`**

```go
package algo

// Sum adds every element of xs.
func Sum(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

// Collect returns a new slice holding the first n integers, without telling
// append how big the result will be.
func Collect(n int) []int {
	var out []int
	for i := 0; i < n; i++ {
		out = append(out, i)
	}
	return out
}
```

**`sum_test.go`**

```go
package algo

import "testing"

// Benchmarks live in _test.go files too, as func BenchmarkXxx(b *testing.B).

var benchInput = Collect(1000)

func BenchmarkSum(b *testing.B) {
	// b.Loop (Go 1.24+) calibrates the iteration count and keeps the call alive.
	for b.Loop() {
		Sum(benchInput)
	}
}

// b.ReportAllocs adds the B/op and allocs/op columns for this benchmark even
// without the -benchmem flag.
func BenchmarkCollect(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		Collect(1000)
	}
}
```

**Run it:**

```bash
go test -bench=. -benchmem -run='^$' ./ex06
```

**Sample output** (timings vary):

```
goos: darwin
goarch: arm64
pkg: l02/ex06
cpu: Apple M4
BenchmarkSum-10        	 4499440	       242.6 ns/op	       0 B/op	       0 allocs/op
BenchmarkCollect-10    	  658558	      1815 ns/op	   25208 B/op	      12 allocs/op
PASS
ok  	l02/ex06	2.640s
```

Reading a row: `BenchmarkSum-10` is the name plus `GOMAXPROCS`; `4499440` is how many iterations it
ran to get a stable measurement; then time per operation, bytes allocated per operation, and number of
allocations per operation. `Collect` needs **12 allocations** to build a 1000-element slice it could
have built in 1 — that's the growth story, measured.

**Rust:**

```rust
// Stable Rust has no built-in benchmark runner (`#[bench]` is nightly-only; the
// real tools are criterion or divan under `cargo bench`, which are crates). So
// this is a std-only harness in an ordinary `main`: a counting allocator gives
// B/op and allocs/op (Go's -benchmem), a calibrating loop gives ns/op (b.Loop).
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- the code under measurement ----

/// Adds every element of `xs`.
fn sum(xs: &[i64]) -> i64 {
    let mut total = 0;
    for &v in xs {
        total += v;
    }
    total
}

/// Returns a new Vec holding the first `n` integers, without telling the Vec
/// how big the result will be.
fn collect(n: i64) -> Vec<i64> {
    let mut out = Vec::new();
    for i in 0..n {
        out.push(i);
    }
    out
}

/// Same result, but the capacity is reserved up front.
fn collect_with_capacity(n: i64) -> Vec<i64> {
    let mut out = Vec::with_capacity(n as usize);
    for i in 0..n {
        out.push(i);
    }
    out
}

// ---- allocation counter: stand-in for -benchmem / b.ReportAllocs() ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static BYTES: AtomicUsize = AtomicUsize::new(0); // bytes requested, summed over all events

unsafe impl GlobalAlloc for Counting {
    // `unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every method only
    // counts and then forwards to the system allocator unchanged.
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested) for ONE call of `f`.
fn allocs_of<R>(f: &mut impl FnMut() -> R) -> (usize, usize) {
    let (a0, b0) = (ALLOCS.load(Relaxed), BYTES.load(Relaxed));
    drop(black_box(f()));
    (ALLOCS.load(Relaxed) - a0, BYTES.load(Relaxed) - b0)
}

/// Calibrates like b.Loop: grows the iteration count until a run takes
/// ~200 ms. Returns (iterations of the final run, ns per iteration).
fn ns_per_op(mut f: impl FnMut()) -> (u64, f64) {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return (n, el.as_nanos() as f64 / n as f64);
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

/// One benchmark row: name, iterations, ns/op, B/op, allocs/op.
fn bench<R>(name: &str, mut f: impl FnMut() -> R) {
    let (allocs, bytes) = allocs_of(&mut f);
    let (iters, ns) = ns_per_op(|| drop(black_box(f())));
    println!("{name:<22}{iters:>10}{ns:>12.1} ns/op{bytes:>8} B/op{allocs:>4} allocs/op");
}

fn main() {
    println!("os: {}", std::env::consts::OS);
    println!("arch: {}", std::env::consts::ARCH);

    let input = collect(1000);
    bench("sum", || sum(black_box(&input)));
    bench("collect", || collect(black_box(1000)));
    bench("collect_with_capacity", || {
        collect_with_capacity(black_box(1000))
    });
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O sum.rs -o sum && ./sum
```

**Rust sample output:**

```
os: macos
arch: aarch64
sum                      4292027        56.0 ns/op       0 B/op   0 allocs/op
collect                   363817       609.3 ns/op   16352 B/op   9 allocs/op
collect_with_capacity     862094       281.4 ns/op    8000 B/op   1 allocs/op
```

> **Go vs Rust:** Stable Rust has no `testing.B`: `#[bench]` is nightly-only and `cargo bench` on stable needs a crate such as criterion or divan (not run here), so this is a std-only harness (`ns_per_op` plus a counting allocator) in an ordinary `main`. The allocation columns are deterministic: `Vec::push` from empty took 9 allocation events and 16,352 bytes (capacities 4, 8, ... 1024, i.e. 8 x 2044 bytes) where Go's `append` took 12 and 25,208, and reserving capacity first took 1 event and 8,000 bytes and ran about 2x faster in each of three runs. I do not compare the ns/op column with Go's row: it is a different run and compiler.

**Takeaway:** `-run='^$'` skips tests, `-benchmem` adds the two columns that catch the mistakes Big-O can't see.

---

> Next tier: [🟡 medium](2-medium.md) — controls, sweeps, benchstat, `-race`, fuzzing.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
