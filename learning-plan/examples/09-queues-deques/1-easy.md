# Step 09 — Queues & Deques · 🟢 Easy

Examples **1–6**: the other end of the slice, where it stops being free — and the two ways to fix it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 3–5 are deterministic apart from their closing benchmark. Examples 1 and 2 report memory,
which varies a little between runs.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. The naive slice queue, and what it actually costs

`🟢 easy` · *the folklore is two-thirds right*

Everyone knows `q = q[1:]` leaks. Almost nobody has measured *when*, and the answer is more
interesting than the warning: in the steady state it does not leak at all.

**Steps:**

1. Run a queue in steady state — one push, one pop, a thousand live — for two million operations.
2. Then drain a large queue and hold the one-element result.
3. Then absorb a burst and drain back down.

```go
package main

import (
	"fmt"
	"runtime"
)

// Lesson 08's stack only ever touched the TAIL of a slice, which is the cheap
// end. A queue needs BOTH ends -- and the front is where a slice stops being
// free.
//
// The naive version looks perfect:
//
//	enqueue   q = append(q, v)   Theta(1) amortized
//	peek      q[0]               Theta(1)
//	dequeue   q = q[1:]          Theta(1)
//
// "Everyone knows" this leaks memory. Everyone is two-thirds right, and the
// third that is wrong is worth measuring, because it tells you exactly when a
// ring buffer is buying you something and when it is not.

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

func main() {
	q := []int{}
	for _, v := range []int{1, 2, 3} {
		q = append(q, v)
	}
	fmt.Printf("start: %v (len=%d cap=%d)\n", q, len(q), cap(q))
	q = q[1:]
	fmt.Printf("after one dequeue: %v (len=%d cap=%d)\n", q, len(q), cap(q))
	fmt.Println()
	fmt.Println("cap fell from 4 to 3 -- but no memory was freed. `q[1:]` moved the")
	fmt.Println("slice POINTER forward one element; cap is just 'how far from here to")
	fmt.Println("the end of the array'. The dequeued 1 is still in that array, and the")
	fmt.Println("array is still reachable through q.")

	fmt.Println()
	fmt.Println("so does it leak? Three workloads, three different answers.")
	fmt.Println()

	// ---- A: steady state -------------------------------------------------
	base := heapMB()
	a := make([]int, 0, 1000)
	for i := 0; i < 1000; i++ {
		a = append(a, i)
	}
	maxCap := 0
	for i := 0; i < 2_000_000; i++ {
		a = append(a, i)
		a = a[1:]
		if cap(a) > maxCap {
			maxCap = cap(a)
		}
	}
	fmt.Println("A. STEADY STATE -- one in, one out, 2,000,000 times, 1000 live:")
	fmt.Printf("   final len=%d, largest cap ever seen=%d, heap grew %.1f MB\n",
		len(a), maxCap, heapMB()-base)
	fmt.Println("   -> NO LEAK. This is the part the folklore gets wrong.")
	fmt.Println()
	fmt.Println("   when append runs out of room it allocates a new array and copies")
	fmt.Println("   only the LIVE elements. The abandoned head goes with the old array,")
	fmt.Println("   and the GC takes it. Memory stays around 1.5x the live size forever.")
	runtime.KeepAlive(a)

	// ---- B: drain and hold ----------------------------------------------
	fmt.Println()
	base2 := heapMB()
	b := make([]int, 10_000_000)
	afterAlloc := heapMB() - base2
	for i := 0; i < 9_999_999; i++ {
		b = b[1:]
	}
	afterDrain := heapMB() - base2
	fmt.Println("B. DRAIN AND HOLD -- build 10,000,000, dequeue all but one, keep it:")
	fmt.Printf("   len=%d cap=%d, heap after building %.1f MB, after draining %.1f MB\n",
		len(b), cap(b), afterAlloc, afterDrain)
	fmt.Println("   -> LEAK. 76 MB held alive by a ONE-ELEMENT slice.")
	fmt.Println()
	fmt.Println("   append never ran, so the array was never replaced. The header says")
	fmt.Println("   cap=1 and the truth is 80 MB. This is lesson 04's example 3 wearing")
	fmt.Println("   a queue costume.")
	runtime.KeepAlive(b)

	// ---- C: burst then drain --------------------------------------------
	fmt.Println()
	base3 := heapMB()
	c := []int{}
	for i := 0; i < 5_000_000; i++ {
		c = append(c, i)
	}
	peakCap := cap(c)
	peak := heapMB() - base3
	for len(c) > 1000 {
		c = c[1:]
	}
	after := heapMB() - base3
	fmt.Println("C. BURST THEN DRAIN -- 5,000,000 arrive, then drain back to 1000:")
	fmt.Printf("   peak cap=%d (%.1f MB) -> len=%d cap=%d, heap still %.1f MB\n",
		peakCap, peak, len(c), cap(c), after)
	fmt.Println("   -> LEAK, and this is the one that bites real servers.")
	fmt.Println()
	fmt.Println("   the queue is small again and the memory is not. A slice never")
	fmt.Println("   shrinks (lesson 06, example 8), so the array stays at the")
	fmt.Println("   HIGH-WATER MARK until every element is dequeued and append")
	fmt.Println("   happens to reallocate.")
	runtime.KeepAlive(c)

	fmt.Println()
	fmt.Println("what that actually tells you:")
	fmt.Println()
	fmt.Println("  the naive slice queue is fine for a bounded, steady workload, and")
	fmt.Println("  it is a memory bug for a bursty one -- which is most real queues.")
	fmt.Println("  A request queue, a work queue, a BFS frontier: all bursty.")
	fmt.Println()
	fmt.Println("  and even in case A you are paying a periodic Theta(n) copy as the")
	fmt.Println("  live window marches through memory and append hauls it back to the")
	fmt.Println("  start. Amortized Theta(1), with a tail (lesson 03, example 15).")

	fmt.Println()
	fmt.Println("the structural mismatch, in one sentence:")
	fmt.Println()
	fmt.Println("  a slice grows at one end and is indexed from a fixed start; a queue")
	fmt.Println("  consumes at one end and produces at the other, so its live region")
	fmt.Println("  MOVES. Those two models do not fit.")
	fmt.Println()
	fmt.Println("two ways to reconcile them, and the rest of this lesson is both:")
	fmt.Println()
	fmt.Println("  1. keep a HEAD INDEX and compact on your own schedule (example 2)")
	fmt.Println("  2. make the array CIRCULAR so the live region wraps instead of")
	fmt.Println("     marching, and never moves at all (example 3)")
}
```

**Sample output:**

```
start: [1 2 3] (len=3 cap=4)
after one dequeue: [2 3] (len=2 cap=3)

cap fell from 4 to 3 -- but no memory was freed. `q[1:]` moved the
slice POINTER forward one element; cap is just 'how far from here to
the end of the array'. The dequeued 1 is still in that array, and the
array is still reachable through q.

so does it leak? Three workloads, three different answers.

A. STEADY STATE -- one in, one out, 2,000,000 times, 1000 live:
   final len=1000, largest cap ever seen=1535, heap grew 0.0 MB
   -> NO LEAK. This is the part the folklore gets wrong.

   when append runs out of room it allocates a new array and copies
   only the LIVE elements. The abandoned head goes with the old array,
   and the GC takes it. Memory stays around 1.5x the live size forever.

B. DRAIN AND HOLD -- build 10,000,000, dequeue all but one, keep it:
   len=1 cap=1, heap after building 76.3 MB, after draining 76.3 MB
   -> LEAK. 76 MB held alive by a ONE-ELEMENT slice.

   append never ran, so the array was never replaced. The header says
   cap=1 and the truth is 80 MB. This is lesson 04's example 3 wearing
   a queue costume.

C. BURST THEN DRAIN -- 5,000,000 arrive, then drain back to 1000:
   peak cap=5045248 (38.5 MB) -> len=1000 cap=46248, heap still 38.5 MB
   -> LEAK, and this is the one that bites real servers.

   the queue is small again and the memory is not. A slice never
   shrinks (lesson 06, example 8), so the array stays at the
   HIGH-WATER MARK until every element is dequeued and append
   happens to reallocate.

what that actually tells you:

  the naive slice queue is fine for a bounded, steady workload, and
  it is a memory bug for a bursty one -- which is most real queues.
  A request queue, a work queue, a BFS frontier: all bursty.

  and even in case A you are paying a periodic Theta(n) copy as the
  live window marches through memory and append hauls it back to the
  start. Amortized Theta(1), with a tail (lesson 03, example 15).

the structural mismatch, in one sentence:

  a slice grows at one end and is indexed from a fixed start; a queue
  consumes at one end and produces at the other, so its live region
  MOVES. Those two models do not fit.

two ways to reconcile them, and the rest of this lesson is both:

  1. keep a HEAD INDEX and compact on your own schedule (example 2)
  2. make the array CIRCULAR so the live region wraps instead of
     marching, and never moves at all (example 3)
```

