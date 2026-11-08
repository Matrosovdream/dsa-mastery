# Step 06 — Dynamic Arrays & Slices · 🟡 Medium

Examples **7–11**: building `Vector[T]`, deriving the amortized bound, and choosing a growth policy.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 7, 8 and 9 **count** rather than time, so they're deterministic. Examples 10 and 11 measure —
sample output from an Apple M4, Go 1.26.3.

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

## 7. Building `Vector[T]`, and deriving amortized Θ(1)

`🟡 medium` · *The resize math*

Lesson 03 told you `append` is amortized Θ(1). Now derive it: build the structure, count every element
copied, and watch copies-per-push converge to a constant.

Then break it — grow by a fixed `+k` instead of a factor — and watch the same number climb without bound.

**Steps:**

1. Implement `Vector[T]` with `Push`/`Pop` and a doubling `grow`.
2. Count total elements copied for n from 16 to 1,048,576.
3. Implement grow-by-+64 and compare.

```go
package main

import "fmt"

// Time to build the thing Go hands you for free, so the amortized-O(1) claim
// from lesson 03 stops being a fact you were told and becomes one you derived.
//
// A dynamic array is three fields and one rule:
//
//   data []T   the backing array, with spare room at the end
//   len  int   how many slots are in use
//   cap  int   how many exist
//
//   RULE: when len == cap, allocate a bigger array and copy. The new size must
//         be a constant FACTOR bigger, never a constant amount bigger.

type Vector[T any] struct {
	data   []T
	length int

	// instrumentation, so we can see what it did
	grows  int
	copied int
}

func (v *Vector[T]) Len() int { return v.length }
func (v *Vector[T]) Cap() int { return len(v.data) }

func (v *Vector[T]) At(i int) T {
	if i < 0 || i >= v.length {
		panic("index out of range")
	}
	return v.data[i]
}

func (v *Vector[T]) grow() {
	newCap := len(v.data) * 2
	if newCap == 0 {
		newCap = 1
	}
	bigger := make([]T, newCap)
	copy(bigger, v.data[:v.length])

	v.grows++
	v.copied += v.length
	v.data = bigger
}

func (v *Vector[T]) Push(x T) {
	if v.length == len(v.data) {
		v.grow()
	}
	v.data[v.length] = x
	v.length++
}

func (v *Vector[T]) Pop() (T, bool) {
	var zero T
	if v.length == 0 {
		return zero, false
	}
	v.length--
	x := v.data[v.length]
	v.data[v.length] = zero // release any pointer inside T (example 2)
	return x, true
}

// growPlusK is the policy people reach for when they think doubling is wasteful.
type growPlusK struct {
	data   []int
	length int
	k      int
	grows  int
	copied int
}

func (v *growPlusK) Push(x int) {
	if v.length == len(v.data) {
		bigger := make([]int, len(v.data)+v.k)
		copy(bigger, v.data)
		v.grows++
		v.copied += v.length
		v.data = bigger
	}
	v.data[v.length] = x
	v.length++
}

func main() {
	fmt.Println("pushing 16 values into a Vector[int], watching it grow:")
	fmt.Println()

	var v Vector[int]
	fmt.Printf("  %6s %6s %6s %8s\n", "push", "len", "cap", "copied")
	for i := 1; i <= 16; i++ {
		before := v.Cap()
		v.Push(i)
		marker := ""
		if v.Cap() != before {
			marker = "  <- grew"
		}
		if i <= 5 || v.Cap() != before {
			fmt.Printf("  %6d %6d %6d %8d%s\n", i, v.Len(), v.Cap(), v.copied, marker)
		}
	}

	fmt.Println()
	fmt.Println("the amortized argument, counted rather than asserted:")
	fmt.Println()
	fmt.Printf("  %10s %10s %14s %16s\n", "n pushes", "grows", "total copied", "copies/push")
	for _, n := range []int{16, 256, 4096, 65536, 1_048_576} {
		var w Vector[int]
		for i := 0; i < n; i++ {
			w.Push(i)
		}
		fmt.Printf("  %10d %10d %14d %16.2f\n", n, w.grows, w.copied, float64(w.copied)/float64(n))
	}

	fmt.Println()
	fmt.Println("copies/push converges to 1 and stops there. THAT is amortized Theta(1):")
	fmt.Println()
	fmt.Println("  the copies form a geometric series. Doubling from 1 to n copies")
	fmt.Println("  1 + 2 + 4 + ... + n/2 = n - 1 elements in total, for n pushes.")
	fmt.Println("  With a growth factor g the sum is n/(g-1), which is a CONSTANT")
	fmt.Println("  multiple of n for any fixed g > 1. (Go's g tapers to ~1.25 for")
	fmt.Println("  large slices, giving 1/(1.25-1) = 4 -- lesson 03, example 7.)")

	fmt.Println()
	fmt.Println("now the policy that seems more frugal -- grow by a fixed +k:")
	fmt.Println()
	fmt.Printf("  %10s %14s %16s %14s %16s\n", "n", "+64 copied", "+64 per push", "x2 copied", "x2 per push")
	for _, n := range []int{1_000, 10_000, 100_000} {
		fixed := &growPlusK{k: 64}
		for i := 0; i < n; i++ {
			fixed.Push(i)
		}
		var dbl Vector[int]
		for i := 0; i < n; i++ {
			dbl.Push(i)
		}
		fmt.Printf("  %10d %14d %16.1f %14d %16.2f\n",
			n, fixed.copied, float64(fixed.copied)/float64(n),
			dbl.copied, float64(dbl.copied)/float64(n))
	}

	fmt.Println()
	fmt.Println("growing by a fixed amount is Theta(n^2) total, and the per-push cost")
	fmt.Println("GROWS WITHOUT BOUND -- exactly the 'copies/n keeps climbing' signature")
	fmt.Println("from lesson 03, example 13.")
	fmt.Println()
	fmt.Println("the whole trick is the word FACTOR. Growing by x1.01 would also be")
	fmt.Println("amortized O(1) (with a constant of 100). Growing by +1,000,000 would not.")
}
```

**Output:**

```
pushing 16 values into a Vector[int], watching it grow:

    push    len    cap   copied
       1      1      1        0  <- grew
       2      2      2        1  <- grew
       3      3      4        3  <- grew
       4      4      4        3
       5      5      8        7  <- grew
       9      9     16       15  <- grew

the amortized argument, counted rather than asserted:

    n pushes      grows   total copied      copies/push
          16          5             15             0.94
         256          9            255             1.00
        4096         13           4095             1.00
       65536         17          65535             1.00
     1048576         21        1048575             1.00

copies/push converges to 1 and stops there. THAT is amortized Theta(1):

  the copies form a geometric series. Doubling from 1 to n copies
  1 + 2 + 4 + ... + n/2 = n - 1 elements in total, for n pushes.
  With a growth factor g the sum is n/(g-1), which is a CONSTANT
  multiple of n for any fixed g > 1. (Go's g tapers to ~1.25 for
  large slices, giving 1/(1.25-1) = 4 -- lesson 03, example 7.)

now the policy that seems more frugal -- grow by a fixed +k:

           n     +64 copied     +64 per push      x2 copied      x2 per push
        1000           7680              7.7           1023             1.02
       10000         783744             78.4          16383             1.64
      100000       78124992            781.2         131071             1.31

growing by a fixed amount is Theta(n^2) total, and the per-push cost
GROWS WITHOUT BOUND -- exactly the 'copies/n keeps climbing' signature
from lesson 03, example 13.

the whole trick is the word FACTOR. Growing by x1.01 would also be
amortized O(1) (with a constant of 100). Growing by +1,000,000 would not.
```

**copies/push converges to 1.00 and stops.** That is amortized Θ(1), derived rather than quoted: the
copies form a geometric series summing to `n/(g−1)`, a constant multiple of n for any fixed `g > 1`.

The `+64` column climbs 7.7 → 78.4 → 781.2 — linear in n, so Θ(n²) total. The whole trick is the word
**factor**.

**Rust:**

```rust
use std::mem;

// Time to build the thing Rust hands you for free, so the amortized-O(1) claim
// from lesson 03 stops being a fact you were told and becomes one you derived.
//
// A dynamic array is three fields and one rule:
//
//   data   Box<[T]>  the backing array, with spare room at the end
//   length usize     how many slots are in use
//   cap    usize     how many exist -- data.len(), so there is no separate field
//
//   RULE: when length == cap, allocate a bigger array and move. The new size
//         must be a constant FACTOR bigger, never a constant amount bigger.
//
// Why Box<[T]> and not Vec<T>: a Vec already owns a growth policy, which is the
// thing we are trying to build. A boxed slice is "just an array": it knows its
// length and nothing else, exactly like Go's `make([]T, n)`.

#[derive(Default)]
struct Vector<T> {
    data: Box<[T]>,
    length: usize,

    // instrumentation, so we can see what it did
    grows: usize,
    copied: usize,
}

// `T: Default` plays the part of Go's zero value: a freshly allocated array has
// to hold *something* in every slot, and Rust has no implicit zero for a T.
impl<T: Default> Vector<T> {
    fn len(&self) -> usize {
        self.length
    }

    fn cap(&self) -> usize {
        self.data.len()
    }

    /// Bounds-checked against len, not cap: slots past len are spare, not elements.
    fn at(&self, i: usize) -> Option<&T> {
        self.data[..self.length].get(i)
    }

    fn grow(&mut self) {
        let new_cap = (self.data.len() * 2).max(1);
        let mut bigger: Box<[T]> = (0..new_cap).map(|_| T::default()).collect();
        // Go copies the elements. Rust MOVES them (T need not be Clone): the live
        // prefix is swapped into the new array, and the old one is freed below.
        bigger[..self.length].swap_with_slice(&mut self.data[..self.length]);

        self.grows += 1;
        self.copied += self.length;
        self.data = bigger;
    }

    fn push(&mut self, x: T) {
        if self.length == self.data.len() {
            self.grow();
        }
        self.data[self.length] = x;
        self.length += 1;
    }

    fn pop(&mut self) -> Option<T> {
        if self.length == 0 {
            return None;
        }
        self.length -= 1;
        // take() moves the value out and leaves T::default() in the slot, so the
        // vector does not keep a second owner of anything inside T (e.g. a String's
        // buffer) alive in the spare space. Go zeroes the slot for the GC's sake.
        Some(mem::take(&mut self.data[self.length]))
    }
}

/// GrowPlusK is the policy people reach for when they think doubling is wasteful.
struct GrowPlusK {
    data: Box<[i64]>,
    length: usize,
    k: usize,
    grows: usize,
    copied: usize,
}

impl GrowPlusK {
    fn new(k: usize) -> Self {
        GrowPlusK {
            data: Box::default(),
            length: 0,
            k,
            grows: 0,
            copied: 0,
        }
    }

    fn push(&mut self, x: i64) {
        if self.length == self.data.len() {
            let old = self.data.len();
            let mut bigger = vec![0; old + self.k].into_boxed_slice();
            bigger[..old].copy_from_slice(&self.data);
            self.grows += 1;
            self.copied += self.length;
            self.data = bigger;
        }
        self.data[self.length] = x;
        self.length += 1;
    }
}

/// What the real std `Vec<i64>` does: first capacity, and the smallest and largest
/// ratio between consecutive capacities, over n pushes. Measured, not assumed.
fn vec_growth(n: usize) -> (usize, f64, f64) {
    let mut v: Vec<i64> = Vec::new();
    let mut caps = Vec::new();
    for i in 0..n {
        let before = v.capacity();
        v.push(i as i64);
        if v.capacity() != before {
            caps.push(v.capacity());
        }
    }
    let ratios = caps.windows(2).map(|w| w[1] as f64 / w[0] as f64);
    let min = ratios.clone().fold(f64::MAX, f64::min);
    let max = ratios.fold(0.0, f64::max);
    (caps[0], min, max)
}

fn main() {
    println!("pushing 16 values into a Vector<i64>, watching it grow:");
    println!();

    let mut v: Vector<i64> = Vector::default();
    println!("  {:>6} {:>6} {:>6} {:>8}", "push", "len", "cap", "copied");
    for i in 1..=16 {
        let before = v.cap();
        v.push(i);
        let grew = v.cap() != before;
        if i <= 5 || grew {
            let marker = if grew { "  <- grew" } else { "" };
            println!(
                "  {:>6} {:>6} {:>6} {:>8}{}",
                i,
                v.len(),
                v.cap(),
                v.copied,
                marker
            );
        }
    }

    // A silent self-check that the rest of the API behaves: `at` sees only live
    // slots (16 is spare capacity, not an element), and `pop` returns what `push`
    // stored, newest first.
    assert_eq!(v.at(15), Some(&16));
    assert_eq!(v.at(16), None);
    assert_eq!((v.pop(), v.pop()), (Some(16), Some(15)));

    println!();
    println!("the amortized argument, counted rather than asserted:");
    println!();
    println!(
        "  {:>10} {:>10} {:>14} {:>16}",
        "n pushes", "grows", "total copied", "copies/push"
    );
    for n in [16, 256, 4096, 65536, 1_048_576] {
        let mut w: Vector<i64> = Vector::default();
        for i in 0..n {
            w.push(i);
        }
        println!(
            "  {:>10} {:>10} {:>14} {:>16.2}",
            n,
            w.grows,
            w.copied,
            w.copied as f64 / n as f64
        );
    }

    println!();
    println!("copies/push converges to 1 and stops there. THAT is amortized Theta(1):");
    println!();
    println!("  the copies form a geometric series. Doubling from 1 to n copies");
    println!("  1 + 2 + 4 + ... + n/2 = n - 1 elements in total, for n pushes.");
    println!("  With a growth factor g the sum is n/(g-1), which is a CONSTANT");
    println!("  multiple of n for any fixed g > 1.");
    let (first, lo, hi) = vec_growth(1_048_576);
    println!("  (std's Vec<i64>, measured over 1,048,576 pushes: first capacity {first}, and");
    println!("  consecutive capacities differ by x{lo:.2} at the least, x{hi:.2} at the most --");
    println!("  g = 2 at every size, so the sum is n/(2-1) = n, the same constant as the");
    println!("  Vector above. lesson 03, example 7 measured the same thing.)");

    println!();
    println!("now the policy that seems more frugal -- grow by a fixed +k:");
    println!();
    println!(
        "  {:>10} {:>14} {:>16} {:>14} {:>16}",
        "n", "+64 copied", "+64 per push", "x2 copied", "x2 per push"
    );
    for n in [1_000, 10_000, 100_000] {
        let mut fixed = GrowPlusK::new(64);
        for i in 0..n {
            fixed.push(i);
        }
        let mut dbl: Vector<i64> = Vector::default();
        for i in 0..n {
            dbl.push(i);
        }
        println!(
            "  {:>10} {:>14} {:>16.1} {:>14} {:>16.2}",
            n,
            fixed.copied,
            fixed.copied as f64 / n as f64,
            dbl.copied,
            dbl.copied as f64 / n as f64
        );
    }

    println!();
    println!("growing by a fixed amount is Theta(n^2) total, and the per-push cost");
    println!("GROWS WITHOUT BOUND -- exactly the 'copies/n keeps climbing' signature");
    println!("from lesson 03, example 13.");
    println!();
    println!("the whole trick is the word FACTOR. Growing by x1.01 would also be");
    println!("amortized O(1) (with a constant of 100). Growing by +1,000,000 would not.");
}
```

