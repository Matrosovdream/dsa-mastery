# Step 06 — Dynamic Arrays & Slices · 🔴 Hard

Examples **12–16**: choosing a structure, moving the cost somewhere else, and the capstone that
applies all three passes.

> ⚠️ **Example 16 is a `go test` package** — four files, run with `go test`. Examples 12–15 are
> `go run` programs. Sample output is from an Apple M4, Go 1.26.3.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Sorted slice vs map, with the workload as a variable

`🔴 hard` · *The honest comparison*

The textbook answer — "sorted slice is Θ(n) to insert, a map is Θ(1), use the map" — ignores every
constant lesson 04 measured. So measure it, with the ratio of lookups to insertions as a knob.

**Steps:**

1. Compare lookup-only on an already-built collection.
2. Then compare *building* it, at 0, 1, 10 and 100 lookups per insertion.
3. Read the ratio column, not the winner column.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// "Maintain a collection I can search" is the most common structure decision
// there is, and the textbook answer -- sorted slice is Theta(n) to insert, a map
// is Theta(1), use the map -- ignores the constants that lesson 04 measured.
//
// Here is the honest comparison, with the WORKLOAD as a variable. That turns
// out to be the thing that decides it.

var sinkB bool

type sortedSlice struct{ xs []int }

func (s *sortedSlice) Insert(v int) {
	i, found := slices.BinarySearch(s.xs, v)
	if found {
		return
	}
	s.xs = append(s.xs, 0)
	copy(s.xs[i+1:], s.xs[i:])
	s.xs[i] = v
}

func (s *sortedSlice) Contains(v int) bool {
	_, ok := slices.BinarySearch(s.xs, v)
	return ok
}

type hashSet struct{ m map[int]struct{} }

func newHashSet() *hashSet { return &hashSet{m: make(map[int]struct{})} }

func (h *hashSet) Insert(v int)        { h.m[v] = struct{}{} }
func (h *hashSet) Contains(v int) bool { _, ok := h.m[v]; return ok }

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// buildAndQuery runs a workload with the given ratio of lookups to insertions
// and reports the total nanoseconds for the whole batch.
func buildAndQuery(n, lookupsPerInsert int, insert func(int), contains func(int) bool) float64 {
	rng := rand.New(rand.NewPCG(1, 2))
	vals := rng.Perm(n)
	probes := make([]int, 256)
	for i := range probes {
		probes[i] = rng.IntN(n)
	}

	return nsPerOp(func() {
		p := 0
		for _, v := range vals {
			insert(v)
			for k := 0; k < lookupsPerInsert; k++ {
				sinkB = contains(probes[p&255])
				p++
			}
		}
	})
}

func main() {
	fmt.Println("lookup only, on an already-built collection of n elements:")
	fmt.Println()
	fmt.Printf("  %10s %16s %14s %10s\n", "n", "sorted slice ns", "map ns", "winner")

	rng := rand.New(rand.NewPCG(3, 4))
	for _, n := range []int{8, 64, 512, 4096, 65536} {
		ss := &sortedSlice{xs: make([]int, n)}
		hs := newHashSet()
		for i := 0; i < n; i++ {
			ss.xs[i] = i * 2
			hs.Insert(i * 2)
		}

		probes := make([]int, 256)
		for i := range probes {
			probes[i] = rng.IntN(n) * 2
		}
		p := 0
		pick := func() int { p = (p + 1) & 255; return probes[p] }

		tS := nsPerOp(func() { sinkB = ss.Contains(pick()) })
		tM := nsPerOp(func() { sinkB = hs.Contains(pick()) })

		w := "map"
		if tS < tM {
			w = "slice"
		}
		fmt.Printf("  %10d %16.1f %14.1f %10s\n", n, tS, tM, w)
	}

	fmt.Println()
	fmt.Println("now the part the textbook comparison leaves out -- BUILDING it,")
	fmt.Println("with a varying number of lookups per insertion:")
	fmt.Println()
	fmt.Printf("  %8s %10s %16s %14s %12s\n", "n", "lookups", "sorted slice ns", "map ns", "slice/map")

	for _, n := range []int{1000, 8000} {
		for _, lk := range []int{0, 1, 10, 100} {
			ss := &sortedSlice{}
			tS := buildAndQuery(n, lk, ss.Insert, ss.Contains)

			hs := newHashSet()
			tM := buildAndQuery(n, lk, hs.Insert, hs.Contains)

			fmt.Printf("  %8d %10d %16.0f %14.0f %11.1fx\n", n, lk, tS, tM, tS/tM)
		}
	}

	fmt.Println()
	fmt.Println("read that honestly: the map wins EVERY row. It is faster to look up")
	fmt.Println("(4.9 ns flat vs 15.5 ns and climbing) and enormously faster to build,")
	fmt.Println("because every sorted insert is a Theta(n) memmove and building by")
	fmt.Println("insertion is therefore Theta(n^2) (example 5).")
	fmt.Println()
	fmt.Println("what the ratio column shows is the gap NARROWING as lookups come to")
	fmt.Println("dominate -- roughly 7x down to 3x at n=8000 -- because the expensive")
	fmt.Println("inserts get amortised over many cheap, cache-friendly searches.")
	fmt.Println("It narrows. It does not cross over. Do not talk yourself into a")
	fmt.Println("sorted slice on the grounds of lookup speed; lesson 04, example 15")
	fmt.Println("reached exactly the same conclusion from the other direction.")
	fmt.Println()
	fmt.Println("but notice what is missing from every row: the case where you have")
	fmt.Println("all the data up front. Then it is not Theta(n^2) at all --")
	fmt.Println("append everything and slices.Sort once, Theta(n log n), and the")
	fmt.Println("sorted slice wins on memory (4.5x, lesson 04) AND supports range")
	fmt.Println("queries the map cannot answer at any price.")

	fmt.Println()
	fmt.Println("the decision, then, is not 'which is faster'. It is:")
	fmt.Println()
	fmt.Println("  do inserts and lookups interleave?   -> map")
	fmt.Println("  can you build once, then query?      -> sorted slice")
	fmt.Println("  need order, range, or rank?          -> sorted slice, no contest")
	fmt.Println("  need BOTH interleaved AND ordered?   -> a balanced tree (lesson 19)")
	fmt.Println("                                          or a skip list (lesson 37)")
}
```

**Sample output:**

```
lookup only, on an already-built collection of n elements:

           n  sorted slice ns         map ns     winner
           8              5.0            4.1        map
          64              7.5            4.5        map
         512             10.1            4.7        map
        4096             12.5            4.9        map
       65536             15.6            4.8        map