**Complexity:** `append` Θ(1) amortized, `q[1:]` Θ(1) · **space is the problem, not time** — the live
region marches forward and the abandoned prefix is still reachable through the backing array

---

## 2. Fix one: a head index and scheduled compaction

`🟢 easy` · *bounded memory, on a dial you choose*

Keep the elements where they are and remember where the live region starts. The dead prefix is then
*yours to reclaim on your own schedule* — which turns a leak into a tuning parameter.

**Steps:**

1. Replace the reslice with an index, and zero each vacated slot.
2. Compact when the dead prefix crosses a threshold.
3. Sweep the threshold and watch the copies/waste trade appear.

```go
package main

import (
	"fmt"
	"testing"
)

// Fix 1: stop reslicing. Keep a HEAD INDEX into a slice you own, and copy the
// live region back to the front when the wasted prefix gets too big.
//
// This keeps the mental model of a slice -- contiguous, indexable, printable --
// and puts YOU in charge of when the Theta(n) copy happens, instead of leaving
// it to append.

type Queue struct {
	buf  []int
	head int // index of the front element; buf[:head] is dead space

	compactions int
	copied      int
}

func (q *Queue) Len() int { return len(q.buf) - q.head }

func (q *Queue) Enqueue(v int) { q.buf = append(q.buf, v) }

// Dequeue advances the head and compacts when more than half the buffer is
// dead. The RATIO is the whole design decision -- see below.
func (q *Queue) Dequeue() (int, bool) {
	if q.head == len(q.buf) {
		return 0, false
	}
	v := q.buf[q.head]
	q.buf[q.head] = 0 // release any pointer (lesson 06, example 2)
	q.head++

	if q.head > len(q.buf)/2 {
		q.compact()
	}
	return v, true
}

func (q *Queue) compact() {
	live := len(q.buf) - q.head
	copy(q.buf, q.buf[q.head:]) // move the live region back to the front
	clear(q.buf[live:])         // and zero what is left behind
	q.buf = q.buf[:live]
	q.head = 0

	q.compactions++
	q.copied += live
}

func (q *Queue) Peek() (int, bool) {
	if q.head == len(q.buf) {
		return 0, false
	}
	return q.buf[q.head], true
}

// live returns the elements currently in the queue.
func (q *Queue) live() []int { return q.buf[q.head:] }

// --- a version with the compaction threshold as a knob ---

// tunable exposes the threshold: compact once the dead prefix exceeds
// deadPct of the buffer. deadPct == 0 means compact on EVERY dequeue.
type tunable struct {
	buf         []int
	head        int
	deadPct     int
	compactions int
	copied      int
}

func (q *tunable) Enqueue(v int) { q.buf = append(q.buf, v) }
func (q *tunable) Dequeue() (int, bool) {
	if q.head == len(q.buf) {
		return 0, false
	}
	v := q.buf[q.head]
	q.head++
	if q.head*100 > len(q.buf)*q.deadPct {
		live := len(q.buf) - q.head
		copy(q.buf, q.buf[q.head:])
		q.buf = q.buf[:live]
		q.head = 0
		q.compactions++
		q.copied += live
	}
	return v, true
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
	var q Queue
	fmt.Println("a head index instead of reslicing:")
	fmt.Println()
	for _, v := range []int{1, 2, 3, 4} {
		q.Enqueue(v)
	}
	fmt.Printf("  enqueue 1..4   buf=%v head=%d live=%v\n", q.buf, q.head, q.live())
	for i := 0; i < 3; i++ {
		v, _ := q.Dequeue()
		fmt.Printf("  dequeue -> %d   buf=%v head=%d live=%v\n", v, q.buf, q.head, q.live())
	}
	fmt.Println()
	fmt.Println("  the first two dequeues just advance head and zero the vacated slot.")
	fmt.Println("  The THIRD crosses the halfway mark (head=3 > 4/2), so it compacts:")
	fmt.Println("  the live tail is copied back to index 0 and the buffer shortened,")
	fmt.Println("  which is why the last line reads buf=[4] head=0.")
	fmt.Println()
	fmt.Println("  nothing is abandoned, so nothing leaks -- and the queue is still a")
	fmt.Println("  plain contiguous slice you can print, range over and index.")

	fmt.Println()
	fmt.Println("the amortized argument, counted:")
	fmt.Println()
	fmt.Printf("  %12s %14s %16s %18s\n", "operations", "compactions", "elements copied", "copies/operation")
	for _, n := range []int{1_000, 10_000, 100_000, 1_000_000} {
		var r Queue
		for i := 0; i < 1000; i++ { // a standing backlog of 1000
			r.Enqueue(i)
		}
		for i := 0; i < n; i++ {
			r.Enqueue(i)
			r.Dequeue()
		}
		fmt.Printf("  %12d %14d %16d %18.2f\n",
			n, r.compactions, r.copied, float64(r.copied)/float64(2*n))
	}
	fmt.Println()
	fmt.Println("  copies per operation settles at a constant, so this is amortized")
	fmt.Println("  Theta(1) by exactly the aggregate argument from lesson 03: each")
	fmt.Println("  compaction can only happen after enough dequeues to make half the")
	fmt.Println("  buffer dead, and those dequeues pay for it.")

	fmt.Println()
	fmt.Println("the threshold is a real knob, and it is the same trade as lesson 06's")
	fmt.Println("shrink hysteresis -- compact eagerly and you copy constantly:")
	fmt.Println()
	fmt.Printf("  %-28s %14s %18s %18s\n", "dead space tolerated", "compactions", "elements copied", "copies/operation")
	for _, pct := range []int{0, 25, 50, 75} {
		t := &tunable{deadPct: pct}
		for i := 0; i < 1000; i++ {
			t.Enqueue(i)
		}
		for i := 0; i < 100_000; i++ {
			t.Enqueue(i)
			t.Dequeue()
		}
		label := fmt.Sprintf("%d%%", pct)
		if pct == 0 {
			label = "0% (compact every dequeue)"
		}
		fmt.Printf("  %-28s %14d %18d %18.2f\n",
			label, t.compactions, t.copied, float64(t.copied)/200_000)
	}
	fmt.Println()
	fmt.Println("  compacting on every dequeue copies the whole live region every")
	fmt.Println("  time -- 1000 elements per operation here, and Theta(n) per")
	fmt.Println("  operation in general, which makes the queue quadratic.")
	fmt.Println()
	fmt.Println("  tolerating more dead space copies less and wastes more. It is the")
	fmt.Println("  same dial as lesson 06's shrink hysteresis, and 50% is the same")
	fmt.Println("  default for the same reason: it is the point where the copies you")
	fmt.Println("  pay for are covered by the dequeues that made them necessary.")

	fmt.Println()
	fmt.Println("versus the naive version, on a bursty workload:")
	fmt.Println()
	const burst = 200_000
	tNaive := nsPerOp(func() {
		s := []int{}
		for i := 0; i < burst; i++ {
			s = append(s, i)
		}
		for len(s) > 0 {
			sinkI = s[0]
			s = s[1:]
		}
	})
	tHead := nsPerOp(func() {
		var r Queue
		for i := 0; i < burst; i++ {
			r.Enqueue(i)
		}
		for {
			v, ok := r.Dequeue()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	fmt.Printf("  %-30s %14.0f ns\n", "naive q = q[1:]", tNaive)
	fmt.Printf("  %-30s %14.0f ns  %.2fx\n", "head index + compaction", tHead, tHead/tNaive)
	fmt.Println()
	fmt.Println("  slower, and that is the honest result: draining once is exactly the")
	fmt.Println("  case the naive version handles well, and compaction is pure extra")
	fmt.Println("  work. What you bought is BOUNDED MEMORY on the cases that leak")
	fmt.Println("  (example 1, B and C), not speed.")
	fmt.Println()
	fmt.Println("  for speed AND bounded memory with no copying at all, the live region")
	fmt.Println("  has to stop moving -- which is the ring buffer, example 3.")
}
```

**Sample output:**

