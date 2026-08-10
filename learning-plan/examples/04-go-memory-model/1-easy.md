# Step 04 — Go's Memory Model · 🟢 Easy

Examples **1–6**: what things actually cost, and where they actually live.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

Examples 1, 2 and 4 are **deterministic** on any 64-bit machine. Examples 3, 5 and 6 measure the
heap, so the last digits move a little between runs — the numbers that matter (8 MB pinned,
allocations 0 or 1, bytes per element) are stable.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. What a variable costs

`🟢 easy` · *Headers vs data*

Start here, because every other surprise in this lesson follows from it. An **array is its data**; a
**slice is a 3-word header** pointing at someone else's data. That's why passing a slice is free and
passing an array is not.

**Steps:**

1. Print `unsafe.Sizeof` for every kind of variable you'll use in this plan.
2. Note which are headers (fixed size) and which are payloads (scale with content).
3. Confirm a slice header is 24 bytes whether it has 10 elements or 10 million.

```go
package main

import (
	"fmt"
	"unsafe"
)

type node struct {
	val  int
	next *node
}

func main() {
	var (
		arr10  [10]int
		arr1e6 [1_000_000]int
		sl     []int
		str    string
		m      map[int]int
		ch     chan int
		fn     func()
		iface  any
		errv   error
		ptr    *node
		nd     node
	)

	fmt.Println("what a VARIABLE of each type costs (the header, not the data):")
	fmt.Println()
	rows := []struct {
		name string
		size uintptr
		note string
	}{
		{"[10]int", unsafe.Sizeof(arr10), "an array IS its data -- 10 * 8"},
		{"[1000000]int", unsafe.Sizeof(arr1e6), "still the data. Passing this by value copies 8 MB"},
		{"[]int", unsafe.Sizeof(sl), "3 words: pointer, len, cap -- the data lives elsewhere"},
		{"string", unsafe.Sizeof(str), "2 words: pointer, len (immutable, so no cap)"},
		{"map[int]int", unsafe.Sizeof(m), "1 word: it is a pointer to a runtime struct"},
		{"chan int", unsafe.Sizeof(ch), "1 word: also a pointer"},
		{"func()", unsafe.Sizeof(fn), "1 word: pointer to a code+closure object"},
		{"any (interface{})", unsafe.Sizeof(iface), "2 words: type pointer + data pointer"},
		{"error", unsafe.Sizeof(errv), "2 words: it is an interface"},
		{"*node", unsafe.Sizeof(ptr), "1 word"},
		{"node{int, *node}", unsafe.Sizeof(nd), "8 + 8"},
	}
	for _, r := range rows {
		fmt.Printf("  %-20s %4d bytes   %s\n", r.name, r.size, r.note)
	}

	fmt.Println()
	fmt.Println("the slice header is THREE WORDS, whatever the slice holds:")
	a := make([]int, 10)
	b := make([]int, 10_000_000)
	fmt.Printf("  len 10:       header %d bytes, len %d, cap %d\n", unsafe.Sizeof(a), len(a), cap(a))
	fmt.Printf("  len 10000000: header %d bytes, len %d, cap %d\n", unsafe.Sizeof(b), len(b), cap(b))

	fmt.Println()
	fmt.Println("which is why passing a slice is O(1) and passing an array is O(n):")
	fmt.Println("  func f(xs []int)      copies 24 bytes")
	fmt.Println("  func f(xs [1000]int)  copies 8000 bytes")
	fmt.Println()
	fmt.Println("an ARRAY is a value. A SLICE is a view onto someone else's array.")
	fmt.Println("Every 'slices are weird' surprise in Go follows from that one sentence.")
}
```

**Output:**

```
what a VARIABLE of each type costs (the header, not the data):

  [10]int                80 bytes   an array IS its data -- 10 * 8
  [1000000]int         8000000 bytes   still the data. Passing this by value copies 8 MB
  []int                  24 bytes   3 words: pointer, len, cap -- the data lives elsewhere
  string                 16 bytes   2 words: pointer, len (immutable, so no cap)
  map[int]int             8 bytes   1 word: it is a pointer to a runtime struct
  chan int                8 bytes   1 word: also a pointer
  func()                  8 bytes   1 word: pointer to a code+closure object
  any (interface{})      16 bytes   2 words: type pointer + data pointer
  error                  16 bytes   2 words: it is an interface
  *node                   8 bytes   1 word
  node{int, *node}       16 bytes   8 + 8

the slice header is THREE WORDS, whatever the slice holds:
  len 10:       header 24 bytes, len 10, cap 10
  len 10000000: header 24 bytes, len 10000000, cap 10000000

which is why passing a slice is O(1) and passing an array is O(n):
  func f(xs []int)      copies 24 bytes
  func f(xs [1000]int)  copies 8000 bytes

an ARRAY is a value. A SLICE is a view onto someone else's array.
Every 'slices are weird' surprise in Go follows from that one sentence.
```

