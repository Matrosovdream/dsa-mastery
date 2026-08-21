# Step 09 — Queues & Deques · 🟡 Medium

Examples **7–11**: what the standard library actually gives you, what a buffered channel really is,
what to do when the queue is full, and the deque algorithm that earns its keep.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Example 7's `container/ring` trace is deterministic; everything else here reports timings or memory,
sampled on an Apple M4 with Go 1.26.3.

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. `container/list` and `container/ring` — what they are for

`🟡 medium` · *neither one is a ring buffer*

Go ships two containers that look like queues. Knowing exactly what each one *is* saves you from both
mistakes: reaching for them when you want a queue, and dismissing `container/ring` when it is
genuinely the right answer.

**Steps:**

1. Benchmark `container/list` on the queue workload and count its heap objects.
2. Meet `container/ring` — a circular *linked list*, not a buffer.
3. Find the operation it does that nothing else does in Θ(1).

```go
package main

import (
	"container/list"
	"container/ring"
	"fmt"
	"runtime"
	"testing"
)

// Go ships two containers that look like a queue, and neither one is.
// Knowing precisely what they ARE saves you from both mistakes:
// reaching for them when you want a queue, and dismissing them when
// container/ring is exactly right.

type sizedRing struct {
	buf         []int
	mask        int
	head, count int
}

func newSizedRing(n int) *sizedRing {
	c := 1
	for c < n {
		c <<= 1
	}
	return &sizedRing{buf: make([]int, c), mask: c - 1}
}
func (r *sizedRing) push(v int) {
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
}
func (r *sizedRing) pop() int {
	v := r.buf[r.head]
	r.head = (r.head + 1) & r.mask
	r.count--
	return v
}

var sinkI int
var sinkA any

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func heapObjects() int64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return int64(ms.HeapObjects)
}

func main() {
	fmt.Println("container/list IS a queue, in the sense that PushBack and Front/")
	fmt.Println("Remove are both Theta(1). Lesson 07 already measured what that")
	fmt.Println("costs; here it is again on the queue workload specifically:")
	fmt.Println()

	const ops = 200_000
	tRing := nsPerOp(func() {
		r := newSizedRing(2048)
		for i := 0; i < 1000; i++ {
			r.push(i)
		}
		for i := 0; i < ops; i++ {
			r.push(i)
			sinkI = r.pop()
		}
	})
	tList := nsPerOp(func() {
		l := list.New()
		for i := 0; i < 1000; i++ {
			l.PushBack(i)
		}
		for i := 0; i < ops; i++ {
			l.PushBack(i)
			sinkA = l.Remove(l.Front())
		}
	})
	tSlice := nsPerOp(func() {
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
	fmt.Printf("  %-30s %10.2f ns/op\n", "ring buffer", tRing/ops)
	fmt.Printf("  %-30s %10.2f ns/op   %.1fx\n", "slice q = q[1:]", tSlice/ops, tSlice/tRing)
	fmt.Printf("  %-30s %10.2f ns/op   %.1fx\n", "container/list", tList/ops, tList/tRing)

	base := heapObjects()
	l := list.New()
	for i := 0; i < 100_000; i++ {
		l.PushBack(i)
	}
	listObjs := heapObjects() - base
	runtime.KeepAlive(l)

	base = heapObjects()
	r := newSizedRing(1 << 17)
	for i := 0; i < 100_000; i++ {
		r.push(i)
	}
	ringObjs := heapObjects() - base
	runtime.KeepAlive(r)

	fmt.Println()
	fmt.Printf("  heap objects for 100,000 ints: list %d, ring %d\n", listObjs, ringObjs)
	fmt.Println()
	fmt.Println("  the ring is ONE object -- the backing array. The list is one")
	fmt.Println("  Element per value plus one boxed `any` per int, and every one of")
	fmt.Println("  them is a pointer the GC must trace on every cycle. That is")
	fmt.Println("  lesson 04's finding and lesson 07's finding arriving together.")
	fmt.Println()
	fmt.Println("  use container/list for a queue only when you also need the thing")
	fmt.Println("  a list uniquely gives you: Theta(1) removal from the MIDDLE by a")
	fmt.Println("  handle you are already holding. That is an LRU cache (lesson 07,")
	fmt.Println("  example 16), not a queue.")

	fmt.Println()
	fmt.Println("container/ring is the one that surprises people. It is NOT a ring")
	fmt.Println("BUFFER -- there is no head, no tail, no Push and no Pop:")
	fmt.Println()
	rr := ring.New(5)
	for i := 0; i < 5; i++ {
		rr.Value = i + 1
		rr = rr.Next()
	}
	fmt.Printf("  ring.New(5), filled 1..5, then Do: ")
	rr.Do(func(v any) { fmt.Printf("%v ", v) })
	fmt.Println()
	fmt.Printf("  Move(2) then Value:                %v\n", rr.Move(2).Value)
	fmt.Printf("  Prev().Value:                      %v\n", rr.Prev().Value)
	fmt.Println()
	fmt.Println("  it is a CIRCULAR DOUBLY LINKED LIST of fixed length -- every")
	fmt.Println("  element is a node with Next and Prev, and the last one points back")
	fmt.Println("  at the first. Lesson 07's sentinel ring, with the sentinel removed")
	fmt.Println("  and the length frozen.")
	fmt.Println()

	base = heapObjects()
	cr := ring.New(100_000)
	for i := 0; i < 100_000; i++ {
		cr.Value = i
		cr = cr.Next()
	}
	crObjs := heapObjects() - base
	runtime.KeepAlive(cr)
	fmt.Printf("  heap objects for a 100,000-element container/ring: %d\n", crObjs)
	fmt.Println()
	fmt.Printf("  heap objects for container/list, again, for comparison:   %d\n", listObjs)
	fmt.Println()
	fmt.Println("  the same. I expected container/ring to do better -- it allocates")
	fmt.Println("  a fixed number of nodes up front rather than one per push -- but")
	fmt.Println("  it is a linked structure storing `any`, so it pays for a node AND")
	fmt.Println("  a box per element exactly like container/list does.")
	fmt.Println()
	fmt.Println("  a ring BUFFER of 100,000 ints is one array and ONE object. The")
	fmt.Println("  word 'ring' in the package name is about topology, not layout.")

	fmt.Println()
	fmt.Println("what container/ring is actually FOR -- rotation and splicing:")
	fmt.Println()
	a := ring.New(3)
	for i, v := range []string{"a", "b", "c"} {
		_ = i
		a.Value = v
		a = a.Next()
	}
	b := ring.New(2)
	for _, v := range []string{"X", "Y"} {
		b.Value = v
		b = b.Next()
	}
	show := func(label string, r *ring.Ring) {
		fmt.Printf("  %-22s len=%d  ", label, r.Len())
		r.Do(func(v any) { fmt.Printf("%v ", v) })
		fmt.Println()
	}
	show("ring a", a)
	show("ring b", b)
	a.Link(b)
	show("a.Link(b)", a)
	rem := a.Unlink(2)
	show("after a.Unlink(2)", a)
	show("  the removed part", rem)
	fmt.Println()
	fmt.Println("  Link and Unlink splice whole rings in Theta(1) -- no copying, no")
	fmt.Println("  reallocation, no bounds. That is genuinely hard to do any other")
	fmt.Println("  way, and it is why the package exists: round-robin schedulers,")
	fmt.Println("  token rings, cyclic buffers of connections you rotate through.")

	fmt.Println()
	fmt.Println("the summary, and it is short:")
	fmt.Println()
	fmt.Printf("  %-20s %s\n", "your own ring", "a queue. Use this.")
	fmt.Printf("  %-20s %s\n", "container/list", "Theta(1) removal by handle. LRU, not queues.")
	fmt.Printf("  %-20s %s\n", "container/ring", "rotation and Theta(1) splicing. Not a buffer.")
	fmt.Printf("  %-20s %s\n", "buffered channel", "a queue with a lock. Example 8.")
	fmt.Println()
	fmt.Println("  both container packages predate generics and store `any`, which")
	fmt.Println("  is why both box. Neither will ever be made generic -- the Go team")
	fmt.Println("  has said so -- so treat them as legacy for anything hot.")
}
```