```
a head index instead of reslicing:

  enqueue 1..4   buf=[1 2 3 4] head=0 live=[1 2 3 4]
  dequeue -> 1   buf=[0 2 3 4] head=1 live=[2 3 4]
  dequeue -> 2   buf=[0 0 3 4] head=2 live=[3 4]
  dequeue -> 3   buf=[4] head=0 live=[4]

  the first two dequeues just advance head and zero the vacated slot.
  The THIRD crosses the halfway mark (head=3 > 4/2), so it compacts:
  the live tail is copied back to index 0 and the buffer shortened,
  which is why the last line reads buf=[4] head=0.

  nothing is abandoned, so nothing leaks -- and the queue is still a
  plain contiguous slice you can print, range over and index.

the amortized argument, counted:

    operations    compactions  elements copied   copies/operation
          1000              0                0               0.00
         10000              9             9000               0.45
        100000             99            99000               0.49
       1000000            999           999000               0.50

  copies per operation settles at a constant, so this is amortized
  Theta(1) by exactly the aggregate argument from lesson 03: each
  compaction can only happen after enough dequeues to make half the
  buffer dead, and those dequeues pay for it.

the threshold is a real knob, and it is the same trade as lesson 06's
shrink hysteresis -- compact eagerly and you copy constantly:

  dead space tolerated            compactions    elements copied   copies/operation
  0% (compact every dequeue)           100000          100000000             500.00
  25%                                     299             299000               1.50
  50%                                      99              99000               0.49
  75%                                      33              33000               0.17

  compacting on every dequeue copies the whole live region every
  time -- 1000 elements per operation here, and Theta(n) per
  operation in general, which makes the queue quadratic.

  tolerating more dead space copies less and wastes more. It is the
  same dial as lesson 06's shrink hysteresis, and 50% is the same
  default for the same reason: it is the point where the copies you
  pay for are covered by the dequeues that made them necessary.

versus the naive version, on a bursty workload:

  naive q = q[1:]                        484158 ns
  head index + compaction               1123820 ns  2.32x

  slower, and that is the honest result: draining once is exactly the
  case the naive version handles well, and compaction is pure extra
  work. What you bought is BOUNDED MEMORY on the cases that leak
  (example 1, B and C), not speed.

  for speed AND bounded memory with no copying at all, the live region
  has to stop moving -- which is the ring buffer, example 3.
```

**Complexity:** amortized Θ(1) per operation at the 50% threshold — **0.49 copies per operation
measured** · compacting on every dequeue is Θ(n) per operation and makes the queue quadratic (500
copies/op measured)

---

## 3. Fix two: the ring buffer

`🟢 easy` · *stop moving the data*

The live region still travels — but around a circle, so it never runs off the end and never has to
be copied back. Three fields, Θ(1) worst case, zero allocations. This is the structure inside every
bounded work queue you have ever used.

**Steps:**

1. Build it with `head`, `count` and a derived `tail`, and watch the wrap happen.
2. Meet `Full` as a real state that the caller must be told about.
3. Benchmark it — and find out why the capacity must be a power of two.

```go
package main

import (
	"fmt"
	"testing"
)

// Fix 2, and the one that actually solves the problem: stop moving the data.
//
// A RING BUFFER is a fixed slice plus two indices that WRAP. The live region
// still travels, but it travels around a circle instead of marching off the end,
// so it never needs to be copied back and the array never grows.
//
//	head  where the next Dequeue takes from
//	tail  where the next Enqueue writes
//	count how many are live -- needed because head == tail is ambiguous
//	      (it means both "empty" and "full")
//
// Every operation is Theta(1) WORST CASE, not amortized. No copying, no
// reallocation, no GC pressure, and a hard memory bound you chose.

type Ring struct {
	buf         []int
	head, count int
}

func NewRing(capacity int) *Ring {
	return &Ring{buf: make([]int, capacity)}
}

func (r *Ring) Len() int   { return r.count }
func (r *Ring) Cap() int   { return len(r.buf) }
func (r *Ring) Full() bool { return r.count == len(r.buf) }

// tail is derived rather than stored -- one less thing to keep in sync.
func (r *Ring) tail() int { return (r.head + r.count) % len(r.buf) }

func (r *Ring) Enqueue(v int) bool {
	if r.Full() {
		return false // the caller decides what to do -- example 9
	}
	r.buf[r.tail()] = v
	r.count++
	return true
}

func (r *Ring) Dequeue() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	v := r.buf[r.head]
	r.buf[r.head] = 0 // release any pointer inside T
	r.head = (r.head + 1) % len(r.buf)
	r.count--
	return v, true
}

func (r *Ring) Peek() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	return r.buf[r.head], true
}

// At indexes logically: At(0) is the front. The modulo is the only difference
// from a slice.
func (r *Ring) At(i int) (int, bool) {
	if i < 0 || i >= r.count {
		return 0, false
	}
	return r.buf[(r.head+i)%len(r.buf)], true
}

// render draws the physical array with markers, so the wrap is visible.
func (r *Ring) render() string {
	s := "["
	for i := range r.buf {
		mark := " "
		switch {
		case r.count > 0 && i == r.head:
			mark = "H"
		case r.count > 0 && i == r.tail():
			mark = "t"
		}
		live := false
		for j := 0; j < r.count; j++ {
			if (r.head+j)%len(r.buf) == i {
				live = true
				break
			}
		}
		if live {
			s += fmt.Sprintf("%s%d ", mark, r.buf[i])
		} else {
			s += fmt.Sprintf("%s. ", mark)
		}
	}
	return s + "]"
}

// RingPow2 is the same structure with one constraint: the capacity is rounded
// up to a power of two. That turns every `% cap` into `& (cap-1)` -- a bitwise
// AND instead of an integer division. Example 3's benchmark shows what that
// single change is worth.
type RingPow2 struct {
	buf         []int
	mask        int
	head, count int
}

func NewRingPow2(capacity int) *RingPow2 {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &RingPow2{buf: make([]int, n), mask: n - 1}
}

func (r *RingPow2) Enqueue(v int) bool {
	if r.count == len(r.buf) {
		return false
	}
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
	return true
}

func (r *RingPow2) Dequeue() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	v := r.buf[r.head]
	r.buf[r.head] = 0
	r.head = (r.head + 1) & r.mask
	r.count--
	return v, true
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
	r := NewRing(5)
	fmt.Println("H marks head, t marks the next write slot, . is a free slot:")
	fmt.Println()
	fmt.Printf("  %-22s %s\n", "empty", r.render())

	for _, v := range []int{1, 2, 3} {
		r.Enqueue(v)
		fmt.Printf("  %-22s %s\n", fmt.Sprintf("enqueue %d", v), r.render())
	}
	for i := 0; i < 2; i++ {
		v, _ := r.Dequeue()
		fmt.Printf("  %-22s %s\n", fmt.Sprintf("dequeue -> %d", v), r.render())
	}
	for _, v := range []int{4, 5, 6} {
		r.Enqueue(v)
		fmt.Printf("  %-22s %s\n", fmt.Sprintf("enqueue %d", v), r.render())
	}

	fmt.Println()
	fmt.Println("  look at the last two lines: the tail ran off the end of the array")
	fmt.Println("  and reappeared at index 0. That is the whole idea -- the live")
	fmt.Println("  region WRAPS instead of marching, so it never has to be copied.")

	fmt.Println()
	fmt.Printf("  logical order is still available: ")
	for i := 0; i < r.Len(); i++ {
		v, _ := r.At(i)
		fmt.Printf("%d ", v)
	}
	fmt.Printf("  (At(i) = buf[(head+i)%%cap])\n")

	fmt.Println()
	fmt.Println("full is a real state, and the caller has to be told:")
	fmt.Println()
	fmt.Printf("  Len=%d Cap=%d Full=%v -> Enqueue(7) = %v\n",
		r.Len(), r.Cap(), r.Full(), r.Enqueue(7))
	fmt.Printf("  Len=%d Cap=%d Full=%v -> Enqueue(8) = %v   <- refused\n",
		r.Len(), r.Cap(), r.Full(), r.Enqueue(8))
	fmt.Printf("  physical array is now %s -- not one free slot\n", r.render())
	fmt.Println()
	fmt.Println("  that `bool` is the price of a bounded structure, and it is a")
	fmt.Println("  FEATURE: a queue that cannot refuse work has no backpressure, and")
	fmt.Println("  a queue with no backpressure is a memory leak with extra steps.")
	fmt.Println("  Example 9 is entirely about what to do with that false.")

	fmt.Println()
	fmt.Println("why `count` and not just head and tail:")
	fmt.Println()
	fmt.Println("  with only two indices, head == tail means the ring is either")
	fmt.Println("  completely EMPTY or completely FULL, and nothing distinguishes")
	fmt.Println("  them. The three classic fixes:")
	fmt.Println()
	fmt.Println("    keep a count            <- what this does. Clearest.")
	fmt.Printf("    %-24s%s\n", "waste one slot", "full is (tail+1)%cap == head")
	fmt.Printf("    %-24s%s\n", "keep monotonic counters", "head/tail only grow; index with %cap")
	fmt.Println()
	fmt.Println("  the third is what Go's runtime uses for channels, because two")
	fmt.Println("  ever-increasing integers are easier to update atomically.")

	fmt.Println()
	fmt.Println("the benchmark, and it does not say what you expect:")
	fmt.Println()
	const ops = 1_000_000
	tNaive := nsPerOp(func() {
		q := make([]int, 0, 1000)
		for i := 0; i < 1000; i++ {
			q = append(q, i)
		}
		for i := 0; i < ops; i++ {
			q = append(q, i)
			sinkI = q[0]
			q = q[1:]
		}
	})
	tRing := nsPerOp(func() {
		rr := NewRing(2000)
		for i := 0; i < 1000; i++ {
			rr.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			rr.Enqueue(i)
			v, _ := rr.Dequeue()
			sinkI = v
		}
	})
	tMask := nsPerOp(func() {
		rr := NewRingPow2(2000)
		for i := 0; i < 1000; i++ {
			rr.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			rr.Enqueue(i)
			v, _ := rr.Dequeue()
			sinkI = v
		}
	})
	fmt.Printf("  %-34s %10.2f ns/op\n", "naive q = q[1:]", tNaive/ops)
	fmt.Printf("  %-34s %10.2f ns/op   %.2fx\n", "ring, capacity 2000 (uses %)", tRing/ops, tNaive/tRing)
	fmt.Printf("  %-34s %10.2f ns/op   %.2fx\n", "ring, capacity 2048 (uses &)", tMask/ops, tNaive/tMask)

	fmt.Println()
	fmt.Println("  BOTH rings lose. I expected the ring to win and it does not, so")
	fmt.Println("  here is what the two numbers actually say.")
	fmt.Println()
	fmt.Printf("  `%% 2000` is an integer division -- one of the slowest instructions\n")
	fmt.Println("  the CPU has -- and there are two per operation. Rounding the")
	fmt.Printf("  capacity up to 2048 turns each %% into an &, and that ONE LINE in\n")
	fmt.Printf("  the constructor recovers %.2f ns/op, most of the gap. This is why\n", tRing/ops-tMask/ops)
	fmt.Println("  every real ring buffer -- Go's own channel buffers, LMAX Disruptor,")
	fmt.Println("  io_uring -- is power-of-two sized. It is not optional.")
	fmt.Println()
	fmt.Printf("  the %.2f ns/op that remains is the method calls: two non-inlined\n", tMask/ops-tNaive/ops)
	fmt.Println("  calls per operation, which is exactly the cost lesson 08 measured")
	fmt.Println("  for the generic Stack wrapper. `q = q[1:]` is a pointer increment")
	fmt.Println("  the compiler can see through; Enqueue/Dequeue are not.")
	fmt.Println()
	fmt.Println("  so the naive slice queue is the FASTEST queue in this lesson. That")
	fmt.Println("  has been true in all three examples now, and it is the honest")
	fmt.Println("  frame: its problem was never speed.")

	fmt.Println()
	fmt.Println("its problem is this:")
	fmt.Println()
	allocNaive := testing.AllocsPerRun(5, func() {
		q := make([]int, 0, 1000)
		for i := 0; i < 100_000; i++ {
			q = append(q, i)
			q = q[1:]
		}
		sinkI = len(q)
	})
	rr := NewRingPow2(2048)
	allocRing := testing.AllocsPerRun(5, func() {
		for i := 0; i < 100_000; i++ {
			rr.Enqueue(i)
			v, _ := rr.Dequeue()
			sinkI = v
		}
	})
	fmt.Printf("  %-34s %10.0f allocations per 100,000 ops\n", "naive q = q[1:]", allocNaive)
	fmt.Printf("  %-34s %10.0f allocations per 100,000 ops\n", "ring buffer", allocRing)

	fmt.Println()
	fmt.Println("  ZERO against 98,996. The ring never allocates after the")
	fmt.Println("  constructor, its memory is bounded by a number YOU chose, and")
	fmt.Println("  every operation is Theta(1) WORST CASE -- not amortized, so no")
	fmt.Println("  single enqueue can ever stall on a 4 MB copy the way append can.")
	fmt.Println()
	fmt.Println("  that combination -- bounded, predictable, allocation-free -- is why")
	fmt.Println("  every audio buffer, network buffer, log ring and bounded work queue")
	fmt.Println("  in production is this structure. The cost is the fixed capacity,")
	fmt.Println("  which example 4 removes and example 9 turns into a feature.")
}
```