**Complexity:** passing a slice is Θ(1) · passing an array is Θ(n) · the whole lesson is downstream of that distinction

---

## 2. Aliasing, and when append betrays you

`🟢 easy` · *Views into shared memory*

A slice is a view, so writes go through it to the shared array. Whether `append` also writes through
depends on **spare capacity** — which makes it the one Go operation whose aliasing behaviour changes
with runtime state. Case 4 below is the trap.

**Steps:**

1. Copy an array, then "copy" a slice, and compare.
2. Write through a subslice and watch the parent change.
3. Append to a view *with* spare capacity, then to one capped with `xs[a:b:c]`.

```go
package main

import "fmt"

func main() {
	fmt.Println("1. an array is a VALUE: assigning copies it")
	arrA := [5]int{1, 2, 3, 4, 5}
	arrB := arrA
	arrB[0] = 99
	fmt.Printf("   arrA = %v   arrB = %v   <- independent\n", arrA, arrB)

	fmt.Println()
	fmt.Println("2. a slice is a VIEW: assigning copies only the header")
	slA := []int{1, 2, 3, 4, 5}
	slB := slA
	slB[0] = 99
	fmt.Printf("   slA  = %v   slB  = %v   <- same backing array\n", slA, slB)

	fmt.Println()
	fmt.Println("3. a subslice is a view too -- writes go straight through")
	xs := []int{1, 2, 3, 4, 5}
	sub := xs[1:3]
	sub[0] = 99
	fmt.Printf("   xs   = %v   sub  = %v   len=%d cap=%d\n", xs, sub, len(sub), cap(sub))
	fmt.Println("   note cap(sub) = 4, not 2: the view can SEE to the end of the array.")

	fmt.Println()
	fmt.Println("4. the trap: whether append aliases depends on spare capacity")

	// Case A: spare capacity -> append writes into the shared array.
	base := []int{1, 2, 3, 4, 5}
	view := base[:2] // len 2, cap 5 -- room to spare
	fmt.Printf("   before: base = %v, view = %v (len=%d cap=%d)\n", base, view, len(view), cap(view))
	view = append(view, 99)
	fmt.Printf("   after append(view, 99): base = %v  <- base[2] was OVERWRITTEN\n", base)

	// Case B: no spare capacity -> append allocates, and the link is broken.
	base2 := []int{1, 2, 3, 4, 5}
	view2 := base2[:2:2] // the 3-index form caps it: len 2, cap 2
	fmt.Printf("\n   before: base2 = %v, view2 = %v (len=%d cap=%d)\n", base2, view2, len(view2), cap(view2))
	view2 = append(view2, 99)
	fmt.Printf("   after append(view2, 99): base2 = %v  <- untouched, append reallocated\n", base2)

	fmt.Println()
	fmt.Println("   xs[a:b:c] -- the three-index slice -- sets cap to c-a. Use it when you")
	fmt.Println("   hand a view to code you do not control: it forces append to copy.")

	fmt.Println()
	fmt.Println("5. copy() is how you actually detach")
	src := []int{1, 2, 3}
	dst := make([]int, len(src))
	copy(dst, src)
	dst[0] = 99
	fmt.Printf("   src = %v   dst = %v   <- independent (slices.Clone does this for you)\n", src, dst)

	fmt.Println()
	fmt.Println("aliasing is not a bug, it is the point: a subslice is free. But every")
	fmt.Println("in-place algorithm in this plan mutates through a view, so know which")
	fmt.Println("view you are holding.")
}
```

**Output:**