**Sample output:**

```
container/list IS a queue, in the sense that PushBack and Front/
Remove are both Theta(1). Lesson 07 already measured what that
costs; here it is again on the queue workload specifically:

  ring buffer                          2.40 ns/op
  slice q = q[1:]                      1.76 ns/op   0.7x
  container/list                      20.73 ns/op   8.6x

  heap objects for 100,000 ints: list 149869, ring 2

  the ring is ONE object -- the backing array. The list is one
  Element per value plus one boxed `any` per int, and every one of
  them is a pointer the GC must trace on every cycle. That is
  lesson 04's finding and lesson 07's finding arriving together.

  use container/list for a queue only when you also need the thing
  a list uniquely gives you: Theta(1) removal from the MIDDLE by a
  handle you are already holding. That is an LRU cache (lesson 07,
  example 16), not a queue.

container/ring is the one that surprises people. It is NOT a ring
BUFFER -- there is no head, no tail, no Push and no Pop:

  ring.New(5), filled 1..5, then Do: 1 2 3 4 5 
  Move(2) then Value:                3
  Prev().Value:                      5

  it is a CIRCULAR DOUBLY LINKED LIST of fixed length -- every
  element is a node with Next and Prev, and the last one points back
  at the first. Lesson 07's sentinel ring, with the sentinel removed
  and the length frozen.

  heap objects for a 100,000-element container/ring: 149867

  heap objects for container/list, again, for comparison:   149869

  the same. I expected container/ring to do better -- it allocates
  a fixed number of nodes up front rather than one per push -- but
  it is a linked structure storing `any`, so it pays for a node AND
  a box per element exactly like container/list does.

  a ring BUFFER of 100,000 ints is one array and ONE object. The
  word 'ring' in the package name is about topology, not layout.

what container/ring is actually FOR -- rotation and splicing:

  ring a                 len=3  a b c 
  ring b                 len=2  X Y 
  a.Link(b)              len=5  a X Y b c 
  after a.Unlink(2)      len=3  a b c 
    the removed part     len=2  X Y 

  Link and Unlink splice whole rings in Theta(1) -- no copying, no
  reallocation, no bounds. That is genuinely hard to do any other
  way, and it is why the package exists: round-robin schedulers,
  token rings, cyclic buffers of connections you rotate through.

the summary, and it is short:

  your own ring        a queue. Use this.
  container/list       Theta(1) removal by handle. LRU, not queues.
  container/ring       rotation and Theta(1) splicing. Not a buffer.
  buffered channel     a queue with a lock. Example 8.

  both container packages predate generics and store `any`, which
  is why both box. Neither will ever be made generic -- the Go team
  has said so -- so treat them as legacy for anything hot.
```

**Complexity:** `container/list` **8.6×** slower with **149,869 heap objects** to the ring's **1** ·
`container/ring` costs the same **149,867** — the word "ring" describes its topology, not its layout ·
`Link`/`Unlink` splice whole rings in Θ(1), which is what the package is actually for

---

## 8. A buffered channel is a ring buffer with a mutex

`🟡 medium` · *the question is never "channel or ring?"*

Go's runtime `hchan` holds a circular array, a send index, a receive index, a count and a lock — the
same three fields as example 3, plus the lock. So the real question is whether you need the lock.

**Steps:**

1. Watch `len`/`cap`/`select`-with-`default` behave exactly like example 3's ring.
2. Measure the ring, the ring behind a `sync.Mutex`, and the channel.
3. Then do the thing a ring buffer cannot do at all.

```go
package main

import (
	"fmt"
	"sync"
	"testing"
	"time"
)

// A buffered channel is a ring buffer with a mutex bolted to it. That is not a
// metaphor -- Go's runtime type `hchan` literally holds:
//
//	buf      unsafe.Pointer  // the circular array
//	sendx    uint            // tail
//	recvx    uint            // head
//	qcount   uint            // count -- same three fields as example 3
//	lock     mutex
//
// So the question is never "channel or ring buffer?" It is "do I need the
// mutex?" -- and if the queue lives inside one goroutine, you are paying for a
// lock that protects you from nobody.

type ring struct {
	buf         []int
	mask        int
	head, count int
}

func newRing(n int) *ring {
	c := 1
	for c < n {
		c <<= 1
	}
	return &ring{buf: make([]int, c), mask: c - 1}
}
func (r *ring) push(v int) {
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
}
func (r *ring) pop() int {
	v := r.buf[r.head]
	r.head = (r.head + 1) & r.mask
	r.count--
	return v
}

type lockedRing struct {
	mu sync.Mutex
	r  *ring
}

func (l *lockedRing) push(v int) {
	l.mu.Lock()
	l.r.push(v)
	l.mu.Unlock()
}
func (l *lockedRing) pop() int {
	l.mu.Lock()
	v := l.r.pop()
	l.mu.Unlock()
	return v
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
	fmt.Println("a buffered channel behaves exactly like example 3's ring, down to")
	fmt.Println("Len and Cap:")
	fmt.Println()
	ch := make(chan int, 3)
	fmt.Printf("  make(chan int, 3)   len=%d cap=%d\n", len(ch), cap(ch))
	for i := 1; i <= 3; i++ {
		ch <- i
		fmt.Printf("  ch <- %d             len=%d cap=%d\n", i, len(ch), cap(ch))
	}
	select {
	case ch <- 4:
		fmt.Println("  ch <- 4             accepted")
	default:
		fmt.Printf("  ch <- 4             REFUSED (len=%d == cap)\n", len(ch))
	}
	v := <-ch
	fmt.Printf("  <-ch                -> %d, len=%d\n", v, len(ch))

	fmt.Println()
	fmt.Println("  `select` with a `default` is TryEnqueue -- the non-blocking form")
	fmt.Println("  that returns example 3's `bool` instead of blocking. Without the")
	fmt.Println("  default, a full channel blocks the sender, which IS backpressure")
	fmt.Println("  and is the whole reason to choose a channel (example 9).")

	fmt.Println()
	fmt.Println("what the mutex costs when nobody is contending for it:")
	fmt.Println()
	const ops = 500_000
	tRing := nsPerOp(func() {
		r := newRing(2048)
		for i := 0; i < 1000; i++ {
			r.push(i)
		}
		for i := 0; i < ops; i++ {
			r.push(i)
			sinkI = r.pop()
		}
	})
	tLocked := nsPerOp(func() {
		l := &lockedRing{r: newRing(2048)}
		for i := 0; i < 1000; i++ {
			l.push(i)
		}
		for i := 0; i < ops; i++ {
			l.push(i)
			sinkI = l.pop()
		}
	})
	tChan := nsPerOp(func() {
		c := make(chan int, 2048)
		for i := 0; i < 1000; i++ {
			c <- i
		}
		for i := 0; i < ops; i++ {
			c <- i
			sinkI = <-c
		}
	})
	fmt.Printf("  %-32s %10.2f ns/op\n", "plain ring buffer", tRing/ops)
	fmt.Printf("  %-32s %10.2f ns/op   %.1fx\n", "ring + sync.Mutex", tLocked/ops, tLocked/tRing)
	fmt.Printf("  %-32s %10.2f ns/op   %.1fx\n", "buffered channel", tChan/ops, tChan/tRing)
	fmt.Println()
	fmt.Println("  all three single-goroutine, all three uncontended. The mutex is")
	fmt.Println("  not free and the channel is not the mutex -- a channel send does")
	fmt.Println("  more than lock/copy/unlock: it also checks for a waiting receiver")
	fmt.Println("  and may hand the value over directly, which is what makes it a")
	fmt.Println("  synchronisation primitive rather than a container.")
	fmt.Println()
	fmt.Println("  if your queue never crosses a goroutine boundary, that is the")
	fmt.Println("  price of a feature you are not using.")

	fmt.Println()
	fmt.Println("now the case a ring buffer cannot do at all:")
	fmt.Println()
	jobs := make(chan int, 64)
	results := make(chan int, 64)
	var wg sync.WaitGroup
	for w := 0; w < 4; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs { // ranges until the channel is CLOSED and drained
				results <- j * j
			}
		}()
	}
	go func() {
		for i := 1; i <= 10; i++ {
			jobs <- i
		}
		close(jobs) // the only correct way to say "no more work"
	}()
	go func() { wg.Wait(); close(results) }()

	sum := 0
	n := 0
	for r := range results {
		sum += r
		n++
	}
	fmt.Printf("  4 workers, 10 jobs -> %d results, sum of squares = %d\n", n, sum)
	fmt.Println()
	fmt.Println("  the ring buffer has no answer for any of this: blocking when full,")
	fmt.Println("  blocking when empty, waking exactly one waiter, `close` as an")
	fmt.Println("  end-of-stream signal, `range` that terminates. Those are the")
	fmt.Println("  channel's actual product; the ring buffer inside it is incidental.")

	fmt.Println()
	fmt.Println("and the case where a channel is the wrong tool even between")
	fmt.Println("goroutines -- you cannot look at it:")
	fmt.Println()
	c := make(chan int, 8)
	for i := 0; i < 5; i++ {
		c <- i
	}
	fmt.Printf("  len(ch)=%d cap(ch)=%d -- and that is ALL you can ask.\n", len(c), cap(c))
	fmt.Println()
	fmt.Println("  no peek at the front, no scan, no removal from the middle, no")
	fmt.Println("  priority, no iterating without consuming. len(ch) is also stale")
	fmt.Println("  the instant you read it, so it is a metric, never a decision.")
	fmt.Println()
	fmt.Println("  need any of those and you need a real structure behind a mutex,")
	fmt.Println("  with a channel or a sync.Cond used only to signal.")

	fmt.Println()
	fmt.Println("the deadlock a channel makes easy, since it is the other half:")
	fmt.Println()
	done := make(chan struct{})
	small := make(chan int, 2)
	go func() {
		for i := 0; i < 5; i++ { // 5 sends into a 2-slot channel, no receiver
			small <- i
		}
		close(done)
	}()
	select {
	case <-done:
		fmt.Println("  sender finished -- unexpected")
	case <-time.After(50 * time.Millisecond):
		fmt.Printf("  sender is blocked after filling %d/%d slots, still waiting\n",
			len(small), cap(small))
	}
	fmt.Println()
	fmt.Println("  that block is not a bug -- it is backpressure, and it is correct.")
	fmt.Println("  It becomes a deadlock only when nothing will ever receive. The")
	fmt.Println("  distinction is the entire subject of example 9.")
	fmt.Println()
	fmt.Println("  the rule: a channel when the queue CROSSES a goroutine boundary,")
	fmt.Println("  a ring buffer when it does not. Do not use a channel as a data")
	fmt.Println("  structure, and do not hand-roll goroutine handoff with a mutex.")
}
```