**Rust output:**

```
pushing 16 values into a Vector<i64>, watching it grow:

    push    len    cap   copied
       1      1      1        0  <- grew
       2      2      2        1  <- grew
       3      3      4        3  <- grew
       4      4      4        3
       5      5      8        7  <- grew
       9      9     16       15  <- grew

the amortized argument, counted rather than asserted:

    n pushes      grows   total copied      copies/push
          16          5             15             0.94
         256          9            255             1.00
        4096         13           4095             1.00
       65536         17          65535             1.00
     1048576         21        1048575             1.00

copies/push converges to 1 and stops there. THAT is amortized Theta(1):

  the copies form a geometric series. Doubling from 1 to n copies
  1 + 2 + 4 + ... + n/2 = n - 1 elements in total, for n pushes.
  With a growth factor g the sum is n/(g-1), which is a CONSTANT
  multiple of n for any fixed g > 1.
  (std's Vec<i64>, measured over 1,048,576 pushes: first capacity 4, and
  consecutive capacities differ by x2.00 at the least, x2.00 at the most --
  g = 2 at every size, so the sum is n/(2-1) = n, the same constant as the
  Vector above. lesson 03, example 7 measured the same thing.)

now the policy that seems more frugal -- grow by a fixed +k:

           n     +64 copied     +64 per push      x2 copied      x2 per push
        1000           7680              7.7           1023             1.02
       10000         783744             78.4          16383             1.64
      100000       78124992            781.2         131071             1.31

growing by a fixed amount is Theta(n^2) total, and the per-push cost
GROWS WITHOUT BOUND -- exactly the 'copies/n keeps climbing' signature
from lesson 03, example 13.

the whole trick is the word FACTOR. Growing by x1.01 would also be
amortized O(1) (with a constant of 100). Growing by +1,000,000 would not.
```

> **Go vs Rust:** Every count is identical to Go's (5/9/13/17/21 grows, 1,048,575 elements copied at n=2^20, 781.2 copies per push for `+64` at n=100,000) because they belong to the algorithm, not the language. The source differs where the languages do: the backing store is a `Box<[T]>` (an array that knows only its length) because a `Vec` would bring its own growth policy; `T: Default` stands in for Go's implicit zero value; the live prefix is *moved* into the new array instead of copied, so `T` need not be `Clone`; and `pop`/`at` return `Option` instead of a `(T, bool)` pair or a panic. The one new line is measured: std's `Vec<i64>` doubles at every size, where this example's Go prose says Go's append tapers toward 1.25x for large slices.

**Complexity:** `Push` **Θ(1) amortized** with any factor `g > 1` · **Θ(n) amortized** with a fixed increment

---

## 8. Shrinking, and how it thrashes

`🟡 medium` · *Hysteresis*

Growing is the easy half. The natural shrink policy — "never waste memory, resize to fit when half
empty" — destroys the amortized bound on a workload that does nothing unusual.

**Steps:**

1. Add `Insert`/`Delete` on top of the same machinery.
2. Implement shrink-to-fit-at-half, and drive a push/pop cycle at the capacity boundary.
3. Replace it with shrink-at-a-quarter-and-halve.

```go
package main

import "fmt"

// Growing is the easy half. SHRINKING is where a hand-rolled dynamic array goes
// wrong, and the bug has a name: thrashing.
//
// The obvious policy -- "shrink when the array is half empty" -- makes an
// alternating push/pop at the boundary cost Theta(n) EVERY TIME, because each
// operation crosses the threshold and triggers a full copy. Amortized O(1) is
// destroyed by a workload that does nothing unusual.
//
// The fix is HYSTERESIS: grow at 100% full, shrink at 25% full. The gap between
// the two thresholds means you must do Theta(n) work to get back to the other
// one, which pays for the copy.

type vec struct {
	data   []int
	length int

	// shrink returns the capacity to shrink to, or -1 to leave it alone.
	shrink func(length, capacity int) int

	grows   int
	shrinks int
	copied  int
}

func (v *vec) resize(newCap int) {
	if newCap < 1 {
		newCap = 1
	}
	bigger := make([]int, newCap)
	copy(bigger, v.data[:v.length])
	v.copied += v.length
	v.data = bigger
}

func (v *vec) Push(x int) {
	if v.length == len(v.data) {
		newCap := len(v.data) * 2
		if newCap == 0 {
			newCap = 1
		}
		v.resize(newCap)
		v.grows++
	}
	v.data[v.length] = x
	v.length++
}

func (v *vec) Pop() (int, bool) {
	if v.length == 0 {
		return 0, false
	}
	v.length--
	x := v.data[v.length]
	v.data[v.length] = 0

	if v.shrink != nil {
		if target := v.shrink(v.length, len(v.data)); target >= 0 {
			v.resize(target)
			v.shrinks++
		}
	}
	return x, true
}

// Insert and Delete, on top of the same machinery.
func (v *vec) Insert(i, x int) {
	v.Push(0) // make room (and maybe grow)
	copy(v.data[i+1:v.length], v.data[i:v.length-1])
	v.data[i] = x
}

func (v *vec) Delete(i int) {
	copy(v.data[i:v.length-1], v.data[i+1:v.length])
	v.Pop()
}

func (v *vec) slice() []int { return v.data[:v.length] }

// shrinkToFit: "never waste memory -- if we are half empty, resize to exactly
// what we are using." Entirely reasonable-sounding, and it thrashes.
func shrinkToFit(length, capacity int) int {
	if capacity > 1 && length*2 <= capacity {
		return length
	}
	return -1
}

// shrinkWithHysteresis: only shrink when THREE QUARTERS empty, and only halve.
// The gap between the grow threshold (100% full) and the shrink threshold
// (25% full) is what makes the amortized argument survive.
func shrinkWithHysteresis(length, capacity int) int {
	if capacity > 1 && length*4 <= capacity {
		return capacity / 2
	}
	return -1
}

func main() {
	fmt.Println("Insert and Delete on the vector:")
	fmt.Println()
	v := &vec{}
	for _, x := range []int{10, 20, 30} {
		v.Push(x)
	}
	fmt.Printf("  after pushes        %v (len=%d cap=%d)\n", v.slice(), v.length, len(v.data))
	v.Insert(1, 15)
	fmt.Printf("  Insert(1, 15)       %v\n", v.slice())
	v.Insert(0, 5)
	fmt.Printf("  Insert(0, 5)        %v\n", v.slice())
	v.Delete(2)
	fmt.Printf("  Delete(2)           %v\n", v.slice())

	fmt.Println()
	fmt.Println("now the shrink policy. The workload: fill up, pop down to just under")
	fmt.Println("the shrink threshold, then alternate push/pop THERE 10,000 times --")
	fmt.Println("which is what a stack or an undo buffer does near its working size.")
	fmt.Println()

	fmt.Printf("  %-22s %8s %8s %16s %14s\n",
		"policy", "grows", "shrinks", "elements copied", "per push/pop")
	for _, p := range []struct {
		name string
		fn   func(int, int) int
	}{
		{"never shrink", nil},
		{"shrink-to-fit at 1/2", shrinkToFit},
		{"halve at 1/4 (correct)", shrinkWithHysteresis},
	} {
		w := &vec{shrink: p.fn}

		// Fill to 1025 so the capacity doubles to 2048, then pop once. That
		// single pop puts a shrink-to-fit vector at len == cap == 1024, which
		// is exactly where the next push has to grow again.
		for i := 0; i < 1025; i++ {
			w.Push(i)
		}
		w.Pop()

		const rounds = 10_000
		base, g0, s0 := w.copied, w.grows, w.shrinks
		for i := 0; i < rounds; i++ {
			w.Push(1)
			w.Pop()
		}
		fmt.Printf("  %-22s %8d %8d %16d %14.1f\n",
			p.name, w.grows-g0, w.shrinks-s0, w.copied-base,
			float64(w.copied-base)/float64(rounds))
	}

	fmt.Println()
	fmt.Println("the middle row is the bug, and it is a spectacular one: 2048 elements")
	fmt.Println("copied per push/pop pair, forever, on a vector holding 1024 items.")
	fmt.Println()
	fmt.Println("the cycle: at len == cap the push grows to 2n and copies n. The pop")
	fmt.Println("then leaves the vector exactly half empty, shrink-to-fit resizes back")
	fmt.Println("down to n and copies n again -- landing precisely on len == cap, ready")
	fmt.Println("to do it all over. The grow and shrink thresholds are TOUCHING, so a")
	fmt.Println("single operation crosses both.")
	fmt.Println()
	fmt.Println("with the 1/4 threshold there is a whole quarter of the array between")
	fmt.Println("the grow point and the shrink point. To trigger a shrink after a grow")
	fmt.Println("you must pop away half the elements -- Theta(n) work -- which is")
	fmt.Println("exactly what pays for the Theta(n) copy. Amortized Theta(1) survives.")

	fmt.Println()
	fmt.Println("this is why Go's own slices NEVER shrink: `xs = xs[:0]` keeps the")
	fmt.Println("whole backing array. The runtime declines to guess, and hands the")
	fmt.Println("decision to you -- which is the reslice leak from lesson 04 and the")
	fmt.Println("buffer-reuse idiom from example 10, depending on what you wanted.")
}
```

**Output:**