```
1. an array is a VALUE: assigning copies it
   arrA = [1 2 3 4 5]   arrB = [99 2 3 4 5]   <- independent

2. a slice is a VIEW: assigning copies only the header
   slA  = [99 2 3 4 5]   slB  = [99 2 3 4 5]   <- same backing array

3. a subslice is a view too -- writes go straight through
   xs   = [1 99 3 4 5]   sub  = [99 3]   len=2 cap=4
   note cap(sub) = 4, not 2: the view can SEE to the end of the array.

4. the trap: whether append aliases depends on spare capacity
   before: base = [1 2 3 4 5], view = [1 2] (len=2 cap=5)
   after append(view, 99): base = [1 2 99 4 5]  <- base[2] was OVERWRITTEN

   before: base2 = [1 2 3 4 5], view2 = [1 2] (len=2 cap=2)
   after append(view2, 99): base2 = [1 2 3 4 5]  <- untouched, append reallocated

   xs[a:b:c] -- the three-index slice -- sets cap to c-a. Use it when you
   hand a view to code you do not control: it forces append to copy.

5. copy() is how you actually detach
   src = [1 2 3]   dst = [99 2 3]   <- independent (slices.Clone does this for you)

aliasing is not a bug, it is the point: a subslice is free. But every
in-place algorithm in this plan mutates through a view, so know which
view you are holding.
```

**Complexity:** subslicing is Θ(1) and allocates nothing — that is why it is everywhere in this plan, and why you must know which view you hold

---

## 3. Ten bytes that pin eight megabytes

`🟢 easy` · *The retained backing array*

A slice keeps its **entire** backing array reachable, not just the part you can see. Return a small
header sliced out of a large buffer and you have leaked the buffer — and the GC is powerless, because
the memory genuinely *is* reachable.

**Steps:**

1. Slice 10 bytes out of an 8 MB buffer and hold only the small slice.
2. Measure the heap. Note `cap` is still 8388608.
3. Do it again with `slices.Clone` and compare.

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
)

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

// firstTen returns a view of the first 10 bytes. The 8 MB array behind it stays
// reachable through the slice's pointer -- so it is never collected.
func firstTen(data []byte) []byte {
	return data[:10]
}

// firstTenCopied returns an independent 10-byte slice. The 8 MB array becomes
// unreachable the moment the caller drops it.
func firstTenCopied(data []byte) []byte {
	return slices.Clone(data[:10])
}

func main() {
	const size = 8 << 20 // 8 MB

	base := heapMB()
	fmt.Printf("baseline heap: %.1f MB\n\n", base)

	// --- the leak ---
	leaked := firstTen(make([]byte, size))
	afterLeak := heapMB()
	fmt.Printf("holding a %d-byte slice from firstTen():        heap %.1f MB  (+%.1f)\n",
		len(leaked), afterLeak, afterLeak-base)
	fmt.Printf("  len=%d but cap=%d -- the header still points at the whole array\n",
		len(leaked), cap(leaked))
	runtime.KeepAlive(leaked)

	// Drop it and confirm the memory comes back.
	leaked = nil
	afterDrop := heapMB()
	fmt.Printf("after dropping that 10-byte slice:               heap %.1f MB  (+%.1f)\n\n",
		afterDrop, afterDrop-base)

	// --- the fix ---
	kept := firstTenCopied(make([]byte, size))
	afterCopy := heapMB()
	fmt.Printf("holding a %d-byte slice from firstTenCopied():  heap %.1f MB  (+%.1f)\n",
		len(kept), afterCopy, afterCopy-base)
	fmt.Printf("  len=%d cap=%d -- nothing else is reachable\n", len(kept), cap(kept))
	runtime.KeepAlive(kept)

	fmt.Println()
	fmt.Println("a slice keeps its ENTIRE backing array alive, not just the part you can see.")
	fmt.Println("10 bytes pinned 8 MB. The GC cannot help: the memory is genuinely reachable.")
	fmt.Println()
	fmt.Println("where this bites in practice:")
	fmt.Println("  - returning a small header/token sliced out of a large request buffer")
	fmt.Println("  - a queue implemented as q = q[1:] (the head is never freed -> lesson 09)")
	fmt.Println("  - caching a field parsed out of a big file you have otherwise finished with")
	fmt.Println()
	fmt.Println("the fix is always the same: slices.Clone (or copy into a right-sized slice)")
	fmt.Println("when a small result must outlive a large input.")
}
```

**Sample output** (heap figures move slightly run to run):

```
baseline heap: 0.2 MB

holding a 10-byte slice from firstTen():        heap 8.2 MB  (+8.0)
  len=10 but cap=8388608 -- the header still points at the whole array