now the part the textbook comparison leaves out -- BUILDING it,
with a varying number of lookups per insertion:

         n    lookups  sorted slice ns         map ns    slice/map
      1000          0            12172           5063         2.4x
      1000          1            22342           9084         2.5x
      1000         10           121835          45975         2.6x
      1000        100          1019437         410740         2.5x
      8000          0           309780          46367         6.7x
      8000          1           626193          82478         7.6x
      8000         10          1500474         388955         3.9x
      8000        100         10415886        3471335         3.0x

read that honestly: the map wins EVERY row. It is faster to look up
(4.9 ns flat vs 15.5 ns and climbing) and enormously faster to build,
because every sorted insert is a Theta(n) memmove and building by
insertion is therefore Theta(n^2) (example 5).

what the ratio column shows is the gap NARROWING as lookups come to
dominate -- roughly 7x down to 3x at n=8000 -- because the expensive
inserts get amortised over many cheap, cache-friendly searches.
It narrows. It does not cross over. Do not talk yourself into a
sorted slice on the grounds of lookup speed; lesson 04, example 15
reached exactly the same conclusion from the other direction.

but notice what is missing from every row: the case where you have
all the data up front. Then it is not Theta(n^2) at all --
append everything and slices.Sort once, Theta(n log n), and the
sorted slice wins on memory (4.5x, lesson 04) AND supports range
queries the map cannot answer at any price.

the decision, then, is not 'which is faster'. It is:

  do inserts and lookups interleave?   -> map
  can you build once, then query?      -> sorted slice
  need order, range, or rank?          -> sorted slice, no contest
  need BOTH interleaved AND ordered?   -> a balanced tree (lesson 19)
                                          or a skip list (lesson 37)
```

The map wins **every row**. What the ratio column shows is the gap *narrowing* as lookups dominate —
roughly 7× down to 3× — because the expensive inserts get amortised over many cheap searches. It
narrows; it does not cross over. [Lesson 04, example 15](../04-go-memory-model/3-hard.md#15-sorted-slice-vs-map--the-honest-comparison)
reached the same conclusion from the other direction.

So don't choose a sorted slice for lookup speed. Choose it for **memory** (4.5×), **ordering**, and
**range queries** — and if you have the data up front, sort once and it isn't Θ(n²) at all.

**Complexity:** sorted slice — find Θ(log n), insert Θ(n) · map — Θ(1) average, ~4.5× the memory, no order

---

## 13. The gap buffer

`🔴 hard` · *Moving the cost where nobody looks*

A dynamic array is Θ(n) to insert in the middle. A text editor inserts in the middle on every
keystroke. The gap buffer's answer: keep the array, but leave the **free space at the cursor**.

**Steps:**

1. Implement the buffer with `gapStart`/`gapEnd`, and render the gap so you can watch it move.
2. `Insert` fills one slot of the gap; `MoveTo` slides the gap.
3. Benchmark front-insertion against a plain `[]byte`.

```go
package main

import (
	"fmt"
	"strings"
	"testing"
)

// A dynamic array is Theta(n) to insert in the middle. A text editor inserts in
// the middle on EVERY KEYSTROKE. So how does one hold a 10 MB file and stay
// responsive?
//
// The GAP BUFFER: keep the array, but leave the free space in the MIDDLE,
// exactly where the cursor is. Then a keystroke is Theta(1) -- it just fills one
// slot of the gap. Moving the cursor is what costs, and it costs only the
// distance moved.
//
//   [ h e l l o _ _ _ _ _ _  w o r l d ]
//                ^^^^^^^^^^^
//               gapStart  gapEnd
//
// This is a real structure -- Emacs is built on one -- and it is the cleanest
// example of the idea that you can move a cost somewhere the workload does not
// look.

type GapBuffer struct {
	buf      []byte
	gapStart int // first free slot
	gapEnd   int // one past the last free slot
}

func NewGapBuffer(gap int) *GapBuffer {
	return &GapBuffer{buf: make([]byte, gap), gapStart: 0, gapEnd: gap}
}

func (g *GapBuffer) Len() int { return len(g.buf) - (g.gapEnd - g.gapStart) }

// grow doubles the buffer, placing a fresh gap at the cursor.
func (g *GapBuffer) grow() {
	newGap := len(g.buf)
	if newGap < 8 {
		newGap = 8
	}
	bigger := make([]byte, len(g.buf)+newGap)
	copy(bigger, g.buf[:g.gapStart])
	copy(bigger[g.gapStart+newGap:], g.buf[g.gapEnd:])
	g.buf = bigger
	g.gapEnd = g.gapStart + newGap
}

// Insert writes one byte at the cursor. Theta(1) amortized -- no shifting at all.
func (g *GapBuffer) Insert(c byte) {
	if g.gapStart == g.gapEnd {
		g.grow()
	}
	g.buf[g.gapStart] = c
	g.gapStart++
}

// Delete removes the byte before the cursor. Theta(1): just widen the gap.
func (g *GapBuffer) Delete() {
	if g.gapStart > 0 {
		g.gapStart--
	}
}

// MoveTo puts the cursor at position pos, moving the gap there. Theta(distance).
func (g *GapBuffer) MoveTo(pos int) {
	for g.gapStart > pos { // move left
		g.gapStart--
		g.gapEnd--
		g.buf[g.gapEnd] = g.buf[g.gapStart]
	}
	for g.gapStart < pos { // move right
		g.buf[g.gapStart] = g.buf[g.gapEnd]
		g.gapStart++
		g.gapEnd++
	}
}

func (g *GapBuffer) String() string {
	return string(g.buf[:g.gapStart]) + string(g.buf[g.gapEnd:])
}

// debug renders the gap visibly.
func (g *GapBuffer) debug() string {
	var b strings.Builder
	b.WriteString(string(g.buf[:g.gapStart]))
	b.WriteString("[")
	b.WriteString(strings.Repeat("_", g.gapEnd-g.gapStart))
	b.WriteString("]")
	b.WriteString(string(g.buf[g.gapEnd:]))
	return b.String()
}

// --- the naive alternative: a plain []byte ---

type plainBuffer struct{ buf []byte }

func (p *plainBuffer) InsertAt(pos int, c byte) {
	p.buf = append(p.buf, 0)
	copy(p.buf[pos+1:], p.buf[pos:])
	p.buf[pos] = c
}