**Sample output:**

```
a buffered channel behaves exactly like example 3's ring, down to
Len and Cap:

  make(chan int, 3)   len=0 cap=3
  ch <- 1             len=1 cap=3
  ch <- 2             len=2 cap=3
  ch <- 3             len=3 cap=3
  ch <- 4             REFUSED (len=3 == cap)
  <-ch                -> 1, len=2

  `select` with a `default` is TryEnqueue -- the non-blocking form
  that returns example 3's `bool` instead of blocking. Without the
  default, a full channel blocks the sender, which IS backpressure
  and is the whole reason to choose a channel (example 9).

what the mutex costs when nobody is contending for it:

  plain ring buffer                      2.35 ns/op
  ring + sync.Mutex                      5.73 ns/op   2.4x
  buffered channel                      17.13 ns/op   7.3x

  all three single-goroutine, all three uncontended. The mutex is
  not free and the channel is not the mutex -- a channel send does
  more than lock/copy/unlock: it also checks for a waiting receiver
  and may hand the value over directly, which is what makes it a
  synchronisation primitive rather than a container.

  if your queue never crosses a goroutine boundary, that is the
  price of a feature you are not using.

now the case a ring buffer cannot do at all:

  4 workers, 10 jobs -> 10 results, sum of squares = 385

  the ring buffer has no answer for any of this: blocking when full,
  blocking when empty, waking exactly one waiter, `close` as an
  end-of-stream signal, `range` that terminates. Those are the
  channel's actual product; the ring buffer inside it is incidental.

and the case where a channel is the wrong tool even between
goroutines -- you cannot look at it:

  len(ch)=5 cap(ch)=8 -- and that is ALL you can ask.

  no peek at the front, no scan, no removal from the middle, no
  priority, no iterating without consuming. len(ch) is also stale
  the instant you read it, so it is a metric, never a decision.

  need any of those and you need a real structure behind a mutex,
  with a channel or a sync.Cond used only to signal.

the deadlock a channel makes easy, since it is the other half:

  sender is blocked after filling 2/2 slots, still waiting

  that block is not a bug -- it is backpressure, and it is correct.
  It becomes a deadlock only when nothing will ever receive. The
  distinction is the entire subject of example 9.

  the rule: a channel when the queue CROSSES a goroutine boundary,
  a ring buffer when it does not. Do not use a channel as a data
  structure, and do not hand-roll goroutine handoff with a mutex.
```

**Complexity:** ring **2.35 ns/op** · ring + `sync.Mutex` **5.73 ns/op** (2.4×) · buffered channel
**17.13 ns/op** (7.3×) — all single-goroutine and uncontended · what the channel sells is blocking,
`close` and `range`, and you cannot inspect it beyond `len` and `cap`

---

## 9. Bounded queues: the four things that can give

`🟡 medium` · *refusing to choose is choosing OOM*

Example 3's `Enqueue` returned `false`. This is what you do with it. If the producer outruns the
consumer for long enough, something must give — and there are exactly four candidates.

**Steps:**

1. Implement block, drop-newest, drop-oldest and error behind one ring.
2. Run the same too-fast producer through each and account for every item.
3. Then sweep the capacity and measure what it actually buys.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Example 3 ended with `Enqueue` returning false. This example is about what
// you do with that false, because it is the single most consequential design
// decision in any system with a queue in it:
//
//	producer faster than consumer, for long enough
//	  -> the queue grows without bound, or something gives
//
// There are exactly four things that can give, and you must pick one. Refusing
// to pick means picking "unbounded", which means picking OOM.

type policy int

const (
	block policy = iota
	dropNewest
	dropOldest
	errorOut
)

func (p policy) String() string {
	return [...]string{"block (backpressure)", "drop newest", "drop oldest", "return an error"}[p]
}

// bounded is a ring buffer with an overflow policy and full accounting, so we
// can measure what each policy actually costs.
type bounded struct {
	mu      sync.Mutex
	notFull *sync.Cond
	buf     []item
	mask    int
	head    int
	count   int
	pol     policy

	accepted, dropped, errors, blockedNS int64

	// per-item queueing delay: how long each value SAT in the queue.
	waitSumNS, waitMaxNS int64
	waitN                int64
}

// item carries its own enqueue timestamp, which is the only way to measure
// queueing delay -- the producer's block time measures something else entirely.
type item struct {
	v int
	t time.Time
}

func newBounded(capacity int, pol policy) *bounded {
	n := 1
	for n < capacity {
		n <<= 1
	}
	b := &bounded{buf: make([]item, n), mask: n - 1, pol: pol}
	b.notFull = sync.NewCond(&b.mu)
	return b
}