```
Insert and Delete on the vector:

  after pushes        [10 20 30] (len=3 cap=4)
  Insert(1, 15)       [10 15 20 30]
  Insert(0, 5)        [5 10 15 20 30]
  Delete(2)           [5 10 20 30]

now the shrink policy. The workload: fill up, pop down to just under
the shrink threshold, then alternate push/pop THERE 10,000 times --
which is what a stack or an undo buffer does near its working size.

  policy                    grows  shrinks  elements copied   per push/pop
  never shrink                  0        0                0            0.0
  shrink-to-fit at 1/2      10000    10000         20480000         2048.0
  halve at 1/4 (correct)        0        0                0            0.0

the middle row is the bug, and it is a spectacular one: 2048 elements
copied per push/pop pair, forever, on a vector holding 1024 items.

the cycle: at len == cap the push grows to 2n and copies n. The pop
then leaves the vector exactly half empty, shrink-to-fit resizes back
down to n and copies n again -- landing precisely on len == cap, ready
to do it all over. The grow and shrink thresholds are TOUCHING, so a
single operation crosses both.

with the 1/4 threshold there is a whole quarter of the array between
the grow point and the shrink point. To trigger a shrink after a grow
you must pop away half the elements -- Theta(n) work -- which is
exactly what pays for the Theta(n) copy. Amortized Theta(1) survives.

this is why Go's own slices NEVER shrink: `xs = xs[:0]` keeps the
whole backing array. The runtime declines to guess, and hands the
decision to you -- which is the reslice leak from lesson 04 and the
buffer-reuse idiom from example 10, depending on what you wanted.
```

**2048 elements copied per push/pop pair**, forever, on a vector holding 1024 items — versus zero for
both other policies. The grow and shrink thresholds were *touching*, so a single operation crossed
both.

Hysteresis fixes it: grow at 100% full, shrink at 25% full, and only halve. To trigger a shrink after
a grow you must now pop away half the elements — Θ(n) work, which is exactly what pays for the Θ(n)
copy.

**Rust:**

```rust
use std::fmt::Display;
use std::mem;

// Growing is the easy half. SHRINKING is where a hand-rolled dynamic array goes
// wrong, and the bug has a name: thrashing.
//
// The obvious policy -- "shrink when the array is half empty" -- makes an
// alternating push/pop at the boundary cost Theta(n) EVERY TIME, because each
// operation crosses the threshold and triggers a full copy. Amortized O(1) is
// destroyed by a workload that does nothing unusual.
//
// The fix is HYSTERESIS: grow at 100% full, shrink at 25% full. The gap between
// the two thresholds means you must do Theta(n) work to get back to the other
// one, which pays for the copy.

/// A shrink policy gets (length, capacity) and returns the capacity to shrink
/// to, or None to leave the array alone. (Go returns -1 for "leave it alone".)
type ShrinkPolicy = fn(length: usize, capacity: usize) -> Option<usize>;

struct Vector {
    // Box<[i64]>, not Vec<i64>: the capacity is exactly data.len(), and WE decide
    // when it changes -- a Vec would quietly apply its own growth policy.
    data: Box<[i64]>,
    length: usize,

    shrink: Option<ShrinkPolicy>,

    grows: usize,
    shrinks: usize,
    copied: usize,
}

impl Vector {
    fn new(shrink: Option<ShrinkPolicy>) -> Self {
        Vector {
            data: Box::default(),
            length: 0,
            shrink,
            grows: 0,
            shrinks: 0,
            copied: 0,
        }
    }

    fn resize(&mut self, new_cap: usize) {
        let mut bigger = vec![0; new_cap.max(1)].into_boxed_slice();
        bigger[..self.length].copy_from_slice(&self.data[..self.length]);
        self.copied += self.length;
        self.data = bigger;
    }

    fn push(&mut self, x: i64) {
        if self.length == self.data.len() {
            self.resize((self.data.len() * 2).max(1));
            self.grows += 1;
        }
        self.data[self.length] = x;
        self.length += 1;
    }

    fn pop(&mut self) -> Option<i64> {
        if self.length == 0 {
            return None;
        }
        self.length -= 1;
        let x = mem::take(&mut self.data[self.length]); // leaves 0 behind, like Go
        if let Some(shrink) = self.shrink
            && let Some(target) = shrink(self.length, self.data.len())
        {
            self.resize(target);
            self.shrinks += 1;
        }
        Some(x)
    }

    // Insert and Delete, on top of the same machinery. `copy_within` is Rust's
    // memmove on one slice: Go needs two overlapping views of the same array
    // for this (`copy(v.data[i+1:], v.data[i:])`); Rust cannot hand out two
    // overlapping &mut views, so the one-slice call is the safe spelling.
    fn insert(&mut self, i: usize, x: i64) {
        self.push(0); // make room (and maybe grow)
        self.data.copy_within(i..self.length - 1, i + 1);
        self.data[i] = x;
    }

    fn delete(&mut self, i: usize) {
        self.data.copy_within(i + 1..self.length, i);
        self.pop();
    }

    fn slice(&self) -> &[i64] {
        &self.data[..self.length]
    }
}

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// shrink_to_fit: "never waste memory -- if we are half empty, resize to exactly
/// what we are using." Entirely reasonable-sounding, and it thrashes.
fn shrink_to_fit(length: usize, capacity: usize) -> Option<usize> {
    (capacity > 1 && length * 2 <= capacity).then_some(length)
}

/// shrink_with_hysteresis: only shrink when THREE QUARTERS empty, and only halve.
/// The gap between the grow threshold (100% full) and the shrink threshold
/// (25% full) is what makes the amortized argument survive.
fn shrink_with_hysteresis(length: usize, capacity: usize) -> Option<usize> {
    (capacity > 1 && length * 4 <= capacity).then_some(capacity / 2)
}

fn main() {
    println!("Insert and Delete on the vector:");
    println!();
    let mut v = Vector::new(None);
    for x in [10, 20, 30] {
        v.push(x);
    }
    println!(
        "  after pushes        {} (len={} cap={})",
        list(v.slice()),
        v.length,
        v.data.len()
    );
    v.insert(1, 15);
    println!("  Insert(1, 15)       {}", list(v.slice()));
    v.insert(0, 5);
    println!("  Insert(0, 5)        {}", list(v.slice()));
    v.delete(2);
    println!("  Delete(2)           {}", list(v.slice()));

    println!();
    println!("now the shrink policy. The workload: fill up, pop down to just under");
    println!("the shrink threshold, then alternate push/pop THERE 10,000 times --");
    println!("which is what a stack or an undo buffer does near its working size.");
    println!();

    println!(
        "  {:<22} {:>8} {:>8} {:>16} {:>14}",
        "policy", "grows", "shrinks", "elements copied", "per push/pop"
    );
    let policies: [(&str, Option<ShrinkPolicy>); 3] = [
        ("never shrink", None),
        ("shrink-to-fit at 1/2", Some(shrink_to_fit)),
        ("halve at 1/4 (correct)", Some(shrink_with_hysteresis)),
    ];
    for (name, shrink) in policies {
        let mut w = Vector::new(shrink);

        // Fill to 1025 so the capacity doubles to 2048, then pop once. That
        // single pop puts a shrink-to-fit vector at len == cap == 1024, which
        // is exactly where the next push has to grow again.
        for i in 0..1025 {
            w.push(i);
        }
        w.pop();

        const ROUNDS: usize = 10_000;
        let (base, g0, s0) = (w.copied, w.grows, w.shrinks);
        for _ in 0..ROUNDS {
            w.push(1);
            w.pop();
        }
        println!(
            "  {:<22} {:>8} {:>8} {:>16} {:>14.1}",
            name,
            w.grows - g0,
            w.shrinks - s0,
            w.copied - base,
            (w.copied - base) as f64 / ROUNDS as f64
        );
    }

    println!();
    println!("the middle row is the bug, and it is a spectacular one: 2048 elements");
    println!("copied per push/pop pair, forever, on a vector holding 1024 items.");
    println!();
    println!("the cycle: at len == cap the push grows to 2n and copies n. The pop");
    println!("then leaves the vector exactly half empty, shrink-to-fit resizes back");
    println!("down to n and copies n again -- landing precisely on len == cap, ready");
    println!("to do it all over. The grow and shrink thresholds are TOUCHING, so a");
    println!("single operation crosses both.");
    println!();
    println!("with the 1/4 threshold there is a whole quarter of the array between");
    println!("the grow point and the shrink point. To trigger a shrink after a grow");
    println!("you must pop away half the elements -- Theta(n) work -- which is");
    println!("exactly what pays for the Theta(n) copy. Amortized Theta(1) survives.");

    // Go's claim here is "Go's own slices NEVER shrink". Rust's Vec is the same
    // kind of thing, so check it instead of assuming it.
    let mut xs: Vec<i64> = Vec::new();
    for i in 0..1025 {
        xs.push(i);
    }
    let full = xs.capacity();
    while xs.pop().is_some() {}
    let after_pops = xs.capacity();
    xs.clear();
    let after_clear = xs.capacity();
    xs.shrink_to_fit();
    let after_shrink = xs.capacity();

    println!();
    println!("Rust's own Vec makes the same choice: it NEVER shrinks by itself.");
    println!("Measured: 1025 pushes gave cap={full}; after 1025 pops, len=0 cap={after_pops};");
    println!("after clear(), cap={after_clear}. Only an explicit shrink_to_fit() gave");
    println!("cap={after_shrink}. The library declines to guess, and hands the decision");
    println!("to you -- which is the retained-buffer bug from lesson 04 and the");
    println!("buffer-reuse idiom from example 10, depending on what you wanted.");
}
```

**Rust output:**

```
Insert and Delete on the vector:

  after pushes        [10 20 30] (len=3 cap=4)
  Insert(1, 15)       [10 15 20 30]
  Insert(0, 5)        [5 10 15 20 30]
  Delete(2)           [5 10 20 30]

now the shrink policy. The workload: fill up, pop down to just under
the shrink threshold, then alternate push/pop THERE 10,000 times --
which is what a stack or an undo buffer does near its working size.

  policy                    grows  shrinks  elements copied   per push/pop
  never shrink                  0        0                0            0.0
  shrink-to-fit at 1/2      10000    10000         20480000         2048.0
  halve at 1/4 (correct)        0        0                0            0.0

the middle row is the bug, and it is a spectacular one: 2048 elements
copied per push/pop pair, forever, on a vector holding 1024 items.

the cycle: at len == cap the push grows to 2n and copies n. The pop
then leaves the vector exactly half empty, shrink-to-fit resizes back
down to n and copies n again -- landing precisely on len == cap, ready
to do it all over. The grow and shrink thresholds are TOUCHING, so a
single operation crosses both.

with the 1/4 threshold there is a whole quarter of the array between
the grow point and the shrink point. To trigger a shrink after a grow
you must pop away half the elements -- Theta(n) work -- which is
exactly what pays for the Theta(n) copy. Amortized Theta(1) survives.

Rust's own Vec makes the same choice: it NEVER shrinks by itself.
Measured: 1025 pushes gave cap=2048; after 1025 pops, len=0 cap=2048;
after clear(), cap=2048. Only an explicit shrink_to_fit() gave
cap=0. The library declines to guess, and hands the decision
to you -- which is the retained-buffer bug from lesson 04 and the
buffer-reuse idiom from example 10, depending on what you wanted.
```

> **Go vs Rust:** Every number is identical to Go's, including the 20,480,000 elements copied and the 2048 per push/pop pair, so thrashing is a property of the thresholds and not of the language. Differences are in the source: the shrink policy returns `Option<usize>` where Go returns `-1`, `Box<[i64]>` keeps the capacity under the vector's control (a `Vec` would apply its own growth policy), and `Insert`/`Delete` use one `copy_within` call because Rust cannot hold two overlapping `&mut` views of the buffer the way Go's `copy(v.data[i+1:], v.data[i:])` does. The last paragraph is measured rather than copied: std's `Vec` also keeps its capacity through every `pop` and `clear` (2048 after 1025 pushes and again after popping to empty) and only an explicit `shrink_to_fit()` releases it.

**Complexity:** with hysteresis, `Push`/`Pop` stay **Θ(1) amortized** · shrink-to-fit makes both **Θ(n) worst case, every time**

---

## 9. Choosing the growth factor

`🟡 medium` · *A real engineering trade*

Any `g > 1` gives amortized Θ(1), so which one? Both sides of the trade are measurable, and they move
in opposite directions.

**Steps:**

1. Run ×1.25, ×1.5, ×2 and ×4 over a million appends.
2. Record copies per push and the worst capacity/length ratio.
3. Check each against the predicted `1/(g−1)`.