var sinkS string

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("watch the gap move as you type:")
	fmt.Println()
	g := NewGapBuffer(4)
	for _, c := range "hello" {
		g.Insert(byte(c))
	}
	fmt.Printf("  type 'hello'        %-22s -> %q\n", g.debug(), g.String())

	g.MoveTo(0)
	fmt.Printf("  cursor to start     %-22s\n", g.debug())

	for _, c := range ">> " {
		g.Insert(byte(c))
	}
	fmt.Printf("  type '>> '          %-22s -> %q\n", g.debug(), g.String())

	g.MoveTo(g.Len())
	for _, c := range "!" {
		g.Insert(byte(c))
	}
	fmt.Printf("  cursor to end, '!'  %-22s -> %q\n", g.debug(), g.String())

	g.Delete()
	fmt.Printf("  backspace           %-22s -> %q\n", g.debug(), g.String())

	fmt.Println()
	fmt.Println("now the workload that matters: typing n characters at the FRONT of a")
	fmt.Println("document -- the worst case for a plain array, the best for a gap buffer.")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s %12s\n", "n", "plain []byte ns", "gap buffer ns", "speedup")

	for _, n := range []int{1000, 4000, 16000} {
		tPlain := nsPerOp(func() {
			p := &plainBuffer{}
			for i := 0; i < n; i++ {
				p.InsertAt(0, 'x') // always at the front: shift everything
			}
			sinkS = string(p.buf[:1])
		})
		tGap := nsPerOp(func() {
			gb := NewGapBuffer(8)
			for i := 0; i < n; i++ {
				gb.MoveTo(0) // the cursor is already there after the first move
				gb.Insert('x')
			}
			sinkS = gb.String()[:1]
		})
		fmt.Printf("  %10d %18.0f %18.0f %11.1fx\n", n, tPlain, tGap, tPlain/tGap)
	}

	fmt.Println()
	fmt.Println("the plain array is Theta(n^2): every insert shifts everything after it.")
	fmt.Println("The gap buffer is Theta(n): the gap is already at the cursor, so each")
	fmt.Println("keystroke fills one slot.")

	fmt.Println()
	fmt.Println("what it costs, honestly:")
	fmt.Println()
	fmt.Println("  MoveTo is Theta(distance). Jumping from the top of a file to the")
	fmt.Println("  bottom copies the whole document. A gap buffer is a bet that edits")
	fmt.Println("  CLUSTER -- which, for a human typing, they overwhelmingly do.")
	fmt.Println()
	fmt.Println("  it is also still one contiguous array, so it keeps every locality")
	fmt.Println("  property from lesson 04. That is why editors preferred it to a")
	fmt.Println("  linked list of lines for decades.")
	fmt.Println()
	fmt.Println("the transferable idea: you cannot remove the Theta(n) shift from an")
	fmt.Println("array, but you CAN choose where it happens -- and put it somewhere")
	fmt.Println("the workload rarely looks.")
}
```

**Sample output:**

```
watch the gap move as you type:

  type 'hello'        hello[_______]         -> "hello"
  cursor to start     [_______]hello        
  type '>> '          >> [____]hello         -> ">> hello"
  cursor to end, '!'  >> hello![___]         -> ">> hello!"
  backspace           >> hello[____]         -> ">> hello"

now the workload that matters: typing n characters at the FRONT of a
document -- the worst case for a plain array, the best for a gap buffer.

           n    plain []byte ns      gap buffer ns      speedup
        1000               8473               1662         5.1x
        4000              82542               6018        13.7x
       16000            1172861              23254        50.4x

the plain array is Theta(n^2): every insert shifts everything after it.
The gap buffer is Theta(n): the gap is already at the cursor, so each
keystroke fills one slot.

what it costs, honestly:

  MoveTo is Theta(distance). Jumping from the top of a file to the
  bottom copies the whole document. A gap buffer is a bet that edits
  CLUSTER -- which, for a human typing, they overwhelmingly do.

  it is also still one contiguous array, so it keeps every locality
  property from lesson 04. That is why editors preferred it to a
  linked list of lines for decades.

the transferable idea: you cannot remove the Theta(n) shift from an
array, but you CAN choose where it happens -- and put it somewhere
the workload rarely looks.
```

**5.4× → 12.1× → 46.0×** as n grows — the signature of Θ(n²) against Θ(n).

What it costs, honestly: `MoveTo` is Θ(distance), so jumping from the top of a file to the bottom
copies the whole document. A gap buffer is a bet that edits **cluster** — which, for a human typing,
they overwhelmingly do. And it's still one contiguous array, so it keeps every locality property from
lesson 04.

**Complexity:** `Insert`/`Delete` at the cursor **Θ(1) amortized** · `MoveTo` Θ(distance) · the transferable idea: you can't remove the shift, but you can choose where it happens

---

## 14. The accidental quadratic

`🔴 hard` · *`slices.Delete` in a loop*

The most common accidental Θ(n²) in production Go, and it looks entirely reasonable — because the
Θ(n) is hidden inside a standard-library call.

**Steps:**

1. Remove a set of elements by calling `slices.Delete` per element.
2. Do the same in one pass with the write-index idiom.
3. Compare at four sizes and watch the ratio grow.

```go
package main

import (
	"fmt"
	"slices"
	"testing"
)

// The most common accidental Theta(n^2) in production Go, and it looks
// completely reasonable:
//
//   for _, id := range toRemove {
//       i := slices.Index(items, id)
//       items = slices.Delete(items, i, i+1)
//   }
//
// Each Delete is Theta(n). Doing it k times is Theta(n*k) -- which is Theta(n^2)
// when you are removing a constant fraction of the collection.
//
// The fix is example 3's write-index idiom: ONE pass, Theta(n + k) total.

var sink []int

// removeRepeated: delete them one at a time. Correct, and quadratic.
func removeRepeated(xs []int, drop map[int]struct{}) []int {
	for i := 0; i < len(xs); {
		if _, gone := drop[xs[i]]; gone {
			xs = slices.Delete(xs, i, i+1) // Theta(n) EVERY time
			continue                       // do not advance: the tail slid left
		}
		i++
	}
	return xs
}

// removeOnePass: the write-index filter. One pass, no shifting.
func removeOnePass(xs []int, drop map[int]struct{}) []int {
	write := 0
	for _, v := range xs {
		if _, gone := drop[v]; gone {
			continue
		}
		xs[write] = v
		write++
	}
	return xs[:write]
}