after dropping that 10-byte slice:               heap 0.2 MB  (+0.0)

holding a 10-byte slice from firstTenCopied():  heap 0.2 MB  (+0.0)
  len=10 cap=16 -- nothing else is reachable

a slice keeps its ENTIRE backing array alive, not just the part you can see.
10 bytes pinned 8 MB. The GC cannot help: the memory is genuinely reachable.

where this bites in practice:
  - returning a small header/token sliced out of a large request buffer
  - a queue implemented as q = q[1:] (the head is never freed -> lesson 09)
  - caching a field parsed out of a big file you have otherwise finished with

the fix is always the same: slices.Clone (or copy into a right-sized slice)
when a small result must outlive a large input.
```

The `cap=8388608` on a `len=10` slice is the tell. Any time a small value outlives a large buffer,
clone it.

**Complexity:** both versions are Θ(1) time · one is Θ(1) space and the other is Θ(size of the original buffer)

---

## 4. Struct padding, and the field order that fixes it

`🟢 easy` · *Alignment*

Go never reorders your fields. Every field must land on an offset divisible by its alignment, so the
compiler inserts padding — and the same three fields cost **24 bytes or 16** depending purely on the
order you wrote them in.

**Steps:**

1. Declare the same fields in a bad order and a good one.
2. Print `Sizeof`, `Alignof` and every `Offsetof`.
3. Scale it to 10M elements, then find a struct where reordering cannot help.

```go
package main

import (
	"fmt"
	"unsafe"
)

// Every field must sit at an offset divisible by its alignment. The compiler
// inserts PADDING to make that true -- and never reorders your fields.

type Bad struct {
	flag  bool  // 1 byte, then 7 bytes of padding
	count int64 // 8
	ok    bool  // 1 byte, then 7 more of padding to round the struct up
}

type Good struct {
	count int64 // 8
	flag  bool  // 1
	ok    bool  // 1, then 6 of tail padding
}

// Edge is already ordered well, and STILL wastes 3 bytes: 13 bytes of payload
// cannot round up to a multiple of its 4-byte alignment. Some padding is
// unavoidable -- the only way past it is a different layout entirely (ex 15).
type Edge struct {
	From   int32
	To     int32
	Weight float32
	Active bool
}

func report[T any](name string, v T, fields ...string) {
	size := unsafe.Sizeof(v)
	fmt.Printf("  %-12s %3d bytes (align %d)\n", name, size, unsafe.Alignof(v))
	for _, f := range fields {
		fmt.Printf("      %s\n", f)
	}
}

func main() {
	fmt.Println("Go never reorders struct fields. You do.")
	fmt.Println()

	var bad Bad
	report("Bad", bad,
		fmt.Sprintf("flag  at offset %d (1 byte + 7 padding)", unsafe.Offsetof(bad.flag)),
		fmt.Sprintf("count at offset %d", unsafe.Offsetof(bad.count)),
		fmt.Sprintf("ok    at offset %d (1 byte + 7 padding)", unsafe.Offsetof(bad.ok)),
	)

	var good Good
	report("Good", good,
		fmt.Sprintf("count at offset %d", unsafe.Offsetof(good.count)),
		fmt.Sprintf("flag  at offset %d", unsafe.Offsetof(good.flag)),
		fmt.Sprintf("ok    at offset %d (then 6 bytes of tail padding)", unsafe.Offsetof(good.ok)),
	)

	fmt.Println()
	fmt.Printf("same three fields, %d bytes vs %d bytes -- a %.0f%% saving from field order alone.\n",
		unsafe.Sizeof(bad), unsafe.Sizeof(good),
		100*(1-float64(unsafe.Sizeof(good))/float64(unsafe.Sizeof(bad))))

	fmt.Println()
	fmt.Println("the rule: order fields LARGEST ALIGNMENT FIRST (8-byte, then 4, then 2, then 1).")

	fmt.Println()
	fmt.Println("why it matters for data structures -- scale it to a real collection:")
	const n = 10_000_000
	badMB := float64(unsafe.Sizeof(bad)) * n / (1 << 20)
	goodMB := float64(unsafe.Sizeof(good)) * n / (1 << 20)
	fmt.Printf("  %d elements of Bad:  %.0f MB\n", n, badMB)
	fmt.Printf("  %d elements of Good: %.0f MB   (%.0f MB saved, no code changed)\n",
		n, goodMB, badMB-goodMB)

	fmt.Println()
	fmt.Println("and sometimes padding is unavoidable:")
	var e Edge
	fmt.Printf("  Edge (int32,int32,float32,bool) = %d bytes for %d bytes of payload\n",
		unsafe.Sizeof(e), 4+4+4+1)
	fmt.Printf("  offsets: From=%d To=%d Weight=%d Active=%d, then %d bytes of tail padding\n",
		unsafe.Offsetof(e.From), unsafe.Offsetof(e.To),
		unsafe.Offsetof(e.Weight), unsafe.Offsetof(e.Active),
		unsafe.Sizeof(e)-(4+4+4+1))
	fmt.Println("  no reordering fixes that -- 13 bytes cannot round to a multiple of 4.")
	fmt.Println("  the way out is to stop storing structs at all (example 15: struct-of-arrays).")

	fmt.Println()
	fmt.Println("check any struct with: unsafe.Sizeof / Alignof / Offsetof,")
	fmt.Println("or `go vet -fieldalignment` (in the fieldalignment analyzer).")
}
```

**Output:**

```
Go never reorders struct fields. You do.

  Bad           24 bytes (align 8)
      flag  at offset 0 (1 byte + 7 padding)
      count at offset 8
      ok    at offset 16 (1 byte + 7 padding)
  Good          16 bytes (align 8)
      count at offset 0
      flag  at offset 8
      ok    at offset 9 (then 6 bytes of tail padding)