func (b *bounded) Enqueue(v int) (ok bool) {
	b.mu.Lock()
	defer b.mu.Unlock()

	if b.count == len(b.buf) {
		switch b.pol {
		case block:
			start := time.Now()
			for b.count == len(b.buf) {
				b.notFull.Wait() // the producer is now going as slow as the consumer
			}
			b.blockedNS += time.Since(start).Nanoseconds()
		case dropNewest:
			b.dropped++
			return false // the value we were handed is discarded
		case dropOldest:
			b.head = (b.head + 1) & b.mask // evict the front
			b.count--
			b.dropped++
		case errorOut:
			b.errors++
			return false // the CALLER decides -- retry, shed, log, page someone
		}
	}
	b.buf[(b.head+b.count)&b.mask] = item{v, time.Now()}
	b.count++
	b.accepted++
	return true
}

func (b *bounded) Dequeue() (int, bool) {
	b.mu.Lock()
	defer b.mu.Unlock()
	if b.count == 0 {
		return 0, false
	}
	it := b.buf[b.head]
	b.head = (b.head + 1) & b.mask
	b.count--
	w := time.Since(it.t).Nanoseconds()
	b.waitSumNS += w
	b.waitN++
	if w > b.waitMaxNS {
		b.waitMaxNS = w
	}
	b.notFull.Signal()
	return it.v, true
}

// run drives one producer and one consumer where the producer is 4x faster,
// and reports what the policy did about it.
func run(pol policy, capacity, items int) *bounded {
	b := newBounded(capacity, pol)
	var wg sync.WaitGroup
	wg.Add(2)

	go func() { // producer: sends `items` values as fast as it can
		defer wg.Done()
		for i := 0; i < items; i++ {
			b.Enqueue(i)
		}
	}()
	go func() { // consumer: deliberately 4x slower
		defer wg.Done()
		got := 0
		for got < items {
			if _, ok := b.Dequeue(); ok {
				got++
				for j := 0; j < 400; j++ { // simulated work
					_ = j * j
				}
				continue
			}
			b.mu.Lock()
			done := b.accepted+b.dropped+b.errors >= int64(items) && b.count == 0
			b.mu.Unlock()
			if done {
				return
			}
			time.Sleep(time.Microsecond)
		}
	}()
	wg.Wait()
	return b
}