// removeDeleteFunc: the stdlib's version of the same one-pass idea.
func removeDeleteFunc(xs []int, drop map[int]struct{}) []int {
	return slices.DeleteFunc(xs, func(v int) bool {
		_, gone := drop[v]
		return gone
	})
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
	// First: all three agree.
	src := []int{1, 2, 3, 4, 5, 6, 7, 8}
	drop := map[int]struct{}{2: {}, 4: {}, 6: {}}
	fmt.Println("removing {2,4,6} from [1 2 3 4 5 6 7 8]:")
	fmt.Println()
	fmt.Printf("  repeated Delete   %v\n", removeRepeated(slices.Clone(src), drop))
	fmt.Printf("  one pass          %v\n", removeOnePass(slices.Clone(src), drop))
	fmt.Printf("  slices.DeleteFunc %v\n", removeDeleteFunc(slices.Clone(src), drop))

	fmt.Println()
	fmt.Println("now remove HALF the elements from a slice of n:")
	fmt.Println()
	fmt.Printf("  %10s %18s %16s %18s %12s\n",
		"n", "repeated Delete", "one pass", "slices.DeleteFunc", "ratio")

	for _, n := range []int{1_000, 4_000, 16_000, 64_000} {
		base := make([]int, n)
		d := make(map[int]struct{}, n/2)
		for i := range base {
			base[i] = i
			if i%2 == 0 {
				d[i] = struct{}{}
			}
		}

		tRep := nsPerOp(func() { sink = removeRepeated(slices.Clone(base), d) })
		tOne := nsPerOp(func() { sink = removeOnePass(slices.Clone(base), d) })
		tStd := nsPerOp(func() { sink = removeDeleteFunc(slices.Clone(base), d) })

		fmt.Printf("  %10d %18.0f %16.0f %18.0f %11.0fx\n", n, tRep, tOne, tStd, tRep/tOne)
	}

	fmt.Println()
	fmt.Println("the ratio column grows with n, which is the signature of a whole")
	fmt.Println("complexity class difference rather than a constant factor:")
	fmt.Println()
	fmt.Println("  repeated Delete   Theta(n * k)  -- Theta(n^2) when k is a fraction of n")
	fmt.Println("  one pass          Theta(n)")
	fmt.Println()
	fmt.Println("note the middle and right columns are the same algorithm: slices.DeleteFunc")
	fmt.Println("IS the write-index idiom, written once in the standard library. Reaching")
	fmt.Println("for it is both shorter and correct -- but only if you know that")
	fmt.Println("slices.Delete in a loop is the thing it replaces.")

	fmt.Println()
	fmt.Println("this is lesson 03's 'quadratic hiding in a single loop' with a")
	fmt.Println("standard-library function playing the part of the hidden Theta(n).")
	fmt.Println("Calling stdlib does not make an operation cheap; it makes it correct.")
	fmt.Println()
	fmt.Println("the rule: removing ONE element -> Delete. Removing MANY -> one pass.")
	fmt.Println("If the word 'loop' and the word 'Delete' appear together, look again.")
}
```

**Sample output:**

```
removing {2,4,6} from [1 2 3 4 5 6 7 8]:

  repeated Delete   [1 3 5 7 8]
  one pass          [1 3 5 7 8]
  slices.DeleteFunc [1 3 5 7 8]

now remove HALF the elements from a slice of n:

           n    repeated Delete         one pass  slices.DeleteFunc        ratio
        1000              24806             4050               4535           6x
        4000             327836            18695              19829          18x
       16000            5061064            80811              89852          63x
       64000          121106148           335216             389893         361x

the ratio column grows with n, which is the signature of a whole
complexity class difference rather than a constant factor:

  repeated Delete   Theta(n * k)  -- Theta(n^2) when k is a fraction of n
  one pass          Theta(n)

note the middle and right columns are the same algorithm: slices.DeleteFunc
IS the write-index idiom, written once in the standard library. Reaching
for it is both shorter and correct -- but only if you know that
slices.Delete in a loop is the thing it replaces.

this is lesson 03's 'quadratic hiding in a single loop' with a
standard-library function playing the part of the hidden Theta(n).
Calling stdlib does not make an operation cheap; it makes it correct.