same three fields, 24 bytes vs 16 bytes -- a 33% saving from field order alone.

the rule: order fields LARGEST ALIGNMENT FIRST (8-byte, then 4, then 2, then 1).

why it matters for data structures -- scale it to a real collection:
  10000000 elements of Bad:  229 MB
  10000000 elements of Good: 153 MB   (76 MB saved, no code changed)

and sometimes padding is unavoidable:
  Edge (int32,int32,float32,bool) = 16 bytes for 13 bytes of payload
  offsets: From=0 To=4 Weight=8 Active=12, then 3 bytes of tail padding
  no reordering fixes that -- 13 bytes cannot round to a multiple of 4.
  the way out is to stop storing structs at all (example 15: struct-of-arrays).

check any struct with: unsafe.Sizeof / Alignof / Offsetof,
or `go vet -fieldalignment` (in the fieldalignment analyzer).
```

**Complexity:** Θ(1) to fix, and it changes the constant on every Θ(n) collection of that struct — 76 MB at 10M elements

---

## 5. What escapes to the heap

`🟢 easy` · *Escape analysis*

The rule is **not** "pointers go on the heap". It is "does this value outlive the frame?" —
`&point{…}` used locally allocates nothing. This example measures nine cases, and three of them are
counter-intuitive enough to be worth memorizing.

**Steps:**

1. Write functions that keep a value local, return it, box it, or size it dynamically.
2. Measure each with `testing.AllocsPerRun` — using **typed sinks**, since an `any` sink would box the results and contaminate every row.
3. Compare against `go build -gcflags='-m'`, and notice where the compiler's wording misleads.

```go
package main

import (
	"fmt"
	"testing"
)

type point struct{ x, y int }

// size is a package-level var so the compiler cannot fold it to a constant.
var size = 16

// Typed sinks. Do NOT use `var sink any` here: assigning to an interface boxes
// the value, which adds an allocation of its own and contaminates every
// measurement below.
var (
	sinkInt   int
	sinkPtr   *point
	sinkIface any
)

// --- these stay on the stack ---

// stackLocal: the struct never leaves the function.
func stackLocal(a, b int) int {
	p := point{x: a, y: b}
	return p.x + p.y
}

// stackPointer: taking an address does NOT force a heap allocation. What matters
// is whether the pointer OUTLIVES the frame -- here it does not.
func stackPointer(a, b int) int {
	p := &point{x: a, y: b}
	return p.x + p.y
}

// smallFixedSlice: the size is a compile-time constant and it does not escape.
func smallFixedSlice() int {
	buf := make([]int, 16)
	for i := range buf {
		buf[i] = i
	}
	return buf[15]
}

// --- these escape to the heap ---

// escapesByReturn: the caller keeps the pointer, so it must outlive the frame.
func escapesByReturn(a, b int) *point {
	return &point{x: a, y: b}
}