```go
package main

import "fmt"

// Any growth factor g > 1 gives amortized Theta(1). So which g?
//
// It is a genuine trade, and both sides are measurable:
//
//   BIG g   fewer copies (cheap), more wasted capacity (expensive)
//   SMALL g more copies (expensive), less waste (cheap)
//
// The theory: with factor g, total copies for n appends is about n/(g-1), and
// the worst-case wasted capacity right after a grow is about n(g-1)/g... but
// rather than trust that, count it.

type policy struct {
	name   string
	next   func(cap int) int
	copied int
	grows  int

	maxRatio float64 // worst capacity/length seen -- the real waste measure
	finalCap int
}

func (p *policy) run(n int) {
	capacity, length := 0, 0
	for i := 0; i < n; i++ {
		if length == capacity {
			newCap := p.next(capacity)
			p.copied += length
			p.grows++
			capacity = newCap
		}
		length++
		if r := float64(capacity) / float64(length); r > p.maxRatio {
			p.maxRatio = r
		}
	}
	p.finalCap = capacity
}

func factor(f float64) func(int) int {
	return func(c int) int {
		if c == 0 {
			return 1
		}
		n := int(float64(c) * f)
		if n <= c {
			n = c + 1
		}
		return n
	}
}

func main() {
	const n = 1_000_000

	policies := []*policy{
		{name: "x1.25", next: factor(1.25)},
		{name: "x1.5", next: factor(1.5)},
		{name: "x2 (classic)", next: factor(2)},
		{name: "x4", next: factor(4)},
	}

	for _, p := range policies {
		p.run(n)
	}

	fmt.Printf("appending %d elements, one at a time:\n\n", n)
	fmt.Printf("%-14s %8s %14s %14s %18s %14s\n",
		"growth", "grows", "copied", "copies/push", "worst cap/len", "final waste")
	for _, p := range policies {
		waste := float64(p.finalCap-n) / float64(n) * 100
		fmt.Printf("%-14s %8d %14d %14.2f %17.2fx %13.1f%%\n",
			p.name, p.grows, p.copied, float64(p.copied)/float64(n), p.maxRatio, waste)
	}
	fmt.Println()
	fmt.Println("(read 'worst cap/len', not 'final waste'. The final column depends on")
	fmt.Println(" where n happens to fall between two growths -- 1,000,000 sits just")
	fmt.Println(" under 2^20, which flatters x2 and x4 by accident.)")

	fmt.Println()
	fmt.Println("the two columns move in opposite directions, exactly as predicted:")
	fmt.Println()
	fmt.Println("  x1.25  4.5 copies per push, and never more than 1.25x over-allocated")
	fmt.Println("  x4     0.35 copies per push, and up to 4x over-allocated")
	fmt.Println()
	fmt.Println("copies/push is very close to 1/(g-1) in every row -- that is the")
	fmt.Println("geometric series from example 7, confirmed empirically.")

	fmt.Println()
	fmt.Println("there is a third consideration that decides it in practice, and it")
	fmt.Println("is not on this table: ALLOCATOR REUSE.")
	fmt.Println()
	fmt.Println("with g = 2, each new block is bigger than the sum of every block")
	fmt.Println("freed so far (1+2+4+...+n/2 = n-1 < n), so the freed space can never")
	fmt.Println("be reused for the next growth. With g < 2 the sum of the freed")
	fmt.Println("blocks eventually exceeds the next request, and the allocator can")
	fmt.Println("recycle them. That is why several real implementations chose 1.5.")

	fmt.Println()
	fmt.Println("what everyone actually does:")
	fmt.Println()
	fmt.Println("  Go slices        x2 for small, tapering toward x1.25 for large")
	fmt.Println("  C++ libstdc++    x2")
	fmt.Println("  C++ MSVC         x1.5")
	fmt.Println("  Python list      about x1.125 plus a constant")
	fmt.Println("  Java ArrayList   x1.5")
	fmt.Println()
	fmt.Println("Go's taper is the interesting one: it takes the cheap-copy end of")
	fmt.Println("the trade while the array is small and nobody cares about waste, then")
	fmt.Println("switches to the low-waste end once the array is big enough that a")
	fmt.Println("doubling would be measured in megabytes.")

	fmt.Println()
	fmt.Println("the practical answer, though, is the one from lesson 03: if you know")
	fmt.Println("n, call make([]T, 0, n) and the growth factor becomes irrelevant --")
	fmt.Println("zero copies, zero waste.")
}
```

**Output:**

```
appending 1000000 elements, one at a time:

growth            grows         copied    copies/push      worst cap/len    final waste
x1.25                62        4489047           4.49              1.25x          12.2%
x1.5                 35        2099753           2.10              1.50x           5.0%
x2 (classic)         21        1048575           1.05              2.00x           4.9%
x4                   11         349525           0.35              4.00x           4.9%

(read 'worst cap/len', not 'final waste'. The final column depends on
 where n happens to fall between two growths -- 1,000,000 sits just
 under 2^20, which flatters x2 and x4 by accident.)

the two columns move in opposite directions, exactly as predicted:

  x1.25  4.5 copies per push, and never more than 1.25x over-allocated
  x4     0.35 copies per push, and up to 4x over-allocated

copies/push is very close to 1/(g-1) in every row -- that is the
geometric series from example 7, confirmed empirically.

there is a third consideration that decides it in practice, and it
is not on this table: ALLOCATOR REUSE.

with g = 2, each new block is bigger than the sum of every block
freed so far (1+2+4+...+n/2 = n-1 < n), so the freed space can never
be reused for the next growth. With g < 2 the sum of the freed
blocks eventually exceeds the next request, and the allocator can
recycle them. That is why several real implementations chose 1.5.

what everyone actually does:

  Go slices        x2 for small, tapering toward x1.25 for large
  C++ libstdc++    x2
  C++ MSVC         x1.5
  Python list      about x1.125 plus a constant
  Java ArrayList   x1.5

Go's taper is the interesting one: it takes the cheap-copy end of
the trade while the array is small and nobody cares about waste, then
switches to the low-waste end once the array is big enough that a
doubling would be measured in megabytes.

the practical answer, though, is the one from lesson 03: if you know
n, call make([]T, 0, n) and the growth factor becomes irrelevant --
zero copies, zero waste.
```

Every row's copies/push matches `1/(g−1)`: 4.49≈4, 2.10≈2, 1.05≈1, 0.35≈⅓. And the worst cap/len
column is exactly `g`.

There's a third consideration the table can't show: with `g = 2` each new block is bigger than the sum
of every block freed so far (`1+2+4+…+n/2 = n−1 < n`), so the allocator can never reuse them. With
`g < 2` it eventually can — which is why several real implementations chose 1.5.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Any growth factor g > 1 gives amortized Theta(1). So which g?
//
// It is a genuine trade, and both sides are measurable:
//
//   BIG g   fewer copies (cheap), more wasted capacity (expensive)
//   SMALL g more copies (expensive), less waste (cheap)
//
// The theory: with factor g, total copies for n appends is about n/(g-1), and
// the worst-case wasted capacity right after a grow is about n(g-1)/g... but
// rather than trust that, count it.

// ---- allocation counter (the kit's stand-in for runtime.MemStats): counts every
// allocation request, including the reallocs a growing Vec makes ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

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

/// (allocation events while running `f`, its result)
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

/// What one policy did over n appends.
#[derive(Default)]
struct Stats {
    grows: usize,
    copied: usize,

    max_ratio: f64, // worst capacity/length seen -- the real waste measure
    final_cap: usize,
}

impl Stats {
    fn see(&mut self, capacity: usize, length: usize) {
        let r = capacity as f64 / length as f64;
        if r > self.max_ratio {
            self.max_ratio = r;
        }
    }
}

struct Policy {
    name: &'static str,
    next: Box<dyn Fn(usize) -> usize>,
}

impl Policy {
    fn run(&self, n: usize) -> Stats {
        let mut s = Stats::default();
        let (mut capacity, mut length) = (0, 0);
        for _ in 0..n {
            if length == capacity {
                let new_cap = (self.next)(capacity);
                s.copied += length;
                s.grows += 1;
                capacity = new_cap;
            }
            length += 1;
            s.see(capacity, length);
        }
        s.final_cap = capacity;
        s
    }
}

fn factor(f: f64) -> impl Fn(usize) -> usize {
    move |c| {
        if c == 0 {
            return 1;
        }
        let n = (c as f64 * f) as usize;
        if n <= c { c + 1 } else { n }
    }
}

/// The real thing: push n values into a std Vec<i64> and watch what IT does. "copied"
/// is what a move to the new buffer would copy (realloc may extend in place and copy
/// nothing, so it is the policy's worst case, as in lesson 03).
struct Measured {
    stats: Stats,
    first_cap: usize,
    min_step: f64,         // smallest ratio between consecutive capacities
    max_step: f64,         // largest
    worst_from_len_8: f64, // worst capacity/length once len >= 8
}

fn measure_vec(n: usize) -> Measured {
    let mut s = Stats::default();
    let mut v: Vec<i64> = Vec::new();
    let mut caps = Vec::new();
    let mut worst_from_8 = 0.0_f64;
    for i in 0..n {
        let (before, len) = (v.capacity(), v.len());
        v.push(i as i64);
        if v.capacity() != before {
            s.copied += len;
            s.grows += 1;
            caps.push(v.capacity());
        }
        s.see(v.capacity(), v.len());
        if v.len() >= 8 {
            worst_from_8 = worst_from_8.max(v.capacity() as f64 / v.len() as f64);
        }
    }
    s.final_cap = v.capacity();
    let steps = caps.windows(2).map(|w| w[1] as f64 / w[0] as f64);
    Measured {
        min_step: steps.clone().fold(f64::MAX, f64::min),
        max_step: steps.fold(0.0, f64::max),
        first_cap: caps[0],
        worst_from_len_8: worst_from_8,
        stats: s,
    }
}

fn main() {
    const N: usize = 1_000_000;

    let policies = [
        Policy {
            name: "x1.25",
            next: Box::new(factor(1.25)),
        },
        Policy {
            name: "x1.5",
            next: Box::new(factor(1.5)),
        },
        Policy {
            name: "x2 (classic)",
            next: Box::new(factor(2.0)),
        },
        Policy {
            name: "x4",
            next: Box::new(factor(4.0)),
        },
    ];
    let mut rows: Vec<(&str, Stats)> = policies.iter().map(|p| (p.name, p.run(N))).collect();
    let real = measure_vec(N);
    let (vec_grows, vec_copies_per_push) = (real.stats.grows, real.stats.copied as f64 / N as f64);
    rows.push(("Vec<i64>", real.stats));

    println!("appending {N} elements, one at a time:\n");
    println!(
        "{:<14} {:>8} {:>14} {:>14} {:>18} {:>14}",
        "growth", "grows", "copied", "copies/push", "worst cap/len", "final waste"
    );
    for (name, s) in &rows {
        let waste = (s.final_cap as f64 - N as f64) / N as f64 * 100.0;
        println!(
            "{:<14} {:>8} {:>14} {:>14.2} {:>17.2}x {:>13.1}%",
            name,
            s.grows,
            s.copied,
            s.copied as f64 / N as f64,
            s.max_ratio,
            waste
        );
    }
    println!();
    println!("(read 'worst cap/len', not 'final waste'. The final column depends on");
    println!(" where n happens to fall between two growths -- 1,000,000 sits just");
    println!(" under 2^20, which flatters x2 and x4 by accident.)");
    println!();
    println!("(the last row is std's own Vec, not a simulation. Its first capacity is");
    println!(
        " {}, so its worst cap/len is that over len=1 ({:.2}x); from len=8 on it never",
        real.first_cap, real.first_cap as f64
    );
    println!(
        " exceeds {:.2}x. Its smallest capacity step is x{:.2} and its largest x{:.2}:",
        real.worst_from_len_8, real.min_step, real.max_step
    );
    println!(" a doubling policy.)");

    println!();
    println!("the two columns move in opposite directions, exactly as predicted:");
    println!();
    println!("  x1.25  4.5 copies per push, and never more than 1.25x over-allocated");
    println!("  x4     0.35 copies per push, and up to 4x over-allocated");
    println!();
    println!("copies/push is very close to 1/(g-1) in every row -- that is the");
    println!("geometric series from example 7, confirmed empirically.");

    println!();
    println!("there is a third consideration that decides it in practice, and it");
    println!("is not on this table: ALLOCATOR REUSE.");
    println!();
    println!("with g = 2, each new block is bigger than the sum of every block");
    println!("freed so far (1+2+4+...+n/2 = n-1 < n), so the freed space can never");
    println!("be reused for the next growth. With g < 2 the sum of the freed");
    println!("blocks eventually exceeds the next request, and the allocator can");
    println!("recycle them. That is why several real implementations chose 1.5.");

    println!();
    println!("what everyone actually does:");
    println!();
    println!("  Rust Vec         x2 at every size, no taper (measured above: {vec_grows} grows)");
    println!("  C++ libstdc++    x2    (documented, not measured here)");
    println!("  C++ MSVC         x1.5  (documented, not measured here)");
    println!("  Python list      about x1.125 plus a constant  (documented, not measured here)");
    println!("  Java ArrayList   x1.5  (documented, not measured here)");
    println!();
    println!("Rust's Vec sits at the x2 point of the trade at every size: the Vec row");
    println!(
        "shows {vec_copies_per_push:.2} copies per push, and the price is the spare capacity a"
    );
    println!("doubling leaves behind once the array is big enough to be measured in");
    println!("megabytes.");

    // Count allocations for the two ways of building the same million-element Vec.
    let (grown_events, grown) = allocs_of(|| {
        let mut v: Vec<i64> = Vec::new();
        for i in 0..N {
            v.push(i as i64);
        }
        v
    });
    let (sized_events, sized) = allocs_of(|| {
        let mut v: Vec<i64> = Vec::with_capacity(N);
        for i in 0..N {
            v.push(i as i64);
        }
        v
    });

    println!();
    println!("the practical answer, though, is the one from lesson 03: if you know");
    println!("n, call Vec::with_capacity(n) and the growth factor becomes irrelevant.");
    println!("Measured with a counting allocator, for {N} pushes:");
    println!();
    for (label, events, v) in [
        ("Vec::new()".to_string(), grown_events, &grown),
        (format!("Vec::with_capacity({N})"), sized_events, &sized),
    ] {
        println!(
            "  {:<27} {:>3} allocation event(s), final capacity {}",
            label,
            events,
            v.capacity()
        );
    }
}
```

**Rust output:**

```
appending 1000000 elements, one at a time:

growth            grows         copied    copies/push      worst cap/len    final waste
x1.25                62        4489047           4.49              1.25x          12.2%
x1.5                 35        2099753           2.10              1.50x           5.0%
x2 (classic)         21        1048575           1.05              2.00x           4.9%
x4                   11         349525           0.35              4.00x           4.9%
Vec<i64>             19        1048572           1.05              4.00x           4.9%

(read 'worst cap/len', not 'final waste'. The final column depends on
 where n happens to fall between two growths -- 1,000,000 sits just
 under 2^20, which flatters x2 and x4 by accident.)

(the last row is std's own Vec, not a simulation. Its first capacity is
 4, so its worst cap/len is that over len=1 (4.00x); from len=8 on it never
 exceeds 2.00x. Its smallest capacity step is x2.00 and its largest x2.00:
 a doubling policy.)

the two columns move in opposite directions, exactly as predicted:

  x1.25  4.5 copies per push, and never more than 1.25x over-allocated
  x4     0.35 copies per push, and up to 4x over-allocated

copies/push is very close to 1/(g-1) in every row -- that is the
geometric series from example 7, confirmed empirically.

there is a third consideration that decides it in practice, and it
is not on this table: ALLOCATOR REUSE.

with g = 2, each new block is bigger than the sum of every block
freed so far (1+2+4+...+n/2 = n-1 < n), so the freed space can never
be reused for the next growth. With g < 2 the sum of the freed
blocks eventually exceeds the next request, and the allocator can
recycle them. That is why several real implementations chose 1.5.

what everyone actually does:

  Rust Vec         x2 at every size, no taper (measured above: 19 grows)
  C++ libstdc++    x2    (documented, not measured here)
  C++ MSVC         x1.5  (documented, not measured here)
  Python list      about x1.125 plus a constant  (documented, not measured here)
  Java ArrayList   x1.5  (documented, not measured here)

Rust's Vec sits at the x2 point of the trade at every size: the Vec row
shows 1.05 copies per push, and the price is the spare capacity a
doubling leaves behind once the array is big enough to be measured in
megabytes.

the practical answer, though, is the one from lesson 03: if you know
n, call Vec::with_capacity(n) and the growth factor becomes irrelevant.
Measured with a counting allocator, for 1000000 pushes:

  Vec::new()                   19 allocation event(s), final capacity 1048576
  Vec::with_capacity(1000000)   1 allocation event(s), final capacity 1000000
```

> **Go vs Rust:** The four simulated rows are identical to Go's: they are arithmetic on a growth rule, not a measurement of either runtime. What changed is the paragraph about real implementations, because Go's "x2 tapering toward x1.25" has no Rust counterpart: the added fifth row is a real std `Vec<i64>` and doubles at every size (19 growths over a million pushes, 1.05 copies per push counted as moves, so it sits on the x2 row). Its worst cap/len is 4.00x only because its first allocation holds 4 elements; from len 8 on it stays under 2x. The closing paragraph is measured too: building the same million-element `Vec` takes 19 allocation events with `Vec::new()` and 1 with `with_capacity`. The C++/Python/Java lines are the Go program's, labelled as not measured here.

**Complexity:** all are Θ(1) amortized · the constant is `1/(g−1)` copies and the waste is up to `g×`

---

## 10. `buf[:0]` — the best optimisation in a hot loop

`🟡 medium` · *Reuse*

Lesson 04 framed "slices never shrink" as a leak. Turned around, it's the reason a hot loop can
allocate exactly once.

**Steps:**

1. Filter into a fresh slice each iteration, then into a reused buffer.
2. Compare ns/op and allocs/op.
3. Learn the three ways to empty a slice, and the pointer trap in the reuse pattern.

```go
package main

import (
	"fmt"
	"testing"
)

// Go's slices never shrink on their own, which lesson 04 framed as a leak.
// Turned around, it is the single best optimisation available to a hot loop:
// `buf = buf[:0]` keeps the whole backing array and resets the length, so the
// next round of appends reuses memory you already have.
//
// This is the difference between "allocates every iteration" and "allocates
// once, ever".

var sink []int

// fresh allocates a new slice every call -- the obvious version.
func fresh(src []int) []int {
	var out []int
	for _, v := range src {
		if v%2 == 0 {
			out = append(out, v)
		}
	}
	return out
}

// reuse appends into a caller-supplied buffer. The caller passes buf[:0].
func reuse(buf []int, src []int) []int {
	for _, v := range src {
		if v%2 == 0 {
			buf = append(buf, v)
		}
	}
	return buf
}

type item struct{ name *string }

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	src := make([]int, 10_000)
	for i := range src {
		src[i] = i
	}

	fmt.Println("filtering 10,000 ints, 1000 times over:")
	fmt.Println()

	tFresh := nsPerOp(func() { sink = fresh(src) })

	buf := make([]int, 0, len(src))
	tReuse := nsPerOp(func() { buf = reuse(buf[:0], src); sink = buf })

	aFresh := testing.AllocsPerRun(20, func() { sink = fresh(src) })
	buf2 := make([]int, 0, len(src))
	aReuse := testing.AllocsPerRun(20, func() { buf2 = reuse(buf2[:0], src); sink = buf2 })

	fmt.Printf("  %-28s %14s %14s\n", "", "ns/op", "allocs/op")
	fmt.Printf("  %-28s %14.0f %14.0f\n", "fresh slice each time", tFresh, aFresh)
	fmt.Printf("  %-28s %14.0f %14.0f\n", "reuse buf[:0]", tReuse, aReuse)
	fmt.Printf("  %-28s %13.1fx\n", "speedup", tFresh/tReuse)

	fmt.Println()
	fmt.Println("`buf[:0]` sets len to 0 and leaves cap alone. Every append then")
	fmt.Println("writes into memory that is already there and already warm in cache.")

	fmt.Println()
	fmt.Println("the three ways to empty a slice, and what each one means:")
	fmt.Println()
	xs := []int{1, 2, 3, 4, 5}
	a := xs[:0]
	fmt.Printf("  xs[:0]        len=%d cap=%d   reuse the memory (this example)\n", len(a), cap(a))

	b := []int{1, 2, 3, 4, 5}
	clear(b)
	fmt.Printf("  clear(xs)     len=%d cap=%d   zero the VALUES, keep the length: %v\n", len(b), cap(b), b)

	var c []int
	fmt.Printf("  xs = nil      len=%d cap=%d   release the memory\n", len(c), cap(c))

	fmt.Println()
	fmt.Println("the trap with reuse: if T contains a pointer, the stale values in")
	fmt.Println("[len:cap] stay reachable (example 2). Zero them before reslicing:")
	fmt.Println()

	name := "leaked"
	items := []item{{&name}, {&name}, {&name}}
	trimmed := items[:0]
	fmt.Printf("  items[:0] -> len=%d, but items[:cap][0].name is still %q\n",
		len(trimmed), *trimmed[:cap(trimmed)][0].name)

	clear(items)
	fmt.Printf("  clear(items) first -> items[:cap][0].name is now %v\n",
		trimmed[:cap(trimmed)][0].name)

	fmt.Println()
	fmt.Println("where this pattern belongs:")
	fmt.Println()
	fmt.Println("  - a per-request scratch buffer in a server loop")
	fmt.Println("  - the frontier/visited slices in a graph search you run repeatedly")
	fmt.Println("  - any 'build a slice, use it, discard it' step inside a hot loop")
	fmt.Println()
	fmt.Println("where it does NOT: when the result outlives the loop. Then you are")
	fmt.Println("handing every caller a view of the same array, and the next iteration")
	fmt.Println("overwrites their data. If the result escapes, clone it (lesson 05,")
	fmt.Println("example 10 -- the same bug in a different costume).")
	fmt.Println()
	fmt.Println("sync.Pool generalises this across goroutines; lesson 39 covers it.")
}
```

**Sample output:**

```
filtering 10,000 ints, 1000 times over:

                                        ns/op      allocs/op
  fresh slice each time                 11132             13
  reuse buf[:0]                          3568              0
  speedup                                3.1x

`buf[:0]` sets len to 0 and leaves cap alone. Every append then
writes into memory that is already there and already warm in cache.

the three ways to empty a slice, and what each one means:

  xs[:0]        len=0 cap=5   reuse the memory (this example)
  clear(xs)     len=5 cap=5   zero the VALUES, keep the length: [0 0 0 0 0]
  xs = nil      len=0 cap=0   release the memory

the trap with reuse: if T contains a pointer, the stale values in
[len:cap] stay reachable (example 2). Zero them before reslicing:

  items[:0] -> len=0, but items[:cap][0].name is still "leaked"
  clear(items) first -> items[:cap][0].name is now <nil>

where this pattern belongs:

  - a per-request scratch buffer in a server loop
  - the frontier/visited slices in a graph search you run repeatedly
  - any 'build a slice, use it, discard it' step inside a hot loop

where it does NOT: when the result outlives the loop. Then you are
handing every caller a view of the same array, and the next iteration
overwrites their data. If the result escapes, clone it (lesson 05,
example 10 -- the same bug in a different costume).