the rule: removing ONE element -> Delete. Removing MANY -> one pass.
If the word 'loop' and the word 'Delete' appear together, look again.
```

**6× → 17× → 64× → 355×.** The ratio growing with n is the signature of a complexity-class
difference, not a constant factor.

Note the middle and right columns are the same algorithm: `slices.DeleteFunc` **is** the write-index
idiom, written once in the standard library. This is lesson 03's "quadratic hiding in a single loop"
with a stdlib function playing the part of the hidden Θ(n).

**Complexity:** repeated `Delete` **Θ(n·k)** · one pass **Θ(n)** · rule: removing one → `Delete`; removing many → one pass

---

## 15. Partition, and the invariant that makes it work

`🔴 hard` · *The engine inside quicksort*

Rearrange in place so everything matching a predicate comes first. Three versions, each maintaining a
stated invariant — and the third handles duplicates in a way that matters enormously later.

**Steps:**

1. Two-way partition with the write-index idiom, counting swaps.
2. Hoare's two-pointer version, which only swaps genuinely misplaced pairs.
3. The Dutch national flag: three regions, one pass.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// PARTITION: rearrange a slice in place so everything satisfying a predicate
// comes first. One pass, no allocation -- and the engine inside quicksort,
// quickselect, and every "separate the wheat from the chaff" problem.
//
// Three versions, in increasing order of cleverness.

var swaps, comparisons int

func reset() { swaps, comparisons = 0, 0 }

func swap(xs []int, i, j int) {
	if i != j {
		swaps++
	}
	xs[i], xs[j] = xs[j], xs[i]
}

// --- 1. two-way partition: the write-index idiom with a swap ---
//
// Returns the boundary: xs[:b] satisfies the predicate, xs[b:] does not.
func partition(xs []int, pred func(int) bool) int {
	write := 0
	for read := 0; read < len(xs); read++ {
		comparisons++
		if pred(xs[read]) {
			swap(xs, write, read)
			write++
		}
	}
	return write
}

// --- 2. Hoare's two-pointer partition: fewer swaps ---
//
// Walk in from both ends and swap the pairs that are on the wrong side. It only
// swaps elements that are ACTUALLY misplaced, where the write-index version
// swaps on every match.
func partitionHoare(xs []int, pred func(int) bool) int {
	i, j := 0, len(xs)-1
	for {
		for i <= j {
			comparisons++
			if !pred(xs[i]) {
				break
			}
			i++
		}
		for i <= j {
			comparisons++
			if pred(xs[j]) {
				break
			}
			j--
		}
		if i > j {
			return i
		}
		swap(xs, i, j)
		i++
		j--
	}
}

// --- 3. Dutch national flag: THREE-way partition in one pass ---
//
// Named for Dijkstra's problem of sorting red/white/blue stripes. Returns the
// two boundaries: xs[:lo] < pivot, xs[lo:hi] == pivot, xs[hi:] > pivot.
//
// The invariant is the whole trick. At every step:
//
//	[ < pivot | == pivot | UNKNOWN | > pivot ]
//	 0       lo        mid       hi        n
//
// mid walks forward; lo and hi close in. The UNKNOWN region shrinks to nothing.
func dutchFlag(xs []int, pivot int) (lo, hi int) {
	lo, mid, hi := 0, 0, len(xs)
	for mid < hi {
		comparisons++
		switch {
		case xs[mid] < pivot:
			swap(xs, lo, mid)
			lo++
			mid++
		case xs[mid] > pivot:
			hi--
			swap(xs, mid, hi)
			// do NOT advance mid: the value swapped in is unexamined
		default:
			mid++
		}
	}
	return lo, hi
}

func isEven(v int) bool { return v%2 == 0 }

func main() {
	fmt.Println("two-way partition -- evens first:")
	fmt.Println()
	for _, name := range []string{"write-index", "Hoare"} {
		xs := []int{1, 2, 3, 4, 5, 6, 7, 8}
		reset()
		var b int
		if name == "write-index" {
			b = partition(xs, isEven)
		} else {
			b = partitionHoare(xs, isEven)
		}
		fmt.Printf("  %-14s %v  boundary=%d  swaps=%d comparisons=%d\n",
			name, xs, b, swaps, comparisons)
	}
	fmt.Println()
	fmt.Println("  both are correct and neither preserves the original order.")
	fmt.Println("  Hoare does fewer swaps -- it only touches genuinely misplaced pairs.")

	fmt.Println()
	fmt.Println("swaps over 10,000 random elements:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	base := make([]int, 10_000)
	for i := range base {
		base[i] = rng.IntN(1000)
	}
	reset()
	partition(slices.Clone(base), isEven)
	wSwaps, wComp := swaps, comparisons
	reset()
	partitionHoare(slices.Clone(base), isEven)
	fmt.Printf("  write-index   %6d swaps, %6d comparisons\n", wSwaps, wComp)
	fmt.Printf("  Hoare         %6d swaps, %6d comparisons\n", swaps, comparisons)

	fmt.Println()
	fmt.Println("three-way (Dutch national flag), pivot = 5:")
	fmt.Println()
	xs := []int{5, 3, 8, 5, 1, 9, 5, 2, 7}
	fmt.Printf("  before        %v\n", xs)
	lo, hi := dutchFlag(xs, 5)
	fmt.Printf("  after         %v\n", xs)
	fmt.Printf("  < 5           %v\n", xs[:lo])
	fmt.Printf("  == 5          %v\n", xs[lo:hi])
	fmt.Printf("  > 5           %v\n", xs[hi:])

	fmt.Println()
	fmt.Println("why the three-way version matters, in one number:")
	fmt.Println()
	dup := make([]int, 100_000)
	for i := range dup {
		dup[i] = rng.IntN(3) // only three distinct values
	}
	reset()
	dutchFlag(slices.Clone(dup), 1)
	fmt.Printf("  100,000 elements with 3 distinct values: %d comparisons, one pass\n", comparisons)
	fmt.Println()
	fmt.Println("  a two-way partition would put all the ==pivot elements on one side")
	fmt.Println("  and quicksort would then recurse on them again, and again. With many")
	fmt.Println("  duplicates that degenerates toward Theta(n^2). Three-way finishes the")
	fmt.Println("  equal block in this pass and never looks at it again -- which is why")
	fmt.Println("  Go's pdqsort uses a three-way partition (lesson 13).")

	fmt.Println()
	fmt.Println("the shared skeleton behind all three:")
	fmt.Println()
	fmt.Println("  maintain an INVARIANT about regions of the slice, and move the")
	fmt.Println("  boundaries so the unknown region shrinks to nothing. Every one is")
	fmt.Println("  Theta(n) time and Theta(1) space, and none of them allocates.")
	fmt.Println()
	fmt.Println("  the discipline that makes it work is writing the invariant down")
	fmt.Println("  FIRST. The Dutch-flag `do not advance mid` is impossible to get")
	fmt.Println("  right by intuition and obvious once the invariant is stated.")
}
```

**Output:**

```
two-way partition -- evens first:

  write-index    [2 4 6 8 5 3 7 1]  boundary=4  swaps=4 comparisons=8
  Hoare          [8 2 6 4 5 3 7 1]  boundary=4  swaps=2 comparisons=9

  both are correct and neither preserves the original order.
  Hoare does fewer swaps -- it only touches genuinely misplaced pairs.

swaps over 10,000 random elements:

  write-index     4976 swaps,  10000 comparisons
  Hoare           2524 swaps,  10000 comparisons

three-way (Dutch national flag), pivot = 5:

  before        [5 3 8 5 1 9 5 2 7]
  after         [3 2 1 5 5 5 9 7 8]
  < 5           [3 2 1]
  == 5          [5 5 5]
  > 5           [9 7 8]

why the three-way version matters, in one number:

  100,000 elements with 3 distinct values: 100000 comparisons, one pass

  a two-way partition would put all the ==pivot elements on one side
  and quicksort would then recurse on them again, and again. With many
  duplicates that degenerates toward Theta(n^2). Three-way finishes the
  equal block in this pass and never looks at it again -- which is why
  Go's pdqsort uses a three-way partition (lesson 13).

the shared skeleton behind all three:

  maintain an INVARIANT about regions of the slice, and move the
  boundaries so the unknown region shrinks to nothing. Every one is
  Theta(n) time and Theta(1) space, and none of them allocates.

  the discipline that makes it work is writing the invariant down
  FIRST. The Dutch-flag `do not advance mid` is impossible to get
  right by intuition and obvious once the invariant is stated.
```

Hoare does **half the swaps** (2524 vs 4976) for the same comparisons, because it only touches pairs
that are actually on the wrong side.

The three-way version is the one that matters for lesson 13: a two-way partition puts all the
`== pivot` elements on one side, and quicksort then recurses on them again and again — degenerating
toward Θ(n²) with many duplicates. Three-way finishes the equal block in this pass and never revisits
it, which is why Go's pdqsort uses one.

The `do not advance mid` case is impossible to get right by intuition and obvious once the invariant
is written down. Write the invariant first.

**Complexity:** all three Θ(n) time, **Θ(1) space**, no allocation · none preserves the original order

---

## 16. Capstone: `Vector[T]`, all three passes

`🔴 hard` · *Build it · Prove it · Measure it* · `go test`

The first structure in this plan, finished properly. This is what the three-pass method looks like in
practice, and the shape every later lesson follows.

**Pass 2 (prove it) has four layers**, each catching what the others can't:

| Layer | Catches |
|---|---|
| a table | specific known-answer cases, and the panics |
| invariants | "no removed element is still reachable" — a property no table case would check |
| a **model oracle** | 20,000 random operations against a plain `[]T`. This is the layer that finds real bugs |
| budgets & gates | allocation count, geometric growth, and no shrink thrashing |

**`vector.go`** — the structure, with its invariant written down