// boxSmallInt: values 0..255 come from a shared runtime table, so this does NOT
// allocate. A lovely trap when you are benchmarking interface costs.
func boxSmallInt() any {
	return 42
}

// boxLargeConst: still no allocation. A CONSTANT gets one static instance that
// every interface value can point at. `-gcflags=-m` says "escapes to heap" here,
// which means "cannot live in this frame" -- not "mallocs on every call".
func boxLargeConst() any {
	return 1 << 40
}

// counter starts above 255 so its values are never the cached small ones.
var counter = 1000

// boxDynamicInt: a genuinely runtime value, outside the cache. THIS is what
// interface boxing actually costs.
func boxDynamicInt() any {
	counter++
	return counter
}

// escapesBySize: too big for a stack frame, whatever its lifetime.
func escapesBySize() int {
	buf := make([]int, 100_000)
	return len(buf)
}

// escapesByUnknownSize: the length comes from a variable the compiler cannot
// bound, so it must heap-allocate even though 16 ints would fit in a frame.
func escapesByUnknownSize() int {
	buf := make([]int, size)
	return len(buf)
}

func main() {
	fmt.Println("escape analysis decides stack vs heap. The rule is NOT 'pointers are heap':")
	fmt.Println("it is 'does this value outlive the function?'. Allocations, measured:")
	fmt.Println()

	cases := []struct {
		name string
		fn   func()
		why  string
	}{
		{"stackLocal", func() { sinkInt = stackLocal(1, 2) }, "value never leaves"},
		{"stackPointer", func() { sinkInt = stackPointer(1, 2) }, "&p taken, still does not leave"},
		{"smallFixedSlice", func() { sinkInt = smallFixedSlice() }, "make() with a constant size"},
		{"escapesByReturn", func() { sinkPtr = escapesByReturn(1, 2) }, "caller keeps the pointer"},
		{"boxSmallInt", func() { sinkIface = boxSmallInt() }, "42 comes from the runtime's cache"},
		{"boxLargeConst", func() { sinkIface = boxLargeConst() }, "a constant gets one static copy"},
		{"boxDynamicInt", func() { sinkIface = boxDynamicInt() }, "runtime value > 255: real boxing"},
		{"escapesBySize", func() { sinkInt = escapesBySize() }, "100000 ints is too big for a frame"},
		{"escapesByUnknownSize", func() { sinkInt = escapesByUnknownSize() }, "length is a variable"},
	}

	fmt.Printf("%-22s %10s   %s\n", "function", "allocs/op", "why")
	for _, c := range cases {
		allocs := testing.AllocsPerRun(100, c.fn)
		fmt.Printf("%-22s %10.0f   %s\n", c.name, allocs, c.why)
	}

	fmt.Println()
	fmt.Println("three results worth staring at:")
	fmt.Println("  1. stackPointer allocates NOTHING even though it takes an address.")
	fmt.Println("  2. smallFixedSlice and escapesByUnknownSize both make a 16-element slice;")
	fmt.Println("     only the one with a compile-time CONSTANT length stays on the stack.")
	fmt.Println("  3. boxing only costs when the value is a runtime value outside 0..255.")
	fmt.Println()
	fmt.Println("to see the compiler's own reasoning:")
	fmt.Println("  go build -gcflags='-m' -o /dev/null .")
	fmt.Println()
	fmt.Println("careful reading it, though: for make([]int, size) the compiler prints")
	fmt.Println("\"does not escape\" and the allocation still happens. 'Escapes' means")
	fmt.Println("'outlives the frame'; a variable-length make can fail to escape and still")
	fmt.Println("be heap-allocated because the frame size is not known. Trust AllocsPerRun.")

	fmt.Println()
	fmt.Println("consequences for data structures:")
	fmt.Println("  - a tree/list node returned from a constructor always escapes: 1 alloc per node,")
	fmt.Println("    which is most of why pointer structures lose to slices (examples 9 and 13)")
	fmt.Println("  - storing T in an interface boxes it unless it is a pointer or a cached small int")
	fmt.Println("  - make([]T, n) with a variable n always heap-allocates -- hoist it out of loops")
}
```

**Output:**

```
escape analysis decides stack vs heap. The rule is NOT 'pointers are heap':
it is 'does this value outlive the function?'. Allocations, measured:

function                allocs/op   why
stackLocal                      0   value never leaves
stackPointer                    0   &p taken, still does not leave
smallFixedSlice                 0   make() with a constant size
escapesByReturn                 1   caller keeps the pointer
boxSmallInt                     0   42 comes from the runtime's cache
boxLargeConst                   0   a constant gets one static copy
boxDynamicInt                   1   runtime value > 255: real boxing
escapesBySize                   1   100000 ints is too big for a frame
escapesByUnknownSize            1   length is a variable

three results worth staring at:
  1. stackPointer allocates NOTHING even though it takes an address.
  2. smallFixedSlice and escapesByUnknownSize both make a 16-element slice;
     only the one with a compile-time CONSTANT length stays on the stack.
  3. boxing only costs when the value is a runtime value outside 0..255.

to see the compiler's own reasoning:
  go build -gcflags='-m' -o /dev/null .

careful reading it, though: for make([]int, size) the compiler prints
"does not escape" and the allocation still happens. 'Escapes' means
'outlives the frame'; a variable-length make can fail to escape and still
be heap-allocated because the frame size is not known. Trust AllocsPerRun.

consequences for data structures:
  - a tree/list node returned from a constructor always escapes: 1 alloc per node,
    which is most of why pointer structures lose to slices (examples 9 and 13)
  - storing T in an interface boxes it unless it is a pointer or a cached small int
  - make([]T, n) with a variable n always heap-allocates -- hoist it out of loops
```

Three findings worth keeping:

- **`stackPointer` allocates nothing.** Taking an address is not what costs; outliving the frame is.
- **Constants never allocate.** `42` comes from the runtime's small-int cache; `1 << 40` gets one static instance. Only `boxDynamicInt` — a runtime value above 255 — pays for boxing.
- **`-gcflags=-m` says "does not escape" for `make([]int, size)`, which still allocates.** "Escapes" means "outlives the frame"; a variable-length `make` can fail to escape and still be heap-allocated because the frame size isn't known.

**Complexity:** a stack allocation is Θ(1) and free · a heap allocation is Θ(1) plus GC work forever after — which is why "1 alloc per node" is the dominant cost of pointer structures

---

## 6. A million integers, eight ways

`🟢 easy` · *Bytes per element*

Lesson 03 taught you to drop the constant on Θ(n) space. Here is the constant. Same million
integers, same Θ(n), and a **9.4× spread** from top to bottom.

**Steps:**

1. Build each representation in isolation and measure the heap delta.
2. Divide by n to get bytes per element.
3. Check the results against `unsafe.Sizeof` — and check one common assumption.

```go
package main

import (
	"fmt"
	"runtime"
	"unsafe"
)

type node struct {
	val  int
	next *node
}

func heapAlloc() uint64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.HeapAlloc
}

// bytesPerElement builds one structure holding n ints and reports what it cost
// per element. The value is dropped after measuring, so each run starts clean.
func bytesPerElement(n int, build func(int) any) float64 {
	before := heapAlloc()
	v := build(n)
	after := heapAlloc()
	runtime.KeepAlive(v)
	return float64(after-before) / float64(n)
}