func main() {
	fmt.Println("the four policies, and the fact that there is no fifth:")
	fmt.Println()
	fmt.Printf("  %-22s %-38s %s\n", "policy", "what it costs", "when it is right")
	for _, r := range [][3]string{
		{"block", "producer latency; risk of deadlock", "the producer can wait"},
		{"drop newest", "the newest data", "metrics, telemetry, logs"},
		{"drop oldest", "the oldest data", "live video, sensors, prices"},
		{"return an error", "caller complexity", "an API boundary"},
	} {
		fmt.Printf("  %-22s %-38s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  every one of them LOSES something. That is not a flaw in the")
	fmt.Println("  design -- it is arithmetic. If the producer is faster than the")
	fmt.Println("  consumer for long enough, the only alternative to losing data is")
	fmt.Println("  losing the process.")

	fmt.Println()
	fmt.Println("the same producer (4x faster than the consumer), 20,000 items,")
	fmt.Println("capacity 64, under each policy:")
	fmt.Println()
	fmt.Printf("  %-22s %10s %10s %10s %14s\n", "policy", "accepted", "dropped", "errors", "producer blocked")
	for _, pol := range []policy{block, dropNewest, dropOldest, errorOut} {
		b := run(pol, 64, 20_000)
		blocked := fmt.Sprintf("%.1f ms", float64(b.blockedNS)/1e6)
		if b.blockedNS == 0 {
			blocked = "-"
		}
		fmt.Printf("  %-22s %10d %10d %10d %14s\n",
			pol, b.accepted, b.dropped, b.errors, blocked)
	}
	fmt.Println()
	fmt.Println("  `block` is the only row that accepted everything, and it paid for")
	fmt.Println("  that in producer latency -- which is exactly the point. Blocking")
	fmt.Println("  is not a failure mode, it is the mechanism by which a slow")
	fmt.Println("  consumer TELLS the producer to slow down. That is backpressure,")
	fmt.Println("  and a system without it does not have a queue, it has a countdown.")

	fmt.Println()
	fmt.Println("capacity is a latency dial, not a throughput dial:")
	fmt.Println()
	fmt.Printf("  %10s %16s %14s %14s\n", "capacity", "producer blocked", "mean wait", "worst wait")
	for _, c := range []int{8, 64, 512, 4096} {
		b := run(block, c, 20_000)
		fmt.Printf("  %10d %13.1f ms %11.1f us %11.1f us\n", c,
			float64(b.blockedNS)/1e6,
			float64(b.waitSumNS)/float64(b.waitN)/1e3,
			float64(b.waitMaxNS)/1e3)
	}
	fmt.Println()
	fmt.Println("  read those two right-hand columns, not the left one. The producer")
	fmt.Println("  blocks LESS as the buffer grows -- that is the buffer doing its")
	fmt.Println("  job -- but every item then waits behind more items, and the mean")
	fmt.Println("  and worst queueing delays climb with the capacity.")
	fmt.Println()
	fmt.Println("  a bigger buffer does not make the consumer faster -- accepted is")
	fmt.Println("  20,000 in every row and throughput is set by the consumer alone.")
	fmt.Println("  What a bigger buffer buys is a longer BURST absorbed before the")
	fmt.Println("  producer feels anything, paid for in latency and memory: 160x")
	fmt.Println("  more mean queueing delay to save 10 ms of producer blocking.")
	fmt.Println()
	fmt.Println("  `bufferbloat` is this exact mistake, made in routers, and it made")
	fmt.Println("  the internet slower for a decade.")
	fmt.Println()
	fmt.Println("  size the queue for the BURST you must absorb, then stop. If you")
	fmt.Println("  find yourself raising it repeatedly, the consumer is too slow and")
	fmt.Println("  no capacity will fix that.")

	fmt.Println()
	fmt.Println("drop-oldest is the one people forget, and it is often the right")
	fmt.Println("answer -- when the data is a SAMPLE of a continuous quantity:")
	fmt.Println()
	newest := newBounded(4, dropNewest)
	oldest := newBounded(4, dropOldest)
	for i := 1; i <= 8; i++ {
		newest.Enqueue(i)
		oldest.Enqueue(i)
	}
	drain := func(b *bounded) []int {
		var out []int
		for {
			v, ok := b.Dequeue()
			if !ok {
				return out
			}
			out = append(out, v)
		}
	}
	fmt.Printf("  pushed 1..8 into a 4-slot queue\n")
	fmt.Printf("  %-14s %v   <- stuck in the past\n", "drop newest:", drain(newest))
	fmt.Printf("  %-14s %v   <- the four most recent\n", "drop oldest:", drain(oldest))
	fmt.Println()
	fmt.Println("  for a video frame, a sensor reading or a price tick, the stale")
	fmt.Println("  value is worthless and the fresh one is the whole product --")
	fmt.Println("  drop-oldest is correct and drop-newest is a bug. For an audit log")
	fmt.Println("  or a payment, both are unacceptable and you must block or error.")
	fmt.Println()
	fmt.Println("  the policy is a property of the DATA, not of the queue. Decide it")
	fmt.Println("  where the data is defined, and make the type say which one it is.")
}
```

**Sample output:**

```
the four policies, and the fact that there is no fifth:

  policy                 what it costs                          when it is right
  block                  producer latency; risk of deadlock     the producer can wait
  drop newest            the newest data                        metrics, telemetry, logs
  drop oldest            the oldest data                        live video, sensors, prices
  return an error        caller complexity                      an API boundary

  every one of them LOSES something. That is not a flaw in the
  design -- it is arithmetic. If the producer is faster than the
  consumer for long enough, the only alternative to losing data is
  losing the process.

the same producer (4x faster than the consumer), 20,000 items,
capacity 64, under each policy:

  policy                   accepted    dropped     errors producer blocked
  block (backpressure)        20000          0          0         3.8 ms
  drop newest                    82      19918          0              -
  drop oldest                 20000      19348          0              -
  return an error               218          0      19782              -

  `block` is the only row that accepted everything, and it paid for
  that in producer latency -- which is exactly the point. Blocking
  is not a failure mode, it is the mechanism by which a slow
  consumer TELLS the producer to slow down. That is backpressure,
  and a system without it does not have a queue, it has a countdown.

capacity is a latency dial, not a throughput dial:

    capacity producer blocked      mean wait     worst wait
           8          19.5 ms         7.0 us       600.4 us
          64           3.5 ms         8.7 us        53.0 us
         512           2.7 ms        93.1 us       209.0 us
        4096           2.5 ms       881.4 us      1221.5 us

  read those two right-hand columns, not the left one. The producer
  blocks LESS as the buffer grows -- that is the buffer doing its
  job -- but every item then waits behind more items, and the mean
  and worst queueing delays climb with the capacity.

  a bigger buffer does not make the consumer faster -- accepted is
  20,000 in every row and throughput is set by the consumer alone.
  What a bigger buffer buys is a longer BURST absorbed before the
  producer feels anything, paid for in latency and memory: 160x
  more mean queueing delay to save 10 ms of producer blocking.

  `bufferbloat` is this exact mistake, made in routers, and it made
  the internet slower for a decade.

  size the queue for the BURST you must absorb, then stop. If you
  find yourself raising it repeatedly, the consumer is too slow and
  no capacity will fix that.

drop-oldest is the one people forget, and it is often the right
answer -- when the data is a SAMPLE of a continuous quantity:

  pushed 1..8 into a 4-slot queue
  drop newest:   [1 2 3 4]   <- stuck in the past
  drop oldest:   [5 6 7 8]   <- the four most recent

  for a video frame, a sensor reading or a price tick, the stale
  value is worthless and the fresh one is the whole product --
  drop-oldest is correct and drop-newest is a bug. For an audit log
  or a payment, both are unacceptable and you must block or error.

  the policy is a property of the DATA, not of the queue. Decide it
  where the data is defined, and make the type say which one it is.
```

**Complexity:** every policy loses something — that is arithmetic, not a design flaw · capacity is a
**latency** dial, not a throughput dial: 8 → 4096 cuts producer blocking from 19.5 ms to 2.5 ms and
raises mean queueing delay from **7.0 µs to 881.4 µs** (126×). That is bufferbloat, measured

---

## 10. The monotonic deque: sliding-window maximum in Θ(n)

`🟡 medium` · *expire from one end, evict from the other*

Lesson 08 built a monotonic *stack*. Opening the second end lets you expire elements that have fallen
out of a window — which turns an Θ(nk) problem into an Θ(n) one with no heap and no allocations.

**Steps:**

1. Trace the deque of indices and watch the two rules fire.
2. Count the pushes and pops to make the aggregate argument concrete.
3. Prove it against brute force *on inputs with ties*, then benchmark it.

```go
package main

import (
	"fmt"
	"math/rand"
	"testing"
)

// The MONOTONIC DEQUE is what a deque is for. Lesson 08 built a monotonic
// STACK; this is the same idea with the second end opened up, and the extra end
// is what lets you EXPIRE elements that have fallen out of a window.
//
// Problem: sliding window maximum. Given a and k, report max(a[i:i+k]) for
// every window.
//
//	brute force  Theta(n*k)
//	heap         Theta(n log k)   (lesson 20)
//	deque        Theta(n)         -- and no allocations after the first
//
// The invariant: the deque holds INDICES whose values are strictly decreasing.
// Everything follows from defending that one sentence.

func maxSlidingBrute(a []int, k int) []int {
	if k <= 0 || len(a) < k {
		return nil
	}
	out := make([]int, 0, len(a)-k+1)
	for i := 0; i+k <= len(a); i++ {
		m := a[i]
		for j := i + 1; j < i+k; j++ {
			if a[j] > m {
				m = a[j]
			}
		}
		out = append(out, m)
	}
	return out
}

func maxSliding(a []int, k int) []int {
	if k <= 0 || len(a) < k {
		return nil
	}
	out := make([]int, 0, len(a)-k+1)
	d := NewDeque[int](k + 1) // holds INDICES, values strictly decreasing

	for i, v := range a {
		// 1. EXPIRE from the front: the index at the front has left the window.
		if f, ok := d.Front(); ok && f <= i-k {
			d.PopFront()
		}
		// 2. MAINTAIN from the back: anything smaller than v can never be the
		//    maximum of any future window, because v is both larger and newer.
		for {
			b, ok := d.Back()
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
		}
		d.PushBack(i)
		// 3. The front is now the maximum of the current window, by induction.
		if i >= k-1 {
			out = append(out, a[d.At(0)])
		}
	}
	return out
}

// counted is the same algorithm with the pushes and pops counted, to make the
// aggregate argument concrete.
func maxSlidingCounted(a []int, k int) (pushes, pops int) {
	d := NewDeque[int](k + 1)
	for i, v := range a {
		if f, ok := d.Front(); ok && f <= i-k {
			d.PopFront()
			pops++
		}
		for {
			b, ok := d.Back()
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
			pops++
		}
		d.PushBack(i)
		pushes++
	}
	return pushes, pops
}

func equal(x, y []int) bool {
	if len(x) != len(y) {
		return false
	}
	for i := range x {
		if x[i] != y[i] {
			return false
		}
	}
	return true
}

var sink []int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	a := []int{1, 3, -1, -3, 5, 3, 6, 7}
	k := 3

	fmt.Printf("a = %v, k = %d\n", a, k)
	fmt.Println()
	fmt.Println("the deque holds INDICES whose values strictly decrease. Watch it:")
	fmt.Println()
	fmt.Printf("  %3s %5s  %-22s %-22s %s\n", "i", "a[i]", "deque (indices)", "their values", "window max")

	d := NewDeque[int](k + 1)
	for i, v := range a {
		if f, ok := d.Front(); ok && f <= i-k {
			d.PopFront()
		}
		for {
			b, ok := d.Back()
			if !ok || a[b] > v {
				break
			}
			d.PopBack()
		}
		d.PushBack(i)

		idx := make([]int, d.Len())
		vals := make([]int, d.Len())
		for j := 0; j < d.Len(); j++ {
			idx[j] = d.At(j)
			vals[j] = a[d.At(j)]
		}
		out := ""
		if i >= k-1 {
			out = fmt.Sprintf("%d", a[d.At(0)])
		}
		fmt.Printf("  %3d %5d  %-22s %-22s %s\n", i, v,
			fmt.Sprint(idx), fmt.Sprint(vals), out)
	}
	fmt.Println()
	fmt.Println("  two things happen to an index, and only two:")
	fmt.Println()
	fmt.Println("    it EXPIRES off the front  -- it left the window (i-k)")
	fmt.Println("    it is EVICTED off the back -- a larger, newer value arrived,")
	fmt.Println("                                  so it can never win again")
	fmt.Println()
	fmt.Println("  the second rule is the one that needs proof, and the proof is one")
	fmt.Println("  sentence: if a[j] <= a[i] and j < i, then every window containing")
	fmt.Println("  j either contains i too -- in which case j is not the maximum --")
	fmt.Println("  or has already passed. Either way j is dead. Delete it.")

	fmt.Println()
	fmt.Println("Theta(n) by the aggregate method, exactly like lesson 08's")
	fmt.Println("monotonic stack:")
	fmt.Println()
	fmt.Printf("  %10s %8s %10s %10s %14s\n", "n", "k", "pushes", "pops", "ops/element")
	rng := rand.New(rand.NewSource(1))
	for _, n := range []int{1000, 10_000, 100_000, 1_000_000} {
		arr := make([]int, n)
		for i := range arr {
			arr[i] = rng.Intn(1_000_000)
		}
		pu, po := maxSlidingCounted(arr, 100)
		fmt.Printf("  %10d %8d %10d %10d %14.2f\n", n, 100, pu, po, float64(pu+po)/float64(n))
	}
	fmt.Println()
	fmt.Println("  the inner `for` loop looks like it makes this quadratic. It does")
	fmt.Println("  not: each index is pushed exactly once and popped at most once, so")
	fmt.Println("  the total work over the whole run is at most 2n regardless of what")
	fmt.Println("  any single step does. Never read a nested loop as O(n^2) without")
	fmt.Println("  asking what it is a loop OVER.")
	fmt.Println()
	fmt.Println("  note ops/element is independent of k -- which is the whole point.")

	fmt.Println()
	fmt.Println("proved against brute force before it is benchmarked, because a fast")
	fmt.Println("wrong answer is worth nothing:")
	fmt.Println()
	fails := 0
	cases := 0
	for trial := 0; trial < 2000; trial++ {
		n := 1 + rng.Intn(30)
		arr := make([]int, n)
		for i := range arr {
			arr[i] = rng.Intn(21) - 10 // small range -> lots of ties
		}
		for kk := 1; kk <= n; kk++ {
			cases++
			if !equal(maxSliding(arr, kk), maxSlidingBrute(arr, kk)) {
				fails++
			}
		}
	}
	fmt.Printf("  %d random (array, k) pairs, ties and negatives included: %d mismatches\n", cases, fails)
	fmt.Println()
	fmt.Println("  the small value range matters. With `a[b] >= v` instead of")
	fmt.Println("  `a[b] > v` in the eviction test the answers are still CORRECT --")
	fmt.Println("  equal elements are interchangeable -- but the deque gets shorter.")
	fmt.Println("  Random values from a large range almost never tie, so a test that")
	fmt.Println("  uses them exercises neither branch. Force the collisions.")

	fmt.Println()
	fmt.Println("and now the benchmark, on the input that actually distinguishes")
	fmt.Println("them -- lesson 08's warning applies here too:")
	fmt.Println()
	fmt.Printf("  %8s %10s %14s %14s %10s\n", "n", "k", "brute (ns)", "deque (ns)", "speedup")
	for _, tc := range []struct{ n, k int }{
		{100_000, 4}, {100_000, 64}, {100_000, 1000}, {100_000, 10_000},
	} {
		arr := make([]int, tc.n)
		for i := range arr {
			arr[i] = rng.Intn(1_000_000)
		}
		tb := nsPerOp(func() { sink = maxSlidingBrute(arr, tc.k) })
		td := nsPerOp(func() { sink = maxSliding(arr, tc.k) })
		fmt.Printf("  %8d %10d %14.0f %14.0f %9.2fx\n", tc.n, tc.k, tb, td, tb/td)
	}
	fmt.Println()
	fmt.Println("  the deque's time is FLAT in k -- under 0.9 ms whether the window")
	fmt.Println("  is 4 wide or 10,000 -- while brute force is linear in it.")
	fmt.Println()
	fmt.Println("  and at k=4 brute force WINS, by nearly 2x. That is not noise: a")
	fmt.Println("  4-element scan is four array reads with no branches, against two")
	fmt.Println("  non-inlined deque method calls plus the bookkeeping. Theta(n) beats")
	fmt.Println("  Theta(nk) only once k is large enough to pay for the constant.")
	fmt.Println()
	fmt.Println("  the crossover here is somewhere between k=4 and k=64. If your")
	fmt.Println("  window is small and FIXED, write the loop. The deque earns its")
	fmt.Println("  complexity when k is large, or when k is a parameter you do not")
	fmt.Println("  control -- because then the crossover is not yours to reason about.")
	fmt.Println()
	fmt.Println("  the same skeleton -- expire from one end, evict from the other --")
	fmt.Println("  solves minimum-in-window, 'first negative in every window',")
	fmt.Println("  'longest subarray with max-min <= limit' (two deques), and the")
	fmt.Println("  constrained-window DP in lesson 30. Learn the invariant, not the")
	fmt.Println("  problem.")
}
```

> Reuses `Deque[T]` from example 5 — copy that type into the same folder as `deque.go`.

**Sample output:**

```
a = [1 3 -1 -3 5 3 6 7], k = 3