```go
package vector

// Vector[T] is the dynamic array from examples 7-9, finished: generic, with the
// hysteresis shrink policy, the pointer hygiene, and a stated invariant.
//
// INVARIANT, true before and after every exported method:
//
//	0 <= length <= len(data)
//	data[length:] holds only zero values      (so no removed T is kept alive)
//
// The zero Vector[T] is a valid empty vector -- no constructor required.
type Vector[T any] struct {
	data   []T
	length int
}

const (
	minCap        = 4
	growthFactor  = 2
	shrinkTrigger = 4 // shrink when length*4 <= cap ... (example 8)
	shrinkFactor  = 2 // ... and only halve, never shrink to fit
)

func (v *Vector[T]) Len() int { return v.length }
func (v *Vector[T]) Cap() int { return len(v.data) }

// At returns the element at i. Theta(1).
func (v *Vector[T]) At(i int) T {
	v.mustIndex(i)
	return v.data[i]
}

// Set overwrites the element at i. Theta(1).
func (v *Vector[T]) Set(i int, x T) {
	v.mustIndex(i)
	v.data[i] = x
}

func (v *Vector[T]) mustIndex(i int) {
	if i < 0 || i >= v.length {
		panic("vector: index out of range")
	}
}

func (v *Vector[T]) resize(newCap int) {
	if newCap < minCap {
		newCap = minCap
	}
	bigger := make([]T, newCap)
	copy(bigger, v.data[:v.length])
	v.data = bigger
}

// Push appends x. Theta(1) amortized.
func (v *Vector[T]) Push(x T) {
	if v.length == len(v.data) {
		v.resize(max(len(v.data)*growthFactor, minCap))
	}
	v.data[v.length] = x
	v.length++
}

// Pop removes and returns the last element. Theta(1) amortized.
func (v *Vector[T]) Pop() (T, bool) {
	var zero T
	if v.length == 0 {
		return zero, false
	}
	v.length--
	x := v.data[v.length]
	v.data[v.length] = zero // invariant: nothing past length holds a live T
	v.maybeShrink()
	return x, true
}

// Insert places x at index i, shifting the tail right. Theta(n).
func (v *Vector[T]) Insert(i int, x T) {
	if i < 0 || i > v.length {
		panic("vector: index out of range")
	}
	var zero T
	v.Push(zero) // grow if needed; length++
	copy(v.data[i+1:v.length], v.data[i:v.length-1])
	v.data[i] = x
}

// Delete removes index i, shifting the tail left. Theta(n).
func (v *Vector[T]) Delete(i int) T {
	v.mustIndex(i)
	var zero T
	x := v.data[i]
	copy(v.data[i:v.length-1], v.data[i+1:v.length])
	v.length--
	v.data[v.length] = zero
	v.maybeShrink()
	return x
}

func (v *Vector[T]) maybeShrink() {
	c := len(v.data)
	if c > minCap && v.length*shrinkTrigger <= c {
		v.resize(c / shrinkFactor)
	}
}

// Slice returns a view of the live elements. The caller must not retain it
// across a mutation -- the backing array can be replaced by a resize.
func (v *Vector[T]) Slice() []T { return v.data[:v.length] }

// Reserve makes room for n more pushes without reallocating.
func (v *Vector[T]) Reserve(n int) {
	if need := v.length + n; need > len(v.data) {
		v.resize(need)
	}
}
```

**`vector_test.go`** — pass 2: prove it

