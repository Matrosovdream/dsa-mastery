# Step 04 — Go's Memory Model · 🔴 Hard

Examples **12–16**: where layout beats asymptotics, ending in a graph the GC never looks at.

> ⚠️ **Sample output** throughout — Apple M4, Go 1.26.3. Examples 14 and 16 take ~30 seconds each.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Previous tier: [🟡 medium](2-medium.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The GC does not look at pointer-free memory

`🔴 hard` · *Pointer maps*

Go's collector only scans blocks whose type **might** contain pointers; the compiler records a pointer
map for every type. So "how much memory do I use" is the wrong question — the question is **how many
pointers are reachable**.

The comparison to watch is rows 2 and 3: same byte count, same element count, same links. One stores
those links as `*T`, the other as `int32`. **22× difference in GC cost.**

**Steps:**

1. Build four structures of 5M elements, differing only in pointer content.
2. Time a full GC cycle with each one live, in isolation.
3. Compare against live heap object counts.

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

// Go's GC only has to scan memory that MIGHT contain pointers. The compiler
// records a pointer map for every type; a block whose type has no pointer
// fields is skipped entirely, no matter how large it is.
//
// So "how much memory do I use" is the wrong question. The question is
// "how many POINTERS are reachable" -- that is what a GC cycle costs.

type withPointer struct {
	id   int
	next *withPointer // one pointer field
}

type withoutPointer struct {
	id   int
	next int32 // an INDEX instead of a pointer -- same job, no tracing
	_    int32
}

func avgGC(rounds int) time.Duration {
	runtime.GC() // warm up
	start := time.Now()
	for i := 0; i < rounds; i++ {
		runtime.GC()
	}
	return time.Since(start) / time.Duration(rounds)
}

type result struct {
	name    string
	mb      float64
	gc      time.Duration
	objects uint64
}

func measure(name string, build func() any) result {
	runtime.GC()
	var before runtime.MemStats
	runtime.ReadMemStats(&before)

	v := build()

	var after runtime.MemStats
	runtime.ReadMemStats(&after)
	gc := avgGC(10)
	runtime.KeepAlive(v)

	return result{
		name:    name,
		mb:      float64(after.HeapAlloc-before.HeapAlloc) / (1 << 20),
		gc:      gc,
		objects: after.HeapObjects - before.HeapObjects,
	}
}

func main() {
	const n = 5_000_000

	results := []result{
		measure("[]int (no pointers)", func() any {
			xs := make([]int, n)
			for i := range xs {
				xs[i] = i
			}
			return xs
		}),
		measure("[]withoutPointer", func() any {
			xs := make([]withoutPointer, n)
			for i := range xs {
				xs[i] = withoutPointer{id: i, next: int32(i)}
			}
			return xs
		}),
		measure("[]withPointer (values)", func() any {
			xs := make([]withPointer, n)
			for i := range xs {
				xs[i].id = i
			}
			return xs
		}),
		measure("[]*withPointer", func() any {
			xs := make([]*withPointer, n)
			for i := range xs {
				xs[i] = &withPointer{id: i}
			}
			return xs
		}),
	}

	fmt.Printf("%d elements, held four ways\n\n", n)
	fmt.Printf("%-26s %10s %14s %16s\n", "", "MB", "GC cycle", "heap objects")
	for _, r := range results {
		fmt.Printf("%-26s %10.1f %14v %16d\n", r.name, r.mb, r.gc.Round(time.Microsecond), r.objects)
	}

	fmt.Println()
	base := results[0].gc
	fmt.Println("GC cost relative to the pointer-free slice:")
	for _, r := range results {
		fmt.Printf("  %-26s %6.0fx\n", r.name, float64(r.gc)/float64(base))
	}

	fmt.Println()
	fmt.Println("row 2 is the one to notice. []withoutPointer holds the SAME 5 million")
	fmt.Println("links as row 3 -- it just stores them as int32 indices rather than")
	fmt.Println("pointers. Identical structure, identical algorithms, and the collector")
	fmt.Println("does not have to look at it at all.")
	fmt.Println()
	fmt.Println("this is the single biggest lever in Go for large data structures:")
	fmt.Println()
	fmt.Println("  pointers  -> nodes allocated separately, traced every cycle, 8 bytes each")
	fmt.Println("  indices   -> one contiguous block, invisible to the GC, 4 bytes each")
	fmt.Println()
	fmt.Println("you give up: compile-time type safety on the link, and the ability to")
	fmt.Println("free one node. You gain: locality, half the link size, and zero GC work.")
	fmt.Println("Example 16 builds a real graph both ways.")
}
```

**Sample output:**

```
5000000 elements, held four ways

                                   MB       GC cycle     heap objects
[]int (no pointers)              38.2          102µs                8
[]withoutPointer                 76.3          141µs                2
[]withPointer (values)           76.3        2.195ms               10
[]*withPointer                  114.4        9.762ms          5000002

GC cost relative to the pointer-free slice:
  []int (no pointers)             1x
  []withoutPointer                1x
  []withPointer (values)         22x
  []*withPointer                 96x

row 2 is the one to notice. []withoutPointer holds the SAME 5 million
links as row 3 -- it just stores them as int32 indices rather than
pointers. Identical structure, identical algorithms, and the collector
does not have to look at it at all.

this is the single biggest lever in Go for large data structures:

  pointers  -> nodes allocated separately, traced every cycle, 8 bytes each
  indices   -> one contiguous block, invisible to the GC, 4 bytes each

you give up: compile-time type safety on the link, and the ability to
free one node. You gain: locality, half the link size, and zero GC work.
Example 16 builds a real graph both ways.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::mem::{needs_drop, size_of};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Rust has no tracing garbage collector, so there is no "GC cycle" to time and
// no pointer map for a collector to consult. What Rust pays for pointers is
// different, and it is paid at known moments:
//
//   * one allocator call per boxed node, when it is built
//   * one allocator call per boxed node, when it is dropped
//   * a dependent load per node, whenever you walk the structure
//
// and while the structure is merely ALIVE it costs nothing: no thread scans it.
//
// The compiler does keep one bit per type that echoes Go's pointer map:
// `needs_drop::<T>()`. A type with no drop glue is freed with a single
// deallocation and no walk, however large. That is the pointer-free fast path.

// A counting allocator stands in for runtime.MemStats: exact, with no GC noise.
// A global allocator is an `unsafe` trait by definition; this one only counts
// and forwards every call to the system allocator.
struct Counting;
static LIVE_BYTES: AtomicUsize = AtomicUsize::new(0);
static LIVE_OBJECTS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        LIVE_BYTES.fetch_add(l.size(), Relaxed);
        LIVE_OBJECTS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE_BYTES.fetch_sub(l.size(), Relaxed);
        LIVE_OBJECTS.fetch_sub(1, Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        LIVE_BYTES.fetch_add(new, Relaxed);
        LIVE_BYTES.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

struct WithPointer {
    id: i64,
    next: Option<Box<WithPointer>>, // one pointer field
}

struct WithoutPointer {
    id: i64,
    next: u32, // an INDEX instead of a pointer: same job, no drop glue
    _pad: u32,
}

// What a "walk" reads, so that the optimiser cannot delete the traversal.
trait Weigh {
    fn weigh(&self) -> i64;
}
impl Weigh for i64 {
    fn weigh(&self) -> i64 {
        *self
    }
}
impl Weigh for WithoutPointer {
    fn weigh(&self) -> i64 {
        self.id + i64::from(self.next)
    }
}
impl Weigh for WithPointer {
    fn weigh(&self) -> i64 {
        self.id + i64::from(self.next.is_some())
    }
}
impl<T: Weigh> Weigh for Box<T> {
    fn weigh(&self) -> i64 {
        (**self).weigh() // one extra dependent load per element
    }
}

struct Row {
    name: &'static str,
    mb: f64,
    objects: usize,
    drop_glue: bool,
    build: Duration,
    scan: Duration,
    free: Duration,
}

// Build, walk and drop one structure. Memory comes from the allocator's own
// books, so it is exact; timings are the best of three whole rounds.
fn measure<T: Weigh>(name: &'static str, build: impl Fn() -> Vec<T>) -> Row {
    const ROUNDS: usize = 3;
    let mut row = Row {
        name,
        mb: 0.0,
        objects: 0,
        drop_glue: needs_drop::<T>(),
        build: Duration::MAX,
        scan: Duration::MAX,
        free: Duration::MAX,
    };
    for round in 0..ROUNDS {
        let (bytes0, objects0) = (LIVE_BYTES.load(Relaxed), LIVE_OBJECTS.load(Relaxed));

        let t = Instant::now();
        let xs = black_box(build());
        row.build = row.build.min(t.elapsed());

        if round == 0 {
            row.mb = (LIVE_BYTES.load(Relaxed) - bytes0) as f64 / (1 << 20) as f64;
            row.objects = LIVE_OBJECTS.load(Relaxed) - objects0;
        }

        for _ in 0..5 {
            let t = Instant::now();
            black_box(xs.iter().map(Weigh::weigh).sum::<i64>());
            row.scan = row.scan.min(t.elapsed());
        }

        let t = Instant::now();
        drop(black_box(xs));
        row.free = row.free.min(t.elapsed());
    }
    row
}

fn main() {
    const N: usize = 5_000_000;

    let rows = [
        measure("Vec<i64> (no pointers)", || (0..N as i64).collect()),
        measure("Vec<WithoutPointer>", || {
            (0..N)
                .map(|i| WithoutPointer {
                    id: i as i64,
                    next: i as u32,
                    _pad: 0,
                })
                .collect()
        }),
        measure("Vec<WithPointer> (values)", || {
            (0..N as i64)
                .map(|id| WithPointer { id, next: None })
                .collect()
        }),
        measure("Vec<Box<WithPointer>>", || {
            (0..N as i64)
                .map(|id| Box::new(WithPointer { id, next: None }))
                .collect()
        }),
    ];

    println!("{N} elements, held four ways\n");
    println!(
        "{:<26} {:>8} {:>12} {:>11} {:>10} {:>10} {:>10}",
        "", "MB", "allocations", "drop glue", "build", "walk", "drop"
    );
    for r in &rows {
        println!(
            "{:<26} {:>8.1} {:>12} {:>11} {:>10} {:>10} {:>10}",
            r.name,
            r.mb,
            r.objects,
            if r.drop_glue { "yes" } else { "no" },
            ms(r.build),
            ms(r.scan),
            ms(r.free)
        );
    }

    // Go's second table is "GC cost relative to the pointer-free slice". The
    // like-for-like pair is rows 2 and 3: same bytes, same links, one allocation
    // each. So the baseline here is row 2, and the quantity is the drop.
    println!();
    println!("drop cost relative to Vec<WithoutPointer> (same bytes, same links):");
    let base = rows[1].free.as_secs_f64();
    for r in &rows {
        println!("  {:<26} {:>6.1}x", r.name, r.free.as_secs_f64() / base);
    }

    // Is the walk slow? The boxes were allocated back to back, so ask where they
    // are: how far is each box from the one before it in the Vec?
    let boxes: Vec<Box<WithPointer>> = (0..N as i64)
        .map(|id| Box::new(WithPointer { id, next: None }))
        .collect();
    let addrs: Vec<usize> = boxes.iter().map(|b| &raw const **b as usize).collect();
    let adjacent = addrs
        .windows(2)
        .filter(|w| w[1].wrapping_sub(w[0]) == size_of::<WithPointer>())
        .count();
    println!();
    println!(
        "size_of::<WithPointer>() = {}    size_of::<WithoutPointer>() = {}    size_of::<Box<WithPointer>>() = {}",
        size_of::<WithPointer>(),
        size_of::<WithoutPointer>(),
        size_of::<Box<WithPointer>>()
    );
    println!(
        "{:.3}% of the boxes sit exactly {} bytes after the previous one",
        adjacent as f64 * 100.0 / (N - 1) as f64,
        size_of::<WithPointer>()
    );
    drop(boxes);

    let [_, indices, values, boxed] = &rows;
    println!();
    println!("row 2 is the one to notice. Vec<WithoutPointer> holds the SAME 5 million");
    println!("links as row 3 -- it just stores them as u32 indices rather than Box");
    println!("pointers. Same size, same single allocation, and the compiler knows it");
    println!(
        "needs no drop glue: freeing it is one deallocation ({}), not a pass over",
        ms(indices.free)
    );
    println!(
        "every element ({} for the values that hold an Option<Box>).",
        ms(values.free)
    );
    println!();
    println!(
        "row 4 is where Rust pays for pointers: {} allocator calls to build and as",
        boxed.objects
    );
    println!(
        "many to drop. Against row 3 the build is {:.0}x and the drop {:.0}x. Nothing is paid",
        ratio(boxed.build, values.build),
        ratio(boxed.free, values.free)
    );
    println!("while it is merely alive: no collector scans it, so there is no per-cycle");
    println!("cost to measure. Go pays row 4's tracing every cycle for as long as the");
    println!("structure lives; Rust pays the malloc and the free once each.");
    println!();
    println!(
        "the walk column moves only {:.1}x from row 3 to row 4, because the boxes were",
        ratio(boxed.scan, values.scan)
    );
    println!("allocated back to back and sit next to each other (the 16-byte steps above).");
    println!("Pointers cost time when they are scattered: that is example 14.");
    println!();
    println!("same lever, different reason:");
    println!();
    println!("  boxes    -> a malloc and a free per node, 8 bytes of pointer each");
    println!("  indices  -> one block, one malloc, one free, 4 bytes each");
    println!();
    println!("you give up: the compiler no longer checks the link (a u32 can name the");
    println!("wrong Vec; an out-of-range one panics at the index instead of being a");
    println!("dangling pointer), and the ability to free one node. You gain: locality,");
    println!("half the link size, and no allocator traffic. Example 16 builds a real");
    println!("graph both ways.");
}

fn ratio(a: Duration, b: Duration) -> f64 {
    a.as_secs_f64() / b.as_secs_f64()
}

fn ms(d: Duration) -> String {
    format!("{:.3}ms", d.as_secs_f64() * 1e3)
}
```

**Rust sample output:**

```
5000000 elements, held four ways

                                 MB  allocations   drop glue      build       walk       drop
Vec<i64> (no pointers)         38.1            1          no    2.693ms    0.506ms    0.291ms
Vec<WithoutPointer>            76.3            1          no    6.063ms    1.238ms    1.046ms
Vec<WithPointer> (values)      76.3            1         yes    5.338ms    1.141ms    2.337ms
Vec<Box<WithPointer>>         114.4      5000001         yes   62.074ms    1.585ms  169.871ms

drop cost relative to Vec<WithoutPointer> (same bytes, same links):
  Vec<i64> (no pointers)        0.3x
  Vec<WithoutPointer>           1.0x
  Vec<WithPointer> (values)     2.2x
  Vec<Box<WithPointer>>       162.3x

size_of::<WithPointer>() = 16    size_of::<WithoutPointer>() = 16    size_of::<Box<WithPointer>>() = 8
99.998% of the boxes sit exactly 16 bytes after the previous one

row 2 is the one to notice. Vec<WithoutPointer> holds the SAME 5 million
links as row 3 -- it just stores them as u32 indices rather than Box
pointers. Same size, same single allocation, and the compiler knows it
needs no drop glue: freeing it is one deallocation (1.046ms), not a pass over
every element (2.337ms for the values that hold an Option<Box>).

row 4 is where Rust pays for pointers: 5000001 allocator calls to build and as
many to drop. Against row 3 the build is 12x and the drop 73x. Nothing is paid
while it is merely alive: no collector scans it, so there is no per-cycle
cost to measure. Go pays row 4's tracing every cycle for as long as the
structure lives; Rust pays the malloc and the free once each.

the walk column moves only 1.4x from row 3 to row 4, because the boxes were
allocated back to back and sit next to each other (the 16-byte steps above).
Pointers cost time when they are scattered: that is example 14.

same lever, different reason:

  boxes    -> a malloc and a free per node, 8 bytes of pointer each
  indices  -> one block, one malloc, one free, 4 bytes each

you give up: the compiler no longer checks the link (a u32 can name the
wrong Vec; an out-of-range one panics at the index instead of being a
dangling pointer), and the ability to free one node. You gain: locality,
half the link size, and no allocator traffic. Example 16 builds a real
graph both ways.
```

> **Go vs Rust:** Rust has no garbage collector, so the Go "GC cycle" column and the pointer map it consults have no counterpart: a live `Vec<Box<T>>` costs nothing per cycle because nothing scans it. What Rust pays is at the two ends, and that is what the table measures instead: 5,000,001 allocator calls to build and as many to free (over three runs, build was 10-13x and drop 60-95x the values row; read orders of magnitude, not digits), while `Vec<WithoutPointer>` is one allocation and one free. The nearest echo of the pointer map is `needs_drop::<T>()`, the "drop glue" column: rows 2 and 3 differ only in that bit, and it costs a pass over the elements at drop. The MB column matches Go (38.1 vs 38.2, 76.3, 76.3, 114.4), but the walk column barely separates rows 3 and 4 because the boxes happen to be 16 bytes apart; scattering them is example 14. Timings are best-of-3 on a shared machine, hence "sample".

**Complexity:** every structure is Θ(n) space · GC cost is Θ(reachable pointers), which is a term Big-O never carries and which you can drive to **zero** by using indices

---

## 13. Array of structs vs struct of arrays

`🔴 hard` · *Layout for the query you actually run*

A 64-byte particle means a scan of one field touches 8 bytes per 64 fetched — the example 7 stride
problem, with stride = `sizeof(struct)`. Storing each field in its own contiguous column fixes it.

**Steps:**

1. Store a million particles both ways.
2. Scan one `float64` field, then one `uint64` field.
3. Compare the speedup against the memory-traffic ratio — and explain the gap.

```go
package main

import (
	"fmt"
	"testing"
	"unsafe"
)

var sink float64
var sinkI int

// A particle: 64 bytes, half of a 128-byte cache line.
type particle struct {
	x, y, z    float64
	vx, vy, vz float64
	id         int64
	flags      uint64
}

// --- Array of Structs: the obvious layout ---

type AoS []particle

func (ps AoS) sumX() float64 {
	total := 0.0
	for i := range ps {
		total += ps[i].x
	}
	return total
}

func (ps AoS) countFlagged() int {
	n := 0
	for i := range ps {
		if ps[i].flags&1 == 1 {
			n++
		}
	}
	return n
}

// --- Struct of Arrays: one contiguous column per field ---

type SoA struct {
	x, y, z    []float64
	vx, vy, vz []float64
	id         []int64
	flags      []uint64
}

func newSoA(n int) *SoA {
	return &SoA{
		x: make([]float64, n), y: make([]float64, n), z: make([]float64, n),
		vx: make([]float64, n), vy: make([]float64, n), vz: make([]float64, n),
		id: make([]int64, n), flags: make([]uint64, n),
	}
}

func (s *SoA) sumX() float64 {
	total := 0.0
	for _, v := range s.x {
		total += v
	}
	return total
}

func (s *SoA) countFlagged() int {
	n := 0
	for _, f := range s.flags {
		if f&1 == 1 {
			n++
		}
	}
	return n
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
	const n = 1_000_000

	aos := make(AoS, n)
	soa := newSoA(n)
	for i := 0; i < n; i++ {
		x := float64(i)
		aos[i] = particle{x: x, y: x, z: x, id: int64(i), flags: uint64(i)}
		soa.x[i], soa.y[i], soa.z[i] = x, x, x
		soa.id[i], soa.flags[i] = int64(i), uint64(i)
	}

	var p particle
	fmt.Printf("particle is %d bytes; a %d-byte cache line holds %d of them\n\n",
		unsafe.Sizeof(p), 128, 128/int(unsafe.Sizeof(p)))

	fmt.Printf("%-28s %14s %14s %12s\n", "reading ONE field of each", "AoS ns/op", "SoA ns/op", "speedup")

	aX := nsPerOp(func() { sink += aos.sumX() })
	sX := nsPerOp(func() { sink += soa.sumX() })
	fmt.Printf("%-28s %14.0f %14.0f %11.1fx\n", "sum x (float64)", aX, sX, aX/sX)

	aF := nsPerOp(func() { sinkI += aos.countFlagged() })
	sF := nsPerOp(func() { sinkI += soa.countFlagged() })
	fmt.Printf("%-28s %14.0f %14.0f %11.1fx\n", "count flagged (uint64)", aF, sF, aF/sF)

	fmt.Println()
	fmt.Printf("why: to read %d x-values, AoS must pull in %d MB of particles;\n",
		n, n*int(unsafe.Sizeof(p))/(1<<20))
	fmt.Printf("SoA pulls in %d MB of x. Same answer, %dx less memory traffic.\n",
		n*8/(1<<20), int(unsafe.Sizeof(p))/8)

	fmt.Println()
	fmt.Println("the speedups are smaller than 8x, and the two rows differ -- worth knowing why.")
	fmt.Println("Less traffic only helps if memory is the bottleneck. Summing float64 is a")
	fmt.Println("SERIAL DEPENDENCY CHAIN (each += waits for the previous one), so part of the")
	fmt.Println("fetch time hides behind arithmetic latency. The flag count has no such chain,")
	fmt.Println("so it is closer to pure memory-bound and gains more.")
	fmt.Println()
	fmt.Println("a good reminder: a layout change gives you HEADROOM, not a guaranteed")
	fmt.Println("speedup. Always measure the actual query, not the byte count.")

	fmt.Println()
	fmt.Println("this is the layout question behind every columnar database, every ECS")
	fmt.Println("game engine, and every vectorized query engine: if you scan ONE field")
	fmt.Println("across MANY records, store that field contiguously.")
	fmt.Println()
	fmt.Println("the trade-off is real, and it goes the other way too:")
	fmt.Println("  AoS wins when you touch MOST fields of ONE record (ps[i] is one line)")
	fmt.Println("  SoA wins when you touch ONE field of MOST records (a full-column scan)")
	fmt.Println("  SoA also costs you: no single `particle` value to pass around, and")
	fmt.Println("  8 slice headers to keep in step -- append to one, append to all.")
	fmt.Println()
	fmt.Println("note this is the same effect as example 7's stride: AoS reading one")
	fmt.Println("field IS a strided access, with stride = sizeof(struct).")
}
```

**Sample output:**

```
particle is 64 bytes; a 128-byte cache line holds 2 of them

reading ONE field of each         AoS ns/op      SoA ns/op      speedup
sum x (float64)                      949895         613788         1.5x
count flagged (uint64)               924385         258669         3.6x

why: to read 1000000 x-values, AoS must pull in 61 MB of particles;
SoA pulls in 7 MB of x. Same answer, 8x less memory traffic.

the speedups are smaller than 8x, and the two rows differ -- worth knowing why.
Less traffic only helps if memory is the bottleneck. Summing float64 is a
SERIAL DEPENDENCY CHAIN (each += waits for the previous one), so part of the
fetch time hides behind arithmetic latency. The flag count has no such chain,
so it is closer to pure memory-bound and gains more.

a good reminder: a layout change gives you HEADROOM, not a guaranteed
speedup. Always measure the actual query, not the byte count.

this is the layout question behind every columnar database, every ECS
game engine, and every vectorized query engine: if you scan ONE field
across MANY records, store that field contiguously.

the trade-off is real, and it goes the other way too:
  AoS wins when you touch MOST fields of ONE record (ps[i] is one line)
  SoA wins when you touch ONE field of MOST records (a full-column scan)
  SoA also costs you: no single `particle` value to pass around, and
  8 slice headers to keep in step -- append to one, append to all.

note this is the same effect as example 7's stride: AoS reading one
field IS a strided access, with stride = sizeof(struct).
```

The measured speedups (1.5× and 3.6×) are **below** the 8× reduction in bytes fetched, and the two
rows differ from each other. Less traffic only helps when memory is the bottleneck: summing `float64`
is a serial dependency chain, so some fetch time hides behind arithmetic latency. The flag count has
no such chain and gains more. A layout change buys **headroom**, not a guaranteed speedup.

**Rust:**

```rust
use std::hint::black_box;
use std::mem::size_of;
use std::time::{Duration, Instant};

// ns/op: stand-in for testing.Benchmark / b.Loop. Doubles the iteration count
// until a run lasts long enough to trust, then reports the mean.
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

// The machine is shared, so report the best of five runs: interference only ever
// makes a run slower, never faster.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..5)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// A particle: 64 bytes, half of a 128-byte cache line.
struct Particle {
    x: f64,
    y: f64,
    z: f64,
    vx: f64,
    vy: f64,
    vz: f64,
    id: i64,
    flags: u64,
}

// --- Array of Structs: the obvious layout ---

fn aos_sum_x(ps: &[Particle]) -> f64 {
    ps.iter().map(|p| p.x).sum()
}

// The same sum with four independent accumulators, so that consecutive adds no
// longer wait for each other. (The x values are whole numbers, so every partial
// sum is exact and the result is bit-identical to the one-accumulator sum.)
fn aos_sum_x4(ps: &[Particle]) -> f64 {
    let mut acc = [0.0; 4];
    let (chunks, rest) = ps.as_chunks::<4>();
    let rest: f64 = rest.iter().map(|p| p.x).sum();
    for chunk in chunks {
        for (a, p) in acc.iter_mut().zip(chunk) {
            *a += p.x;
        }
    }
    acc.iter().sum::<f64>() + rest
}

fn aos_count_flagged(ps: &[Particle]) -> usize {
    ps.iter().filter(|p| p.flags & 1 == 1).count()
}

// --- Struct of Arrays: one contiguous column per field ---

struct SoA {
    x: Vec<f64>,
    y: Vec<f64>,
    z: Vec<f64>,
    vx: Vec<f64>,
    vy: Vec<f64>,
    vz: Vec<f64>,
    id: Vec<i64>,
    flags: Vec<u64>,
}

impl SoA {
    fn new(n: usize) -> Self {
        SoA {
            x: vec![0.0; n],
            y: vec![0.0; n],
            z: vec![0.0; n],
            vx: vec![0.0; n],
            vy: vec![0.0; n],
            vz: vec![0.0; n],
            id: vec![0; n],
            flags: vec![0; n],
        }
    }

    fn sum_x(&self) -> f64 {
        self.x.iter().sum()
    }

    fn sum_x4(&self) -> f64 {
        let mut acc = [0.0; 4];
        let (chunks, rest) = self.x.as_chunks::<4>();
        let rest: f64 = rest.iter().sum();
        for chunk in chunks {
            for (a, v) in acc.iter_mut().zip(chunk) {
                *a += v;
            }
        }
        acc.iter().sum::<f64>() + rest
    }

    fn count_flagged(&self) -> usize {
        self.flags.iter().filter(|&&f| f & 1 == 1).count()
    }
}

// Touch every field of every record, in each layout. Used once, to check that
// the two layouts hold the same data (and so that no column is dead weight).
fn aos_checksum(ps: &[Particle]) -> f64 {
    ps.iter()
        .map(|p| p.x + p.y + p.z + p.vx + p.vy + p.vz + p.id as f64 + p.flags as f64)
        .sum()
}

fn soa_checksum(s: &SoA) -> f64 {
    (0..s.x.len())
        .map(|i| {
            s.x[i]
                + s.y[i]
                + s.z[i]
                + s.vx[i]
                + s.vy[i]
                + s.vz[i]
                + s.id[i] as f64
                + s.flags[i] as f64
        })
        .sum()
}

fn main() {
    const N: usize = 1_000_000;

    let mut aos = Vec::with_capacity(N);
    let mut soa = SoA::new(N);
    for i in 0..N {
        let x = i as f64;
        aos.push(Particle {
            x,
            y: x,
            z: x,
            vx: 0.0,
            vy: 0.0,
            vz: 0.0,
            id: i as i64,
            flags: i as u64,
        });
        soa.x[i] = x;
        soa.y[i] = x;
        soa.z[i] = x;
        soa.id[i] = i as i64;
        soa.flags[i] = i as u64;
    }
    assert_eq!(aos_checksum(&aos), soa_checksum(&soa));
    assert_eq!(aos_sum_x4(&aos), aos_sum_x(&aos));
    assert_eq!(soa.sum_x4(), soa.sum_x());

    let size = size_of::<Particle>();
    println!(
        "particle is {size} bytes; a 128-byte cache line holds {} of them\n",
        128 / size
    );

    println!(
        "{:<28} {:>14} {:>14} {:>12}",
        "reading ONE field of each", "AoS ns/op", "SoA ns/op", "speedup"
    );

    let a_x = best_ns_per_op(|| {
        black_box(aos_sum_x(black_box(&aos)));
    });
    let s_x = best_ns_per_op(|| {
        black_box(black_box(&soa).sum_x());
    });
    println!(
        "{:<28} {a_x:>14.0} {s_x:>14.0} {:>11.1}x",
        "sum x (f64)",
        a_x / s_x
    );

    let a_f = best_ns_per_op(|| {
        black_box(aos_count_flagged(black_box(&aos)));
    });
    let s_f = best_ns_per_op(|| {
        black_box(black_box(&soa).count_flagged());
    });
    println!(
        "{:<28} {a_f:>14.0} {s_f:>14.0} {:>11.1}x",
        "count flagged (u64)",
        a_f / s_f
    );

    let mb = |bytes: usize| bytes as f64 / (1 << 20) as f64;
    println!();
    println!(
        "why: to read {N} x-values, AoS must pull in {:.1} MB of particles;",
        mb(N * size)
    );
    println!(
        "SoA pulls in {:.1} MB of x. Same answer, {}x less memory traffic.",
        mb(N * 8),
        size / 8
    );

    // Go's text explains the gap between the two rows with a claim about a
    // dependency chain. Rust can test it: break the chain and watch the row move.
    println!();
    println!("checking the explanation: the same sum with FOUR accumulators");
    let a_x4 = best_ns_per_op(|| {
        black_box(aos_sum_x4(black_box(&aos)));
    });
    let s_x4 = best_ns_per_op(|| {
        black_box(black_box(&soa).sum_x4());
    });
    println!(
        "{:<28} {a_x4:>14.0} {s_x4:>14.0} {:>11.1}x",
        "sum x, 4 accumulators",
        a_x4 / s_x4
    );

    let per = |ns: f64| ns / N as f64;
    println!();
    println!(
        "SoA sum with one accumulator: {:.2} ns per element. Each += waits for the previous",
        per(s_x)
    );
    println!("one, and Rust will not reorder float adds for you, so the loop runs at add");
    println!(
        "latency. With four independent chains it takes {:.2} ns per element ({:.1}x faster).",
        per(s_x4),
        s_x / s_x4
    );
    println!(
        "AoS barely notices: {a_x:.0}, {a_f:.0} and {a_x4:.0} ns for the sum, the count and the"
    );
    println!(
        "4-chain sum. Whatever the loop does, {:.0} MB takes about the same time to stream in,",
        mb(N * size)
    );
    println!("so AoS is waiting for memory, not for arithmetic.");
    println!();
    println!(
        "so the sum's speedup is {:.1}x with the chain and {:.1}x without it. The count's is",
        a_x / s_x,
        a_x4 / s_x4
    );
    println!(
        "{:.1}x, against an {}x traffic ratio: the byte count only caps the speedup if both",
        a_f / s_f,
        size / 8
    );
    println!("loops chew through bytes at the same rate. They do not: the compiler turns the");
    println!("SoA loop into vector adds on 16-byte registers, and did not do that for AoS,");
    println!("whose flags are 64 bytes apart (seen in the assembly, not timed on its own");
    println!("here). A 7.6 MB column may also stay in cache between runs where");
    println!("{:.0} MB cannot (plausible, not measured).", mb(N * size));
    println!();
    println!(
        "a layout change gives you HEADROOM, not a number: {:.1}x or {:.1}x, depending on what",
        a_x / s_x,
        a_f / s_f
    );
    println!("the loop does with the bytes. Always measure the actual query, not the byte count.");
    println!();
    println!("this is the layout question behind every columnar database, every ECS game");
    println!("engine, and every vectorised query engine: if you scan ONE field across MANY");
    println!("records, store that field contiguously. The trade-off goes the other way too");
    println!("(design guidance, not measured here):");
    println!("  AoS wins when you touch MOST fields of ONE record (ps[i] is 64 contiguous bytes)");
    println!("  SoA wins when you touch ONE field of MOST records (a full-column scan)");
    println!("  SoA also costs you: no single Particle value to pass around, and eight Vecs");
    println!("  to keep in step -- push to one, push to all");
    println!();
    println!("AoS reading one field IS a strided access, with stride = size_of::<Particle>().");
}
```

**Rust sample output:**

```
particle is 64 bytes; a 128-byte cache line holds 2 of them

reading ONE field of each         AoS ns/op      SoA ns/op      speedup
sum x (f64)                          931625         559114         1.7x
count flagged (u64)                 1079826          91753        11.8x

why: to read 1000000 x-values, AoS must pull in 61.0 MB of particles;
SoA pulls in 7.6 MB of x. Same answer, 8x less memory traffic.

checking the explanation: the same sum with FOUR accumulators
sum x, 4 accumulators               1127807         170138         6.6x

SoA sum with one accumulator: 0.56 ns per element. Each += waits for the previous
one, and Rust will not reorder float adds for you, so the loop runs at add
latency. With four independent chains it takes 0.17 ns per element (3.3x faster).
AoS barely notices: 931625, 1079826 and 1127807 ns for the sum, the count and the
4-chain sum. Whatever the loop does, 61 MB takes about the same time to stream in,
so AoS is waiting for memory, not for arithmetic.

so the sum's speedup is 1.7x with the chain and 6.6x without it. The count's is
11.8x, against an 8x traffic ratio: the byte count only caps the speedup if both
loops chew through bytes at the same rate. They do not: the compiler turns the
SoA loop into vector adds on 16-byte registers, and did not do that for AoS,
whose flags are 64 bytes apart (seen in the assembly, not timed on its own
here). A 7.6 MB column may also stay in cache between runs where
61 MB cannot (plausible, not measured).

a layout change gives you HEADROOM, not a number: 1.7x or 11.8x, depending on what
the loop does with the bytes. Always measure the actual query, not the byte count.

this is the layout question behind every columnar database, every ECS game
engine, and every vectorised query engine: if you scan ONE field across MANY
records, store that field contiguously. The trade-off goes the other way too
(design guidance, not measured here):
  AoS wins when you touch MOST fields of ONE record (ps[i] is 64 contiguous bytes)
  SoA wins when you touch ONE field of MOST records (a full-column scan)
  SoA also costs you: no single Particle value to pass around, and eight Vecs
  to keep in step -- push to one, push to all

AoS reading one field IS a strided access, with stride = size_of::<Particle>().
```

> **Go vs Rust:** The layout effect is the same in both languages, but the Go claim "the speedups are smaller than 8x" does not survive the port: the AoS rows match Go (about 0.9 ms for either scan, memory-bound), and the SoA sum matches too (1.6-2.1x over my runs, against Go's 1.5x), but the SoA flag count is 10.8-12.0x here against Go's 3.6x because rustc auto-vectorises that dense loop (NEON adds in the assembly) and Go's compiler does not. The Go explanation of the sum row is testable and holds: with four accumulators the SoA sum is about 3.5x faster and its speedup rises from under 2x to about 6x. Rust also has no `testing.Benchmark`, so `ns_per_op` is hand-written, and it reports the best of five runs because the machine is shared.

**Complexity:** both scans are Θ(n) · SoA fetches `sizeof(field)/sizeof(struct)` of the bytes — the layout behind every columnar database and ECS game engine

---

## 14. The same tree, in two places in memory

`🔴 hard` · *Layout vs asymptotics*

Four structures over identical sorted keys. The last two are the **same balanced tree**, doing the
same comparisons in the same order — differing only in which addresses the nodes were given.

**Steps:**

1. Build a balanced BST from a pool of nodes allocated back-to-back.
2. Build the identical tree from a **shuffled** pool.
3. Compare both against a linear scan and `slices.BinarySearch` across seven orders of magnitude.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

var sink bool

// Four ways to answer "is this key present?" over the same sorted keys:
//
//   linear scan of a slice          Theta(n)
//   binary search of a slice        Theta(log n)
//   balanced BST, compact nodes     Theta(log n)
//   balanced BST, scattered nodes   Theta(log n)
//
// The last two are the SAME TREE: same shape, same comparisons, same code.
// They differ only in where in memory the nodes happen to live.

type treeNode struct {
	key         int
	left, right *treeNode
}

// buildFrom assembles a perfectly balanced tree out of pre-allocated nodes,
// taking them from pool in order. Hand it a pool in allocation order and the
// tree is laid out compactly; hand it a shuffled pool and the identical tree
// is scattered across the heap.
func buildFrom(sorted []int, pool []*treeNode, next *int) *treeNode {
	if len(sorted) == 0 {
		return nil
	}
	mid := len(sorted) / 2
	n := pool[*next]
	*next++
	n.key = sorted[mid]
	n.left = buildFrom(sorted[:mid], pool, next)
	n.right = buildFrom(sorted[mid+1:], pool, next)
	return n
}

func newPool(n int) []*treeNode {
	pool := make([]*treeNode, n)
	for i := range pool {
		pool[i] = &treeNode{} // allocated back-to-back
	}
	return pool
}

func (t *treeNode) contains(key int) bool {
	for n := t; n != nil; {
		switch {
		case key == n.key:
			return true
		case key < n.key:
			n = n.left
		default:
			n = n.right
		}
	}
	return false
}

func linearContains(xs []int, key int) bool {
	for _, v := range xs {
		if v == key {
			return true
		}
	}
	return false
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
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("one lookup, four structures, identical sorted keys.")
	fmt.Println("the last two are the same tree -- only the node ADDRESSES differ.")
	fmt.Println()
	fmt.Printf("%9s %12s %12s %14s %14s   %s\n",
		"n", "scan", "bsearch", "BST compact", "BST scattered", "winner")

	for _, n := range []int{8, 128, 4096, 65536, 1 << 20, 1 << 22} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}

		compactPool := newPool(n)
		var i0 int
		compact := buildFrom(xs, compactPool, &i0)

		scatteredPool := newPool(n)
		rng.Shuffle(n, func(i, j int) {
			scatteredPool[i], scatteredPool[j] = scatteredPool[j], scatteredPool[i]
		})
		var i1 int
		scattered := buildFrom(xs, scatteredPool, &i1)

		probes := make([]int, 256)
		for i := range probes {
			probes[i] = rng.IntN(n) * 2
		}
		next := 0
		pick := func() int {
			next = (next + 1) % len(probes)
			return probes[next]
		}

		scan := nsPerOp(func() { sink = linearContains(xs, pick()) })
		bs := nsPerOp(func() {
			_, ok := slices.BinarySearch(xs, pick())
			sink = ok
		})
		tc := nsPerOp(func() { sink = compact.contains(pick()) })
		ts := nsPerOp(func() { sink = scattered.contains(pick()) })

		winner, best := "scan", scan
		for name, v := range map[string]float64{"bsearch": bs, "BST compact": tc, "BST scattered": ts} {
			if v < best {
				winner, best = name, v
			}
		}
		fmt.Printf("%9d %12.1f %12.1f %14.1f %14.1f   %s\n", n, scan, bs, tc, ts, winner)
	}

	fmt.Println()
	fmt.Println("three things Big-O did not tell you:")
	fmt.Println()
	fmt.Println("1. the O(n) scan wins at n=8. It walks consecutive memory -- 16 keys per")
	fmt.Println("   cache line, prefetcher always ahead -- so a few dozen sequential reads")
	fmt.Println("   cost less than 3 dependent pointer hops. By n=128 it has lost badly.")
	fmt.Println()
	fmt.Println("2. the two BST columns are the SAME TREE doing the SAME comparisons in the")
	fmt.Println("   SAME order. The ONLY difference is which addresses the nodes got.")
	fmt.Println("   Up to 65536 they are identical -- the whole tree fits in cache, so")
	fmt.Println("   layout cannot matter. Past that the scattered version falls behind:")
	fmt.Println("   about 1.5x at n=2^20 and 1.8x at n=2^22, widening as n grows.")
	fmt.Println("   No complexity table contains that column.")
	fmt.Println()
	fmt.Println("3. a freshly built tree IS the compact column. That is what a naive")
	fmt.Println("   benchmark measures, and it flatters trees badly: it is the layout you")
	fmt.Println("   get for exactly as long as you never insert or delete anything.")
	fmt.Println()
	fmt.Println("(the compact tree and binary search stay within a few percent of each other,")
	fmt.Println(" trading places at the top end: slices.BinarySearch pays for a generic")
	fmt.Println(" comparison per probe, while contains() compares ints directly. Do not read")
	fmt.Println(" much into which wins. The instructive column is compact-vs-scattered.)")
	fmt.Println()
	fmt.Println("so why use a tree at all? Because a sorted slice cannot be UPDATED cheaply:")
	fmt.Println("insertion is a Theta(n) memmove. The tree buys O(log n) insert and delete.")
	fmt.Println("Read-mostly data belongs in a sorted slice you binary-search (lesson 15).")
}
```

**Sample output:**

```
one lookup, four structures, identical sorted keys.
the last two are the same tree -- only the node ADDRESSES differ.

        n         scan      bsearch    BST compact  BST scattered   winner
        8          3.2          4.8            3.2            3.3   scan
      128         22.0          7.7            5.7            5.8   BST compact
     4096        493.7         11.8            9.5            9.3   BST scattered
    65536       7397.5         15.4           12.5           12.6   BST compact
  1048576     132904.0         18.4           16.6           26.4   BST compact
  4194304     504932.2         20.6           20.4           38.7   BST compact

three things Big-O did not tell you:

1. the O(n) scan wins at n=8. It walks consecutive memory -- 16 keys per
   cache line, prefetcher always ahead -- so a few dozen sequential reads
   cost less than 3 dependent pointer hops. By n=128 it has lost badly.

2. the two BST columns are the SAME TREE doing the SAME comparisons in the
   SAME order. The ONLY difference is which addresses the nodes got.
   Up to 65536 they are identical -- the whole tree fits in cache, so
   layout cannot matter. Past that the scattered version falls behind:
   about 1.5x at n=2^20 and 1.8x at n=2^22, widening as n grows.
   No complexity table contains that column.

3. a freshly built tree IS the compact column. That is what a naive
   benchmark measures, and it flatters trees badly: it is the layout you
   get for exactly as long as you never insert or delete anything.

(the compact tree and binary search stay within a few percent of each other,
 trading places at the top end: slices.BinarySearch pays for a generic
 comparison per probe, while contains() compares ints directly. Do not read
 much into which wins. The instructive column is compact-vs-scattered.)

so why use a tree at all? Because a sorted slice cannot be UPDATED cheaply:
insertion is a Theta(n) memmove. The tree buys O(log n) insert and delete.
Read-mostly data belongs in a sorted slice you binary-search (lesson 15).
```

Three readings:

- **The Θ(n) scan wins at n=8.** Sequential memory, 16 keys per line, prefetcher always ahead — a few dozen sequential reads beat three dependent pointer hops. By n=128 it has lost badly.
- **The two tree columns are identical up to n=65536**, because the whole tree fits in cache and layout cannot matter. Past that the scattered version falls behind: ~1.6× at 2²⁰, ~1.9× at 2²², widening.
- **A freshly built tree is the compact column.** That is what a naive benchmark measures, and it is the layout you keep for exactly as long as you never insert or delete.

**Rust:**

```rust
use std::cmp::Ordering;
use std::hint::black_box;
use std::mem::size_of;
use std::time::{Duration, Instant};

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

// The machine is shared, so report the best of three runs: interference only
// ever makes a run slower, never faster.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// ---- RNG: stand-in for math/rand (splitmix64; seeded, so runs are reproducible) ----
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
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
    /// like rand.Perm(n) / Shuffle
    fn shuffle<T>(&mut self, xs: &mut [T]) {
        for i in (1..xs.len()).rev() {
            xs.swap(i, self.below(i as u64 + 1) as usize);
        }
    }
}

// Four ways to answer "is this key present?" over the same sorted keys:
//
//   linear scan of a slice          Theta(n)
//   binary search of a slice        Theta(log n)
//   balanced BST, compact nodes     Theta(log n)
//   balanced BST, scattered nodes   Theta(log n)
//
// The last two are the SAME TREE: same shape, same comparisons, same code.
// They differ only in where in memory the nodes happen to live.

// Cycle through the probes forever. The length is a power of two so that the
// wrap-around is a mask: with `%` on a runtime length the compiler emits a real
// division, which cost about 2 ns per lookup when measured here, as much as the
// whole lookup it sits next to.
fn cycle(probes: &[i64]) -> impl FnMut() -> i64 + '_ {
    assert!(probes.len().is_power_of_two());
    let mut next = 0;
    move || {
        next = (next + 1) & (probes.len() - 1);
        probes[next]
    }
}

struct TreeNode {
    key: i64,
    left: Option<Box<TreeNode>>,
    right: Option<Box<TreeNode>>,
}

// build_from assembles a perfectly balanced tree out of pre-allocated nodes,
// taking them from the pool in order. Hand it a pool in allocation order and the
// tree is laid out compactly; hand it a shuffled pool and the identical tree is
// scattered across the heap. In Rust this works because a Box is a pointer:
// moving it around the pool Vec, or into a parent, never moves the node it owns.
fn build_from(
    sorted: &[i64],
    pool: &mut std::vec::IntoIter<Box<TreeNode>>,
) -> Option<Box<TreeNode>> {
    if sorted.is_empty() {
        return None;
    }
    let mid = sorted.len() / 2;
    let mut node = pool.next().expect("the pool holds one node per key");
    node.key = sorted[mid];
    node.left = build_from(&sorted[..mid], pool);
    node.right = build_from(&sorted[mid + 1..], pool);
    Some(node)
}

// A Vec of Boxes is exactly what clippy::vec_box says to avoid, and exactly what
// this example needs: each node keeps the heap address it was allocated at, no
// matter how the Vec of pointers is shuffled afterwards.
#[allow(clippy::vec_box)]
fn new_pool(n: usize) -> Vec<Box<TreeNode>> {
    (0..n)
        .map(|_| {
            // allocated back-to-back
            Box::new(TreeNode {
                key: 0,
                left: None,
                right: None,
            })
        })
        .collect()
}

fn contains(root: Option<&TreeNode>, key: i64) -> bool {
    let mut cur = root;
    while let Some(n) = cur {
        match key.cmp(&n.key) {
            Ordering::Equal => return true,
            Ordering::Less => cur = n.left.as_deref(),
            Ordering::Greater => cur = n.right.as_deref(),
        }
    }
    false
}

fn linear_contains(xs: &[i64], key: i64) -> bool {
    xs.contains(&key)
}

// The distance in bytes from each node to the child it points at; the median
// of all of them. This is the number that says what "compact" and "scattered"
// mean, since both trees have exactly the same edges.
fn median_hop(root: &TreeNode, edges: usize) -> usize {
    fn walk(n: &TreeNode, hops: &mut Vec<usize>) {
        let from = n as *const TreeNode as usize;
        for child in [&n.left, &n.right].into_iter().flatten() {
            hops.push((&**child as *const TreeNode as usize).abs_diff(from));
            walk(child, hops);
        }
    }
    let mut hops = Vec::with_capacity(edges);
    walk(root, &mut hops);
    let mid = hops.len() / 2;
    *hops.select_nth_unstable(mid).1
}

struct Row {
    n: usize,
    scan: f64,
    bsearch: f64,
    compact: f64,
    scattered: f64,
    hop_compact: usize,
    hop_scattered: usize,
}

fn main() {
    let mut rng = Rng::new(1);

    println!("one lookup, four structures, identical sorted keys.");
    println!("the last two are the same tree -- only the node ADDRESSES differ.");
    println!();
    println!(
        "{:>9} {:>12} {:>12} {:>14} {:>14}   winner",
        "n", "scan", "bsearch", "BST compact", "BST scattered"
    );

    let mut rows = Vec::new();
    for n in [8usize, 128, 4096, 65536, 1 << 20, 1 << 22] {
        let xs: Vec<i64> = (0..n as i64).map(|i| i * 2).collect();

        let compact = build_from(&xs, &mut new_pool(n).into_iter());

        let mut scattered_pool = new_pool(n);
        rng.shuffle(&mut scattered_pool);
        let scattered = build_from(&xs, &mut scattered_pool.into_iter());

        let probes: Vec<i64> = (0..256).map(|_| rng.below(n as u64) as i64 * 2).collect();
        let mut pick = cycle(&probes);

        let scan = best_ns_per_op(|| {
            black_box(linear_contains(black_box(&xs), pick()));
        });
        let bsearch = best_ns_per_op(|| {
            black_box(black_box(&xs).binary_search(&pick()).is_ok());
        });
        let tc = best_ns_per_op(|| {
            black_box(contains(black_box(compact.as_deref()), pick()));
        });
        let ts = best_ns_per_op(|| {
            black_box(contains(black_box(scattered.as_deref()), pick()));
        });

        let (winner, _) = [
            ("scan", scan),
            ("bsearch", bsearch),
            ("BST compact", tc),
            ("BST scattered", ts),
        ]
        .into_iter()
        .min_by(|a, b| a.1.total_cmp(&b.1))
        .expect("four candidates");
        println!("{n:>9} {scan:>12.1} {bsearch:>12.1} {tc:>14.1} {ts:>14.1}   {winner}");

        rows.push(Row {
            n,
            scan,
            bsearch,
            compact: tc,
            scattered: ts,
            hop_compact: median_hop(compact.as_deref().expect("n > 0"), n - 1),
            hop_scattered: median_hop(scattered.as_deref().expect("n > 0"), n - 1),
        });
    }

    // Both trees have exactly the same edges, so "compact" and "scattered" have
    // to mean something about addresses. This table is what they mean.
    println!();
    println!("same two trees, by address: median distance in bytes from a node to its child",);
    println!(
        "(a TreeNode is {} bytes; the allocator rounds each block up).",
        size_of::<TreeNode>()
    );
    println!();
    println!(
        "{:>9} {:>14} {:>14} {:>22}",
        "n", "BST compact", "BST scattered", "time scattered/compact"
    );
    for r in &rows {
        println!(
            "{:>9} {:>14} {:>14} {:>21.2}x",
            r.n,
            r.hop_compact,
            r.hop_scattered,
            r.scattered / r.compact
        );
    }

    println!();
    println!("(at tiny n the allocator hands out whatever freed blocks it has, so the 'compact'");
    println!("pool need not be contiguous there; it is the large rows that the tables compare.)");

    let [r8, r128, r4096, r64k, r1m, r4m] = &rows[..] else {
        unreachable!("six sizes")
    };
    let small_gap = [r8, r128, r4096]
        .iter()
        .map(|r| (r.scattered / r.compact - 1.0).abs() * 100.0)
        .fold(0.0, f64::max);
    println!();
    println!("three things Big-O did not tell you:");
    println!();
    println!(
        "1. the O(n) scan wins at n=8: {:.1} ns against {:.1} for binary search. It walks",
        r8.scan, r8.bsearch
    );
    println!("   consecutive memory, and slice::contains on integers is vectorised (vector");
    println!("   compares in the assembly), so a few dozen sequential reads cost less than a");
    println!(
        "   few dependent pointer hops. It has lost by n=128 ({:.1} vs {:.1}) and by n=4096",
        r128.scan, r128.bsearch
    );
    println!(
        "   it is {:.0}x slower than binary search.",
        r4096.scan / r4096.bsearch
    );
    println!();
    println!("2. the two BST columns are the SAME TREE doing the SAME comparisons in the");
    println!("   SAME order. The ONLY difference is which addresses the nodes got (the hop");
    println!(
        "   table). Through n=4096 their times are within {small_gap:.0}% of each other: the tree"
    );
    println!("   is small enough that its layout hardly matters. At 65536 the scattered tree is");
    println!(
        "   FASTER ({:.1} vs {:.1} ns), which is not what you would predict and which I have",
        r64k.scattered, r64k.compact
    );
    println!(
        "   not explained. At 2^20 the ratio is {:.2}x and at 2^22 the scattered tree is",
        r1m.scattered / r1m.compact
    );
    println!(
        "   {:.1}x slower: the gap opens only at the largest sizes, where I did not isolate",
        r4m.scattered / r4m.compact
    );
    println!("   which cache level is the cause. No complexity table has that column.");
    println!();
    println!("3. a freshly built tree IS the compact column, because the pool was allocated in");
    println!("   one go. That is what a naive benchmark measures, and it flatters trees: it is");
    println!("   the layout you get for exactly as long as you never insert or delete anything.");
    println!();
    println!(
        "(binary search is NOT within a few percent of the compact tree here: at 2^22 it takes {:.1} ns",
        r4m.bsearch
    );
    println!(
        " against {:.1} for the compact tree. std's binary_search is deliberately branchless",
        r4m.compact
    );
    println!(" (a csel in the assembly, and a comment in core::slice saying binary search");
    println!(" 'interacts poorly with branch prediction'). With no branch to guess, each probe");
    println!(" waits for the previous load; a branchy search lets the CPU guess ahead and start");
    println!(" the next load early whenever the branches are predictable, as they may be for 256");
    println!(" probes repeating forever. That is a plausible mechanism, not one I measured.)");
    println!();
    println!("so why use a tree at all? Because a sorted slice cannot be UPDATED cheaply:");
    println!("Vec::insert is a Theta(n) memmove. The tree buys O(log n) insert and delete.");
    println!("Read-mostly data belongs in a sorted slice you binary-search (lesson 15).");
}
```

**Rust sample output:**

```
one lookup, four structures, identical sorted keys.
the last two are the same tree -- only the node ADDRESSES differ.

        n         scan      bsearch    BST compact  BST scattered   winner
        8          1.0          1.2            1.5            1.5   scan
      128          5.1          2.8            4.8            4.6   bsearch
     4096        166.6          5.7            8.2            7.2   bsearch
    65536       2462.6         17.4           16.5           10.9   BST scattered
  1048576      41669.6         32.6           20.4           24.5   BST compact
  4194304     194404.3         45.3           22.9           38.2   BST compact

same two trees, by address: median distance in bytes from a node to its child
(a TreeNode is 24 bytes; the allocator rounds each block up).

        n    BST compact  BST scattered time scattered/compact
        8            304             96                  1.00x
      128             32           1408                  0.95x
     4096             64          38656                  0.88x
    65536             64         618080                  0.66x
  1048576             64       11032352                  1.20x
  4194304             64       56311456                  1.67x

(at tiny n the allocator hands out whatever freed blocks it has, so the 'compact'
pool need not be contiguous there; it is the large rows that the tables compare.)

three things Big-O did not tell you:

1. the O(n) scan wins at n=8: 1.0 ns against 1.2 for binary search. It walks
   consecutive memory, and slice::contains on integers is vectorised (vector
   compares in the assembly), so a few dozen sequential reads cost less than a
   few dependent pointer hops. It has lost by n=128 (5.1 vs 2.8) and by n=4096
   it is 29x slower than binary search.

2. the two BST columns are the SAME TREE doing the SAME comparisons in the
   SAME order. The ONLY difference is which addresses the nodes got (the hop
   table). Through n=4096 their times are within 12% of each other: the tree
   is small enough that its layout hardly matters. At 65536 the scattered tree is
   FASTER (10.9 vs 16.5 ns), which is not what you would predict and which I have
   not explained. At 2^20 the ratio is 1.20x and at 2^22 the scattered tree is
   1.7x slower: the gap opens only at the largest sizes, where I did not isolate
   which cache level is the cause. No complexity table has that column.

3. a freshly built tree IS the compact column, because the pool was allocated in
   one go. That is what a naive benchmark measures, and it flatters trees: it is
   the layout you get for exactly as long as you never insert or delete anything.

(binary search is NOT within a few percent of the compact tree here: at 2^22 it takes 45.3 ns
 against 22.9 for the compact tree. std's binary_search is deliberately branchless
 (a csel in the assembly, and a comment in core::slice saying binary search
 'interacts poorly with branch prediction'). With no branch to guess, each probe
 waits for the previous load; a branchy search lets the CPU guess ahead and start
 the next load early whenever the branches are predictable, as they may be for 256
 probes repeating forever. That is a plausible mechanism, not one I measured.)

so why use a tree at all? Because a sorted slice cannot be UPDATED cheaply:
Vec::insert is a Theta(n) memmove. The tree buys O(log n) insert and delete.
Read-mostly data belongs in a sorted slice you binary-search (lesson 15).
```

> **Go vs Rust:** Same experiment, partly different answers. The scattered tree is clearly slower at 2^22 in both languages (1.6-2.3x over my runs; Go: 1.8x), but at 2^20 it is only 1.0-1.25x slower here (Go: 1.5x), and at 65536 it is 1.3-1.7x FASTER than the compact one in every run I made (Go: "identical up to 65536"); I do not know why. The hop table shows the two trees really do differ in addresses, which Go never checked. `binary_search` is not within a few percent of the compact tree either: at 2^22 it takes about 2x as long, and std's search is deliberately branchless. The Rust scan stays competitive longer than Go's because `slice::contains` is vectorised. Moving a `Box` never moves its node, which is what makes the shuffled-pool trick work in safe Rust.

**Complexity:** scan Θ(n) · binary search and both trees Θ(log n) · the gap between the two Θ(log n) trees is entirely allocation addresses

---

## 15. Sorted slice vs map — the honest comparison

`🔴 hard` · *Choosing on more than lookup speed*

The textbook says map is O(1) and binary search is O(log n), so use the map. The measurement agrees —
**the map is faster at every size here**. So the sorted slice is not competing on lookup speed. It is
competing on three other things.

**Steps:**

1. Measure both at six sizes, tracking time and memory.
2. Confirm the shapes: map flat, binary search growing with log n.
3. Then read the memory column, which never improves.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"testing"
)

var sink bool

// The textbook says: map is O(1), sorted slice is O(log n), so use the map.
// The textbook is not wrong. It is just not the whole decision -- a map costs
// ~4.7x the memory (example 6) and its "O(1)" has a large constant.

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// heapBytes is signed: at small n the GC can free more between two readings
// than the map allocated, and an unsigned subtraction would wrap to nonsense.
func heapBytes() int64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return int64(ms.HeapAlloc)
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("lookup: sorted slice + binary search vs map")
	fmt.Println()
	fmt.Printf("%9s %14s %14s %10s %14s %14s %8s\n",
		"n", "bsearch ns", "map ns", "map wins", "slice MB", "map MB", "x mem")

	for _, n := range []int{16, 128, 1024, 16384, 262144, 1 << 21} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}

		before := heapBytes()
		m := make(map[int]struct{}, n)
		for _, v := range xs {
			m[v] = struct{}{}
		}
		mapBytes := heapBytes() - before

		probes := make([]int, 256)
		for i := range probes {
			probes[i] = rng.IntN(n) * 2
		}
		next := 0
		pick := func() int {
			next = (next + 1) % len(probes)
			return probes[next]
		}

		bs := nsPerOp(func() {
			_, ok := slices.BinarySearch(xs, pick())
			sink = ok
		})
		mp := nsPerOp(func() {
			_, ok := m[pick()]
			sink = ok
		})

		sliceMB := float64(n*8) / (1 << 20)
		mapMB := float64(mapBytes) / (1 << 20)

		verdict := "no"
		if mp < bs {
			verdict = "yes"
		}
		// Below a few thousand keys the heap delta is smaller than GC noise.
		ratio := "-"
		if n >= 1024 {
			ratio = fmt.Sprintf("%.1fx", mapMB/sliceMB)
		}
		fmt.Printf("%9d %14.1f %14.1f %10s %14.3f %14.3f %8s\n",
			n, bs, mp, verdict, sliceMB, mapMB, ratio)

		runtime.KeepAlive(m)
	}

	fmt.Println()
	fmt.Println("the shapes are exactly as advertised: the map is FLAT (~4-5 ns at every")
	fmt.Println("size) and binary search GROWS with log n (5.8 -> 20 ns). And unlike the")
	fmt.Println("tree-vs-slice comparison in example 14, the map is genuinely faster at")
	fmt.Println("every size measured -- hashing one int is cheap and it is a single probe,")
	fmt.Println("versus up to 21 dependent halvings.")
	fmt.Println()
	fmt.Println("so the sorted slice is NOT competing on lookup speed. It competes on:")
	fmt.Println()
	fmt.Println("  memory   ~4.5x less, at every size, forever (example 6)")
	fmt.Println("  order    binary search gives you predecessor, successor, and RANGE")
	fmt.Println("           queries. A map gives you none of those at any price.")
	fmt.Println("  layout   one contiguous block: no per-key objects, no GC tracing")
	fmt.Println("           (example 12), and it can be memory-mapped or written to disk")
	fmt.Println("           as-is (which is why every on-disk index is sorted, lesson 40)")
	fmt.Println()
	fmt.Println("how to choose, in practice:")
	fmt.Println("  static or read-mostly data      -> sort a slice, binary-search it")
	fmt.Println("  frequent inserts/deletes        -> map")
	fmt.Println("  memory-constrained, huge n      -> sorted slice")
	fmt.Println("  need ordered iteration or range -> sorted slice (a map has neither)")
	fmt.Println("  keys are dense small integers   -> neither: index a slice directly, O(1)")
	fmt.Println()
	fmt.Println("that last line is worth more than the rest. If your keys are 0..n-1,")
	fmt.Println("xs[key] is a real O(1) with a constant of ~1 and no memory overhead at all.")
	fmt.Println("Example 16 builds a graph on exactly that idea.")
}
```

**Sample output:**

```
lookup: sorted slice + binary search vs map

        n     bsearch ns         map ns   map wins       slice MB         map MB    x mem
       16            5.8            4.3        yes          0.000         -0.001        -
      128            8.3            4.3        yes          0.001          0.003        -
     1024           11.2            4.6        yes          0.008          0.034     4.3x
    16384           14.2            4.6        yes          0.125          0.562     4.5x
   262144           17.8            5.3        yes          2.000          9.019     4.5x
  2097152           20.4            5.4        yes         16.000         72.156     4.5x