sync.Pool generalises this across goroutines; lesson 39 covers it.
```

**3.2× faster, 13 allocations down to 0.** `buf[:0]` sets len to 0 and leaves cap alone, so every
append writes into memory that's already there and already warm in cache.

The trap: if `T` contains a pointer, the stale values in `[len:cap]` stay reachable. `clear` first.
And never hand a caller a view of a buffer the next iteration will overwrite — that's lesson 05's
backtracking bug in a different costume.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::fmt::Display;
use std::hint::black_box;
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Slices that never shrink on their own were framed in lesson 04 as a leak.
// Turned around, that is the single best optimisation available to a hot loop:
// keep the whole backing buffer and reset the length, so the next round of
// pushes reuses memory you already have. In Go that is `buf = buf[:0]`; in Rust
// it is `buf.clear()`, which keeps the capacity too.
//
// This is the difference between "allocates every iteration" and "allocates
// once, ever".

// ---- allocation counter: stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0); // how many of those were realloc
static IN_PLACE: AtomicUsize = AtomicUsize::new(0); // realloc that kept the same address

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
        REALLOCS.fetch_add(1, Relaxed);
        let q = unsafe { System.realloc(p, l, new) };
        if q == p {
            IN_PLACE.fetch_add(1, Relaxed); // the allocator extended it: nothing was copied
        }
        q
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// allocation events per call of `f`, averaged over `runs` (AllocsPerRun)
fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> f64 {
    let a0 = ALLOCS.load(Relaxed);
    for _ in 0..runs {
        f();
    }
    (ALLOCS.load(Relaxed) - a0) as f64 / runs as f64
}

/// (allocation events, of which reallocs, of which extended in place) for one call of `f`
fn realloc_profile(f: impl FnOnce()) -> (usize, usize, usize) {
    let (a, r, i) = (
        ALLOCS.load(Relaxed),
        REALLOCS.load(Relaxed),
        IN_PLACE.load(Relaxed),
    );
    f();
    (
        ALLOCS.load(Relaxed) - a,
        REALLOCS.load(Relaxed) - r,
        IN_PLACE.load(Relaxed) - i,
    )
}

// ---- ns/op: stand-in for testing.Benchmark / b.Loop ----
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

/// fresh allocates a new Vec every call -- the obvious version.
fn fresh(src: &[i64]) -> Vec<i64> {
    let mut out = Vec::new();
    for &v in src {
        if v % 2 == 0 {
            out.push(v);
        }
    }
    out
}

// reuse pushes into a caller-supplied buffer; the caller calls buf.clear() first.
// Go has to RETURN the slice (append may hand back a new header, and forgetting to
// assign it loses the data). Rust takes `&mut Vec`, so the caller's buffer is
// updated in place and there is nothing to forget.
fn reuse(buf: &mut Vec<i64>, src: &[i64]) {
    for &v in src {
        if v % 2 == 0 {
            buf.push(v);
        }
    }
}

// Go's *string. Option so a slot can be emptied ("nil"); Rc so we can SEE how many
// owners the string has (Go's GC would only show you as a leak).
struct Item {
    name: Option<Rc<String>>,
}

/// A hand-rolled reusable buffer: `slots` is the backing array and `len` says how
/// many are live -- the same shape as Go's slice, and as the Vector in example 7.
struct Buf {
    slots: Vec<Item>,
    len: usize,
}

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    let src: Vec<i64> = (0..10_000).collect();

    println!("filtering 10,000 ints, 1000 times over:");
    println!();

    let t_fresh = ns_per_op(|| {
        black_box(fresh(black_box(&src)));
    });

    let mut buf: Vec<i64> = Vec::with_capacity(src.len());
    let t_reuse = ns_per_op(|| {
        buf.clear();
        reuse(&mut buf, black_box(&src));
        black_box(&buf);
    });

    let a_fresh = allocs_per_run(20, || {
        black_box(fresh(black_box(&src)));
    });
    let mut buf2: Vec<i64> = Vec::with_capacity(src.len());
    let a_reuse = allocs_per_run(20, || {
        buf2.clear();
        reuse(&mut buf2, black_box(&src));
        black_box(&buf2);
    });

    println!("  {:<28} {:>14} {:>14}", "", "ns/op", "allocs/op");
    println!(
        "  {:<28} {:>14.0} {:>14.0}",
        "fresh Vec each time", t_fresh, a_fresh
    );
    println!(
        "  {:<28} {:>14.0} {:>14.0}",
        "reuse buf.clear()", t_reuse, a_reuse
    );
    println!("  {:<28} {:>13.1}x", "speedup", t_fresh / t_reuse);

    println!();
    let (events, reallocs, in_place) = realloc_profile(|| {
        black_box(fresh(black_box(&src)));
    });
    println!("`buf.clear()` sets len to 0 and leaves capacity alone. Every push then");
    println!("writes into memory that is already allocated, so the allocation count");
    println!("above is the measured difference between the two rows.");
    println!();
    println!(
        "one call of fresh(): {events} allocation events ({} alloc + {reallocs} reallocs).",
        events - reallocs
    );
    println!("{in_place} of the reallocs came back at the same address: the allocator extended");
    println!("the buffer in place and nothing was copied. That is one plausible reason");
    println!("growing is cheap here; how much of the time gap comes from allocator calls,");
    println!("growth copies or cache warmth was not separated.");

    println!();
    println!("the three ways to empty a Vec, and what each one means:");
    println!();
    let mut xs = vec![1i64, 2, 3, 4, 5];
    xs.clear();
    println!(
        "  {:<15} len={} cap={}   reuse the memory (this example)",
        "xs.clear()",
        xs.len(),
        xs.capacity()
    );

    let mut b = vec![1i64, 2, 3, 4, 5];
    b.fill(0);
    println!(
        "  {:<15} len={} cap={}   zero the VALUES, keep the length: {}",
        "xs.fill(0)",
        b.len(),
        b.capacity(),
        list(&b)
    );

    let mut c = vec![1i64, 2, 3, 4, 5];
    let cap_before = c.capacity();
    c = Vec::new(); // the old buffer is freed right here: its owner was overwritten
    println!(
        "  {:<15} len={} cap={}   release the memory (it was cap={cap_before})",
        "xs = Vec::new()",
        c.len(),
        c.capacity()
    );

    println!();
    println!("the trap with reuse, in the shape Go has it: if T owns something, the stale");
    println!("values in [len..cap] stay alive in a hand-rolled buffer (example 7's `pop`");
    println!("uses mem::take for this reason). Empty the slots before resetting len:");
    println!();

    let name = Rc::new(String::from("leaked"));
    let item = || Item {
        name: Some(Rc::clone(&name)),
    };
    let mut hand = Buf {
        slots: vec![item(), item(), item()],
        len: 3,
    };
    println!(
        "  three items share one string: strong count = {}",
        Rc::strong_count(&name)
    );

    hand.len = 0;
    println!(
        "  hand.len = 0 -> len={}, but slots[0].name is still {:?} (strong count {})",
        hand.len,
        hand.slots[0].name,
        Rc::strong_count(&name)
    );

    for slot in &mut hand.slots {
        slot.name = None;
    }
    println!(
        "  empty the slots first -> slots[0].name is now {:?} (strong count {})",
        hand.slots[0].name,
        Rc::strong_count(&name)
    );

    // Vec::clear does that emptying for you: it DROPS the elements and keeps the buffer.
    let mut items = vec![item(), item(), item()];
    let before = Rc::strong_count(&name);
    items.clear();
    println!(
        "  Vec::clear() on a fresh Vec<Item> -> len={} cap={}, strong count {} -> {}",
        items.len(),
        items.capacity(),
        before,
        Rc::strong_count(&name)
    );

    println!();
    println!("where this pattern belongs:");
    println!();
    println!("  - a per-request scratch buffer in a server loop");
    println!("  - the frontier/visited vectors in a graph search you run repeatedly");
    println!("  - any 'build a Vec, use it, discard it' step inside a hot loop");
    println!();
    println!("where it does NOT: when the result outlives the loop. In Go that hands every");
    println!("caller a view of the same array and the next iteration overwrites their data.");
    println!("In Rust it does not compile: `results.push(&buf)` followed by the next");
    println!("`buf.clear()` is error[E0502] (cannot borrow `buf` as mutable because it is");
    println!("also borrowed as immutable). If the result escapes, clone it: `buf.clone()` or");
    println!("`buf.to_vec()` (lesson 05, example 10 -- the same bug, caught at compile time).");
    println!();
    println!("Go's sync.Pool generalises this across goroutines; lesson 39 covers it.");
}
```

**Rust sample output:**

```
filtering 10,000 ints, 1000 times over:

                                        ns/op      allocs/op
  fresh Vec each time                    4982             12
  reuse buf.clear()                      3944              0
  speedup                                1.3x

`buf.clear()` sets len to 0 and leaves capacity alone. Every push then
writes into memory that is already allocated, so the allocation count
above is the measured difference between the two rows.

one call of fresh(): 12 allocation events (1 alloc + 11 reallocs).
9 of the reallocs came back at the same address: the allocator extended
the buffer in place and nothing was copied. That is one plausible reason
growing is cheap here; how much of the time gap comes from allocator calls,
growth copies or cache warmth was not separated.

the three ways to empty a Vec, and what each one means:

  xs.clear()      len=0 cap=5   reuse the memory (this example)
  xs.fill(0)      len=5 cap=5   zero the VALUES, keep the length: [0 0 0 0 0]
  xs = Vec::new() len=0 cap=0   release the memory (it was cap=5)

the trap with reuse, in the shape Go has it: if T owns something, the stale
values in [len..cap] stay alive in a hand-rolled buffer (example 7's `pop`
uses mem::take for this reason). Empty the slots before resetting len:

  three items share one string: strong count = 4
  hand.len = 0 -> len=0, but slots[0].name is still Some("leaked") (strong count 4)
  empty the slots first -> slots[0].name is now None (strong count 1)
  Vec::clear() on a fresh Vec<Item> -> len=0 cap=3, strong count 4 -> 1

where this pattern belongs:

  - a per-request scratch buffer in a server loop
  - the frontier/visited vectors in a graph search you run repeatedly
  - any 'build a Vec, use it, discard it' step inside a hot loop

where it does NOT: when the result outlives the loop. In Go that hands every
caller a view of the same array and the next iteration overwrites their data.
In Rust it does not compile: `results.push(&buf)` followed by the next
`buf.clear()` is error[E0502] (cannot borrow `buf` as mutable because it is
also borrowed as immutable). If the result escapes, clone it: `buf.clone()` or
`buf.to_vec()` (lesson 05, example 10 -- the same bug, caught at compile time).

Go's sync.Pool generalises this across goroutines; lesson 39 covers it.
```