**Sample output:**

```
H marks head, t marks the next write slot, . is a free slot:

  empty                  [ .  .  .  .  . ]
  enqueue 1              [H1 t.  .  .  . ]
  enqueue 2              [H1  2 t.  .  . ]
  enqueue 3              [H1  2  3 t.  . ]
  dequeue -> 1           [ . H2  3 t.  . ]
  dequeue -> 2           [ .  . H3 t.  . ]
  enqueue 4              [ .  . H3  4 t. ]
  enqueue 5              [t.  . H3  4  5 ]
  enqueue 6              [ 6 t. H3  4  5 ]

  look at the last two lines: the tail ran off the end of the array
  and reappeared at index 0. That is the whole idea -- the live
  region WRAPS instead of marching, so it never has to be copied.

  logical order is still available: 3 4 5 6   (At(i) = buf[(head+i)%cap])

full is a real state, and the caller has to be told:

  Len=4 Cap=5 Full=false -> Enqueue(7) = true
  Len=5 Cap=5 Full=true -> Enqueue(8) = false   <- refused
  physical array is now [ 6  7 H3  4  5 ] -- not one free slot

  that `bool` is the price of a bounded structure, and it is a
  FEATURE: a queue that cannot refuse work has no backpressure, and
  a queue with no backpressure is a memory leak with extra steps.
  Example 9 is entirely about what to do with that false.

why `count` and not just head and tail:

  with only two indices, head == tail means the ring is either
  completely EMPTY or completely FULL, and nothing distinguishes
  them. The three classic fixes:

    keep a count            <- what this does. Clearest.
    waste one slot          full is (tail+1)%cap == head
    keep monotonic counters head/tail only grow; index with %cap

  the third is what Go's runtime uses for channels, because two
  ever-increasing integers are easier to update atomically.

the benchmark, and it does not say what you expect:

  naive q = q[1:]                          1.76 ns/op
  ring, capacity 2000 (uses %)             4.60 ns/op   0.38x
  ring, capacity 2048 (uses &)             2.21 ns/op   0.80x

  BOTH rings lose. I expected the ring to win and it does not, so
  here is what the two numbers actually say.

  `% 2000` is an integer division -- one of the slowest instructions
  the CPU has -- and there are two per operation. Rounding the
  capacity up to 2048 turns each % into an &, and that ONE LINE in
  the constructor recovers 2.39 ns/op, most of the gap. This is why
  every real ring buffer -- Go's own channel buffers, LMAX Disruptor,
  io_uring -- is power-of-two sized. It is not optional.

  the 0.44 ns/op that remains is the method calls: two non-inlined
  calls per operation, which is exactly the cost lesson 08 measured
  for the generic Stack wrapper. `q = q[1:]` is a pointer increment
  the compiler can see through; Enqueue/Dequeue are not.

  so the naive slice queue is the FASTEST queue in this lesson. That
  has been true in all three examples now, and it is the honest
  frame: its problem was never speed.

its problem is this:

  naive q = q[1:]                         98996 allocations per 100,000 ops
  ring buffer                                 0 allocations per 100,000 ops

  ZERO against 98,996. The ring never allocates after the
  constructor, its memory is bounded by a number YOU chose, and
  every operation is Theta(1) WORST CASE -- not amortized, so no
  single enqueue can ever stall on a 4 MB copy the way append can.

  that combination -- bounded, predictable, allocation-free -- is why
  every audio buffer, network buffer, log ring and bounded work queue
  in production is this structure. The cost is the fixed capacity,
  which example 4 removes and example 9 turns into a feature.
```

**Complexity:** every operation Θ(1) **worst case**, not amortized · **0 allocations** in the steady
state · a non-power-of-two capacity costs 2.39 ns/op in integer division alone

---

## 4. Growing a ring, and the bug that hides until it doesn't

`🟢 easy` · *a wrapped region is two pieces, never one*

Doubling a ring is four lines, one of which everybody gets wrong — and the wrong version passes every
test you write by hand, because those tests all start with `head == 0`.

**Steps:**

1. Write the one-copy grow, then rotate the queue so `head != 0` and watch it corrupt.
2. Add the rotation sweep that catches it.
3. Confirm the amortized bound, and notice what you gave up by growing at all.