the shapes are exactly as advertised: the map is FLAT (~4-5 ns at every
size) and binary search GROWS with log n (5.8 -> 20 ns). And unlike the
tree-vs-slice comparison in example 14, the map is genuinely faster at
every size measured -- hashing one int is cheap and it is a single probe,
versus up to 21 dependent halvings.

so the sorted slice is NOT competing on lookup speed. It competes on:

  memory   ~4.5x less, at every size, forever (example 6)
  order    binary search gives you predecessor, successor, and RANGE
           queries. A map gives you none of those at any price.
  layout   one contiguous block: no per-key objects, no GC tracing
           (example 12), and it can be memory-mapped or written to disk
           as-is (which is why every on-disk index is sorted, lesson 40)

how to choose, in practice:
  static or read-mostly data      -> sort a slice, binary-search it
  frequent inserts/deletes        -> map
  memory-constrained, huge n      -> sorted slice
  need ordered iteration or range -> sorted slice (a map has neither)
  keys are dense small integers   -> neither: index a slice directly, O(1)

that last line is worth more than the rest. If your keys are 0..n-1,
xs[key] is a real O(1) with a constant of ~1 and no memory overhead at all.
Example 16 builds a graph on exactly that idea.
```

Note the signed heap delta: at small n the GC can free more between two readings than the map
allocated, and an unsigned subtraction wraps to nonsense. (My first version printed 17592186044416 MB.)

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashSet;
use std::hash::{BuildHasherDefault, Hasher};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// A counting allocator stands in for runtime.MemStats: exact, so there is no
// forced GC and no noisy small-n readings. A global allocator is an `unsafe`
// trait by definition; this one only counts and forwards to the system.
struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0);

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

fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// ---- RNG: stand-in for math/rand (splitmix64; seeded, so runs are reproducible) ----
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
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// The textbook says: set is O(1), sorted slice is O(log n), so use the set.
// The textbook is not wrong. It is just not the whole decision: a set costs
// extra memory (example 6) and its "O(1)" includes hashing the key, which in
// Rust is a choice. HashSet's default hasher is SipHash, built to resist
// hostile keys; a cheap multiplicative one is the other end of that choice.
//
// A multiply-and-rotate hasher (the idea behind rustc's own FxHash): about one
// multiplication per integer, no randomisation, no protection against hostile keys.
#[derive(Default)]
struct Fx(u64);
impl Hasher for Fx {
    fn finish(&self) -> u64 {
        self.0
    }
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.write_u64(u64::from(b));
        }
    }
    fn write_u64(&mut self, x: u64) {
        self.0 = (self.0.rotate_left(5) ^ x).wrapping_mul(0x517c_c1b7_2722_0a95);
    }
    fn write_i64(&mut self, x: i64) {
        self.write_u64(x as u64);
    }
}
type FxSet = HashSet<i64, BuildHasherDefault<Fx>>;

// Cycle through the probes forever. The length is a power of two so that the
// wrap-around is a mask: with `%` on a runtime length the compiler emits a real
// division, which cost about 2 ns per lookup when measured here, as much as the
// whole lookup it sits next to.
fn cycle(probes: &[i64]) -> impl FnMut() -> i64 + '_ {
    assert!(probes.len().is_power_of_two());
    let mut next = 0;
    move || {
        next = (next + 1) & (probes.len() - 1);
        probes[next]
    }
}

struct Row {
    n: usize,
    bsearch: f64,
    set: f64,
    fx: f64,
    slice_bytes: usize,
    set_bytes: usize,
}

// Heap bytes taken by a set of the first n even numbers, from the allocator's
// own books: exact, so unlike a MemStats delta it needs no GC and no signed
// arithmetic, and it is not noisy at small n.
fn bytes_of<S: Extend<i64>>(make: impl FnOnce() -> S, n: usize) -> (usize, S) {
    let before = LIVE.load(Relaxed);
    let mut set = make();
    set.extend((0..n as i64).map(|i| i * 2));
    (LIVE.load(Relaxed) - before, set)
}

fn main() {
    let mut rng = Rng::new(1);

    println!("lookup: sorted slice + binary search vs HashSet (std SipHash, and a cheap hasher)");
    println!();
    println!(
        "{:>9} {:>11} {:>11} {:>11} {:>9} {:>8} {:>10} {:>10} {:>7}",
        "n",
        "bsearch ns",
        "set ns",
        "fx set ns",
        "set wins",
        "fx wins",
        "slice MB",
        "set MB",
        "x mem"
    );

    let mb = |bytes: usize| bytes as f64 / (1 << 20) as f64;
    let verdict = |set: f64, bsearch: f64| if set < bsearch { "yes" } else { "no" };
    let mut rows = Vec::new();
    let mut biggest = None; // the last (largest) tables, kept for the probe-count check
    for n in [16usize, 128, 1024, 16384, 262144, 1 << 21] {
        let xs: Vec<i64> = (0..n as i64).map(|i| i * 2).collect();

        let (set_bytes, set) = bytes_of(|| HashSet::with_capacity(n), n);
        let (fx_bytes, fx) = bytes_of(|| FxSet::with_capacity_and_hasher(n, Default::default()), n);
        assert_eq!(
            set_bytes, fx_bytes,
            "same table layout, whatever the hasher"
        );

        let probes: Vec<i64> = (0..256).map(|_| rng.below(n as u64) as i64 * 2).collect();
        let mut pick = cycle(&probes);

        let bsearch = best_ns_per_op(|| {
            black_box(black_box(&xs).binary_search(&pick()).is_ok());
        });
        let std_set = best_ns_per_op(|| {
            black_box(black_box(&set).contains(&pick()));
        });
        let fx_set = best_ns_per_op(|| {
            black_box(black_box(&fx).contains(&pick()));
        });

        let row = Row {
            n,
            bsearch,
            set: std_set,
            fx: fx_set,
            slice_bytes: n * 8,
            set_bytes,
        };
        println!(
            "{n:>9} {bsearch:>11.1} {std_set:>11.1} {fx_set:>11.1} {:>9} {:>8} {:>10.3} {:>10.3} {:>6.2}x",
            verdict(std_set, bsearch),
            verdict(fx_set, bsearch),
            mb(row.slice_bytes),
            mb(row.set_bytes),
            row.set_bytes as f64 / row.slice_bytes as f64
        );
        rows.push(row);
        biggest = Some((xs, set, fx));
    }
    let (xs, set, fx) = biggest.expect("the loop ran");

    let (first, last) = (&rows[0], &rows[rows.len() - 1]);
    let set_lo = rows.iter().map(|r| r.set).fold(f64::INFINITY, f64::min);
    let set_hi = rows.iter().map(|r| r.set).fold(0.0, f64::max);
    let ratio_lo = rows
        .iter()
        .map(|r| r.set_bytes as f64 / r.slice_bytes as f64)
        .fold(f64::INFINITY, f64::min);
    let ratio_hi = rows
        .iter()
        .map(|r| r.set_bytes as f64 / r.slice_bytes as f64)
        .fold(0.0, f64::max);
    let crossover = rows.iter().find(|r| r.set < r.bsearch).map(|r| r.n);
    println!();
    println!(
        "the shapes: the std set is nearly FLAT ({set_lo:.1} to {set_hi:.1} ns at every size) and binary"
    );
    println!(
        "search GROWS with log n ({:.1} -> {:.1} ns). But the textbook winner does not win",
        first.bsearch, last.bsearch
    );
    match crossover {
        Some(n) => println!(
            "everywhere: with std's default hasher the set only overtakes the slice at n={n}."
        ),
        None => println!("anywhere: with std's default hasher the slice wins at every size here."),
    }
    println!(
        "Hashing is the difference. The fx set has the same table, the same bytes and the same"
    );
    println!(
        "probe, with a hash that costs one multiplication: {:.1} to {:.1} ns, and it wins at",
        rows.iter().map(|r| r.fx).fold(f64::INFINITY, f64::min),
        rows.iter().map(|r| r.fx).fold(0.0, f64::max)
    );
    println!("every size, even n=16. Hash cost is part of 'O(1)'; it is not free, and it is a");
    println!("setting (SipHash buys protection against keys chosen to collide).");

    // Both flat lines come from a loop that cycles through 256 probes forever:
    // the same 256 buckets stay hot in cache. Spread the probes out and see.
    let n = last.n;
    println!();
    println!("how flat is the flat line? n = {n}, cycling through 256 probes vs 1048576:");
    println!(
        "{:>12} {:>11} {:>11} {:>11}",
        "probes", "bsearch ns", "set ns", "fx set ns"
    );
    for count in [256usize, 1 << 20] {
        let probes: Vec<i64> = (0..count).map(|_| rng.below(n as u64) as i64 * 2).collect();
        let mut pick = cycle(&probes);
        let bsearch = best_ns_per_op(|| {
            black_box(black_box(&xs).binary_search(&pick()).is_ok());
        });
        let std_set = best_ns_per_op(|| {
            black_box(black_box(&set).contains(&pick()));
        });
        let fx_set = best_ns_per_op(|| {
            black_box(black_box(&fx).contains(&pick()));
        });
        println!("{count:>12} {bsearch:>11.1} {std_set:>11.1} {fx_set:>11.1}");
    }
    println!("the ranking survives, the flatness does not. A loop that reuses 256 keys keeps");
    println!("their 256 buckets hot in cache and so never sees a miss; spread the probes");
    println!("out and every lookup lands somewhere cold (plausible mechanism, not measured).");

    println!();
    println!("so the sorted slice is NOT competing on lookup speed alone. Memory: the set costs");
    println!(
        "{ratio_lo:.2}x to {ratio_hi:.2}x the slice at every size above. These sizes are all powers of two."
    );
    println!("hashbrown keeps its table at most 7/8 full and rounds the bucket count up to a");
    println!("power of two, so the cost steps with n (9 bytes per bucket: key plus control byte):");
    for n in [14336usize, 14337, 16384] {
        let (bytes, _set) = bytes_of(|| HashSet::with_capacity(n), n);
        println!(
            "  n={n:>6}: {:5.2} bytes per key, {:.2}x the slice",
            bytes as f64 / n as f64,
            bytes as f64 / (n * 8) as f64
        );
    }
    println!();
    println!("  order    binary search gives you predecessor, successor and RANGE queries");
    println!("           (partition_point). A HashSet gives you none of those at any price;");
    println!("           the std answer for an ordered AND updatable set is BTreeSet.");
    println!("  layout   one contiguous block that can be memory-mapped or written to disk");
    println!("           as-is (which is why every on-disk index is sorted, lesson 40)");
    println!();
    println!("how to choose, in practice:");
    println!("  static or read-mostly data      -> sort a Vec, binary-search it");
    println!("  frequent inserts/deletes        -> HashSet (pick the hasher on purpose)");
    println!("  memory-constrained, huge n      -> sorted Vec");
    println!("  need ordered iteration or range -> sorted Vec, or BTreeSet if it also changes");
    println!("  keys are dense small integers   -> neither: index a Vec directly, O(1)");
    println!();
    println!("that last line is worth more than the rest. If your keys are 0..n-1,");
    println!("xs[key] is a real O(1) with a constant of ~1 and no memory overhead at all.");
    println!("Example 16 builds a graph on exactly that idea.");
}
```