> **Go vs Rust:** Rust does not reproduce Go's headline: across several runs on a busy machine the reuse speedup was 1.1x to 1.4x (Go's sample: 3.1x), though the allocation counts do carry over (12 events to 0, Go: 13 to 0). The measured piece of the explanation is that most of `Vec`'s reallocs (9 of 11 here) extended the buffer in place instead of copying; whether Go's append can do that was not checked. Three Go-specific things change shape: `clear(xs)` zeroes values in Go but `Vec::clear` drops the elements (so `fill(0)` is the twin of Go's `clear`), the reuse function takes `&mut Vec` instead of returning the slice, and the pointer trap only exists in a hand-rolled buffer, because `Vec::clear` drops the elements immediately (strong count 4 to 1) while keeping the capacity. The escape bug is a compile error (E0502), not a runtime overwrite.

**Complexity:** both are Θ(n) · one allocates Θ(log n) times per call and the other **zero**

---

## 11. Three ways to lay out a grid

`🟡 medium` · *`[][]T` is a slice of slices*

Go has no 2-D slice. `[][]int` is one array of headers plus a separate array per row, scattered
wherever the allocator put them.

**Steps:**

1. Reproduce the shared-row bug first.
2. Build a 1000×1000 grid three ways and compare heap objects and scan time.
3. Repeat at 250,000×4 and explain why the ranking changes.

```go
package main

import (
	"fmt"
	"runtime"
	"testing"
)

// Go has no 2-D slice. `[][]int` is a slice OF slices: one array of headers,
// plus a separate array per row, scattered wherever the allocator put them.
//
// A flat []int with index math is one contiguous block. Same Theta(r*c) space,
// completely different layout -- which lesson 04 taught you to care about.

var sink int

// --- the row-of-slices version ---

func newJagged(rows, cols int) [][]int {
	g := make([][]int, rows)
	for i := range g {
		g[i] = make([]int, cols) // one allocation PER ROW
	}
	return g
}

// --- the flat version: one allocation, index math ---

type Grid struct {
	data []int
	rows int
	cols int
}

func NewGrid(rows, cols int) *Grid {
	return &Grid{data: make([]int, rows*cols), rows: rows, cols: cols}
}

func (g *Grid) At(r, c int) int { return g.data[r*g.cols+c] }
func (g *Grid) Set(r, c, v int) { g.data[r*g.cols+c] = v }
func (g *Grid) Row(r int) []int { return g.data[r*g.cols : (r+1)*g.cols] }

// --- the one-allocation jagged version: best of both ---

func newBacked(rows, cols int) [][]int {
	backing := make([]int, rows*cols) // ONE allocation
	g := make([][]int, rows)
	for i := range g {
		g[i] = backing[i*cols : (i+1)*cols : (i+1)*cols]
	}
	return g
}

func sumJagged(g [][]int) int {
	t := 0
	for _, row := range g {
		for _, v := range row {
			t += v
		}
	}
	return t
}

func (g *Grid) Sum() int {
	t := 0
	for _, v := range g.data { // one linear walk, no indirection at all
		t += v
	}
	return t
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func heapObjects(build func() any) uint64 {
	runtime.GC()
	var before runtime.MemStats
	runtime.ReadMemStats(&before)
	v := build()
	runtime.GC()
	var after runtime.MemStats
	runtime.ReadMemStats(&after)
	runtime.KeepAlive(v)
	return after.HeapObjects - before.HeapObjects
}

func main() {
	fmt.Println("the classic beginner bug first -- one row, shared by everybody:")
	fmt.Println()
	row := make([]int, 3)
	shared := [][]int{row, row, row} // all three headers point at ONE array
	shared[0][0] = 99
	fmt.Printf("  [][]int{row, row, row} then shared[0][0] = 99  ->  %v\n", shared)
	fmt.Println("  every row changed, because there is only one row.")
	fmt.Println("  make the rows inside the loop, or use a backing array (below).")

	fmt.Println()
	const rows, cols = 1000, 1000

	fmt.Printf("a %dx%d grid, three ways:\n\n", rows, cols)
	fmt.Printf("  %-26s %16s %14s %16s\n", "layout", "heap objects", "sum ns/op", "contiguous?")

	jObjs := heapObjects(func() any { return newJagged(rows, cols) })
	bObjs := heapObjects(func() any { return newBacked(rows, cols) })
	gObjs := heapObjects(func() any { return NewGrid(rows, cols) })

	jag := newJagged(rows, cols)
	bak := newBacked(rows, cols)
	grid := NewGrid(rows, cols)

	tJag := nsPerOp(func() { sink += sumJagged(jag) })
	tBak := nsPerOp(func() { sink += sumJagged(bak) })
	tGrid := nsPerOp(func() { sink += grid.Sum() })

	fmt.Printf("  %-26s %16d %14.0f %16s\n", "[][]int, row per alloc", jObjs, tJag, "no")
	fmt.Printf("  %-26s %16d %14.0f %16s\n", "[][]int, one backing array", bObjs, tBak, "yes")
	fmt.Printf("  %-26s %16d %14.0f %16s\n", "flat []int + index math", gObjs, tGrid, "yes")

	fmt.Println()
	fmt.Println("the heap-object column is dramatic; the TIME column is barely different.")
	fmt.Println("That is worth understanding rather than glossing over: at 1000 ints per")
	fmt.Println("row, each row is already 8 KB of contiguous memory, so the header")
	fmt.Println("indirection happens once per 1000 elements. It is noise.")
	fmt.Println()
	fmt.Println("make the rows SHORT and the same total size, and it becomes measurable:")
	fmt.Println()

	const rows2, cols2 = 250_000, 4
	jag2 := newJagged(rows2, cols2)
	bak2 := newBacked(rows2, cols2)
	grid2 := NewGrid(rows2, cols2)

	fmt.Printf("  a %dx%d grid (same %d ints, %d rows):\n\n", rows2, cols2, rows2*cols2, rows2)
	t2Jag := nsPerOp(func() { sink += sumJagged(jag2) })
	t2Bak := nsPerOp(func() { sink += sumJagged(bak2) })
	t2Grid := nsPerOp(func() { sink += grid2.Sum() })
	fmt.Printf("  %-26s %14.0f ns %10.1fx\n", "[][]int, row per alloc", t2Jag, t2Jag/t2Grid)
	fmt.Printf("  %-26s %14.0f ns %10.1fx\n", "[][]int, one backing array", t2Bak, t2Bak/t2Grid)
	fmt.Printf("  %-26s %14.0f ns %10.1fx\n", "flat []int + index math", t2Grid, 1.0)

	fmt.Println()
	fmt.Println("so the three findings are:")
	fmt.Println()
	fmt.Println("  1. the jagged version makes one allocation PER ROW. Every one is a")
	fmt.Println("     separate heap object the GC traces (lesson 04, example 12), and")
	fmt.Println("     that cost is there regardless of row width.")
	fmt.Println()
	fmt.Println("  2. the backed version keeps the [][]int ERGONOMICS -- g[r][c] still")
	fmt.Println("     works -- while being one contiguous block. Four lines, and it is")
	fmt.Println("     usually the right answer.")
	fmt.Println()
	fmt.Println("  3. the flat version wins on SCAN SPEED only when rows are short")
	fmt.Println("     enough that the per-row indirection is a real fraction of the")
	fmt.Println("     work -- and even then it is ~1.2x, not 10x. With wide rows,")
	fmt.Println("     choose on ALLOCATIONS, not on speed.")
	fmt.Println()
	fmt.Println("(the single-digit heap-object counts are all within noise of each")
	fmt.Println(" other -- the meaningful contrast is 997 against a handful.)")

	fmt.Println()
	fmt.Println("the index math, once:")
	fmt.Println()
	fmt.Println("  row-major (Go, C):     index = r*cols + c")
	fmt.Println("  column-major (Fortran): index = c*rows + r")
	fmt.Println()
	fmt.Println("row-major means a ROW is contiguous. Iterate rows in the outer loop")
	fmt.Println("and columns in the inner one, and you walk memory in order. Swap them")
	fmt.Println("and every step is a stride of `cols` -- lesson 04's example 7, and the")
	fmt.Println("reason a transposed loop can be several times slower for free.")

	fmt.Println()
	fmt.Println("this flat-array-plus-index-math shape comes back repeatedly:")
	fmt.Println("  a heap is a flat array with index math (lesson 20)")
	fmt.Println("  a DP table is a flat array with index math (lesson 31)")
	fmt.Println("  CSR adjacency is flat arrays with index math (lesson 04, example 16)")

	g2 := NewGrid(3, 4)
	g2.Set(1, 2, 7)
	fmt.Printf("\n  Grid(3,4).Set(1,2,7) -> At(1,2)=%d, Row(1)=%v\n", g2.At(1, 2), g2.Row(1))
}
```

**Sample output:**

```
the classic beginner bug first -- one row, shared by everybody:

  [][]int{row, row, row} then shared[0][0] = 99  ->  [[99 0 0] [99 0 0] [99 0 0]]
  every row changed, because there is only one row.
  make the rows inside the loop, or use a backing array (below).

a 1000x1000 grid, three ways:

  layout                         heap objects      sum ns/op      contiguous?
  [][]int, row per alloc                  997         274236               no
  [][]int, one backing array                3         270930              yes
  flat []int + index math                   9         260805              yes

the heap-object column is dramatic; the TIME column is barely different.
That is worth understanding rather than glossing over: at 1000 ints per
row, each row is already 8 KB of contiguous memory, so the header
indirection happens once per 1000 elements. It is noise.

make the rows SHORT and the same total size, and it becomes measurable:

  a 250000x4 grid (same 1000000 ints, 250000 rows):

  [][]int, row per alloc             326257 ns        1.2x
  [][]int, one backing array         326180 ns        1.2x
  flat []int + index math            264566 ns        1.0x

so the three findings are:

  1. the jagged version makes one allocation PER ROW. Every one is a
     separate heap object the GC traces (lesson 04, example 12), and
     that cost is there regardless of row width.

  2. the backed version keeps the [][]int ERGONOMICS -- g[r][c] still
     works -- while being one contiguous block. Four lines, and it is
     usually the right answer.

  3. the flat version wins on SCAN SPEED only when rows are short
     enough that the per-row indirection is a real fraction of the
     work -- and even then it is ~1.2x, not 10x. With wide rows,
     choose on ALLOCATIONS, not on speed.

(the single-digit heap-object counts are all within noise of each
 other -- the meaningful contrast is 997 against a handful.)

the index math, once:

  row-major (Go, C):     index = r*cols + c
  column-major (Fortran): index = c*rows + r

row-major means a ROW is contiguous. Iterate rows in the outer loop
and columns in the inner one, and you walk memory in order. Swap them
and every step is a stride of `cols` -- lesson 04's example 7, and the
reason a transposed loop can be several times slower for free.

this flat-array-plus-index-math shape comes back repeatedly:
  a heap is a flat array with index math (lesson 20)
  a DP table is a flat array with index math (lesson 31)
  CSR adjacency is flat arrays with index math (lesson 04, example 16)

  Grid(3,4).Set(1,2,7) -> At(1,2)=7, Row(1)=[0 0 7 0]
```

**997 heap objects versus a handful** — that's the real difference, and it's there regardless of row
width. The *time* difference is nearly nothing at 1000-wide rows (each row is already 8 KB of
contiguous memory) and only ~1.2× even at 4-wide.

So the four-line **backing-array** version is usually the right answer: it keeps `g[r][c]` syntax and
turns n+1 allocations into 2. Choose on allocations, not on speed.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::RefCell;
use std::fmt::Display;
use std::hint::black_box;
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Neither Go nor Rust has a built-in 2-D slice. `Vec<Vec<T>>` is a Vec OF Vecs:
// one array of (ptr, cap, len) headers, plus a separate heap array per row,
// scattered wherever the allocator put them.
//
// A flat Vec<i64> with index math is one contiguous block. Same Theta(r*c)
// space, completely different layout -- which lesson 04 taught you to care about.

// ---- allocation counter: stand-in for runtime.MemStats.HeapObjects ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

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

/// (allocation events while running `f`, its result). Exact -- no GC to wait for.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

// ---- ns/op: stand-in for testing.Benchmark / b.Loop ----
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

// --- the row-of-Vecs version ---

fn new_jagged(rows: usize, cols: usize) -> Vec<Vec<i64>> {
    // one allocation PER ROW (plus one for the outer Vec)
    (0..rows).map(|_| vec![0; cols]).collect()
}

// --- the flat version: one allocation, index math ---

struct Grid {
    data: Vec<i64>,
    rows: usize,
    cols: usize,
}

impl Grid {
    fn new(rows: usize, cols: usize) -> Grid {
        Grid {
            data: vec![0; rows * cols],
            rows,
            cols,
        }
    }

    // Index math alone would let at(0, cols) silently read at(1, 0): the flat
    // buffer cannot tell that c is out of range for the row. Check both.
    fn at(&self, r: usize, c: usize) -> i64 {
        assert!(r < self.rows && c < self.cols, "grid index out of range");
        self.data[r * self.cols + c]
    }

    fn set(&mut self, r: usize, c: usize, v: i64) {
        assert!(r < self.rows && c < self.cols, "grid index out of range");
        self.data[r * self.cols + c] = v;
    }

    fn row(&self, r: usize) -> &[i64] {
        assert!(r < self.rows, "grid row out of range");
        &self.data[r * self.cols..(r + 1) * self.cols]
    }

    fn sum(&self) -> i64 {
        self.data.iter().sum() // one linear walk, no indirection at all
    }

    // The same contiguous memory, but visited one row at a time: a separate short
    // loop per row, as the Vec-of-rows layouts have to do. Added so the timings can
    // tell "where the rows live" apart from "how many loops it takes".
    fn sum_by_rows(&self) -> i64 {
        self.data
            .chunks(self.cols)
            .map(|r| r.iter().sum::<i64>())
            .sum()
    }
}

// --- the one-allocation jagged version: best of both ---
//
// Go slices the rows out of one backing array with `backing[i*cols:(i+1)*cols:(i+1)*cols]`
// (the third index caps each row so an append cannot bleed into the next one).
// In Rust the rows are `&mut [i64]` borrowed from the backing array: `chunks_mut`
// hands out DISJOINT mutable pieces, which the borrow checker accepts without
// `unsafe`. A `&mut [T]` has no capacity and cannot grow, so there is nothing to cap.
// The backing array is owned by the caller, so the rows cannot outlive it.
fn backed_rows(backing: &mut [i64], cols: usize) -> Vec<&mut [i64]> {
    backing.chunks_mut(cols).collect()
}

// One sum for both `Vec<Vec<i64>>` and `Vec<&mut [i64]>`: anything that views as &[i64].
fn sum_jagged<R: AsRef<[i64]>>(g: &[R]) -> i64 {
    let mut t = 0;
    for row in g {
        for &v in row.as_ref() {
            t += v;
        }
    }
    t
}

fn fill<R: AsMut<[i64]>>(g: &mut [R]) {
    let mut next = 0;
    for row in g {
        for x in row.as_mut() {
            *x = next;
            next += 1;
        }
    }
}

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    println!("the classic beginner bug first -- one row, shared by everybody:");
    println!();
    // `vec![row, row, row]` does not compile: row is MOVED into the first slot, and
    //   error[E0382]: use of moved value: `row`
    // so the bug cannot happen by accident. To share one row you have to say so, with
    // reference counting and interior mutability; `vec![x; 3]` clones x three times,
    // and cloning an Rc clones the POINTER.
    let row = Rc::new(RefCell::new(vec![0i64; 3]));
    let shared = vec![row; 3]; // all three Rc's point at ONE array
    shared[0].borrow_mut()[0] = 99;
    let shown: Vec<String> = shared.iter().map(|r| list(&r.borrow())).collect();
    println!(
        "  vec![Rc<RefCell<Vec>>; 3] then shared[0][0] = 99  ->  [{}]",
        shown.join(" ")
    );
    println!("  every row changed, because there is only one row.");
    println!("  the ordinary spelling clones the CONTENTS instead:");
    let mut own = vec![vec![0i64; 3]; 3];
    own[0][0] = 99;
    let shown: Vec<String> = own.iter().map(|r| list(r)).collect();
    println!(
        "  vec![vec![0; 3]; 3] then own[0][0] = 99  ->  [{}]",
        shown.join(" ")
    );

    println!();
    const ROWS: usize = 1000;
    const COLS: usize = 1000;

    println!("a {ROWS}x{COLS} grid, three ways:\n");
    println!(
        "  {:<28} {:>16} {:>14} {:>16}",
        "layout", "allocations", "sum ns/op", "contiguous?"
    );

    let (j_allocs, mut jag) = allocs_of(|| new_jagged(ROWS, COLS));
    let (b_alloc_data, mut backing) = allocs_of(|| vec![0i64; ROWS * COLS]);
    let (b_alloc_rows, mut bak) = allocs_of(|| backed_rows(&mut backing, COLS));
    let b_allocs = b_alloc_data + b_alloc_rows;
    let (g_allocs, mut grid) = allocs_of(|| Grid::new(ROWS, COLS));

    // Fill all three with the same 0, 1, 2, ... so the sums mean something, and
    // check that the three layouts agree on the answer.
    fill(&mut jag);
    fill(&mut bak);
    for (i, x) in grid.data.iter_mut().enumerate() {
        *x = i as i64;
    }
    assert_eq!(sum_jagged(&jag), sum_jagged(&bak));
    assert_eq!(sum_jagged(&jag), grid.sum());

    let mut sink = 0i64;
    let t_jag = ns_per_op(|| sink = sink.wrapping_add(sum_jagged(black_box(&jag))));
    let t_bak = ns_per_op(|| sink = sink.wrapping_add(sum_jagged(black_box(&bak))));
    let t_grid = ns_per_op(|| sink = sink.wrapping_add(black_box(&grid).sum()));

    println!(
        "  {:<28} {:>16} {:>14.0} {:>16}",
        "Vec<Vec<i64>>, row per alloc", j_allocs, t_jag, "no"
    );
    println!(
        "  {:<28} {:>16} {:>14.0} {:>16}",
        "Vec<&mut [i64]>, one backing", b_allocs, t_bak, "yes"
    );
    println!(
        "  {:<28} {:>16} {:>14.0} {:>16}",
        "flat Vec<i64> + index math", g_allocs, t_grid, "yes"
    );

    println!();
    println!(
        "the allocation column is dramatic: {j_allocs} against {b_allocs} against {g_allocs}, and exact -- there is no"
    );
    println!("GC to wait for, so unlike a GC-sampled heap-object count it does not wobble. The");
    println!("TIME column is the same order of magnitude for all three, and the");
    println!("order changed from run to run on this shared machine: do not read a");
    println!("ranking out of it. At 1000 ints per row each row is already 8 KB of");
    println!("contiguous memory, so the hop from one row to the next happens once per");
    println!("thousand elements.");
    println!();
    println!("make the rows SHORT and the same total size, and it becomes measurable:");
    println!();

    // Free the big grids before building the next ones.
    drop(bak);
    drop(backing);
    drop(jag);
    drop(grid);

    const ROWS2: usize = 250_000;
    const COLS2: usize = 4;
    let mut jag2 = new_jagged(ROWS2, COLS2);
    let mut backing2 = vec![0i64; ROWS2 * COLS2];
    let mut bak2 = backed_rows(&mut backing2, COLS2);
    let mut grid2 = Grid::new(ROWS2, COLS2);
    fill(&mut jag2);
    fill(&mut bak2);
    for (i, x) in grid2.data.iter_mut().enumerate() {
        *x = i as i64;
    }

    println!(
        "  a {ROWS2}x{COLS2} grid (same {} ints, {ROWS2} rows):\n",
        ROWS2 * COLS2
    );
    let t2_jag = ns_per_op(|| sink = sink.wrapping_add(sum_jagged(black_box(&jag2))));
    let t2_bak = ns_per_op(|| sink = sink.wrapping_add(sum_jagged(black_box(&bak2))));
    let t2_grid = ns_per_op(|| sink = sink.wrapping_add(black_box(&grid2).sum()));
    let t2_rows = ns_per_op(|| sink = sink.wrapping_add(black_box(&grid2).sum_by_rows()));
    println!(
        "  {:<28} {:>14.0} ns {:>10.1}x",
        "Vec<Vec<i64>>, row per alloc",
        t2_jag,
        t2_jag / t2_grid
    );
    println!(
        "  {:<28} {:>14.0} ns {:>10.1}x",
        "Vec<&mut [i64]>, one backing",
        t2_bak,
        t2_bak / t2_grid
    );
    println!(
        "  {:<28} {:>14.0} ns {:>10.1}x",
        "flat Vec<i64> + index math", t2_grid, 1.0
    );
    println!(
        "  {:<28} {:>14.0} ns {:>10.1}x",
        "  same, summed per row",
        t2_rows,
        t2_rows / t2_grid
    );
    black_box(sink);

    println!();
    println!("so the findings are:");
    println!();
    println!(
        "  1. the jagged version makes one allocation PER ROW ({j_allocs} here, counted). Each"
    );
    println!("     is its own malloc and its own free on drop. No GC traces them, but");
    println!("     the cost is there regardless of row width.");
    println!();
    println!("  2. the backed version keeps the g[r][c] ERGONOMICS while being one");
    println!(
        "     contiguous block plus one Vec of row views ({b_allocs} allocations). The price is"
    );
    println!("     a borrow: the rows borrow `backing`, so it must outlive them and cannot");
    println!("     be resized or read while they are alive.");
    println!();
    println!(
        "  3. at 4 ints per row the flat sum was {:.1}x faster than the jagged layout",
        t2_jag / t2_grid
    );
    println!(
        "     and {:.1}x faster than the backed one; at 1000 per row the three were close",
        t2_bak / t2_grid
    );
    println!("     and in no stable order. The last row of the 4-wide table says where the");
    println!(
        "     gap comes from: the SAME flat memory summed one 4-element row at a time is {:.1}x",
        t2_rows / t2_grid
    );
    println!("     slower than one long loop, in the same range as the two Vec-of-rows");
    println!("     layouts -- and the backed rows are contiguous too. So at this width the");
    println!("     cost is the per-row loop, which never gets long enough to pay for its");
    println!("     setup, not where the rows live. (Plausible mechanism for why the one long");
    println!("     loop is so cheap: LLVM vectorised it. I checked the assembly of a");
    println!("     stand-alone copy of the two loops, not of this binary.)");
    println!();
    println!(
        "(the allocation counts are exact: {j_allocs} against {b_allocs} against {g_allocs}. The timings are not --"
    );
    println!(" trust the ratios at 4-wide, not the digits.)");

    println!();
    println!("the index math, once:");
    println!();
    println!("  row-major (Rust, Go, C):  index = r*cols + c");
    println!("  column-major (Fortran):   index = c*rows + r");
    println!();
    println!("row-major means a ROW is contiguous. Iterate rows in the outer loop");
    println!("and columns in the inner one, and you walk memory in order. Swap them");
    println!("and every step is a stride of `cols` -- lesson 04's example 7 measures");
    println!("what that costs.");

    println!();
    println!("this flat-array-plus-index-math shape comes back repeatedly:");
    println!("  a heap is a flat array with index math (lesson 20)");
    println!("  a DP table is a flat array with index math (lesson 31)");
    println!("  CSR adjacency is flat arrays with index math (lesson 04, example 16)");

    let mut g2 = Grid::new(3, 4);
    g2.set(1, 2, 7);
    println!(
        "\n  Grid(3,4).set(1,2,7) -> at(1,2)={}, row(1)={}",
        g2.at(1, 2),
        list(g2.row(1))
    );
}
```

**Rust sample output:**

```
the classic beginner bug first -- one row, shared by everybody:

  vec![Rc<RefCell<Vec>>; 3] then shared[0][0] = 99  ->  [[99 0 0] [99 0 0] [99 0 0]]
  every row changed, because there is only one row.
  the ordinary spelling clones the CONTENTS instead:
  vec![vec![0; 3]; 3] then own[0][0] = 99  ->  [[99 0 0] [0 0 0] [0 0 0]]

