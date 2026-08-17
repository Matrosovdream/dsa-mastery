# Step 06 — Dynamic Arrays & Slices · 🟡 Medium

Examples **7–11**: building `Vector[T]`, deriving the amortized bound, and choosing a growth policy.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 7, 8 and 9 **count** rather than time, so they're deterministic. Examples 10 and 11 measure —
sample output from an Apple M4, Go 1.26.3.

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

**Complexity:** all Θ(r·c) space · the jagged version costs **one heap object per row**, which the GC traces on every cycle

---

> Next tier: [🔴 hard](3-hard.md) — where the Θ(n) shift goes to hide, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