**Rust sample output:**

```
lookup: sorted slice + binary search vs HashSet (std SipHash, and a cheap hasher)

        n  bsearch ns      set ns   fx set ns  set wins  fx wins   slice MB     set MB   x mem
       16         2.0         4.8         1.1        no      yes      0.000      0.000   2.31x
      128         3.0         5.0         1.1        no      yes      0.001      0.002   2.26x
     1024         4.5         5.2         1.2        no      yes      0.008      0.018   2.25x
    16384         8.9         5.3         1.3       yes      yes      0.125      0.281   2.25x
   262144        28.8         5.9         1.3       yes      yes      2.000      4.500   2.25x
  2097152        57.2         7.2         1.6       yes      yes     16.000     36.000   2.25x

the shapes: the std set is nearly FLAT (4.8 to 7.2 ns at every size) and binary
search GROWS with log n (2.0 -> 57.2 ns). But the textbook winner does not win
everywhere: with std's default hasher the set only overtakes the slice at n=16384.
Hashing is the difference. The fx set has the same table, the same bytes and the same
probe, with a hash that costs one multiplication: 1.1 to 1.6 ns, and it wins at
every size, even n=16. Hash cost is part of 'O(1)'; it is not free, and it is a
setting (SipHash buys protection against keys chosen to collide).

how flat is the flat line? n = 2097152, cycling through 256 probes vs 1048576:
      probes  bsearch ns      set ns   fx set ns
         256        51.8         6.6         1.8
     1048576       163.0        40.9        11.3
the ranking survives, the flatness does not. A loop that reuses 256 keys keeps
their 256 buckets hot in cache and so never sees a miss; spread the probes
out and every lookup lands somewhere cold (plausible mechanism, not measured).

so the sorted slice is NOT competing on lookup speed alone. Memory: the set costs
2.25x to 2.31x the slice at every size above. These sizes are all powers of two.
hashbrown keeps its table at most 7/8 full and rounds the bucket count up to a
power of two, so the cost steps with n (9 bytes per bucket: key plus control byte):
  n= 14336: 10.29 bytes per key, 1.29x the slice
  n= 14337: 20.57 bytes per key, 2.57x the slice
  n= 16384: 18.00 bytes per key, 2.25x the slice

  order    binary search gives you predecessor, successor and RANGE queries
           (partition_point). A HashSet gives you none of those at any price;
           the std answer for an ordered AND updatable set is BTreeSet.
  layout   one contiguous block that can be memory-mapped or written to disk
           as-is (which is why every on-disk index is sorted, lesson 40)

how to choose, in practice:
  static or read-mostly data      -> sort a Vec, binary-search it
  frequent inserts/deletes        -> HashSet (pick the hasher on purpose)
  memory-constrained, huge n      -> sorted Vec
  need ordered iteration or range -> sorted Vec, or BTreeSet if it also changes
  keys are dense small integers   -> neither: index a Vec directly, O(1)

that last line is worth more than the rest. If your keys are 0..n-1,
xs[key] is a real O(1) with a constant of ~1 and no memory overhead at all.
Example 16 builds a graph on exactly that idea.
```