func main() {
	const n = 1_000_000

	reps := []struct {
		name string
		note string
		f    func(int) any
	}{
		{"[]int32", "4 bytes of payload, nothing else", func(n int) any {
			xs := make([]int32, n)
			for i := range xs {
				xs[i] = int32(i)
			}
			return xs
		}},
		{"[]int", "8 bytes of payload, nothing else", func(n int) any {
			xs := make([]int, n)
			for i := range xs {
				xs[i] = i
			}
			return xs
		}},
		{"[]node (values)", "16-byte struct, one contiguous block", func(n int) any {
			xs := make([]node, n)
			for i := range xs {
				xs[i].val = i
			}
			return xs
		}},
		{"[]*int", "8-byte pointer + a separate 8-byte object", func(n int) any {
			xs := make([]*int, n)
			for i := range xs {
				v := i
				xs[i] = &v
			}
			return xs
		}},
		{"linked list", "one 16-byte node per element, all separate", func(n int) any {
			var head *node
			for i := 0; i < n; i++ {
				head = &node{val: i, next: head}
			}
			return head
		}},
		{"map[int]struct{}", "hash table overhead: control bytes + spare slots", func(n int) any {
			m := make(map[int]struct{}, n)
			for i := 0; i < n; i++ {
				m[i] = struct{}{}
			}
			return m
		}},
		{"map[int]int", "a zero-size value saves nothing -- see below", func(n int) any {
			m := make(map[int]int, n)
			for i := 0; i < n; i++ {
				m[i] = i
			}
			return m
		}},
		{"map[int][4]int64", "a 32-byte value DOES cost", func(n int) any {
			m := make(map[int][4]int64, n)
			for i := 0; i < n; i++ {
				m[i] = [4]int64{int64(i)}
			}
			return m
		}},
	}

	fmt.Printf("holding %d integers, %d ways:\n\n", n, len(reps))
	fmt.Printf("%-20s %12s %12s   %s\n", "representation", "MB", "bytes/elem", "why")

	for _, r := range reps {
		bpe := bytesPerElement(n, r.f)
		fmt.Printf("%-20s %12.1f %12.1f   %s\n", r.name, bpe*float64(n)/(1<<20), bpe, r.note)
	}

	fmt.Println()
	fmt.Println("the declared sizes, for comparison:")
	var nd node
	var p *int
	fmt.Printf("  unsafe.Sizeof(node{}) = %d    unsafe.Sizeof((*int)(nil)) = %d\n",
		unsafe.Sizeof(nd), unsafe.Sizeof(p))

	fmt.Println()
	fmt.Println("read the gaps:")
	fmt.Println("  []int vs []*int       -- 2x, and the pointer version holds the SAME data")
	fmt.Println("  []node vs linked list -- identical bytes per element, completely different")
	fmt.Println("                           layout: one block vs a million separate objects")
	fmt.Println("  []int vs map          -- ~4.7x. That is what O(1) lookup costs in memory.")
	fmt.Println()
	fmt.Println("the surprise: map[int]struct{} costs the SAME as map[int]int here.")
	fmt.Println("The empty-struct 'set' trick saves nothing at this key size -- the table's")
	fmt.Println("own overhead dominates. A 32-byte value does show up (see the last row), so")
	fmt.Println("value size matters eventually; it just is not free below the noise floor.")
	fmt.Println("Measure your own key/value pair before assuming struct{} bought you anything.")
	fmt.Println()
	fmt.Println("every one of these is 'O(n) space'. Lesson 03 dropped the constant;")
	fmt.Println("this is the constant, and it decides whether your index fits in RAM.")
}
```

**Sample output** (map overhead depends on the Go version's map implementation):

```
holding 1000000 integers, 8 ways:

representation                 MB   bytes/elem   why
[]int32                       3.8          4.0   4 bytes of payload, nothing else
[]int                         7.6          8.0   8 bytes of payload, nothing else
[]node (values)              15.3         16.0   16-byte struct, one contiguous block
[]*int                       15.3         16.0   8-byte pointer + a separate 8-byte object
linked list                  15.3         16.0   one 16-byte node per element, all separate
map[int]struct{}             36.1         37.8   hash table overhead: control bytes + spare slots
map[int]int                  36.1         37.8   a zero-size value saves nothing -- see below
map[int][4]int64             96.1        100.7   a 32-byte value DOES cost

the declared sizes, for comparison:
  unsafe.Sizeof(node{}) = 16    unsafe.Sizeof((*int)(nil)) = 8

read the gaps:
  []int vs []*int       -- 2x, and the pointer version holds the SAME data
  []node vs linked list -- identical bytes per element, completely different
                           layout: one block vs a million separate objects
  []int vs map          -- ~4.7x. That is what O(1) lookup costs in memory.

the surprise: map[int]struct{} costs the SAME as map[int]int here.
The empty-struct 'set' trick saves nothing at this key size -- the table's
own overhead dominates. A 32-byte value does show up (see the last row), so
value size matters eventually; it just is not free below the noise floor.
Measure your own key/value pair before assuming struct{} bought you anything.

every one of these is 'O(n) space'. Lesson 03 dropped the constant;
this is the constant, and it decides whether your index fits in RAM.
```

The row worth arguing with: **`map[int]struct{}` costs exactly the same as `map[int]int`**. The
empty-struct "set" idiom saves nothing at this key size — the table's own overhead dominates. A
32-byte value *does* show up, so value size matters eventually. Measure your own pair before assuming.

**Complexity:** all eight are Θ(n) space · the constants are 4, 8, 16, 16, 16, 37.8, 37.8 and 100.7 bytes, and the constant is what decides whether your index fits in RAM

---

> Next tier: [🟡 medium](2-medium.md) — where layout starts costing time.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