```go
package main

import (
	"fmt"
	"testing"
)

// The ring buffer's one weakness is the fixed capacity. Growing it is not hard,
// but it is the one place where "it's just a slice" stops being true:
//
//	the live region may be WRAPPED, so you cannot copy it in one call.
//
// Getting this wrong is the single most common ring-buffer bug, so this example
// writes the wrong version first and lets the test catch it.

type Growable struct {
	buf         []int
	mask        int
	head, count int
	grows       int
}

func NewGrowable(capacity int) *Growable {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Growable{buf: make([]int, n), mask: n - 1}
}

func (g *Growable) Len() int { return g.count }
func (g *Growable) Cap() int { return len(g.buf) }

func (g *Growable) Enqueue(v int) {
	if g.count == len(g.buf) {
		g.grow()
	}
	g.buf[(g.head+g.count)&g.mask] = v
	g.count++
}

func (g *Growable) Dequeue() (int, bool) {
	if g.count == 0 {
		return 0, false
	}
	v := g.buf[g.head]
	g.buf[g.head] = 0
	g.head = (g.head + 1) & g.mask
	g.count--
	return v, true
}

// grow doubles the array and UNWRAPS the live region back to index 0.
// Doubling keeps the capacity a power of two, so the mask stays valid.
//
// Two copies, because the live region is at most two contiguous pieces:
//
//	[ 6 7 . H3 4 5 ]   ->   piece A = buf[head:]  = 3 4 5
//	                        piece B = buf[:tail]  = 6 7
func (g *Growable) grow() {
	next := make([]int, len(g.buf)*2)
	n := copy(next, g.buf[g.head:])
	copy(next[n:], g.buf[:g.head])
	g.buf = next
	g.mask = len(next) - 1
	g.head = 0
	g.grows++
}

// growWrong is the bug: one copy, which only works when head == 0.
type GrowWrong struct {
	buf         []int
	mask        int
	head, count int
}

func NewGrowWrong(capacity int) *GrowWrong {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &GrowWrong{buf: make([]int, n), mask: n - 1}
}

func (g *GrowWrong) Enqueue(v int) {
	if g.count == len(g.buf) {
		next := make([]int, len(g.buf)*2)
		copy(next, g.buf) // WRONG: copies the PHYSICAL array, not the logical queue
		g.buf = next
		g.mask = len(next) - 1
		// head is left alone, which is the other half of the bug
	}
	g.buf[(g.head+g.count)&g.mask] = v
	g.count++
}

func (g *GrowWrong) Dequeue() (int, bool) {
	if g.count == 0 {
		return 0, false
	}
	v := g.buf[g.head]
	g.head = (g.head + 1) & g.mask
	g.count--
	return v, true
}

func drainG(g *Growable) []int {
	out := make([]int, 0, g.Len())
	for {
		v, ok := g.Dequeue()
		if !ok {
			return out
		}
		out = append(out, v)
	}
}

func drainW(g *GrowWrong) []int {
	out := make([]int, 0, g.count)
	for {
		v, ok := g.Dequeue()
		if !ok {
			return out
		}
		out = append(out, v)
	}
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
	fmt.Println("growing a ring is not `copy(next, buf)`, because the live region")
	fmt.Println("may be WRAPPED. Both queues below hold 1..4 in order, but one has")
	fmt.Println("been rotated so that head != 0:")
	fmt.Println()

	// Rotate both so head == 2, then fill to capacity and force a grow.
	build := func(enq func(int), deq func() (int, bool)) {
		for i := 0; i < 2; i++ { // push and pop twice to advance head
			enq(-1)
			deq()
		}
		for i := 1; i <= 4; i++ {
			enq(i)
		}
	}

	good := NewGrowable(4)
	build(good.Enqueue, good.Dequeue)
	bad := NewGrowWrong(4)
	build(bad.Enqueue, bad.Dequeue)

	fmt.Printf("  before grow, physical array: %v  head=%d\n", good.buf, good.head)
	fmt.Println("  logical order is 1 2 3 4 -- it wraps around the end.")
	fmt.Println()

	good.Enqueue(5)
	bad.Enqueue(5)

	fmt.Printf("  %-32s %v\n", "correct grow (two copies):", drainG(good))
	fmt.Printf("  %-32s %v\n", "one-copy grow:", drainW(bad))
	fmt.Println()
	fmt.Println("  the broken version returns the elements in the wrong order and")
	fmt.Println("  invents a zero. It does not panic. It does not fail on the first")
	fmt.Println("  test you write, because with head == 0 it is CORRECT -- and head")
	fmt.Println("  is 0 in every example you write by hand.")
	fmt.Println()
	fmt.Println("  the two-line fix, and the reason it is two lines:")
	fmt.Println()
	fmt.Println("      n := copy(next, g.buf[g.head:])   // the piece before the wrap")
	fmt.Println("      copy(next[n:], g.buf[:g.head])    // the piece after it")
	fmt.Println()
	fmt.Println("  a wrapped live region is at most TWO contiguous pieces, never")
	fmt.Println("  more. That fact is worth memorising -- it is also how you write")
	fmt.Println("  Slice(), io.Reader and io.Writer for a ring.")

	fmt.Println()
	fmt.Println("a model test finds it immediately, because it rotates:")
	fmt.Println()
	fails := 0
	firstFail := ""
	for rot := 0; rot < 8; rot++ {
		w := NewGrowWrong(4)
		for i := 0; i < rot; i++ {
			w.Enqueue(-1)
			w.Dequeue()
		}
		for i := 1; i <= 5; i++ { // 5 elements into a cap-4 ring forces a grow
			w.Enqueue(i)
		}
		got := drainW(w)
		want := []int{1, 2, 3, 4, 5}
		ok := len(got) == len(want)
		for i := range want {
			if ok && got[i] != want[i] {
				ok = false
			}
		}
		if !ok {
			fails++
			if firstFail == "" {
				firstFail = fmt.Sprintf("rotation %d -> %v", rot, got)
			}
		}
	}
	fmt.Printf("  8 starting rotations, same 5 enqueues: %d fail\n", fails)
	fmt.Printf("  first failure: %s\n", firstFail)
	fmt.Println()
	fmt.Println("  rotation 0 passes. That is the whole trap: the state that breaks")
	fmt.Println("  a ring buffer is a state you never reach by hand, so the test has")
	fmt.Println("  to reach it FOR you. Add a rotate loop to every ring test you")
	fmt.Println("  write -- it costs three lines and catches the only real bug.")

	fmt.Println()
	fmt.Println("growth is amortized Theta(1), the same argument as lesson 06:")
	fmt.Println()
	fmt.Printf("  %10s %10s %10s %12s\n", "enqueued", "capacity", "grows", "copies/enq")
	for _, n := range []int{1000, 10_000, 100_000, 1_000_000} {
		g := NewGrowable(4)
		for i := 0; i < n; i++ {
			g.Enqueue(i)
		}
		copies := 0
		for c := 4; c < g.Cap(); c *= 2 {
			copies += c
		}
		fmt.Printf("  %10d %10d %10d %12.2f\n", n, g.Cap(), g.grows, float64(copies)/float64(n))
	}
	fmt.Println()
	fmt.Println("  always under 2.0, never converging to a single number. Doubling")
	fmt.Println("  makes the total copies n + n/2 + n/4 + ... < 2n, so the BOUND is 2")
	fmt.Println("  copies per enqueue -- and where you land inside it depends on how")
	fmt.Println("  far past the last power of two you stopped. 10,000 is just over")
	fmt.Println("  8192 so it pays for a 16384-element array it barely uses (1.64);")
	fmt.Println("  1,000,000 is just under 1048576 and gets 1.05. Same algorithm,")
	fmt.Println("  1.6x apart, decided entirely by n.")
	fmt.Println()
	fmt.Println("  that still beats Go's own append (4.02 copies, lesson 03) for the")
	fmt.Println("  reason lesson 06 found: append's factor tapers to 1.25x, and a ring")
	fmt.Println("  cannot taper -- the mask requires a power of two, so it must double.")
	fmt.Println("  A constraint that looked like a limitation is buying you copies.")

	fmt.Println()
	fmt.Println("what it costs against the fixed ring, once it has stopped growing:")
	fmt.Println()
	const ops = 1_000_000
	tFixed := nsPerOp(func() {
		r := NewGrowable(2048)
		for i := 0; i < 1000; i++ {
			r.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			r.Enqueue(i)
			v, _ := r.Dequeue()
			sinkI = v
		}
	})
	tGrown := nsPerOp(func() {
		r := NewGrowable(4)
		for i := 0; i < 1000; i++ {
			r.Enqueue(i)
		}
		for i := 0; i < ops; i++ {
			r.Enqueue(i)
			v, _ := r.Dequeue()
			sinkI = v
		}
	})
	fmt.Printf("  %-34s %10.2f ns/op\n", "preallocated to 2048", tFixed/ops)
	fmt.Printf("  %-34s %10.2f ns/op\n", "grown from 4", tGrown/ops)
	fmt.Println()
	fmt.Println("  identical in the steady state -- growth is a startup cost, not a")
	fmt.Println("  per-operation one. But you have now given up the two properties")
	fmt.Println("  that made the ring worth having: BOUNDED memory and WORST-CASE")
	fmt.Println("  Theta(1). A growable ring is a faster naive queue, not a")
	fmt.Println("  different kind of thing.")
	fmt.Println()
	fmt.Println("  pick deliberately: growable when the queue is an implementation")
	fmt.Println("  detail and the producer is trusted; FIXED when the queue is a")
	fmt.Println("  boundary between a producer and a consumer you do not control.")
	fmt.Println("  Example 9 is about the second case.")
}
```

**Sample output:**