> **Go vs Rust:** The Go claim "the map is faster at every size measured" does not hold for std's `HashSet`: with its default SipHash the sorted slice wins clearly at n=16 and n=128 (roughly 2-4 ns against 5-6), is a toss-up at n=1024, and the set pulls ahead only from n=16384; that ordering held in every run I made. The cause is the hasher: the same table (identical bytes, asserted in the program) with a one-multiplication hasher, ~10 lines of std-only code, takes about 1.1-1.6 ns and wins at every size, which Go cannot show because its map has one built-in hash. Memory is not Go's "~4.5x" either: the set costs 2.25-2.31x the slice here, and between 1.29x and 2.57x depending on where n falls. The flat line is partly the benchmark: reusing 256 probes keeps them hot, and with 1,048,576 probes the std set costs about 4x more per lookup at n=2^21. Memory is read from a counting allocator (exact), so there is no GC noise and no "-" column.

**Complexity:** map Θ(1) average, ~4–5 ns flat · binary search Θ(log n), 5.8 → 21 ns · the slice's case is memory (4.5×), ordered/range queries, and a layout you can write to disk unchanged

---

## 16. One graph, three layouts, one BFS

`🔴 hard` · *The capstone*

A million vertices, eight million edges, the **same Θ(V+E) BFS** run over three representations.
Nothing algorithmic differs. Everything in the results is layout.