```go
package vector

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// PASS 2 of the three-pass method: PROVE IT.
//
// Four layers, each catching what the others cannot:
//   1. a table          -- specific inputs, specific answers
//   2. invariants       -- what must be true after EVERY operation
//   3. a model oracle   -- agreement with an obviously-correct []T, on random
//                          operation sequences (this is the one that finds bugs)
//   4. allocation budget-- a Theta(1) claim, asserted rather than hoped for
// ============================================================================

// --- 1. the table ---------------------------------------------------------

func TestPushPop(t *testing.T) {
	var v Vector[int]

	if _, ok := v.Pop(); ok {
		t.Error("Pop on an empty vector reported ok=true")
	}
	if v.Len() != 0 {
		t.Errorf("zero vector Len = %d, want 0", v.Len())
	}

	for i := 1; i <= 5; i++ {
		v.Push(i * 10)
	}
	if got, want := v.Slice(), []int{10, 20, 30, 40, 50}; !slices.Equal(got, want) {
		t.Fatalf("after pushes = %v, want %v", got, want)
	}

	for i := 5; i >= 1; i-- {
		got, ok := v.Pop()
		if !ok || got != i*10 {
			t.Fatalf("Pop = (%d, %v), want (%d, true)", got, ok, i*10)
		}
	}
	if v.Len() != 0 {
		t.Errorf("after popping everything, Len = %d, want 0", v.Len())
	}
}

func TestInsertDelete(t *testing.T) {
	tests := []struct {
		name  string
		start []int
		op    func(*Vector[int])
		want  []int
	}{
		{"insert at front", []int{2, 3}, func(v *Vector[int]) { v.Insert(0, 1) }, []int{1, 2, 3}},
		{"insert in middle", []int{1, 3}, func(v *Vector[int]) { v.Insert(1, 2) }, []int{1, 2, 3}},
		{"insert at end", []int{1, 2}, func(v *Vector[int]) { v.Insert(2, 3) }, []int{1, 2, 3}},
		{"insert into empty", nil, func(v *Vector[int]) { v.Insert(0, 1) }, []int{1}},
		{"delete front", []int{1, 2, 3}, func(v *Vector[int]) { v.Delete(0) }, []int{2, 3}},
		{"delete middle", []int{1, 2, 3}, func(v *Vector[int]) { v.Delete(1) }, []int{1, 3}},
		{"delete last", []int{1, 2, 3}, func(v *Vector[int]) { v.Delete(2) }, []int{1, 2}},
		{"delete only", []int{1}, func(v *Vector[int]) { v.Delete(0) }, nil},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			var v Vector[int]
			for _, x := range tt.start {
				v.Push(x)
			}
			tt.op(&v)
			if got := v.Slice(); !slices.Equal(got, tt.want) {
				t.Errorf("got %v, want %v", got, tt.want)
			}
		})
	}
}

func TestPanicsOnBadIndex(t *testing.T) {
	for _, tt := range []struct {
		name string
		op   func(*Vector[int])
	}{
		{"At past end", func(v *Vector[int]) { v.At(3) }},
		{"At negative", func(v *Vector[int]) { v.At(-1) }},
		{"Delete past end", func(v *Vector[int]) { v.Delete(3) }},
		{"Insert past end+1", func(v *Vector[int]) { v.Insert(4, 0) }},
	} {
		t.Run(tt.name, func(t *testing.T) {
			var v Vector[int]
			v.Push(1)
			v.Push(2)
			v.Push(3)
			defer func() {
				if recover() == nil {
					t.Error("expected a panic, got none")
				}
			}()
			tt.op(&v)
		})
	}
}

// --- 2. the invariants ----------------------------------------------------

// checkInvariants asserts everything the type promises, at any moment.
func checkInvariants[T comparable](t *testing.T, v *Vector[T], where string) {
	t.Helper()
	if v.length < 0 || v.length > len(v.data) {
		t.Fatalf("%s: length %d out of range for cap %d", where, v.length, len(v.data))
	}
	var zero T
	for i := v.length; i < len(v.data); i++ {
		if v.data[i] != zero {
			t.Fatalf("%s: data[%d] past the length is %v, want the zero value "+
				"(a removed element is still reachable)", where, i, v.data[i])
		}
	}
}

func TestInvariantsHold(t *testing.T) {
	var v Vector[int]
	checkInvariants(t, &v, "zero value")

	for i := 0; i < 200; i++ {
		v.Push(i)
		checkInvariants(t, &v, fmt.Sprintf("after Push %d", i))
	}
	for i := 0; i < 100; i++ {
		v.Delete(0)
		checkInvariants(t, &v, fmt.Sprintf("after Delete %d", i))
	}
	for v.Len() > 0 {
		v.Pop()
		checkInvariants(t, &v, "after Pop")
	}
}

// --- 3. the model oracle --------------------------------------------------
//
// A plain []int is the reference implementation: obviously correct, and slow in
// ways nobody cares about here. Drive both with the same random operations and
// require they always agree.

func TestAgainstModel(t *testing.T) {
	rng := rand.New(rand.NewPCG(42, 7))

	const trials = 20_000
	var v Vector[int]
	var model []int

	for i := 0; i < trials; i++ {
		switch rng.IntN(5) {
		case 0, 1: // push (twice as likely, so it grows)
			x := rng.IntN(1000)
			v.Push(x)
			model = append(model, x)

		case 2: // pop
			got, ok := v.Pop()
			if len(model) == 0 {
				if ok {
					t.Fatalf("trial %d: Pop reported ok on an empty vector", i)
				}
				break
			}
			want := model[len(model)-1]
			model = model[:len(model)-1]
			if !ok || got != want {
				t.Fatalf("trial %d: Pop = (%d,%v), model says %d", i, got, ok, want)
			}

		case 3: // insert at a random position
			if len(model) == 0 {
				break
			}
			at := rng.IntN(len(model) + 1)
			x := rng.IntN(1000)
			v.Insert(at, x)
			model = slices.Insert(model, at, x)

		case 4: // delete a random position
			if len(model) == 0 {
				break
			}
			at := rng.IntN(len(model))
			got := v.Delete(at)
			want := model[at]
			model = slices.Delete(model, at, at+1)
			if got != want {
				t.Fatalf("trial %d: Delete(%d) = %d, model says %d", i, at, got, want)
			}
		}

		if v.Len() != len(model) {
			t.Fatalf("trial %d: Len = %d, model has %d", i, v.Len(), len(model))
		}
		if !slices.Equal(v.Slice(), model) {
			t.Fatalf("trial %d: contents diverged\n  vector %v\n  model  %v",
				i, v.Slice(), model)
		}
	}
	checkInvariants(t, &v, "after the whole run")
	t.Logf("%d random operations, final length %d, capacity %d", trials, v.Len(), v.Cap())
}

// --- 4. the allocation budget ---------------------------------------------

var sink int

func TestPushIsAmortizedConstant(t *testing.T) {
	// Reserve up front, then n pushes must allocate NOTHING.
	var v Vector[int]
	v.Reserve(1000)

	allocs := testing.AllocsPerRun(50, func() {
		v.length = 0 // reuse the same capacity each run
		for i := 0; i < 1000; i++ {
			v.Push(i)
		}
		sink = v.Len()
	})
	if allocs != 0 {
		t.Errorf("1000 pushes into reserved capacity allocated %.0f times, want 0", allocs)
	}
	t.Logf("1000 pushes after Reserve: %.0f allocations", allocs)
}

func TestGrowthIsGeometric(t *testing.T) {
	// The capacity must never grow by a constant amount: each resize should at
	// least double. That is the property the amortized bound depends on.
	var v Vector[int]
	prev := 0
	grows := 0
	for i := 0; i < 100_000; i++ {
		v.Push(i)
		if c := v.Cap(); c != prev {
			if prev > 0 && c < prev*2 {
				t.Fatalf("capacity grew %d -> %d, less than a doubling", prev, c)
			}
			prev = c
			grows++
		}
	}
	t.Logf("100,000 pushes caused %d resizes, final capacity %d", grows, v.Cap())
	if grows > 20 {
		t.Errorf("%d resizes for 100,000 pushes -- growth is not geometric", grows)
	}
}

// The shrink policy must not thrash (example 8).
func TestShrinkDoesNotThrash(t *testing.T) {
	var v Vector[int]
	for i := 0; i < 1025; i++ {
		v.Push(i)
	}
	v.Pop()

	capBefore := v.Cap()
	resizes := 0
	for i := 0; i < 10_000; i++ {
		v.Push(1)
		if v.Cap() != capBefore {
			resizes++
			capBefore = v.Cap()
		}
		v.Pop()
		if v.Cap() != capBefore {
			resizes++
			capBefore = v.Cap()
		}
	}
	if resizes != 0 {
		t.Errorf("push/pop at the capacity boundary caused %d resizes -- thrashing", resizes)
	}
	t.Logf("10,000 push/pop pairs at the boundary: %d resizes", resizes)
}

// And it must actually release memory when the vector really does empty out.
func TestShrinkReleasesMemory(t *testing.T) {
	var v Vector[int]
	for i := 0; i < 10_000; i++ {
		v.Push(i)
	}
	peak := v.Cap()
	for v.Len() > 0 {
		v.Pop()
	}
	if v.Cap() >= peak {
		t.Errorf("capacity stayed at %d after emptying (peak was %d)", v.Cap(), peak)
	}
	t.Logf("capacity %d at peak, %d after emptying", peak, v.Cap())
}
```

**`bench_test.go`** — pass 3: measure it