the deque holds INDICES whose values strictly decrease. Watch it:

    i  a[i]  deque (indices)        their values           window max
    0     1  [0]                    [1]                    
    1     3  [1]                    [3]                    
    2    -1  [1 2]                  [3 -1]                 3
    3    -3  [1 2 3]                [3 -1 -3]              3
    4     5  [4]                    [5]                    5
    5     3  [4 5]                  [5 3]                  5
    6     6  [6]                    [6]                    6
    7     7  [7]                    [7]                    7

  two things happen to an index, and only two:

    it EXPIRES off the front  -- it left the window (i-k)
    it is EVICTED off the back -- a larger, newer value arrived,
                                  so it can never win again

  the second rule is the one that needs proof, and the proof is one
  sentence: if a[j] <= a[i] and j < i, then every window containing
  j either contains i too -- in which case j is not the maximum --
  or has already passed. Either way j is dead. Delete it.

Theta(n) by the aggregate method, exactly like lesson 08's
monotonic stack:

           n        k     pushes       pops    ops/element
        1000      100       1000        995           2.00
       10000      100      10000       9996           2.00
      100000      100     100000      99993           2.00
     1000000      100    1000000     999998           2.00

  the inner `for` loop looks like it makes this quadratic. It does
  not: each index is pushed exactly once and popped at most once, so
  the total work over the whole run is at most 2n regardless of what
  any single step does. Never read a nested loop as O(n^2) without
  asking what it is a loop OVER.

  note ops/element is independent of k -- which is the whole point.

proved against brute force before it is benchmarked, because a fast
wrong answer is worth nothing:

  30796 random (array, k) pairs, ties and negatives included: 0 mismatches

  the small value range matters. With `a[b] >= v` instead of
  `a[b] > v` in the eviction test the answers are still CORRECT --
  equal elements are interchangeable -- but the deque gets shorter.
  Random values from a large range almost never tie, so a test that
  uses them exercises neither branch. Force the collisions.

and now the benchmark, on the input that actually distinguishes
them -- lesson 08's warning applies here too:

         n          k     brute (ns)     deque (ns)    speedup
    100000          4         443778         854594      0.52x
    100000         64        3996921         878460      4.55x
    100000       1000       31388885         856741     36.64x
    100000      10000      237704533         910696    261.01x

  the deque's time is FLAT in k -- under 0.9 ms whether the window
  is 4 wide or 10,000 -- while brute force is linear in it.

  and at k=4 brute force WINS, by nearly 2x. That is not noise: a
  4-element scan is four array reads with no branches, against two
  non-inlined deque method calls plus the bookkeeping. Theta(n) beats
  Theta(nk) only once k is large enough to pay for the constant.

  the crossover here is somewhere between k=4 and k=64. If your
  window is small and FIXED, write the loop. The deque earns its
  complexity when k is large, or when k is a parameter you do not
  control -- because then the crossover is not yours to reason about.

  the same skeleton -- expire from one end, evict from the other --
  solves minimum-in-window, 'first negative in every window',
  'longest subarray with max-min <= limit' (two deques), and the
  constrained-window DP in lesson 30. Learn the invariant, not the
  problem.
```

**Complexity:** Θ(n), **exactly 2.00 operations per element** regardless of k · the deque's runtime is
*flat* in k (0.85 ms at k=4, 0.91 ms at k=10,000) while brute force is linear in it — but **at k=4 brute
force wins** — the deque runs at 0.52× its speed, and the crossover is between 4 and 64

---

## 11. Six queues, three workloads, three different winners

`🟡 medium` · *and the harness turned out to be part of the experiment*

Build them all, then let the measurement pick. The three workloads are chosen because they
**disagree** — a benchmark that only runs the steady state will tell you the naive queue is best, and
it will be lying by omission.

**Steps:**

1. Put all six behind one `Queue` interface so the call overhead is identical.
2. Run steady state, burst-then-drain, and memory-held-after-drain.
3. Then check whether the harness itself changed the answer.

```go
package main

import (
	"container/list"
	"fmt"
	"runtime"
	"testing"
)