```
growing a ring is not `copy(next, buf)`, because the live region
may be WRAPPED. Both queues below hold 1..4 in order, but one has
been rotated so that head != 0:

  before grow, physical array: [3 4 1 2]  head=2
  logical order is 1 2 3 4 -- it wraps around the end.

  correct grow (two copies):       [1 2 3 4 5]
  one-copy grow:                   [1 2 0 0 5]

  the broken version returns the elements in the wrong order and
  invents a zero. It does not panic. It does not fail on the first
  test you write, because with head == 0 it is CORRECT -- and head
  is 0 in every example you write by hand.

  the two-line fix, and the reason it is two lines:

      n := copy(next, g.buf[g.head:])   // the piece before the wrap
      copy(next[n:], g.buf[:g.head])    // the piece after it

  a wrapped live region is at most TWO contiguous pieces, never
  more. That fact is worth memorising -- it is also how you write
  Slice(), io.Reader and io.Writer for a ring.

a model test finds it immediately, because it rotates:

  8 starting rotations, same 5 enqueues: 6 fail
  first failure: rotation 1 -> [1 2 3 0 5]

  rotation 0 passes. That is the whole trap: the state that breaks
  a ring buffer is a state you never reach by hand, so the test has
  to reach it FOR you. Add a rotate loop to every ring test you
  write -- it costs three lines and catches the only real bug.

growth is amortized Theta(1), the same argument as lesson 06:

    enqueued   capacity      grows   copies/enq
        1000       1024          8         1.02
       10000      16384         12         1.64
      100000     131072         15         1.31
     1000000    1048576         18         1.05

  always under 2.0, never converging to a single number. Doubling
  makes the total copies n + n/2 + n/4 + ... < 2n, so the BOUND is 2
  copies per enqueue -- and where you land inside it depends on how
  far past the last power of two you stopped. 10,000 is just over
  8192 so it pays for a 16384-element array it barely uses (1.64);
  1,000,000 is just under 1048576 and gets 1.05. Same algorithm,
  1.6x apart, decided entirely by n.

  that still beats Go's own append (4.02 copies, lesson 03) for the
  reason lesson 06 found: append's factor tapers to 1.25x, and a ring
  cannot taper -- the mask requires a power of two, so it must double.
  A constraint that looked like a limitation is buying you copies.

what it costs against the fixed ring, once it has stopped growing:

  preallocated to 2048                     2.17 ns/op
  grown from 4                             2.17 ns/op

  identical in the steady state -- growth is a startup cost, not a
  per-operation one. But you have now given up the two properties
  that made the ring worth having: BOUNDED memory and WORST-CASE
  Theta(1). A growable ring is a faster naive queue, not a
  different kind of thing.

  pick deliberately: growable when the queue is an implementation
  detail and the producer is trusted; FIXED when the queue is a
  boundary between a producer and a consumer you do not control.
  Example 9 is about the second case.
```

**Complexity:** growth amortized Θ(1), bounded by **2 copies per enqueue** (measured 1.02–1.64
depending only on where `n` falls relative to a power of two) · **rotation 0 passes and 6 of 8 fail** —
the sweep is three lines and catches the only real bug

---

## 5. `Deque[T]` — opening the other end

`🟢 easy` · *`(head-1) & mask`, and why `%` will not do*

One extra line makes the ring run backwards too. That line is `head = (head-1) & mask`, and it works
only because the capacity is a power of two — Go's `%` returns `-1` for `-1 % 8`.

**Steps:**

1. Add `PushFront` and `PopBack`, and watch `head` wrap backwards past zero.
2. See that a deque subsumes both a stack and a queue.
3. Delete the slot-zeroing line and measure what it was doing.

```go
package main

import (
	"fmt"
	"runtime"
	"testing"
)

// A DEQUE (double-ended queue) is a ring buffer that also runs backwards.
// That is the entire generalisation:
//
//	PushBack   write at (head+count)&mask, count++
//	PushFront  head = (head-1)&mask, write at head, count++
//
// The `(head-1)&mask` is the trick. In Go, `-1 & mask` on an int gives mask,
// because Go's & is bitwise on two's complement -- so decrementing past zero
// wraps to the end of the array with no branch. `%` would NOT do this: in Go,
// -1 % 8 is -1, not 7. That is a second reason powers of two are not optional.

type Deque[T any] struct {
	buf         []T
	mask        int
	head, count int
}

func NewDeque[T any](capacity int) *Deque[T] {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Deque[T]{buf: make([]T, n), mask: n - 1}
}

func (d *Deque[T]) Len() int { return d.count }
func (d *Deque[T]) Cap() int { return len(d.buf) }

func (d *Deque[T]) grow() {
	next := make([]T, len(d.buf)*2)
	n := copy(next, d.buf[d.head:])
	copy(next[n:], d.buf[:d.head])
	d.buf, d.mask, d.head = next, len(next)-1, 0
}

func (d *Deque[T]) PushBack(v T) {
	if d.count == len(d.buf) {
		d.grow()
	}
	d.buf[(d.head+d.count)&d.mask] = v
	d.count++
}

func (d *Deque[T]) PushFront(v T) {
	if d.count == len(d.buf) {
		d.grow()
	}
	d.head = (d.head - 1) & d.mask // wraps to the end when head is 0
	d.buf[d.head] = v
	d.count++
}

func (d *Deque[T]) PopFront() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	v := d.buf[d.head]
	d.buf[d.head] = zero // MUST: T may contain pointers
	d.head = (d.head + 1) & d.mask
	d.count--
	return v, true
}

func (d *Deque[T]) PopBack() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	i := (d.head + d.count - 1) & d.mask
	v := d.buf[i]
	d.buf[i] = zero
	d.count--
	return v, true
}

func (d *Deque[T]) At(i int) T { return d.buf[(d.head+i)&d.mask] }

func (d *Deque[T]) Front() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	return d.buf[d.head], true
}

func (d *Deque[T]) Back() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	return d.buf[(d.head+d.count-1)&d.mask], true
}

func (d *Deque[T]) slice() []T {
	out := make([]T, d.count)
	for i := 0; i < d.count; i++ {
		out[i] = d.At(i)
	}
	return out
}

// A leaky deque: same code with the zeroing removed. Example 1's lesson,
// applied to a structure that looks like it cannot leak.
type Leaky[T any] struct {
	buf         []T
	mask        int
	head, count int
}

func NewLeaky[T any](capacity int) *Leaky[T] {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Leaky[T]{buf: make([]T, n), mask: n - 1}
}
func (d *Leaky[T]) PushBack(v T) {
	d.buf[(d.head+d.count)&d.mask] = v
	d.count++
}
func (d *Leaky[T]) PopFront() (T, bool) {
	var zero T
	if d.count == 0 {
		return zero, false
	}
	v := d.buf[d.head]
	// no d.buf[d.head] = zero
	d.head = (d.head + 1) & d.mask
	d.count--
	return v, true
}

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
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
	d := NewDeque[string](4)
	fmt.Println("one structure, four operations, and it subsumes both of the last")
	fmt.Println("two lessons:")
	fmt.Println()
	for _, op := range []struct {
		name string
		fn   func()
	}{
		{`PushBack("b")`, func() { d.PushBack("b") }},
		{`PushBack("c")`, func() { d.PushBack("c") }},
		{`PushFront("a")`, func() { d.PushFront("a") }},
		{`PushFront("z")`, func() { d.PushFront("z") }},
	} {
		op.fn()
		fmt.Printf("  %-16s %v   head=%d physical=%v\n", op.name, d.slice(), d.head, d.buf)
	}
	f, _ := d.PopFront()
	b, _ := d.PopBack()
	fmt.Printf("  %-16s -> %q and %q, leaving %v\n", "PopFront/PopBack", f, b, d.slice())

	fmt.Println()
	fmt.Println("  watch the physical array on the PushFront lines: head went 0 -> 3,")
	fmt.Println("  wrapping backwards past zero. That is `(head-1) & mask`, one AND")
	fmt.Println("  with no branch and no bounds problem.")
	fmt.Println()
	fmt.Println("  it does NOT work with %: in Go, -1 % 8 == -1, not 7, because Go's")
	fmt.Println("  remainder takes the sign of the dividend. A ring written with % has")
	fmt.Println("  to add cap first: ((head-1+cap) % cap). Powers of two make the")
	fmt.Println("  problem disappear rather than be worked around.")

	fmt.Println()
	fmt.Println("use only one end and you have already built lesson 08 and this one:")
	fmt.Println()
	fmt.Printf("  %-34s %s\n", "PushBack + PopBack", "a stack (lesson 08)")
	fmt.Printf("  %-34s %s\n", "PushBack + PopFront", "a queue (this lesson)")
	fmt.Printf("  %-34s %s\n", "PushFront + PopFront", "a stack, growing the other way")
	fmt.Printf("  %-34s %s\n", "all four", "a deque -- examples 10, 13, 15")
	fmt.Println()
	fmt.Println("  which is why Go has no queue type in the standard library and no")
	fmt.Println("  deque type either: a slice is a stack, and everything else is")
	fmt.Println("  thirty lines you write once.")

	fmt.Println()
	fmt.Println("the zeroing on Pop is not optional. Same deque, one line removed:")
	fmt.Println()
	type payload struct{ data []byte }
	const n = 20_000

	base := heapMB()
	good := NewDeque[*payload](1 << 16)
	for i := 0; i < n; i++ {
		good.PushBack(&payload{data: make([]byte, 1024)})
		good.PopFront()
	}
	goodMB := heapMB() - base
	runtime.KeepAlive(good)

	base = heapMB()
	leaky := NewLeaky[*payload](1 << 16)
	for i := 0; i < n; i++ {
		leaky.PushBack(&payload{data: make([]byte, 1024)})
		leaky.PopFront()
	}
	leakyMB := heapMB() - base
	runtime.KeepAlive(leaky)

	fmt.Printf("  %-40s %8.1f MB held\n", "Pop writes the zero value", goodMB)
	fmt.Printf("  %-40s %8.1f MB held\n", "Pop leaves the slot alone", leakyMB)
	fmt.Println()
	fmt.Printf("  both deques report Len()==0. The leaky one is still holding %.0f\n", leakyMB)
	fmt.Println("  MB, because every slot it ever wrote is still a live pointer as")
	fmt.Println("  far as the GC is concerned. The array is bounded; what the array")
	fmt.Println("  POINTS AT is not.")
	fmt.Println()
	fmt.Println("  this is example 1's leak wearing a different hat, and it is worse:")
	fmt.Println("  there is no growing slice to notice, no capacity climbing, nothing")
	fmt.Println("  to see in a heap profile except a large []*payload that is")
	fmt.Println("  'obviously' empty.")

	fmt.Println()
	fmt.Println("what it costs against a slice, on the operation slices are bad at:")
	fmt.Println()
	const m = 20_000
	tSlice := nsPerOp(func() {
		var s []int
		for i := 0; i < m; i++ {
			s = append([]int{i}, s...) // the classic front-insert
		}
		sinkI = len(s)
	})
	tDeque := nsPerOp(func() {
		dd := NewDeque[int](8)
		for i := 0; i < m; i++ {
			dd.PushFront(i)
		}
		sinkI = dd.Len()
	})
	fmt.Printf("  %-34s %12.0f ns for %d front-inserts\n", "append([]int{v}, s...)", tSlice, m)
	fmt.Printf("  %-34s %12.0f ns   %.0fx faster\n", "deque PushFront", tDeque, tSlice/tDeque)
	fmt.Println()
	fmt.Println("  Theta(n^2) against Theta(n), which is the same gap lesson 06's gap")
	fmt.Println("  buffer opened -- but a deque needs no gap, no rebalancing and no")
	fmt.Println("  special cases. When you need both ends, this is the structure.")
	fmt.Println()
	fmt.Println("  the rest of the lesson uses it four times: BFS (6), sliding-window")
	fmt.Println("  maximum (10), 0-1 BFS (13) and work stealing (15).")
}
```