The third is **CSR** (Compressed Sparse Row): one `offsets` array and one `targets` array, with the
neighbours of `v` living at `targets[offsets[v]:offsets[v+1]]`. It is how real graph and
sparse-matrix libraries store adjacency.

**Steps:**

1. Build the graph as `map[int][]int`, `[][]int`, and CSR.
2. Measure each structure's memory in isolation — **GC between build and measurement**, or CSR's temporary scratch gets counted as part of the result.
3. Run the same BFS on all three.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
)

var sink int

// The same graph, three ways. Identical vertices, identical edges, identical
// BFS algorithm -- Theta(V+E) in every case. Only the LAYOUT changes.
//
//   1. map[int][]int   the shape people reach for first
//   2. [][]int         one slice per vertex
//   3. CSR             two flat arrays, no per-vertex allocation at all
//
// CSR (Compressed Sparse Row) is how real graph and sparse-matrix libraries
// store adjacency, and this example is why.

// --- representation 1: map of slices ---

type MapGraph map[int][]int

func (g MapGraph) BFS(start int, visited []bool, queue []int) int {
	for i := range visited {
		visited[i] = false
	}
	queue = queue[:0]
	queue = append(queue, start)
	visited[start] = true
	seen := 0

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		seen++
		for _, w := range g[v] {
			if !visited[w] {
				visited[w] = true
				queue = append(queue, w)
			}
		}
	}
	return seen
}