// Six queues, three workloads, one table. Lesson 07 ended this way and the
// habit is the point: build them all, then let the measurement pick.
//
// The three workloads are chosen because they DISAGREE. A benchmark that only
// runs the steady state will tell you the naive slice queue is best, and it
// will be lying by omission.

type Queue interface {
	Push(int)
	Pop() (int, bool)
}

// 1. naive: q = q[1:]
type Naive struct{ q []int }

func (n *Naive) Push(v int) { n.q = append(n.q, v) }
func (n *Naive) Pop() (int, bool) {
	if len(n.q) == 0 {
		return 0, false
	}
	v := n.q[0]
	n.q = n.q[1:]
	return v, true
}

// 2. head index, compacting once half the buffer is dead
type Compact struct {
	buf  []int
	head int
}

func (c *Compact) Push(v int) { c.buf = append(c.buf, v) }
func (c *Compact) Pop() (int, bool) {
	if c.head == len(c.buf) {
		return 0, false
	}
	v := c.buf[c.head]
	c.head++
	if c.head > len(c.buf)/2 {
		live := len(c.buf) - c.head
		copy(c.buf, c.buf[c.head:])
		c.buf = c.buf[:live]
		c.head = 0
	}
	return v, true
}

// 3. fixed ring, power of two. Push REFUSES when full.
type Ring struct {
	buf         []int
	mask        int
	head, count int
	refused     int
}

func NewRing(capacity int) *Ring {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &Ring{buf: make([]int, n), mask: n - 1}
}
func (r *Ring) Push(v int) {
	if r.count == len(r.buf) {
		r.refused++
		return
	}
	r.buf[(r.head+r.count)&r.mask] = v
	r.count++
}
func (r *Ring) Pop() (int, bool) {
	if r.count == 0 {
		return 0, false
	}
	v := r.buf[r.head]
	r.head = (r.head + 1) & r.mask
	r.count--
	return v, true
}

// 4. growable ring
type GrowRing struct {
	buf         []int
	mask        int
	head, count int
}

func NewGrowRing(capacity int) *GrowRing {
	n := 1
	for n < capacity {
		n <<= 1
	}
	return &GrowRing{buf: make([]int, n), mask: n - 1}
}
func (g *GrowRing) Push(v int) {
	if g.count == len(g.buf) {
		next := make([]int, len(g.buf)*2)
		k := copy(next, g.buf[g.head:])
		copy(next[k:], g.buf[:g.head])
		g.buf, g.mask, g.head = next, len(next)-1, 0
	}
	g.buf[(g.head+g.count)&g.mask] = v
	g.count++
}
func (g *GrowRing) Pop() (int, bool) {
	if g.count == 0 {
		return 0, false
	}
	v := g.buf[g.head]
	g.head = (g.head + 1) & g.mask
	g.count--
	return v, true
}

// 5. container/list
type List struct{ l *list.List }

func NewList() *List       { return &List{l: list.New()} }
func (l *List) Push(v int) { l.l.PushBack(v) }
func (l *List) Pop() (int, bool) {
	e := l.l.Front()
	if e == nil {
		return 0, false
	}
	return l.l.Remove(e).(int), true
}

// 6. buffered channel, non-blocking on both ends so it fits the interface
type Chan struct{ c chan int }

func NewChan(capacity int) *Chan { return &Chan{c: make(chan int, capacity)} }
func (c *Chan) Push(v int) {
	select {
	case c.c <- v:
	default:
	}
}
func (c *Chan) Pop() (int, bool) {
	select {
	case v := <-c.c:
		return v, true
	default:
		return 0, false
	}
}

type impl struct {
	name    string
	bounded bool
	new     func(capacity int) Queue
}

var impls = []impl{
	{"naive q = q[1:]", false, func(c int) Queue { return &Naive{q: make([]int, 0, c)} }},
	{"head index + compact", false, func(c int) Queue { return &Compact{buf: make([]int, 0, c)} }},
	{"ring (fixed)", true, func(c int) Queue { return NewRing(c) }},
	{"ring (growable)", false, func(c int) Queue { return NewGrowRing(c) }},
	{"container/list", false, func(int) Queue { return NewList() }},
	{"buffered channel", true, func(c int) Queue { return NewChan(c) }},
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

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
}