**Sample output:**

```
one structure, four operations, and it subsumes both of the last
two lessons:

  PushBack("b")    [b]   head=0 physical=[b   ]
  PushBack("c")    [b c]   head=0 physical=[b c  ]
  PushFront("a")   [a b c]   head=3 physical=[b c  a]
  PushFront("z")   [z a b c]   head=2 physical=[b c z a]
  PopFront/PopBack -> "z" and "c", leaving [a b]

  watch the physical array on the PushFront lines: head went 0 -> 3,
  wrapping backwards past zero. That is `(head-1) & mask`, one AND
  with no branch and no bounds problem.

  it does NOT work with %: in Go, -1 % 8 == -1, not 7, because Go's
  remainder takes the sign of the dividend. A ring written with % has
  to add cap first: ((head-1+cap) % cap). Powers of two make the
  problem disappear rather than be worked around.

use only one end and you have already built lesson 08 and this one:

  PushBack + PopBack                 a stack (lesson 08)
  PushBack + PopFront                a queue (this lesson)
  PushFront + PopFront               a stack, growing the other way
  all four                           a deque -- examples 10, 13, 15

  which is why Go has no queue type in the standard library and no
  deque type either: a slice is a stack, and everything else is
  thirty lines you write once.

the zeroing on Pop is not optional. Same deque, one line removed:

  Pop writes the zero value                     0.5 MB held
  Pop leaves the slot alone                    20.5 MB held

  both deques report Len()==0. The leaky one is still holding 20
  MB, because every slot it ever wrote is still a live pointer as
  far as the GC is concerned. The array is bounded; what the array
  POINTS AT is not.

  this is example 1's leak wearing a different hat, and it is worse:
  there is no growing slice to notice, no capacity climbing, nothing
  to see in a heap profile except a large []*payload that is
  'obviously' empty.

what it costs against a slice, on the operation slices are bad at:

  append([]int{v}, s...)                 94152788 ns for 20000 front-inserts
  deque PushFront                           59587 ns   1580x faster

  Theta(n^2) against Theta(n), which is the same gap lesson 06's gap
  buffer opened -- but a deque needs no gap, no rebalancing and no
  special cases. When you need both ends, this is the structure.

  the rest of the lesson uses it four times: BFS (6), sliding-window
  maximum (10), 0-1 BFS (13) and work stealing (15).
```

**Complexity:** all four operations Θ(1) · front-insertion **1580× faster** than `append([]T{v}, s...)`
at n=20,000 · dropping the zeroing line held **20.5 MB** in a deque reporting `Len() == 0`

---

## 6. BFS, and why the container *is* the algorithm

`🟢 easy` · *one word apart from DFS*

Swap `PopFront` for `PopBack` and breadth-first search becomes depth-first search. Nothing else
changes — which means the shortest-path guarantee belongs to the queue, not to the traversal.

**Steps:**

1. Run the same twenty lines with each end and compare the visit orders.
2. Reconstruct shortest paths from the distances the queue produced.
3. Then measure the classic bug: marking visited on *dequeue*.