a 1000x1000 grid, three ways:

  layout                            allocations      sum ns/op      contiguous?
  Vec<Vec<i64>>, row per alloc             1001          92601               no
  Vec<&mut [i64]>, one backing                2          83929              yes
  flat Vec<i64> + index math                  1          82609              yes

the allocation column is dramatic: 1001 against 2 against 1, and exact -- there is no
GC to wait for, so unlike a GC-sampled heap-object count it does not wobble. The
TIME column is the same order of magnitude for all three, and the
order changed from run to run on this shared machine: do not read a
ranking out of it. At 1000 ints per row each row is already 8 KB of
contiguous memory, so the hop from one row to the next happens once per
thousand elements.

make the rows SHORT and the same total size, and it becomes measurable:

  a 250000x4 grid (same 1000000 ints, 250000 rows):

  Vec<Vec<i64>>, row per alloc         349546 ns        4.3x
  Vec<&mut [i64]>, one backing         325222 ns        4.0x
  flat Vec<i64> + index math            82129 ns        1.0x
    same, summed per row               329805 ns        4.0x

so the findings are:

  1. the jagged version makes one allocation PER ROW (1001 here, counted). Each
     is its own malloc and its own free on drop. No GC traces them, but
     the cost is there regardless of row width.

  2. the backed version keeps the g[r][c] ERGONOMICS while being one
     contiguous block plus one Vec of row views (2 allocations). The price is
     a borrow: the rows borrow `backing`, so it must outlive them and cannot
     be resized or read while they are alive.

  3. at 4 ints per row the flat sum was 4.3x faster than the jagged layout
     and 4.0x faster than the backed one; at 1000 per row the three were close
     and in no stable order. The last row of the 4-wide table says where the
     gap comes from: the SAME flat memory summed one 4-element row at a time is 4.0x
     slower than one long loop, in the same range as the two Vec-of-rows
     layouts -- and the backed rows are contiguous too. So at this width the
     cost is the per-row loop, which never gets long enough to pay for its
     setup, not where the rows live. (Plausible mechanism for why the one long
     loop is so cheap: LLVM vectorised it. I checked the assembly of a
     stand-alone copy of the two loops, not of this binary.)

(the allocation counts are exact: 1001 against 2 against 1. The timings are not --
 trust the ratios at 4-wide, not the digits.)

the index math, once:

  row-major (Rust, Go, C):  index = r*cols + c
  column-major (Fortran):   index = c*rows + r

row-major means a ROW is contiguous. Iterate rows in the outer loop
and columns in the inner one, and you walk memory in order. Swap them
and every step is a stride of `cols` -- lesson 04's example 7 measures
what that costs.

this flat-array-plus-index-math shape comes back repeatedly:
  a heap is a flat array with index math (lesson 20)
  a DP table is a flat array with index math (lesson 31)
  CSR adjacency is flat arrays with index math (lesson 04, example 16)

  Grid(3,4).set(1,2,7) -> at(1,2)=7, row(1)=[0 0 7 0]
```

> **Go vs Rust:** Two things the Go program does cannot be written in safe Rust, and the twin shows what replaces them. The shared-row bug does not happen by accident: `vec![row, row, row]` is a compile error (use of moved value), `vec![vec![0; 3]; 3]` clones the contents, and the Go output's `[[99 0 0] [99 0 0] [99 0 0]]` needs an explicit `Rc<RefCell<..>>`. And the backed grid is `Vec<&mut [i64]>` built with `chunks_mut` (disjoint mutable rows, no `unsafe`, nothing to cap because a slice cannot `append`), which borrows its backing array. Allocation counts are exact (1001 / 2 / 1; Go's HeapObjects reading said 997 / 3 / 9). The timings disagree with Go's prose: Go measured about 1.2x for the flat layout at 4-wide rows, here it was about 4x to 5x in every run, and the extra last row shows that the same contiguous memory summed row by row is just as slow, so in Rust the gap at this width comes from the per-row loop rather than from where the rows live. Go does not vectorise this loop, which is a plausible reason its gap is smaller (not measured).

**Complexity:** all Θ(r·c) space · the jagged version costs **one heap object per row**, which the GC traces on every cycle

---

> Next tier: [🔴 hard](3-hard.md) — where the Θ(n) shift goes to hide, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