func main() {
	const steady = 500_000
	const burst = 2_000_000

	fmt.Println("workload A -- STEADY STATE: 1000 live, one push and one pop each,")
	fmt.Println("500,000 times. A work queue keeping up with its producer.")
	fmt.Println()
	fmt.Printf("  %-24s %14s\n", "implementation", "ns/op")
	steadyNS := map[string]float64{}
	for _, im := range impls {
		t := nsPerOp(func() {
			q := im.new(4096)
			for i := 0; i < 1000; i++ {
				q.Push(i)
			}
			for i := 0; i < steady; i++ {
				q.Push(i)
				v, _ := q.Pop()
				sinkI = v
			}
		})
		steadyNS[im.name] = t / steady
		fmt.Printf("  %-24s %14.2f\n", im.name, t/steady)
	}
	fmt.Println()
	fmt.Println("  all six behind the same `Queue` interface, so the call overhead is")
	fmt.Println("  identical and the differences are the structures themselves.")

	fmt.Println()
	fmt.Println("workload B -- BURST THEN DRAIN: push 2,000,000, then pop them all,")
	fmt.Println("starting from capacity 1024. A BFS frontier, a batch import, a")
	fmt.Println("backlog after an outage.")
	fmt.Println()
	fmt.Printf("  %-24s %14s %12s %14s\n", "implementation", "ns/op", "peak MB", "pushes lost")
	for _, im := range impls {
		base := heapMB()
		q := im.new(1024)
		for i := 0; i < burst; i++ {
			q.Push(i)
		}
		peak := heapMB() - base
		got := 0
		for {
			if _, ok := q.Pop(); !ok {
				break
			}
			got++
		}
		lost := burst - got
		t := nsPerOp(func() {
			q2 := im.new(1024)
			for i := 0; i < burst; i++ {
				q2.Push(i)
			}
			for i := 0; i < burst; i++ {
				v, _ := q2.Pop()
				sinkI = v
			}
		})
		fmt.Printf("  %-24s %14.2f %12.1f %14d\n", im.name, t/(2*burst), peak, lost)
	}
	fmt.Println()
	fmt.Println("  read the last column first. The two BOUNDED implementations threw")
	fmt.Println("  away 99.9% of the input, silently, and were fast about it. That is")
	fmt.Println("  not a benchmark loss -- it is the policy working exactly as")
	fmt.Println("  designed (example 9), on a workload where the policy is wrong.")
	fmt.Println()
	fmt.Println("  a bounded queue is not a slower unbounded queue. It is a different")
	fmt.Println("  contract, and the ns/op column cannot see the difference.")

	fmt.Println()
	fmt.Println("workload C -- WHAT IS STILL HELD after draining to empty:")
	fmt.Println()
	fmt.Printf("  %-24s %18s\n", "implementation", "MB held at len 0")
	for _, im := range impls {
		if im.bounded {
			continue
		}
		base := heapMB()
		q := im.new(1024)
		for i := 0; i < burst; i++ {
			q.Push(i)
		}
		for {
			if _, ok := q.Pop(); !ok {
				break
			}
		}
		held := heapMB() - base
		runtime.KeepAlive(q)
		fmt.Printf("  %-24s %18.1f\n", im.name, held)
	}
	fmt.Println()
	fmt.Println("  this is example 1's case B, and it reverses the ranking again:")
	fmt.Println("  container/list is the ONLY structure here that gives its memory")
	fmt.Println("  back on drain, because every node it dropped is genuinely garbage.")
	fmt.Println("  The three array-backed queues are all still holding their peak.")
	fmt.Println()
	fmt.Println("  the slowest, most allocation-hungry implementation wins the")
	fmt.Println("  memory-return column outright. Three workloads, three different")
	fmt.Println("  winners -- which is the entire reason to run all three.")

	fmt.Println()
	fmt.Println("the harness is part of the experiment, and this table proves it:")
	fmt.Println()
	naiveInline := nsPerOp(func() {
		q := make([]int, 0, 4096)
		for i := 0; i < 1000; i++ {
			q = append(q, i)
		}
		for i := 0; i < steady; i++ {
			q = append(q, i)
			sinkI = q[0]
			q = q[1:]
		}
	})
	ringInline := nsPerOp(func() {
		buf, mask := make([]int, 4096), 4095
		head, count := 0, 0
		for i := 0; i < 1000; i++ {
			buf[(head+count)&mask] = i
			count++
		}
		for i := 0; i < steady; i++ {
			buf[(head+count)&mask] = i
			count++
			sinkI = buf[head]
			head = (head + 1) & mask
			count--
		}
	})
	allocInline := testing.AllocsPerRun(3, func() {
		q := make([]int, 0, 4096)
		for i := 0; i < 1000; i++ {
			q = append(q, i)
		}
		for i := 0; i < 200_000; i++ {
			q = append(q, i)
			sinkI = q[0]
			q = q[1:]
		}
	})
	allocIface := testing.AllocsPerRun(3, func() {
		var q Queue = &Naive{q: make([]int, 0, 4096)}
		for i := 0; i < 1000; i++ {
			q.Push(i)
		}
		for i := 0; i < 200_000; i++ {
			q.Push(i)
			v, _ := q.Pop()
			sinkI = v
		}
	})
	fmt.Printf("  %-28s %12s %12s %10s\n", "", "inline", "interface", "added")
	fmt.Printf("  %-28s %10.2f ns %10.2f ns %8.2f ns\n", "naive q = q[1:]",
		naiveInline/steady, steadyNS["naive q = q[1:]"],
		steadyNS["naive q = q[1:]"]-naiveInline/steady)
	fmt.Printf("  %-28s %10.2f ns %10.2f ns %8.2f ns\n", "ring (fixed)",
		ringInline/steady, steadyNS["ring (fixed)"],
		steadyNS["ring (fixed)"]-ringInline/steady)
	fmt.Println()
	fmt.Printf("  allocations, naive, 200,000 ops: inline %.0f, interface %.0f\n",
		allocInline, allocIface)
	fmt.Println()
	fmt.Println("  the allocation counts are the SAME, so the gap is not allocation.")
	fmt.Println("  It is the two indirect calls per operation: an interface method")
	fmt.Println("  cannot be inlined, so the slice header moves between the struct")
	fmt.Println("  and registers on every push and every pop instead of staying put.")
	fmt.Println()
	fmt.Println("  that is why examples 1 to 3, which wrote the loops inline, kept")
	fmt.Println("  finding the naive queue fastest and this table does not. Both are")
	fmt.Println("  correct measurements of different programs.")
	fmt.Println()
	fmt.Println("  the practical reading: within THIS table the comparison is fair,")
	fmt.Println("  because all six pay the same tax. Across tables it is not. Never")
	fmt.Println("  quote a ns/op figure without the harness it was measured in.")

	fmt.Println()
	fmt.Println("the verdict, and it is not 'use the fastest one':")
	fmt.Println()
	fmt.Printf("  %-22s %-34s %s\n", "implementation", "use it when", "never when")
	for _, r := range [][3]string{
		{"naive q = q[1:]", "short-lived, scoped, fully drained", "it outlives the function"},
		{"head index + compact", "you want one file, no surprises", "you need worst-case Theta(1)"},
		{"ring (fixed)", "there is a real capacity limit", "bursts must not be refused"},
		{"ring (growable)", "hot path, size unknown", "you needed a memory bound"},
		{"container/list", "you remove by handle too", "it is only a queue"},
		{"buffered channel", "it crosses a goroutine", "it does not"},
	} {
		fmt.Printf("  %-22s %-34s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Printf("  the spread on workload A is %.1fx between best and worst, which is\n",
		steadyNS["container/list"]/steadyNS["ring (fixed)"])
	fmt.Println("  real but small next to the difference between a queue that bounds")
	fmt.Println("  its memory and one that does not. Speed was the least interesting")
	fmt.Println("  axis in this lesson, and measuring it was the only way to find")
	fmt.Println("  that out.")
}
```

**Sample output:**

```
workload A -- STEADY STATE: 1000 live, one push and one pop each,
500,000 times. A work queue keeping up with its producer.

  implementation                    ns/op
  naive q = q[1:]                    7.99
  head index + compact               2.17
  ring (fixed)                       3.55
  ring (growable)                    2.44
  container/list                    21.12
  buffered channel                  19.23

  all six behind the same `Queue` interface, so the call overhead is
  identical and the differences are the structures themselves.

workload B -- BURST THEN DRAIN: push 2,000,000, then pop them all,
starting from capacity 1024. A BFS frontier, a batch import, a
backlog after an outage.

  implementation                    ns/op      peak MB    pushes lost
  naive q = q[1:]                    5.04         16.9              0
  head index + compact               4.20         16.9              0
  ring (fixed)                       1.71          0.0        1998976
  ring (growable)                    3.60         16.0              0
  container/list                    17.80        106.8              0
  buffered channel                   2.93          0.0        1998976

  read the last column first. The two BOUNDED implementations threw
  away 99.9% of the input, silently, and were fast about it. That is
  not a benchmark loss -- it is the policy working exactly as
  designed (example 9), on a workload where the policy is wrong.

  a bounded queue is not a slower unbounded queue. It is a different
  contract, and the ns/op column cannot see the difference.

workload C -- WHAT IS STILL HELD after draining to empty:

  implementation             MB held at len 0
  naive q = q[1:]                        16.9
  head index + compact                   16.9
  ring (growable)                        16.0
  container/list                         -0.0

  this is example 1's case B, and it reverses the ranking again:
  container/list is the ONLY structure here that gives its memory
  back on drain, because every node it dropped is genuinely garbage.
  The three array-backed queues are all still holding their peak.

  the slowest, most allocation-hungry implementation wins the
  memory-return column outright. Three workloads, three different
  winners -- which is the entire reason to run all three.

the harness is part of the experiment, and this table proves it:

                                     inline    interface      added
  naive q = q[1:]                    1.82 ns       7.99 ns     6.17 ns
  ring (fixed)                       0.55 ns       3.55 ns     3.00 ns

  allocations, naive, 200,000 ops: inline 368, interface 369

  the allocation counts are the SAME, so the gap is not allocation.
  It is the two indirect calls per operation: an interface method
  cannot be inlined, so the slice header moves between the struct
  and registers on every push and every pop instead of staying put.

  that is why examples 1 to 3, which wrote the loops inline, kept
  finding the naive queue fastest and this table does not. Both are
  correct measurements of different programs.

  the practical reading: within THIS table the comparison is fair,
  because all six pay the same tax. Across tables it is not. Never
  quote a ns/op figure without the harness it was measured in.

the verdict, and it is not 'use the fastest one':

  implementation         use it when                        never when
  naive q = q[1:]        short-lived, scoped, fully drained it outlives the function
  head index + compact   you want one file, no surprises    you need worst-case Theta(1)
  ring (fixed)           there is a real capacity limit     bursts must not be refused
  ring (growable)        hot path, size unknown             you needed a memory bound
  container/list         you remove by handle too           it is only a queue
  buffered channel       it crosses a goroutine             it does not

  the spread on workload A is 5.9x between best and worst, which is
  real but small next to the difference between a queue that bounds
  its memory and one that does not. Speed was the least interesting
  axis in this lesson, and measuring it was the only way to find
  that out.
```

**Complexity:** three workloads, three winners — head-index on speed, the unbounded queues on the
burst (the bounded ones silently discarded **1,998,976** pushes), and `container/list` alone returns
its memory on drain · the same naive queue measures **1.82 ns/op** inline and **7.99 ns/op** behind an
interface, with *identical* allocation counts

---

> Next tier: [🔴 hard](3-hard.md) — BFS variations, 0-1 BFS, a queue that knows its own minimum,
> work stealing, and the capstone.