// --- representation 2: slice of slices ---

type SliceGraph [][]int

func (g SliceGraph) BFS(start int, visited []bool, queue []int) int {
	for i := range visited {
		visited[i] = false
	}
	queue = queue[:0]
	queue = append(queue, start)
	visited[start] = true
	seen := 0

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		seen++
		for _, w := range g[v] {
			if !visited[w] {
				visited[w] = true
				queue = append(queue, w)
			}
		}
	}
	return seen
}

// --- representation 3: CSR ---
//
// offsets has V+1 entries; the neighbours of v are targets[offsets[v]:offsets[v+1]].
// Two allocations for the whole graph, both pointer-free, both contiguous.
type CSR struct {
	offsets []int32
	targets []int32
}

func (g *CSR) BFS(start int32, visited []bool, queue []int32) int {
	for i := range visited {
		visited[i] = false
	}
	queue = queue[:0]
	queue = append(queue, start)
	visited[start] = true
	seen := 0

	for len(queue) > 0 {
		v := queue[0]
		queue = queue[1:]
		seen++
		for _, w := range g.targets[g.offsets[v]:g.offsets[v+1]] {
			if !visited[w] {
				visited[w] = true
				queue = append(queue, w)
			}
		}
	}
	return seen
}

func buildAll(v, degree int, rng *rand.Rand) (MapGraph, SliceGraph, *CSR) {
	adj := make([][]int, v)
	for i := range adj {
		adj[i] = make([]int, 0, degree)
		for d := 0; d < degree; d++ {
			adj[i] = append(adj[i], rng.IntN(v))
		}
	}

	mg := make(MapGraph, v)
	for i, ns := range adj {
		mg[i] = ns
	}

	sg := SliceGraph(adj)

	csr := &CSR{
		offsets: make([]int32, v+1),
		targets: make([]int32, 0, v*degree),
	}
	for i, ns := range adj {
		csr.offsets[i] = int32(len(csr.targets))
		for _, w := range ns {
			csr.targets = append(csr.targets, int32(w))
		}
	}
	csr.offsets[v] = int32(len(csr.targets))

	return mg, sg, csr
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// measureMem reports what the FINISHED structure costs. The GC between build
// and the second reading is essential: CSR is assembled from a temporary
// [][]int, and without collecting that scratch first the measurement reports
// the intermediate as though it were part of the result.
func measureMem(build func() any) (float64, uint64) {
	runtime.GC()
	var before runtime.MemStats
	runtime.ReadMemStats(&before)

	v := build()

	runtime.GC() // drop everything the build allocated but did not keep
	var after runtime.MemStats
	runtime.ReadMemStats(&after)
	runtime.KeepAlive(v)

	return float64(after.HeapAlloc-before.HeapAlloc) / (1 << 20),
		after.HeapObjects - before.HeapObjects
}

func main() {
	const (
		v      = 1_000_000
		degree = 8
	)
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Printf("graph: %d vertices, %d edges (out-degree %d)\n\n", v, v*degree, degree)

	// --- memory, each built in isolation ---
	adjFor := func() [][]int {
		adj := make([][]int, v)
		for i := range adj {
			adj[i] = make([]int, 0, degree)
			for d := 0; d < degree; d++ {
				adj[i] = append(adj[i], rng.IntN(v))
			}
		}
		return adj
	}

	mapMB, mapObjs := measureMem(func() any {
		g := make(MapGraph, v)
		for i, ns := range adjFor() {
			g[i] = ns
		}
		return g
	})
	sliceMB, sliceObjs := measureMem(func() any { return SliceGraph(adjFor()) })
	csrMB, csrObjs := measureMem(func() any {
		adj := adjFor()
		c := &CSR{offsets: make([]int32, v+1), targets: make([]int32, 0, v*degree)}
		for i, ns := range adj {
			c.offsets[i] = int32(len(c.targets))
			for _, w := range ns {
				c.targets = append(c.targets, int32(w))
			}
		}
		c.offsets[v] = int32(len(c.targets))
		return c
	})

	// --- speed, all three live at once so the comparison is like-for-like ---
	mg, sg, csr := buildAll(v, degree, rng)
	visited := make([]bool, v)
	queueInt := make([]int, 0, v)
	queue32 := make([]int32, 0, v)

	tMap := nsPerOp(func() { sink += mg.BFS(0, visited, queueInt) })
	tSlice := nsPerOp(func() { sink += sg.BFS(0, visited, queueInt) })
	tCSR := nsPerOp(func() { sink += csr.BFS(0, visited, queue32) })

	fmt.Printf("%-18s %12s %14s %16s %12s\n", "", "BFS ms", "vs CSR", "heap objects", "MB")
	fmt.Printf("%-18s %12.1f %13.1fx %16d %12.1f\n", "map[int][]int", tMap/1e6, tMap/tCSR, mapObjs, mapMB)
	fmt.Printf("%-18s %12.1f %13.1fx %16d %12.1f\n", "[][]int", tSlice/1e6, tSlice/tCSR, sliceObjs, sliceMB)
	fmt.Printf("%-18s %12.1f %13.1fx %16d %12.1f\n", "CSR", tCSR/1e6, 1.0, csrObjs, csrMB)

	fmt.Println()
	fmt.Println("every lesson in this file, in one table:")
	fmt.Println()
	fmt.Println("  map[int][]int  a hash lookup per vertex visit (example 15), plus a")
	fmt.Println("                 separate slice per vertex: 1M+ heap objects, all traced")
	fmt.Println("  [][]int        no hashing -- g[v] is an index -- but still one")
	fmt.Println("                 allocation per vertex, so still 1M objects and 24 bytes")
	fmt.Println("                 of slice header each (examples 1, 10, 12)")
	fmt.Println("  CSR            TWO allocations for the entire graph. Neighbours of")
	fmt.Println("                 every vertex are contiguous (example 7), int32 instead")
	fmt.Println("                 of int halves the edge array (example 6), and there is")
	fmt.Println("                 not one pointer in it, so the GC skips it (example 12)")
	fmt.Println()
	fmt.Println("what CSR gives up: you cannot add an edge without rebuilding. That is the")
	fmt.Println("trade -- it is a BUILD-ONCE, QUERY-MANY structure, which is exactly what a")
	fmt.Println("road network, a dependency graph or a sparse matrix is.")
	fmt.Println()
	fmt.Println("this is the payoff for the whole lesson: same Theta(V+E), same algorithm,")
	fmt.Println("and the difference between the first row and the last is pure layout.")
}
```

**Sample output:**

```
graph: 1000000 vertices, 8000000 edges (out-degree 8)

                         BFS ms         vs CSR     heap objects           MB
map[int][]int              80.6           2.9x          1004118        141.1
[][]int                    47.1           1.7x          1000009         83.9
CSR                        28.2           1.0x                4         34.3

every lesson in this file, in one table:

  map[int][]int  a hash lookup per vertex visit (example 15), plus a
                 separate slice per vertex: 1M+ heap objects, all traced
  [][]int        no hashing -- g[v] is an index -- but still one
                 allocation per vertex, so still 1M objects and 24 bytes
                 of slice header each (examples 1, 10, 12)
  CSR            TWO allocations for the entire graph. Neighbours of
                 every vertex are contiguous (example 7), int32 instead
                 of int halves the edge array (example 6), and there is
                 not one pointer in it, so the GC skips it (example 12)

what CSR gives up: you cannot add an edge without rebuilding. That is the
trade -- it is a BUILD-ONCE, QUERY-MANY structure, which is exactly what a
road network, a dependency graph or a sparse matrix is.

this is the payoff for the whole lesson: same Theta(V+E), same algorithm,
and the difference between the first row and the last is pure layout.
```

**Four heap objects instead of a million.** Every concept in this lesson is in that one row:

| | mechanism | example |
|---|---|---|
| `map[int][]int` | a hash probe per vertex visit, plus a slice per vertex | 15, 6 |
| `[][]int` | indexing instead of hashing, but still one allocation and one 24-byte header per vertex | 1, 10, 12 |
| CSR | two allocations total; neighbours contiguous; `int32` instead of `int`; **not one pointer**, so the GC skips it | 7, 6, 12 |

What CSR gives up: you cannot add an edge without rebuilding. It is a **build-once, query-many**
structure — which is exactly what a road network, a dependency graph, or a sparse matrix is.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::hash::{BuildHasherDefault, Hasher};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// A counting allocator stands in for runtime.MemStats: it keeps the bytes and
// the number of blocks that are alive right now. That is exact, and it makes
// "GC between build and measurement" unnecessary: Rust frees the build's scratch
// the moment it goes out of scope. A global allocator is an `unsafe` trait by
// definition; this one only counts and forwards to the system allocator.
struct Counting;
static LIVE_BYTES: AtomicUsize = AtomicUsize::new(0);
static LIVE_OBJECTS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        LIVE_BYTES.fetch_add(l.size(), Relaxed);
        LIVE_OBJECTS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE_BYTES.fetch_sub(l.size(), Relaxed);
        LIVE_OBJECTS.fetch_sub(1, Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        LIVE_BYTES.fetch_add(new, Relaxed);
        LIVE_BYTES.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

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

// The machine is shared, so report the best of three runs: interference only
// ever makes a run slower, never faster.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// ---- RNG: stand-in for math/rand (splitmix64; seeded, so runs are reproducible) ----
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
    /// like rand.Intn(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// A multiply-and-rotate hasher (the idea behind rustc's own FxHash): about one
// multiplication per integer, no randomisation, no protection against hostile
// keys. It is here to separate "the map hashes" from "the map is a different
// layout"; std's default is SipHash.
#[derive(Default)]
struct Fx(u64);
impl Hasher for Fx {
    fn finish(&self) -> u64 {
        self.0
    }
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.write_u64(u64::from(b));
        }
    }
    fn write_u64(&mut self, x: u64) {
        self.0 = (self.0.rotate_left(5) ^ x).wrapping_mul(0x517c_c1b7_2722_0a95);
    }
    fn write_usize(&mut self, x: usize) {
        self.write_u64(x as u64);
    }
}

// The same graph, ways. Identical vertices, identical edges, identical BFS
// algorithm -- Theta(V+E) in every case. Only the LAYOUT changes.
//
//   1. HashMap<usize, Vec<usize>>   the shape people reach for first
//   2. Vec<Vec<usize>>              one Vec per vertex
//   3. CSR                          two flat arrays, no per-vertex allocation at all
//
// CSR (Compressed Sparse Row) is how real graph and sparse-matrix libraries
// store adjacency, and this example is why. Row 1 is shown twice: with std's
// default SipHash and with the cheap hasher above.

const V: usize = 1_000_000;
const DEGREE: usize = 8;

// --- representation 1: map of vectors ---

type MapGraph = HashMap<usize, Vec<usize>>;
type FxMapGraph = HashMap<usize, Vec<usize>, BuildHasherDefault<Fx>>;

// Written once, generic over the hasher, so both map rows run the same code.
fn bfs_map<H: std::hash::BuildHasher>(
    g: &HashMap<usize, Vec<usize>, H>,
    start: usize,
    visited: &mut [bool],
    queue: &mut Vec<usize>,
) -> usize {
    visited.fill(false);
    queue.clear();
    queue.push(start);
    visited[start] = true;
    let mut head = 0; // Go's `queue = queue[1:]`: move a cursor, never shift
    let mut seen = 0;

    while head < queue.len() {
        let v = queue[head];
        head += 1;
        seen += 1;
        for &w in &g[&v] {
            if !visited[w] {
                visited[w] = true;
                queue.push(w);
            }
        }
    }
    seen
}

// --- representation 2: vector of vectors ---

type SliceGraph = Vec<Vec<usize>>;

fn bfs_slice(g: &SliceGraph, start: usize, visited: &mut [bool], queue: &mut Vec<usize>) -> usize {
    visited.fill(false);
    queue.clear();
    queue.push(start);
    visited[start] = true;
    let mut head = 0;
    let mut seen = 0;

    while head < queue.len() {
        let v = queue[head];
        head += 1;
        seen += 1;
        for &w in &g[v] {
            if !visited[w] {
                visited[w] = true;
                queue.push(w);
            }
        }
    }
    seen
}

// --- representation 3: CSR ---
//
// offsets has V+1 entries; the neighbours of v are targets[offsets[v]..offsets[v+1]].
// Two allocations for the whole graph, both contiguous, neither holding a pointer.
struct Csr {
    offsets: Vec<u32>,
    targets: Vec<u32>,
}

impl Csr {
    fn from_adjacency(adj: &SliceGraph) -> Csr {
        let mut offsets = Vec::with_capacity(adj.len() + 1);
        let mut targets = Vec::with_capacity(adj.len() * DEGREE);
        for ns in adj {
            offsets.push(targets.len() as u32);
            targets.extend(ns.iter().map(|&w| w as u32));
        }
        offsets.push(targets.len() as u32);
        Csr { offsets, targets }
    }

    fn neighbours(&self, v: u32) -> &[u32] {
        let v = v as usize;
        &self.targets[self.offsets[v] as usize..self.offsets[v + 1] as usize]
    }

    fn bfs(&self, start: u32, visited: &mut [bool], queue: &mut Vec<u32>) -> usize {
        visited.fill(false);
        queue.clear();
        queue.push(start);
        visited[start as usize] = true;
        let mut head = 0;
        let mut seen = 0;

        while head < queue.len() {
            let v = queue[head];
            head += 1;
            seen += 1;
            for &w in self.neighbours(v) {
                if !visited[w as usize] {
                    visited[w as usize] = true;
                    queue.push(w);
                }
            }
        }
        seen
    }
}

fn random_adjacency(rng: &mut Rng) -> SliceGraph {
    (0..V)
        .map(|_| (0..DEGREE).map(|_| rng.below(V as u64) as usize).collect())
        .collect()
}

// measure_mem reports what the FINISHED structure costs: bytes and live
// allocations, taken after the build's scratch has been dropped. (Go needs a
// forced GC here, or CSR's temporary [][]int is counted as part of the result;
// in Rust the temporary is freed when the closure returns.)
fn measure_mem<T>(build: impl FnOnce() -> T) -> (f64, usize) {
    let (bytes0, objects0) = (LIVE_BYTES.load(Relaxed), LIVE_OBJECTS.load(Relaxed));
    let v = black_box(build());
    let mb = (LIVE_BYTES.load(Relaxed) - bytes0) as f64 / (1 << 20) as f64;
    let objects = LIVE_OBJECTS.load(Relaxed) - objects0;
    drop(v);
    (mb, objects)
}

fn main() {
    let mut rng = Rng::new(1);

    println!(
        "graph: {V} vertices, {} edges (out-degree {DEGREE})\n",
        V * DEGREE
    );

    // --- memory, each built in isolation ---
    let (map_mb, map_objs) = measure_mem(|| {
        let mut g = MapGraph::with_capacity(V);
        g.extend(random_adjacency(&mut rng).into_iter().enumerate());
        g
    });
    let (fx_mb, fx_objs) = measure_mem(|| {
        let mut g = FxMapGraph::with_capacity_and_hasher(V, Default::default());
        g.extend(random_adjacency(&mut rng).into_iter().enumerate());
        g
    });
    let (slice_mb, slice_objs) = measure_mem(|| random_adjacency(&mut rng));
    let (csr_mb, csr_objs) = measure_mem(|| Csr::from_adjacency(&random_adjacency(&mut rng)));

    // --- speed, all of them live at once, over ONE graph, so the comparison is like-for-like ---
    let sg = random_adjacency(&mut rng);
    let mut mg = MapGraph::with_capacity(V);
    mg.extend(sg.iter().cloned().enumerate());
    let mut fg = FxMapGraph::with_capacity_and_hasher(V, Default::default());
    fg.extend(sg.iter().cloned().enumerate());
    let csr = Csr::from_adjacency(&sg);

    let mut visited = vec![false; V];
    let mut queue_usize = Vec::with_capacity(V);
    let mut queue_u32 = Vec::with_capacity(V);

    let reached = bfs_slice(&sg, 0, &mut visited, &mut queue_usize);
    assert_eq!(bfs_map(&mg, 0, &mut visited, &mut queue_usize), reached);
    assert_eq!(bfs_map(&fg, 0, &mut visited, &mut queue_usize), reached);
    assert_eq!(csr.bfs(0, &mut visited, &mut queue_u32), reached);

    let t_map = best_ns_per_op(|| {
        black_box(bfs_map(black_box(&mg), 0, &mut visited, &mut queue_usize));
    });
    let t_fx = best_ns_per_op(|| {
        black_box(bfs_map(black_box(&fg), 0, &mut visited, &mut queue_usize));
    });
    let t_slice = best_ns_per_op(|| {
        black_box(bfs_slice(black_box(&sg), 0, &mut visited, &mut queue_usize));
    });
    let t_csr = best_ns_per_op(|| {
        black_box(black_box(&csr).bfs(0, &mut visited, &mut queue_u32));
    });

    println!(
        "{:<26} {:>10} {:>10} {:>14} {:>10}",
        "", "BFS ms", "vs CSR", "heap objects", "MB"
    );
    let rows = [
        ("HashMap<usize, Vec<usize>>", t_map, map_objs, map_mb),
        ("  same, fx hasher", t_fx, fx_objs, fx_mb),
        ("Vec<Vec<usize>>", t_slice, slice_objs, slice_mb),
        ("CSR", t_csr, csr_objs, csr_mb),
    ];
    for (name, t, objs, mb) in rows {
        println!(
            "{name:<26} {:>10.1} {:>9.1}x {objs:>14} {mb:>10.1}",
            t / 1e6,
            t / t_csr
        );
    }
    println!();
    println!("all four reach the same {reached} vertices from vertex 0.");

    println!();
    println!("every lesson in this file, in one table:");
    println!();
    println!("  HashMap<usize, Vec<usize>>");
    println!("                 a hash lookup per vertex visit (example 15), plus a separate");
    println!("                 Vec per vertex: 1M+ allocations, each freed one by one when the");
    println!("                 graph is dropped (example 12)");
    println!("  Vec<Vec<usize>>");
    println!("                 no hashing -- g[v] is an index -- but still one allocation per");
    println!("                 vertex, so still 1M objects and 24 bytes of Vec header each");
    println!("                 (examples 1, 10, 12)");
    println!("  CSR            TWO allocations for the entire graph. Neighbours of every");
    println!("                 vertex are contiguous (example 7), u32 instead of usize halves");
    println!("                 the edge array (example 6), and there is nothing to drop but");
    println!("                 two blocks (example 12)");
    println!();
    println!(
        "the hasher barely matters here, unlike example 15: the same map with the cheap hasher is only"
    );
    println!(
        "{:.2}x faster than SipHash and still {:.1}x behind CSR. A BFS visits vertices in random",
        t_map / t_fx,
        t_fx / t_csr
    );
    println!("order, so each lookup is a cold memory access whatever the hash costs (plausible");
    println!("mechanism, not measured). The fx map holds the same bytes in the same number of");
    println!("allocations as row 1, so the gap that remains is the extra table lookup per vertex.");
    println!();
    println!("what CSR gives up: you cannot add an edge without rebuilding. That is the");
    println!("trade -- it is a BUILD-ONCE, QUERY-MANY structure, which is exactly what a");
    println!("road network, a dependency graph or a sparse matrix is.");
    println!();
    println!("this is the payoff for the whole lesson: same Theta(V+E), same algorithm,");
    println!("and the difference between the first row and the last is layout; the hash is");
    println!("the small part of it.");
}
```

**Rust sample output:**

```
graph: 1000000 vertices, 8000000 edges (out-degree 8)

                               BFS ms     vs CSR   heap objects         MB
HashMap<usize, Vec<usize>>       72.0       2.4x        1000001      127.0
  same, fx hasher                67.8       2.3x        1000001      127.0
Vec<Vec<usize>>                  49.4       1.7x        1000001       83.9
CSR                              29.5       1.0x              2       34.3

all four reach the same 999645 vertices from vertex 0.

every lesson in this file, in one table:

  HashMap<usize, Vec<usize>>
                 a hash lookup per vertex visit (example 15), plus a separate
                 Vec per vertex: 1M+ allocations, each freed one by one when the
                 graph is dropped (example 12)
  Vec<Vec<usize>>
                 no hashing -- g[v] is an index -- but still one allocation per
                 vertex, so still 1M objects and 24 bytes of Vec header each
                 (examples 1, 10, 12)
  CSR            TWO allocations for the entire graph. Neighbours of every
                 vertex are contiguous (example 7), u32 instead of usize halves
                 the edge array (example 6), and there is nothing to drop but
                 two blocks (example 12)

the hasher barely matters here, unlike example 15: the same map with the cheap hasher is only
1.06x faster than SipHash and still 2.3x behind CSR. A BFS visits vertices in random
order, so each lookup is a cold memory access whatever the hash costs (plausible
mechanism, not measured). The fx map holds the same bytes in the same number of
allocations as row 1, so the gap that remains is the extra table lookup per vertex.

what CSR gives up: you cannot add an edge without rebuilding. That is the
trade -- it is a BUILD-ONCE, QUERY-MANY structure, which is exactly what a
road network, a dependency graph or a sparse matrix is.

this is the payoff for the whole lesson: same Theta(V+E), same algorithm,
and the difference between the first row and the last is layout; the hash is
the small part of it.
```

> **Go vs Rust:** This one ports almost number for number: BFS 71-81 / 48-51 / 27-35 ms and 2.0-3.0x / 1.4-1.9x against CSR over my runs (Go: 80.6 / 47.1 / 28.2 ms, 2.9x / 1.7x), 83.9 MB and 34.3 MB exactly as in Go, 127.0 MB for the map (Go: 141.1), and 1,000,001 / 1,000,001 / 2 live allocations (Go: 1,004,118 / 1,000,009 / 4, which include runtime objects). Rust needs no forced GC between build and measurement: the temporary adjacency list is freed when its scope ends, and the counting allocator reads the live total. Go's `queue = queue[1:]` becomes a head cursor into a `Vec`. The "fx hasher" row is an addition: std's SipHash costs only about 1.1x on this BFS (plausibly because each lookup is a cache miss anyway), where in example 15's hot-cache lookups the same choice was worth about 4x.

**Complexity:** all three are Θ(V+E) time and Θ(V+E) space · 2.9× time and 4.1× memory separate the first row from the last, and none of it appears in the complexity

---

> That's Part 1's memory chapter. Next: [05 — Recursion & the Call Stack](../../05-recursion.md),
> which closes the foundations — then Part 2 starts building structures with everything measured here.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