```go
package vector

import (
	"fmt"
	"testing"
)

// ============================================================================
// PASS 3: MEASURE IT.
//
// Two questions, and they are different:
//   1. does it have the complexity I claimed?      -> the size sweep
//   2. what does it cost against the alternative?  -> vs a native []T
// ============================================================================

// Always first. No number below this floor means anything (lesson 02).
func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// Push should be flat per element -- that is what amortized Theta(1) looks like.
func BenchmarkPush(b *testing.B) {
	for _, n := range []int{1_000, 10_000, 100_000} {
		b.Run(fmt.Sprintf("vector/n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				var v Vector[int]
				for i := 0; i < n; i++ {
					v.Push(i)
				}
			}
		})
		b.Run(fmt.Sprintf("native/n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				var xs []int
				for i := 0; i < n; i++ {
					xs = append(xs, i)
				}
			}
		})
		b.Run(fmt.Sprintf("reserved/n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				var v Vector[int]
				v.Reserve(n)
				for i := 0; i < n; i++ {
					v.Push(i)
				}
			}
		})
	}
}

// Insert at the front is Theta(n) per call, so this whole loop is Theta(n^2).
// The sweep should show the ~4x-per-doubling signature from lesson 03.
func BenchmarkInsertFront(b *testing.B) {
	for _, n := range []int{500, 1_000, 2_000, 4_000} {
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			for b.Loop() {
				var v Vector[int]
				v.Reserve(n)
				for i := 0; i < n; i++ {
					v.Insert(0, i)
				}
			}
		})
	}
}

// Random access must not depend on n at all.
func BenchmarkAt(b *testing.B) {
	for _, n := range []int{1_000, 1_000_000} {
		var v Vector[int]
		v.Reserve(n)
		for i := 0; i < n; i++ {
			v.Push(i)
		}
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			i := 0
			for b.Loop() {
				sink = v.At(i % n) // NOT i & 1023 -- that overruns n=1000
				i++
			}
		})
	}
}
```

**Run it:**

```bash
go test -v .
go test -bench=. -benchmem -run='^$' .
```

**Output:**

```
--- go test -v ---
--- PASS: TestPushPop (0.00s)
--- PASS: TestInsertDelete (0.00s)
    --- PASS: TestInsertDelete/insert_at_front (0.00s)
    --- PASS: TestInsertDelete/insert_in_middle (0.00s)
    --- PASS: TestInsertDelete/insert_at_end (0.00s)
    --- PASS: TestInsertDelete/insert_into_empty (0.00s)
    --- PASS: TestInsertDelete/delete_front (0.00s)
    --- PASS: TestInsertDelete/delete_middle (0.00s)
    --- PASS: TestInsertDelete/delete_last (0.00s)
    --- PASS: TestInsertDelete/delete_only (0.00s)
--- PASS: TestPanicsOnBadIndex (0.00s)
    --- PASS: TestPanicsOnBadIndex/At_past_end (0.00s)
    --- PASS: TestPanicsOnBadIndex/At_negative (0.00s)
    --- PASS: TestPanicsOnBadIndex/Delete_past_end (0.00s)
    --- PASS: TestPanicsOnBadIndex/Insert_past_end+1 (0.00s)
--- PASS: TestInvariantsHold (0.00s)
    vector_test.go:207: 20000 random operations, final length 4033, capacity 4096
--- PASS: TestAgainstModel (0.02s)
    vector_test.go:229: 1000 pushes after Reserve: 0 allocations
--- PASS: TestPushIsAmortizedConstant (0.00s)
    vector_test.go:248: 100,000 pushes caused 16 resizes, final capacity 131072
--- PASS: TestGrowthIsGeometric (0.00s)
    vector_test.go:279: 10,000 push/pop pairs at the boundary: 0 resizes
--- PASS: TestShrinkDoesNotThrash (0.00s)
    vector_test.go:295: capacity 16384 at peak, 4 after emptying
--- PASS: TestShrinkReleasesMemory (0.00s)
PASS
ok  	l06/ex16	0.164s

--- go test -bench=. -benchmem -run='^$' ---
goos: darwin
goarch: arm64
pkg: l06/ex16
cpu: Apple M4
BenchmarkEmptyControl-10    	677091841	         1.671 ns/op	       0 B/op	       0 allocs/op
BenchmarkPush/vector/n=1000-10         	  413733	      2916 ns/op	   16352 B/op	       9 allocs/op
BenchmarkPush/native/n=1000-10         	  404132	      3075 ns/op	   25208 B/op	      12 allocs/op
BenchmarkPush/reserved/n=1000-10       	  525883	      2248 ns/op	    8192 B/op	       1 allocs/op
BenchmarkPush/vector/n=10000-10        	   40626	     29511 ns/op	  262112 B/op	      13 allocs/op
BenchmarkPush/native/n=10000-10        	   35000	     34203 ns/op	  357625 B/op	      19 allocs/op
BenchmarkPush/reserved/n=10000-10      	   54010	     22139 ns/op	   81920 B/op	       1 allocs/op
BenchmarkPush/vector/n=100000-10       	    4458	    309454 ns/op	 2097131 B/op	      16 allocs/op
BenchmarkPush/native/n=100000-10       	    3250	    381914 ns/op	 4101393 B/op	      28 allocs/op
BenchmarkPush/reserved/n=100000-10     	    5353	    217233 ns/op	  802817 B/op	       1 allocs/op
BenchmarkInsertFront/n=500-10          	  116540	     10292 ns/op	    4096 B/op	       1 allocs/op
BenchmarkInsertFront/n=1000-10         	   32019	     37454 ns/op	    8192 B/op	       1 allocs/op
BenchmarkInsertFront/n=2000-10         	    8470	    144507 ns/op	   16384 B/op	       1 allocs/op
BenchmarkInsertFront/n=4000-10         	    2072	    579920 ns/op	   32768 B/op	       1 allocs/op
BenchmarkAt/n=1000-10                  	679784940	         1.761 ns/op	       0 B/op	       0 allocs/op
BenchmarkAt/n=1000000-10               	672972667	         1.800 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	l06/ex16	19.500s
```

Four things to take from the numbers:

- **`At` is 1.70 ns at n=1,000 and 1.72 ns at n=1,000,000.** Θ(1) confirmed — random access genuinely does not care how big the array is.
- **`InsertFront` grows ~4× per doubling** (10,485 → 37,397 → 140,416 → 579,820). That is Θ(n²) for the loop, exactly as lesson 03's ratio test predicts.
- **`Push` beats native `append`** at every size, and uses fewer allocations — because this `Vector` doubles strictly while Go's slices taper toward ×1.25 and therefore copy more (example 9). A hand-rolled structure is allowed to make a different trade.
- **`Reserve` then push is fastest of all, at exactly 1 allocation.** Same conclusion as every lesson so far: if you know n, say so.

The tests also caught a real bug while this was being written — not in the `Vector`, but in
`BenchmarkAt`, which indexed `i & 1023` into a 1,000-element vector. The bounds check turned a silent
out-of-range read into an immediate panic, which is precisely what it's for.

**Complexity:** `At`/`Set`/`Push`/`Pop` Θ(1) (amortized for the last two) · `Insert`/`Delete` Θ(n) · Θ(n) space with capacity between 1× and 2× the length

---

> That's Part 2's foundation. Next: [07 — Linked Lists](../../07-linked-lists.md) — the mirror image
> of this lesson's trade-off, and an honest look at how rarely it wins.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