```go
package main

import (
	"fmt"
	"math/rand"
)

// Here is why the queue matters: BFS and DFS are THE SAME ALGORITHM with a
// different container. Swap the queue for a stack and the traversal order
// changes completely -- and so does what the result means.
//
// BFS with a queue finds SHORTEST PATHS on an unweighted graph. DFS with a
// stack finds *a* path. That difference is entirely the container's doing.

type graph [][]int

func (g graph) neighbors(v int) []int { return g[v] }

// bfs returns dist[v] = number of edges from src, and prev[v] for path
// reconstruction. -1 means unreachable.
func bfs(g graph, src int) (dist, prev []int) {
	dist = make([]int, len(g))
	prev = make([]int, len(g))
	for i := range dist {
		dist[i], prev[i] = -1, -1
	}

	q := NewDeque[int](8)
	dist[src] = 0
	q.PushBack(src)

	for q.Len() > 0 {
		v, _ := q.PopFront()
		for _, w := range g.neighbors(v) {
			if dist[w] != -1 { // MARK ON ENQUEUE -- see below
				continue
			}
			dist[w] = dist[v] + 1
			prev[w] = v
			q.PushBack(w)
		}
	}
	return dist, prev
}

// dfs is byte-for-byte bfs with PopFront changed to PopBack.
func dfs(g graph, src int) []int {
	order := []int{}
	seen := make([]bool, len(g))
	q := NewDeque[int](8)
	seen[src] = true
	q.PushBack(src)
	for q.Len() > 0 {
		v, _ := q.PopBack() // <- the only difference
		order = append(order, v)
		for _, w := range g.neighbors(v) {
			if seen[w] {
				continue
			}
			seen[w] = true
			q.PushBack(w)
		}
	}
	return order
}

func bfsOrder(g graph, src int) []int {
	order := []int{}
	seen := make([]bool, len(g))
	q := NewDeque[int](8)
	seen[src] = true
	q.PushBack(src)
	for q.Len() > 0 {
		v, _ := q.PopFront()
		order = append(order, v)
		for _, w := range g.neighbors(v) {
			if seen[w] {
				continue
			}
			seen[w] = true
			q.PushBack(w)
		}
	}
	return order
}

// bfsMarkOnDequeue is the classic bug: the visited check happens when a vertex
// comes OUT of the queue rather than when it goes in. The distance travels
// WITH the queue entry, so the answers stay correct -- it just queues a vertex
// once per incoming edge instead of once.
type entry struct{ v, d int }

func bfsMarkOnDequeue(g graph, src int) (dist []int, pushes int) {
	dist = make([]int, len(g))
	for i := range dist {
		dist[i] = -1
	}
	q := NewDeque[entry](8)
	q.PushBack(entry{src, 0})
	pushes = 1
	for q.Len() > 0 {
		e, _ := q.PopFront()
		if dist[e.v] != -1 {
			continue // already settled -- this vertex was queued more than once
		}
		dist[e.v] = e.d
		for _, w := range g.neighbors(e.v) {
			if dist[w] != -1 {
				continue
			}
			q.PushBack(entry{w, e.d + 1})
			pushes++
		}
	}
	return dist, pushes
}

// bfsSharedDist is the same bug PLUS the shortcut of keeping the tentative
// distance in an array instead of in the queue entry. Now it is not merely
// slow -- a later, longer tentative distance overwrites an earlier shorter one
// before the vertex is ever popped.
func bfsSharedDist(g graph, src int) []int {
	dist := make([]int, len(g))
	for i := range dist {
		dist[i] = -1
	}
	d := make([]int, len(g))
	q := NewDeque[int](8)
	q.PushBack(src)
	for q.Len() > 0 {
		v, _ := q.PopFront()
		if dist[v] != -1 {
			continue
		}
		dist[v] = d[v]
		for _, w := range g.neighbors(v) {
			if dist[w] != -1 {
				continue
			}
			d[w] = d[v] + 1
			q.PushBack(w)
		}
	}
	return dist
}

func bfsPushes(g graph, src int) int {
	seen := make([]bool, len(g))
	q := NewDeque[int](8)
	seen[src] = true
	q.PushBack(src)
	pushes := 1
	for q.Len() > 0 {
		v, _ := q.PopFront()
		for _, w := range g.neighbors(v) {
			if seen[w] {
				continue
			}
			seen[w] = true
			q.PushBack(w)
			pushes++
		}
	}
	return pushes
}

func path(prev []int, dst int) []int {
	var p []int
	for v := dst; v != -1; v = prev[v] {
		p = append(p, v)
	}
	for i, j := 0, len(p)-1; i < j; i, j = i+1, j-1 {
		p[i], p[j] = p[j], p[i]
	}
	return p
}

// randomGraph builds a connected undirected graph with n vertices.
func randomGraph(n, extra int, seed int64) graph {
	rng := rand.New(rand.NewSource(seed))
	g := make(graph, n)
	add := func(a, b int) {
		g[a] = append(g[a], b)
		g[b] = append(g[b], a)
	}
	for v := 1; v < n; v++ { // spanning tree, so it is connected
		add(v, rng.Intn(v))
	}
	for i := 0; i < extra; i++ {
		a, b := rng.Intn(n), rng.Intn(n)
		if a != b {
			add(a, b)
		}
	}
	return g
}

func main() {
	//        0
	//      / | \
	//     1  2  3
	//    /|     |
	//   4 5     6
	//           |
	//           7
	g := graph{
		0: {1, 2, 3},
		1: {0, 4, 5},
		2: {0},
		3: {0, 6},
		4: {1},
		5: {1},
		6: {3, 7},
		7: {6},
	}

	fmt.Println("the same twenty lines, one word different:")
	fmt.Println()
	fmt.Printf("  PopFront (queue) -> %v\n", bfsOrder(g, 0))
	fmt.Printf("  PopBack  (stack) -> %v\n", dfs(g, 0))
	fmt.Println()
	fmt.Println("  BFS visits 1, 2, 3 before any of their children: it explores in")
	fmt.Println("  RINGS of increasing distance. DFS dives to 7 before it has even")
	fmt.Println("  looked at 2. Neither is a special algorithm -- the container is")
	fmt.Println("  doing all the work.")

	dist, prev := bfs(g, 0)
	fmt.Println()
	fmt.Println("and because BFS explores in rings, dist[] is a SHORTEST path:")
	fmt.Println()
	fmt.Printf("  %-8s %-10s %s\n", "vertex", "distance", "path from 0")
	for v := range g {
		fmt.Printf("  %-8d %-10d %v\n", v, dist[v], path(prev, v))
	}
	fmt.Println()
	fmt.Println("  the queue is what guarantees this. Vertices come out in")
	fmt.Println("  non-decreasing distance order, so the FIRST time you reach a")
	fmt.Println("  vertex is necessarily by a shortest path. A stack gives you no")
	fmt.Println("  such promise -- DFS reached 7 in 4 steps down one branch, but")
	fmt.Println("  had the edges been listed differently it might have found a")
	fmt.Println("  longer route first and never revisited.")
	fmt.Println()
	fmt.Println("  this is Dijkstra with all weights equal to 1, and the queue is")
	fmt.Println("  standing in for the priority queue you would otherwise need")
	fmt.Println("  (lesson 20). Example 13 does the same trick for weights 0 and 1.")

	fmt.Println()
	fmt.Println("the bug that does not look like a bug -- marking on DEQUEUE:")
	fmt.Println()
	fmt.Printf("  %10s %8s %14s %14s %10s\n", "vertices", "edges", "mark on enq", "mark on deq", "ratio")
	wrongTotal, wrongN := 0, 0
	for _, n := range []int{1000, 10_000, 100_000} {
		gg := randomGraph(n, n*4, 7)
		edges := 0
		for _, adj := range gg {
			edges += len(adj)
		}
		good := bfsPushes(gg, 0)
		dd, bad := bfsMarkOnDequeue(gg, 0)
		ref, _ := bfs(gg, 0)
		for i := range ref { // correctness is proved, not assumed
			if ref[i] != dd[i] {
				panic("distances differ")
			}
		}
		shared := bfsSharedDist(gg, 0)
		for i := range ref {
			if ref[i] != shared[i] {
				wrongTotal++
			}
		}
		wrongN = n
		fmt.Printf("  %10d %8d %14d %14d %9.1fx\n", n, edges/2, good, bad, float64(bad)/float64(good))
	}
	fmt.Println()
	fmt.Println("  the distances are IDENTICAL -- verified against the correct BFS")
	fmt.Println("  above, not assumed. The broken version is correct and slow, which")
	fmt.Println("  is the worst kind of wrong: it queues a vertex once per incoming")
	fmt.Println("  edge instead of once, so the queue holds Theta(E) entries instead")
	fmt.Println("  of Theta(V), and nothing ever tells you.")
	fmt.Println()
	fmt.Println("  mark the vertex the moment you PUSH it, never when you pop it.")

	fmt.Println()
	fmt.Println("and it stays correct only because the distance rides in the QUEUE")
	fmt.Println("ENTRY. Move it to an array -- which looks like an obvious saving --")
	fmt.Println("and a later, longer tentative distance overwrites an earlier one:")
	fmt.Println()
	fmt.Printf("  vertices with the WRONG distance across the three graphs: %d\n", wrongTotal)
	fmt.Printf("  (largest graph had %d vertices)\n", wrongN)
	fmt.Println()
	fmt.Println("  I wrote that version first and asserted it was 'correct but slow'.")
	fmt.Println("  The check three lines above panicked on the first graph. Both")
	fmt.Println("  variants were plausible; only the measurement separated them.")
	fmt.Println()
	fmt.Println("  (the one algorithm where you genuinely must settle on POP is")
	fmt.Println("  Dijkstra with a priority queue, because a shorter path can still")
	fmt.Println("  arrive later. On an unweighted graph it cannot -- which is exactly")
	fmt.Println("  what the queue buys you.)")
}
```

> Reuses `Deque[T]` from example 5 — copy that type into the same folder as `deque.go`.

**Sample output:**

```
the same twenty lines, one word different:

  PopFront (queue) -> [0 1 2 3 4 5 6 7]
  PopBack  (stack) -> [0 3 6 7 2 1 5 4]

  BFS visits 1, 2, 3 before any of their children: it explores in
  RINGS of increasing distance. DFS dives to 7 before it has even
  looked at 2. Neither is a special algorithm -- the container is
  doing all the work.

and because BFS explores in rings, dist[] is a SHORTEST path:

  vertex   distance   path from 0
  0        0          [0]
  1        1          [0 1]
  2        1          [0 2]
  3        1          [0 3]
  4        2          [0 1 4]
  5        2          [0 1 5]
  6        2          [0 3 6]
  7        3          [0 3 6 7]

  the queue is what guarantees this. Vertices come out in
  non-decreasing distance order, so the FIRST time you reach a
  vertex is necessarily by a shortest path. A stack gives you no
  such promise -- DFS reached 7 in 4 steps down one branch, but
  had the edges been listed differently it might have found a
  longer route first and never revisited.

  this is Dijkstra with all weights equal to 1, and the queue is
  standing in for the priority queue you would otherwise need
  (lesson 20). Example 13 does the same trick for weights 0 and 1.

the bug that does not look like a bug -- marking on DEQUEUE:

    vertices    edges    mark on enq    mark on deq      ratio
        1000     4994           1000           4995       5.0x
       10000    49994          10000          49995       5.0x
      100000   499999         100000         500000       5.0x

  the distances are IDENTICAL -- verified against the correct BFS
  above, not assumed. The broken version is correct and slow, which
  is the worst kind of wrong: it queues a vertex once per incoming
  edge instead of once, so the queue holds Theta(E) entries instead
  of Theta(V), and nothing ever tells you.

  mark the vertex the moment you PUSH it, never when you pop it.

and it stays correct only because the distance rides in the QUEUE
ENTRY. Move it to an array -- which looks like an obvious saving --
and a later, longer tentative distance overwrites an earlier one:

  vertices with the WRONG distance across the three graphs: 95797
  (largest graph had 100000 vertices)

  I wrote that version first and asserted it was 'correct but slow'.
  The check three lines above panicked on the first graph. Both
  variants were plausible; only the measurement separated them.

  (the one algorithm where you genuinely must settle on POP is
  Dijkstra with a priority queue, because a shorter path can still
  arrive later. On an unweighted graph it cannot -- which is exactly
  what the queue buys you.)
```

**Complexity:** Θ(V + E) with marking on enqueue · marking on dequeue is still **correct** and queues
**5× more** entries — Θ(E) instead of Θ(V) · moving the tentative distance out of the queue entry into
an array makes it *wrong*, on **95,797** vertices here

---

> Next tier: [🟡 medium](2-medium.md) — what the standard library gives you, what a channel really is,
> and what to do when the queue is full.
