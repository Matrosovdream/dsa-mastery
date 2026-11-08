# Step 10 — Hash Tables & Sets · 🟡 Medium

Examples **7–11**: sets and why `struct{}`, the counter idioms, what can legally be a key, what
preallocation actually buys, and four implementations measured head to head.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 8 and 9 are deterministic apart from their closing benchmarks; 7, 10 and 11 report memory
and timings, sampled on an Apple M4 with Go 1.26.3.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Sets, and why `map[T]struct{}` — for the right reason

`🟡 medium` · *the usual argument for it is folklore*

Go has no set type; it has `map[T]struct{}`. Everyone will tell you that saves a byte per entry over
`map[T]bool`. That is measurably false, and the real reason is much stronger.

**Steps:**

1. Measure five value types and find four of them identical.
2. Get the explanation out of `unsafe.Sizeof`.
3. Then meet the actual argument: `map[T]bool` has two ways to be absent.

```go
package main

import (
	"fmt"
	"maps"
	"runtime"
	"slices"
	"testing"
	"unsafe"
)

// Go has no set type. It has `map[T]struct{}`, and the reason that is enough --
// rather than a gap in the standard library -- is worth being precise about.
//
//	map[T]struct{}   a set. The value carries no information.
//	map[T]bool       a set with a footgun: two ways to say "absent".
//
// The usual argument for struct{} is that it saves a byte per entry. That
// argument is folklore, and the first measurement below is the disproof. The
// real argument is the second one, and it is much stronger.

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

// A Set[T] worth having is about thirty lines, and the operations are the
// interesting part -- not the storage.
type Set[T comparable] map[T]struct{}

func NewSet[T comparable](items ...T) Set[T] {
	s := make(Set[T], len(items))
	for _, v := range items {
		s[v] = struct{}{}
	}
	return s
}

func (s Set[T]) Add(v T)       { s[v] = struct{}{} }
func (s Set[T]) Delete(v T)    { delete(s, v) }
func (s Set[T]) Has(v T) bool  { _, ok := s[v]; return ok }
func (s Set[T]) Len() int      { return len(s) }
func (s Set[T]) Items() []T    { return slices.Collect(maps.Keys(s)) }
func (s Set[T]) Clone() Set[T] { return maps.Clone(s) }

// Union, Intersection and Difference are each three lines, and each is Theta(n)
// -- but only if you iterate the SMALLER set. Getting that backwards is the one
// performance bug in set code.
func (s Set[T]) Union(o Set[T]) Set[T] {
	out := make(Set[T], len(s)+len(o))
	for v := range s {
		out[v] = struct{}{}
	}
	for v := range o {
		out[v] = struct{}{}
	}
	return out
}

func (s Set[T]) Intersect(o Set[T]) Set[T] {
	if len(o) < len(s) {
		s, o = o, s // ALWAYS iterate the smaller one
	}
	out := make(Set[T])
	for v := range s {
		if _, ok := o[v]; ok {
			out[v] = struct{}{}
		}
	}
	return out
}

// IntersectNaive always iterates the receiver, so it is Theta(|s|) rather than
// Theta(min(|s|,|o|)).
func (s Set[T]) IntersectNaive(o Set[T]) Set[T] {
	out := make(Set[T])
	for v := range s {
		if _, ok := o[v]; ok {
			out[v] = struct{}{}
		}
	}
	return out
}

func (s Set[T]) Difference(o Set[T]) Set[T] {
	out := make(Set[T], len(s))
	for v := range s {
		if _, ok := o[v]; !ok {
			out[v] = struct{}{}
		}
	}
	return out
}

func (s Set[T]) IsSubsetOf(o Set[T]) bool {
	if len(s) > len(o) {
		return false // a cheap early exit that is also correct
	}
	for v := range s {
		if _, ok := o[v]; !ok {
			return false
		}
	}
	return true
}

func sorted[T interface{ ~int | ~string }](s Set[T]) []T {
	out := s.Items()
	slices.Sort(out)
	return out
}

func main() {
	fmt.Println("map[T]struct{} vs map[T]bool -- the memory question first,")
	fmt.Println("because the usual answer is wrong:")
	fmt.Println()
	const n = 1 << 19
	keys := make([]int, n)
	for i := range keys {
		keys[i] = i * 2654435761
	}
	fmt.Printf("  %-24s %11s %12s\n", "map type", "KB", "bytes/key")
	report := func(name string, kb float64) {
		fmt.Printf("  %-24s %11.0f %12.1f\n", name, kb, kb*1024/n)
	}
	report("map[int]struct{}", liveKB(func() any {
		m := make(map[int]struct{}, n)
		for _, k := range keys {
			m[k] = struct{}{}
		}
		return m
	}))
	report("map[int]bool", liveKB(func() any {
		m := make(map[int]bool, n)
		for _, k := range keys {
			m[k] = true
		}
		return m
	}))
	report("map[int]int8", liveKB(func() any {
		m := make(map[int]int8, n)
		for _, k := range keys {
			m[k] = 1
		}
		return m
	}))
	report("map[int]int", liveKB(func() any {
		m := make(map[int]int, n)
		for _, k := range keys {
			m[k] = 1
		}
		return m
	}))
	report("map[int][16]byte", liveKB(func() any {
		m := make(map[int][16]byte, n)
		for _, k := range keys {
			m[k] = [16]byte{}
		}
		return m
	}))
	fmt.Println()
	fmt.Println("  the first FOUR are identical. Not close -- identical. Swapping")
	fmt.Println("  struct{} for a full int saves nothing, and the [16]byte row")
	fmt.Println("  proves the measurement can see a difference when there is one.")
	fmt.Println()
	fmt.Println("  the reason is padding, and `unsafe.Sizeof` says it outright:")
	fmt.Println()
	type kvBool struct {
		k int
		v bool
	}
	type kvEmpty struct {
		k int
		v struct{}
	}
	type kvInt struct {
		k int
		v int
	}
	fmt.Printf("    struct{ k int; v struct{} }  %2d bytes\n", unsafe.Sizeof(kvEmpty{}))
	fmt.Printf("    struct{ k int; v bool }      %2d bytes\n", unsafe.Sizeof(kvBool{}))
	fmt.Printf("    struct{ k int; v int }       %2d bytes\n", unsafe.Sizeof(kvInt{}))
	fmt.Println()
	fmt.Println("  a zero-sized field at the END of a struct is padded, because")
	fmt.Println("  otherwise taking its address would produce a pointer just past")
	fmt.Println("  the allocation. So an empty struct costs a full alignment unit")
	fmt.Println("  here, exactly like a bool does -- and exactly like an int does.")
	fmt.Println()
	fmt.Println("  I wrote 'struct{} occupies no space' into this file before")
	fmt.Println("  measuring it. The type occupies no space; the SLOT does.")

	fmt.Println()
	fmt.Println("so the real argument for struct{} is not memory at all -- it is")
	fmt.Println("that map[T]bool has TWO ways to be absent:")
	fmt.Println()
	b := map[string]bool{"yes": true, "no": false}
	fmt.Printf("  b := map[string]bool{\"yes\": true, \"no\": false}\n")
	fmt.Println()
	for _, k := range []string{"yes", "no", "missing"} {
		v, ok := b[k]
		fmt.Printf("  %-10s b[k]=%-6v  comma-ok=%-6v  in set by `if b[k]`? %v\n", k, v, ok, b[k])
	}
	fmt.Println()
	fmt.Println("  \"no\" IS in the map and `if b[k]` says it is not. Every set")
	fmt.Println("  operation written the obvious way is now wrong for that key, and")
	fmt.Println("  the type system will never mention it.")
	fmt.Println()
	fmt.Println("  with map[T]struct{} the only question you CAN ask is comma-ok,")
	fmt.Println("  so the bug is unrepresentable. That is the whole reason to prefer")
	fmt.Println("  it, and it is a better reason than the one usually given.")
	fmt.Println()
	fmt.Println("  (map[T]bool is the right choice when you genuinely need three")
	fmt.Println("  states: present-and-true, present-and-false, absent. A feature-")
	fmt.Println("  flag override table, for instance.)")

	fmt.Println()
	fmt.Println("the operations, on small sets so you can check them by eye:")
	fmt.Println()
	a := NewSet(1, 2, 3, 4, 5)
	c := NewSet(4, 5, 6, 7)
	fmt.Printf("  %-16s %v\n", "a", sorted(a))
	fmt.Printf("  %-16s %v\n", "b", sorted(c))
	fmt.Printf("  %-16s %v\n", "a union b", sorted(a.Union(c)))
	fmt.Printf("  %-16s %v\n", "a intersect b", sorted(a.Intersect(c)))
	fmt.Printf("  %-16s %v\n", "a minus b", sorted(a.Difference(c)))
	fmt.Printf("  %-16s %v\n", "b minus a", sorted(c.Difference(a)))
	fmt.Printf("  %-16s %v\n", "{4,5} subset a?", NewSet(4, 5).IsSubsetOf(a))
	fmt.Printf("  %-16s %v\n", "{4,9} subset a?", NewSet(4, 9).IsSubsetOf(a))

	fmt.Println()
	fmt.Println("intersection must iterate the SMALLER set. Why it matters:")
	fmt.Println()
	big := make(Set[int], 1_000_000)
	for i := 0; i < 1_000_000; i++ {
		big.Add(i)
	}
	small := NewSet(1, 2, 3, 500_000, 999_999)
	var res Set[int]
	tSmart := nsPerOp(func() { res = big.Intersect(small) })
	tNaive := nsPerOp(func() { res = big.IntersectNaive(small) })
	fmt.Printf("  %-40s %14.0f ns\n", "iterate the smaller (5 elements)", tSmart)
	fmt.Printf("  %-40s %14.0f ns   %.0fx slower\n",
		"iterate the receiver (1,000,000)", tNaive, tNaive/tSmart)
	fmt.Printf("  both return %d elements\n", len(res))
	fmt.Println()
	fmt.Println("  Theta(min(|a|,|b|)) against Theta(|a|). The two functions differ")
	fmt.Println("  by one `if` and one swap, and they are in different complexity")
	fmt.Println("  classes. This is the single most common inefficiency in set code.")

	fmt.Println()
	fmt.Println("a set is not always the answer. Against a sorted slice:")
	fmt.Println()
	fmt.Printf("  %10s %16s %16s %12s\n", "n", "set build ns", "sort+dedup ns", "set/slice")
	for _, sz := range []int{8, 64, 1024, 100_000} {
		src := make([]int, sz)
		for i := range src {
			src[i] = (i * 7919) % (sz * 2)
		}
		tSet := nsPerOp(func() {
			s := make(Set[int], len(src))
			for _, v := range src {
				s.Add(v)
			}
			sinkI = s.Len()
		})
		tSlice := nsPerOp(func() {
			cp := slices.Clone(src)
			slices.Sort(cp)
			cp = slices.Compact(cp)
			sinkI = len(cp)
		})
		fmt.Printf("  %10d %16.0f %16.0f %11.2fx\n", sz, tSet, tSlice, tSet/tSlice)
	}
	fmt.Println()
	fmt.Println("  sort-and-compact wins at small n and loses at large n, which is")
	fmt.Println("  the same crossover story as example 1 -- and it gives you sorted")
	fmt.Println("  output, contiguous memory and no hashing at all.")
	fmt.Println()
	fmt.Println("  use a slice when the set is small, built once and read many")
	fmt.Println("  times. Use a map when it is large, or when membership is tested")
	fmt.Println("  as often as it is built, or when elements come and go.")

	fmt.Println()
	fmt.Println("the stdlib helpers, which people still write by hand:")
	fmt.Println()
	src := NewSet("a", "b", "c")
	cl := src.Clone()
	cl.Add("d")
	fmt.Printf("  maps.Clone       %v -> %v\n", sorted(src), sorted(cl))
	fmt.Printf("  maps.Keys        %v\n", sorted(src))
	fmt.Printf("  maps.Equal       %v\n", maps.Equal(src, NewSet("a", "b", "c")))
	fmt.Printf("  slices.Collect   %v\n", len(slices.Collect(maps.Keys(src))))
	fmt.Println()
	fmt.Println("  maps.Clone is a SHALLOW copy -- if T is a pointer, both sets")
	fmt.Println("  point at the same things. For a Set[T comparable] that is")
	fmt.Println("  usually what you want, because comparable mostly means small.")
}
```

**Sample output:**

```
map[T]struct{} vs map[T]bool -- the memory question first,
because the usual answer is wrong:

  map type                          KB    bytes/key
  map[int]struct{}               18473         36.1
  map[int]bool                   18479         36.1
  map[int]int8                   18473         36.1
  map[int]int                    18473         36.1
  map[int][16]byte               23209         45.3

  the first FOUR are identical. Not close -- identical. Swapping
  struct{} for a full int saves nothing, and the [16]byte row
  proves the measurement can see a difference when there is one.

  the reason is padding, and `unsafe.Sizeof` says it outright:

    struct{ k int; v struct{} }  16 bytes
    struct{ k int; v bool }      16 bytes
    struct{ k int; v int }       16 bytes

  a zero-sized field at the END of a struct is padded, because
  otherwise taking its address would produce a pointer just past
  the allocation. So an empty struct costs a full alignment unit
  here, exactly like a bool does -- and exactly like an int does.

  I wrote 'struct{} occupies no space' into this file before
  measuring it. The type occupies no space; the SLOT does.

so the real argument for struct{} is not memory at all -- it is
that map[T]bool has TWO ways to be absent:

  b := map[string]bool{"yes": true, "no": false}

  yes        b[k]=true    comma-ok=true    in set by `if b[k]`? true
  no         b[k]=false   comma-ok=true    in set by `if b[k]`? false
  missing    b[k]=false   comma-ok=false   in set by `if b[k]`? false

  "no" IS in the map and `if b[k]` says it is not. Every set
  operation written the obvious way is now wrong for that key, and
  the type system will never mention it.

  with map[T]struct{} the only question you CAN ask is comma-ok,
  so the bug is unrepresentable. That is the whole reason to prefer
  it, and it is a better reason than the one usually given.

  (map[T]bool is the right choice when you genuinely need three
  states: present-and-true, present-and-false, absent. A feature-
  flag override table, for instance.)

the operations, on small sets so you can check them by eye:

  a                [1 2 3 4 5]
  b                [4 5 6 7]
  a union b        [1 2 3 4 5 6 7]
  a intersect b    [4 5]
  a minus b        [1 2 3]
  b minus a        [6 7]
  {4,5} subset a?  true
  {4,9} subset a?  false

intersection must iterate the SMALLER set. Why it matters:

  iterate the smaller (5 elements)                    109 ns
  iterate the receiver (1,000,000)                8678279 ns   79796x slower
  both return 5 elements

  Theta(min(|a|,|b|)) against Theta(|a|). The two functions differ
  by one `if` and one swap, and they are in different complexity
  classes. This is the single most common inefficiency in set code.

a set is not always the answer. Against a sorted slice:

           n     set build ns    sort+dedup ns    set/slice
           8               44               31        1.40x
          64              519              322        1.61x
        1024             7272             9409        0.77x
      100000          1025166          4215741        0.24x

  sort-and-compact wins at small n and loses at large n, which is
  the same crossover story as example 1 -- and it gives you sorted
  output, contiguous memory and no hashing at all.

  use a slice when the set is small, built once and read many
  times. Use a map when it is large, or when membership is tested
  as often as it is built, or when elements come and go.

the stdlib helpers, which people still write by hand:

  maps.Clone       [a b c] -> [a b c d]
  maps.Keys        [a b c]
  maps.Equal       true
  slices.Collect   3

  maps.Clone is a SHALLOW copy -- if T is a pointer, both sets
  point at the same things. For a Set[T comparable] that is
  usually what you want, because comparable mostly means small.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{HashMap, HashSet};
use std::fmt::Display;
use std::hash::{BuildHasherDefault, Hash, Hasher};
use std::hint::black_box;
use std::mem::{align_of, size_of};
use std::panic::{self, AssertUnwindSafe};
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Rust has a set type: HashSet<T>, which is literally HashMap<T, ()>. Go has
// `map[T]struct{}`, and the question this example asks is the one Go's version
// of the argument raises -- what does the empty value actually buy?
//
//	HashSet<T>        a set. The value type is `()`, which carries no information.
//	HashMap<T, bool>  a set with a footgun: two ways to say "absent".
//
// The usual argument for struct{} is that it saves a byte per entry. In Go that
// argument is folklore (a trailing zero-sized field is padded). Whether it is
// folklore in Rust is a measurement, and the first one below is that measurement.

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

// ---- a second hasher, so the timing comparisons are fair ----
//
// std's default is SipHash-1-3 with a random key per map: built to resist
// collision attacks (example 14), and not built to be the fastest hash there
// is. `Fast` is the same shape as example 2's finalized FNV-1a: FNV-1a over the
// bytes of a string, ONE xor-multiply round over the whole word for an integer,
// then the splitmix64 finalizer (bit-identical to Go's example 2). It has no
// random seed, so it is exactly the hash an attacker could aim at.
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
type FastSet<T> = HashSet<T, BuildHasherDefault<Fast>>;

// A Set<T> worth having is about thirty lines, and the operations are the
// interesting part -- not the storage. (std already has lazy `union`,
// `intersection`, `difference` and `is_subset` on HashSet; these are the eager,
// owned versions, written out so the cost of each one is visible. `delete` is
// `HashSet::remove` and is left out because main never calls it.)
#[derive(Clone, PartialEq)]
struct Set<T: Eq + Hash>(HashSet<T>);

impl<T: Eq + Hash + Clone> Set<T> {
    fn new(items: impl IntoIterator<Item = T>) -> Self {
        Set(items.into_iter().collect())
    }
    fn with_capacity(n: usize) -> Self {
        Set(HashSet::with_capacity(n))
    }
    fn add(&mut self, v: T) {
        self.0.insert(v);
    }
    fn len(&self) -> usize {
        self.0.len()
    }

    // Union, Intersection and Difference are each a few lines, and each is
    // Theta(n) -- but only if you iterate the SMALLER set. Getting that
    // backwards is the one performance bug in set code.
    fn union(&self, o: &Set<T>) -> Set<T> {
        let mut out = HashSet::with_capacity(self.len() + o.len());
        out.extend(self.0.iter().cloned());
        out.extend(o.0.iter().cloned());
        Set(out)
    }

    fn intersect(&self, o: &Set<T>) -> Set<T> {
        // ALWAYS iterate the smaller one: a swap of two references, where Go
        // swaps two map headers.
        let (small, large) = if o.len() < self.len() {
            (o, self)
        } else {
            (self, o)
        };
        Set(small
            .0
            .iter()
            .filter(|v| large.0.contains(*v))
            .cloned()
            .collect())
    }

    // intersect_naive always iterates the receiver, so it is Theta(|s|) rather
    // than Theta(min(|s|,|o|)).
    fn intersect_naive(&self, o: &Set<T>) -> Set<T> {
        Set(self
            .0
            .iter()
            .filter(|v| o.0.contains(*v))
            .cloned()
            .collect())
    }

    fn difference(&self, o: &Set<T>) -> Set<T> {
        Set(self
            .0
            .iter()
            .filter(|v| !o.0.contains(*v))
            .cloned()
            .collect())
    }

    fn is_subset_of(&self, o: &Set<T>) -> bool {
        if self.len() > o.len() {
            return false; // a cheap early exit that is also correct
        }
        self.0.iter().all(|v| o.0.contains(v))
    }
}

/// the items in order, which a HashSet cannot give you by itself
fn sorted<T: Ord + Hash + Clone>(s: &Set<T>) -> Vec<T> {
    let mut out: Vec<T> = s.0.iter().cloned().collect();
    out.sort_unstable();
    out
}

/// `[1 2 3]`, the way Go's %v prints a slice ({:?} would give `[1, 2, 3]`)
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// A HashMap<i64, V> holding every key, built with a size hint (as in Go).
fn filled<V: Copy>(keys: &[i64], v: V) -> HashMap<i64, V> {
    let mut m = HashMap::with_capacity(keys.len());
    for &k in keys {
        m.insert(k, v);
    }
    m
}

fn main() {
    println!("HashSet<T> vs HashMap<T, bool> -- the memory question first,");
    println!("because Go's answer is 'no difference' and Rust's may not be:");
    println!();
    const N: usize = 1 << 19;
    let keys: Vec<i64> = (0..N as i64).map(|i| i * 2654435761).collect();
    println!(
        "  {:<24} {:>11} {:>12} {:>9}",
        "map type", "KB", "bytes/key", "(K,V) B"
    );
    // (name, bytes of one (K,V) slot, KB) -- kept for the commentary below
    let mut rows: Vec<(&str, usize, f64)> = Vec::new();
    let mut report = |name: &'static str, slot: usize, kb: f64| {
        println!(
            "  {:<24} {:>11.0} {:>12.1} {:>9}",
            name,
            kb,
            kb * 1024.0 / N as f64,
            slot
        );
        rows.push((name, slot, kb));
    };
    report(
        "HashSet<i64>",
        size_of::<(i64, ())>(),
        live_kb(|| {
            let mut m = HashSet::with_capacity(N);
            for &k in &keys {
                m.insert(k);
            }
            m
        }),
    );
    report(
        "HashMap<i64, bool>",
        size_of::<(i64, bool)>(),
        live_kb(|| filled(&keys, true)),
    );
    report(
        "HashMap<i64, i8>",
        size_of::<(i64, i8)>(),
        live_kb(|| filled(&keys, 1i8)),
    );
    report(
        "HashMap<i64, i64>",
        size_of::<(i64, i64)>(),
        live_kb(|| filled(&keys, 1i64)),
    );
    report(
        "HashMap<i64, [u8; 16]>",
        size_of::<(i64, [u8; 16])>(),
        live_kb(|| filled(&keys, [0u8; 16])),
    );
    // the claims below are asserted, so the program cannot print them if false
    assert!(rows[1].2 == rows[2].2 && rows[2].2 == rows[3].2);
    assert!(rows[0].2 < rows[1].2 && rows[3].2 < rows[4].2);
    let slots = rows[0].2 * 1024.0 / (rows[0].1 + 1) as f64;
    assert!(
        rows.iter()
            .all(|r| (r.2 * 1024.0 / (r.1 + 1) as f64 - slots).abs() < 1.0)
    );
    println!();
    println!(
        "  the middle THREE are identical (bool, i8, i64: {:.0} KB each). The set is not:",
        rows[1].2
    );
    println!(
        "  it is {:.2}x the size of the bool map, because its slot is {} bytes where theirs",
        rows[0].2 / rows[1].2,
        rows[0].1
    );
    println!(
        "  is {}. The [u8; 16] row ({} B slot) shows the measurement can see a change in",
        rows[1].1, rows[4].1
    );
    println!("  either direction. Every slot also carries ONE control byte, and every row has");
    println!(
        "  the same number of slots: KB / (slot + 1 control byte) = {:.0} slots for {N} keys.",
        slots
    );
    println!("  That is 2x the keys, and it is the hint's doing: hashbrown keeps its table at");
    println!("  most 7/8 full, so 2^19 keys need at least 2^19 * 8/7 slots, which rounds up to");
    println!("  the next power of two, 2^20. (Go's 36.1 bytes/key is the same kind of number.)");
    println!();
    println!("  so in Rust the folklore is TRUE, and `size_of` says why:");
    println!();
    println!(
        "    (i64, ())       {:2} bytes   align {}",
        size_of::<(i64, ())>(),
        align_of::<(i64, ())>()
    );
    println!(
        "    (i64, bool)     {:2} bytes   align {}",
        size_of::<(i64, bool)>(),
        align_of::<(i64, bool)>()
    );
    println!(
        "    (i64, i64)      {:2} bytes   align {}",
        size_of::<(i64, i64)>(),
        align_of::<(i64, i64)>()
    );
    println!(
        "    ()              {:2} bytes   align {}",
        size_of::<()>(),
        align_of::<()>()
    );
    println!();
    println!("  a struct's size is its fields' sizes rounded up to its alignment. `()` is");
    println!("  0 bytes with alignment 1, so it adds nothing and forces no padding, whereas a");
    println!("  bool is 1 byte that drags 7 bytes of padding in behind it. Go pads a trailing");
    println!("  zero-sized field (its stated reason: a pointer to it must not point past the");
    println!("  allocation) and Rust's layout does not -- this output is the whole evidence.");

    println!();
    println!("so the second argument for a set type is not memory -- it is that a");
    println!("map to bool has THREE answers, and only some spellings show them:");
    println!();
    let b: HashMap<&str, bool> = HashMap::from([("yes", true), ("no", false)]);
    println!("  let b = HashMap::from([(\"yes\", true), (\"no\", false)]);");
    println!();
    println!("  (the last column is `if b[k]` as Go spells it: absent turns into false)");
    println!();
    // `b[k]` on a missing key panics. Catch it (quietly) so the table can show
    // it; an unwinding panic is an ordinary exit, not a crash report.
    let quiet = panic::take_hook();
    panic::set_hook(Box::new(|_| {}));
    for k in ["yes", "no", "missing"] {
        let get = format!("{:?}", b.get(k));
        let idx = match panic::catch_unwind(AssertUnwindSafe(|| b[k])) {
            Ok(v) => v.to_string(),
            Err(_) => "panics".to_string(),
        };
        println!(
            "  {:<10} b.get(k)={:<12}  b[k]={:<7}  .copied().unwrap_or(false)={}",
            k,
            get,
            idx,
            b.get(k).copied().unwrap_or(false)
        );
    }
    panic::set_hook(quiet);
    println!();
    println!("  \"no\" IS in the map and `unwrap_or(false)` says it is not. That spelling is");
    println!(
        "  the Go bug, and Rust lets you write it: it is also what `*b.get(k).unwrap_or(&false)`"
    );
    println!("  does. What Rust adds is that the other two spellings are honest: `get` hands");
    println!("  back an Option<&bool>, so absent is a different VALUE from false, and `b[k]`");
    println!("  does not return a default -- it panics. Go's `m[k]` silently returns false.");
    println!();
    println!("  with HashSet<T> the only question you CAN ask is `contains`, so the bug is");
    println!("  unrepresentable, as with map[T]struct{{}}. The reason to prefer it is the same");
    println!("  in both languages; Rust's Option just makes the bool map less dangerous.");
    println!();
    println!("  (HashMap<T, bool> is still the right choice when you genuinely need three");
    println!("  states: present-and-true, present-and-false, absent. A feature-flag");
    println!("  override table, for instance -- and match on the Option to say so.)");

    println!();
    println!("the operations, on small sets so you can check them by eye:");
    println!();
    let a = Set::new([1, 2, 3, 4, 5]);
    let c = Set::new([4, 5, 6, 7]);
    println!("  {:<16} {}", "a", show(&sorted(&a)));
    println!("  {:<16} {}", "b", show(&sorted(&c)));
    println!("  {:<16} {}", "a union b", show(&sorted(&a.union(&c))));
    println!(
        "  {:<16} {}",
        "a intersect b",
        show(&sorted(&a.intersect(&c)))
    );
    println!("  {:<16} {}", "a minus b", show(&sorted(&a.difference(&c))));
    println!("  {:<16} {}", "b minus a", show(&sorted(&c.difference(&a))));
    println!(
        "  {:<16} {}",
        "{4,5} subset a?",
        Set::new([4, 5]).is_subset_of(&a)
    );
    println!(
        "  {:<16} {}",
        "{4,9} subset a?",
        Set::new([4, 9]).is_subset_of(&a)
    );

    println!();
    println!("intersection must iterate the SMALLER set. Why it matters:");
    println!();
    let mut big = Set::with_capacity(1_000_000);
    for i in 0..1_000_000i64 {
        big.add(i);
    }
    let small = Set::new([1, 2, 3, 500_000, 999_999]);
    let res = big.intersect(&small);
    let t_smart = ns_per_op(|| {
        black_box(black_box(&big).intersect(black_box(&small)));
    });
    let t_naive = ns_per_op(|| {
        black_box(black_box(&big).intersect_naive(black_box(&small)));
    });
    println!(
        "  {:<40} {:>14.0} ns",
        "iterate the smaller (5 elements)", t_smart
    );
    println!(
        "  {:<40} {:>14.0} ns   {:.0}x slower",
        "iterate the receiver (1,000,000)",
        t_naive,
        t_naive / t_smart
    );
    // std's own lazy `intersection`, collected into a new set so that it builds
    // the same result ours does: does it already pick the smaller side?
    let t_std_bs = ns_per_op(|| {
        let r: HashSet<i64> = black_box(&big.0)
            .intersection(black_box(&small.0))
            .copied()
            .collect();
        black_box(r);
    });
    let t_std_sb = ns_per_op(|| {
        let r: HashSet<i64> = black_box(&small.0)
            .intersection(black_box(&big.0))
            .copied()
            .collect();
        black_box(r);
    });
    println!(
        "  {:<40} {:>14.0} ns",
        "std intersection(big, small)", t_std_bs
    );
    println!(
        "  {:<40} {:>14.0} ns",
        "std intersection(small, big)", t_std_sb
    );
    assert_eq!(res.len(), big.intersect_naive(&small).len());
    println!("  both return {} elements", res.len());
    println!();
    println!("  Theta(min(|a|,|b|)) against Theta(|a|). The two functions differ");
    println!("  by one `if` and one swap, and they are in different complexity");
    println!("  classes. This is the single most common inefficiency in set code.");
    println!("  std's HashSet::intersection is the cheap kind for BOTH argument orders");
    println!("  (the last two rows): it does the swap for you. Only code that loops");
    println!("  over one set and probes the other by hand has to remember it.");

    println!();
    println!("a set is not always the answer. Against a sorted slice, with std's default");
    println!("SipHash-1-3 hasher and with the `Fast` one defined above:");
    println!();
    println!(
        "  {:>10} {:>14} {:>14} {:>14} {:>10} {:>10}",
        "n", "set sip ns", "set fast ns", "sort+dedup ns", "sip/slice", "fast/slice"
    );
    // (n, sip, fast, slice) per row
    let mut table: Vec<(usize, f64, f64, f64)> = Vec::new();
    for sz in [8usize, 64, 1024, 100_000] {
        let src: Vec<i64> = (0..sz as i64)
            .map(|i| (i * 7919) % (sz as i64 * 2))
            .collect();
        let t_sip = ns_per_op(|| {
            let mut s = HashSet::with_capacity(src.len());
            for &v in black_box(&src) {
                s.insert(v);
            }
            black_box(s.len());
        });
        let t_fast = ns_per_op(|| {
            let mut s = FastSet::with_capacity_and_hasher(src.len(), Default::default());
            for &v in black_box(&src) {
                s.insert(v);
            }
            black_box(s.len());
        });
        let t_slice = ns_per_op(|| {
            let mut cp = black_box(&src).clone();
            cp.sort_unstable();
            cp.dedup();
            black_box(cp.len());
        });
        println!(
            "  {:>10} {:>14.0} {:>14.0} {:>14.0} {:>9.2}x {:>9.2}x",
            sz,
            t_sip,
            t_fast,
            t_slice,
            t_sip / t_slice,
            t_fast / t_slice
        );
        table.push((sz, t_sip, t_fast, t_slice));
    }
    let wins = |pick: fn(&(usize, f64, f64, f64)) -> f64| -> Vec<usize> {
        table
            .iter()
            .filter(|r| r.3 < pick(r))
            .map(|r| r.0)
            .collect()
    };
    let list = |ns: &[usize]| -> String {
        if ns.is_empty() {
            "no row".to_string()
        } else {
            format!("n in {}", show(ns))
        }
    };
    let (w_sip, w_fast) = (wins(|r| r.1), wins(|r| r.2));
    println!();
    println!(
        "  sort-and-dedup beats the SipHash set at:   {}",
        list(&w_sip)
    );
    println!(
        "  sort-and-dedup beats the Fast-hash set at: {}",
        list(&w_fast)
    );
    let (lo, hi) = table
        .iter()
        .map(|r| r.1 / r.2)
        .fold((f64::INFINITY, 0.0f64), |(lo, hi), x| {
            (lo.min(x), hi.max(x))
        });
    println!();
    println!("  the SipHash set costs {lo:.1}x to {hi:.1}x the Fast-hash set on these rows: the");
    if w_sip == w_fast {
        println!("  two hashers happen to give the same crossover in this run.");
    } else {
        println!("  hasher alone moves the crossover, which is the same story as example 1 (what");
        println!("  the hash costs decides how long a scan or a sort stays ahead).");
    }
    println!("  The slice also gives you sorted output, contiguous memory and no hashing at all.");
    println!();
    println!("  use a slice when the set is small, built once and read many");
    println!("  times. Use a map when it is large, or when membership is tested");
    println!("  as often as it is built, or when elements come and go.");

    println!();
    println!("the stdlib helpers, which people still write by hand:");
    println!();
    let src = Set::new(["a", "b", "c"]);
    let mut cl = src.clone();
    cl.add("d");
    println!(
        "  clone()          {} -> {}",
        show(&sorted(&src)),
        show(&sorted(&cl))
    );
    println!("  iter()           {}", show(&sorted(&src)));
    println!("  ==               {}", src == Set::new(["a", "b", "c"]));
    println!(
        "  collect().len()  {}",
        src.0.iter().collect::<Vec<_>>().len()
    );
    println!();
    // Go's maps.Clone is SHALLOW. Rust's Clone is whatever T::clone is: deep for
    // String, a shared pointer for Rc. Look at where the element actually lives.
    let owned = Set::new(["a", "b", "c"].map(String::from));
    let owned_cl = owned.clone();
    let shared = Set::new(["a", "b", "c"].map(Rc::<str>::from));
    let shared_cl = shared.clone();
    let (o1, o2) = (owned.0.get("a").unwrap(), owned_cl.0.get("a").unwrap());
    let (s1, s2) = (shared.0.get("a").unwrap(), shared_cl.0.get("a").unwrap());
    println!(
        "  Set<String>  clone: element \"a\" lives at the same address in both? {}",
        o1.as_ptr() == o2.as_ptr()
    );
    println!(
        "  Set<Rc<str>> clone: element \"a\" lives at the same address in both? {} (strong count {})",
        Rc::ptr_eq(s1, s2),
        Rc::strong_count(s1)
    );
    println!();
    println!("  there is no 'shallow' or 'deep' switch: clone() calls T::clone() on every");
    println!("  element, so a Set<String> copies every string and a Set<Rc<_>> (or a set of");
    println!("  references) shares them -- Go's \"if T is a pointer, both sets point at the");
    println!("  same things\", except that Rust lets T choose.");
}
```

**Rust sample output:**

```
HashSet<T> vs HashMap<T, bool> -- the memory question first,
because Go's answer is 'no difference' and Rust's may not be:

  map type                          KB    bytes/key   (K,V) B
  HashSet<i64>                    9216         18.0         8
  HashMap<i64, bool>             17408         34.0        16
  HashMap<i64, i8>               17408         34.0        16
  HashMap<i64, i64>              17408         34.0        16
  HashMap<i64, [u8; 16]>         25600         50.0        24

  the middle THREE are identical (bool, i8, i64: 17408 KB each). The set is not:
  it is 0.53x the size of the bool map, because its slot is 8 bytes where theirs
  is 16. The [u8; 16] row (24 B slot) shows the measurement can see a change in
  either direction. Every slot also carries ONE control byte, and every row has
  the same number of slots: KB / (slot + 1 control byte) = 1048577 slots for 524288 keys.
  That is 2x the keys, and it is the hint's doing: hashbrown keeps its table at
  most 7/8 full, so 2^19 keys need at least 2^19 * 8/7 slots, which rounds up to
  the next power of two, 2^20. (Go's 36.1 bytes/key is the same kind of number.)

  so in Rust the folklore is TRUE, and `size_of` says why:

    (i64, ())        8 bytes   align 8
    (i64, bool)     16 bytes   align 8
    (i64, i64)      16 bytes   align 8
    ()               0 bytes   align 1

  a struct's size is its fields' sizes rounded up to its alignment. `()` is
  0 bytes with alignment 1, so it adds nothing and forces no padding, whereas a
  bool is 1 byte that drags 7 bytes of padding in behind it. Go pads a trailing
  zero-sized field (its stated reason: a pointer to it must not point past the
  allocation) and Rust's layout does not -- this output is the whole evidence.

so the second argument for a set type is not memory -- it is that a
map to bool has THREE answers, and only some spellings show them:

  let b = HashMap::from([("yes", true), ("no", false)]);

  (the last column is `if b[k]` as Go spells it: absent turns into false)

  yes        b.get(k)=Some(true)    b[k]=true     .copied().unwrap_or(false)=true
  no         b.get(k)=Some(false)   b[k]=false    .copied().unwrap_or(false)=false
  missing    b.get(k)=None          b[k]=panics   .copied().unwrap_or(false)=false

  "no" IS in the map and `unwrap_or(false)` says it is not. That spelling is
  the Go bug, and Rust lets you write it: it is also what `*b.get(k).unwrap_or(&false)`
  does. What Rust adds is that the other two spellings are honest: `get` hands
  back an Option<&bool>, so absent is a different VALUE from false, and `b[k]`
  does not return a default -- it panics. Go's `m[k]` silently returns false.

  with HashSet<T> the only question you CAN ask is `contains`, so the bug is
  unrepresentable, as with map[T]struct{}. The reason to prefer it is the same
  in both languages; Rust's Option just makes the bool map less dangerous.

  (HashMap<T, bool> is still the right choice when you genuinely need three
  states: present-and-true, present-and-false, absent. A feature-flag
  override table, for instance -- and match on the Option to say so.)

the operations, on small sets so you can check them by eye:

  a                [1 2 3 4 5]
  b                [4 5 6 7]
  a union b        [1 2 3 4 5 6 7]
  a intersect b    [4 5]
  a minus b        [1 2 3]
  b minus a        [6 7]
  {4,5} subset a?  true
  {4,9} subset a?  false

intersection must iterate the SMALLER set. Why it matters:

  iterate the smaller (5 elements)                    200 ns
  iterate the receiver (1,000,000)                8872161 ns   44256x slower
  std intersection(big, small)                        209 ns
  std intersection(small, big)                        272 ns
  both return 5 elements

  Theta(min(|a|,|b|)) against Theta(|a|). The two functions differ
  by one `if` and one swap, and they are in different complexity
  classes. This is the single most common inefficiency in set code.
  std's HashSet::intersection is the cheap kind for BOTH argument orders
  (the last two rows): it does the swap for you. Only code that loops
  over one set and probes the other by hand has to remember it.

a set is not always the answer. Against a sorted slice, with std's default
SipHash-1-3 hasher and with the `Fast` one defined above:

           n     set sip ns    set fast ns  sort+dedup ns  sip/slice fast/slice
           8            121             67             32      3.83x      2.11x
          64            625            349            206      3.03x      1.69x
        1024           9934           2932           4736      2.10x      0.62x
      100000        1445378         378187         774485      1.87x      0.49x

  sort-and-dedup beats the SipHash set at:   n in [8 64 1024 100000]
  sort-and-dedup beats the Fast-hash set at: n in [8 64]

  the SipHash set costs 1.8x to 3.8x the Fast-hash set on these rows: the
  hasher alone moves the crossover, which is the same story as example 1 (what
  the hash costs decides how long a scan or a sort stays ahead).
  The slice also gives you sorted output, contiguous memory and no hashing at all.

  use a slice when the set is small, built once and read many
  times. Use a map when it is large, or when membership is tested
  as often as it is built, or when elements come and go.

the stdlib helpers, which people still write by hand:

  clone()          [a b c] -> [a b c d]
  iter()           [a b c]
  ==               true
  collect().len()  3

  Set<String>  clone: element "a" lives at the same address in both? false
  Set<Rc<str>> clone: element "a" lives at the same address in both? true (strong count 2)

  there is no 'shallow' or 'deep' switch: clone() calls T::clone() on every
  element, so a Set<String> copies every string and a Set<Rc<_>> (or a set of
  references) shares them -- Go's "if T is a pointer, both sets point at the
  same things", except that Rust lets T choose.
```

> **Go vs Rust:** Rust contradicts Go's headline here: `HashSet<i64>` is 18.0 bytes/key against 34.0 for `HashMap<i64, bool>`, `<i64, i8>` and `<i64, i64>` (those three tie, as in Go), because a `(i64, ())` slot is 8 bytes and `(i64, bool)` is 16 -- Rust does not pad a trailing zero-sized field, so for Rust the "saves a byte" folklore is true (memory is exact via the counting allocator, no GC readings). The "two ways to be absent" trap needs more effort in Rust: `b[k]` panics on a missing key and `get` returns an `Option`, so Go's bug takes an explicit `unwrap_or(false)`. For timings (std's default SipHash-1-3 with a random per-set key against a `Fast` finalized-FNV `BuildHasher`; Go's built-in map uses its own runtime hash), over seven runs the SipHash set loses to sort+dedup at n=8, 64 and 1,024 and is level with it at 100,000 (0.9-1.7x slower), while the `Fast` set crosses over between 64 and 1,024 as Go's does (0.26-0.71x at n >= 1,024); the naive intersection is 41,000-48,000x slower (Go: 78,000-80,000x) and std's own `intersection` is already the cheap kind in both argument orders.

**Complexity:** `struct{}`, `bool`, `int8` and `int` values all cost **36.1 bytes/key** — a struct
ending in a zero-sized field is padded, so `{int, struct{}}` is **16 bytes**, same as `{int, int}` ·
intersection must iterate the **smaller** set: Θ(min) vs Θ(|a|) measured at **79,796×**

---

## 8. Frequency counters and the idioms that surround them

`🟡 medium` · *three language decisions make it a one-liner*

Reading a missing key returns the zero value, so `m[k]++` works on a key never seen; `delete` on a
missing key is a no-op. Together those mean a counter needs no initialisation and no existence checks.

**Steps:**

1. Write the counter, then rank it — and find out why the tie-break is mandatory.
2. Map of slices, map of maps, composite keys.
3. Delete while ranging (legal), add while ranging (unspecified), and the three ways to clear.

```go
package main

import (
	"fmt"
	"maps"
	"slices"
	"sort"
	"strings"
	"testing"
)

// The frequency counter is the single most useful thing a map does, and Go's
// map is designed around it. Three language decisions make it a one-liner:
//
//	1. reading a missing key returns the ZERO VALUE, not an error
//	2. so m[k]++ works on a key that has never been seen
//	3. and `delete` on a missing key is a no-op
//
// Together those mean a counter needs no initialisation and no existence
// checks. This example is the idioms, and the three places they bite.

var sinkI int
var sinkS []string

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func words(s string) []string { return strings.Fields(strings.ToLower(s)) }

const text = `the quick brown fox jumps over the lazy dog
the dog barks and the fox runs the fox is quick
a quick brown dog jumps over a lazy fox`

func main() {
	fmt.Println("the counter, in one line, with no initialisation:")
	fmt.Println()
	count := map[string]int{}
	for _, w := range words(text) {
		count[w]++
	}
	fmt.Println("      for _, w := range words { count[w]++ }")
	fmt.Println()
	fmt.Printf("  %d distinct words from %d total\n", len(count), len(words(text)))
	fmt.Println()
	fmt.Println("  `count[w]++` on a key that has never been seen reads the zero")
	fmt.Println("  value, adds one, and writes it back. No `if _, ok :=`, no")
	fmt.Println("  initialisation, no branch. In a language where a missing key is")
	fmt.Println("  an error, this is four lines and a nested block.")

	fmt.Println()
	fmt.Println("ranking it needs a slice, because a map has no order:")
	fmt.Println()
	type kv struct {
		k string
		n int
	}
	pairs := make([]kv, 0, len(count))
	for k, n := range count {
		pairs = append(pairs, kv{k, n})
	}
	// Sort by count DESCENDING, then by key ASCENDING. The second key is not
	// decoration -- without it the output is nondeterministic, because the
	// input order came from a randomised map iteration.
	slices.SortFunc(pairs, func(a, b kv) int {
		if a.n != b.n {
			return b.n - a.n
		}
		return strings.Compare(a.k, b.k)
	})
	for _, p := range pairs[:6] {
		fmt.Printf("  %-8s %s %d\n", p.k, strings.Repeat("#", p.n), p.n)
	}
	fmt.Println()
	fmt.Println("  the tie-break on the key is REQUIRED, not tidiness. Map iteration")
	fmt.Println("  is randomised (example 6), so `quick` and `dog` -- both 3 -- would")
	fmt.Println("  swap places between runs and your test would flake.")
	fmt.Println()
	fmt.Println("  this is the general rule: any time a map's contents become an")
	fmt.Println("  ordered output, the ordering must be TOTAL.")

	fmt.Println()
	fmt.Println("the comma-ok idiom, and when you actually need it:")
	fmt.Println()
	scores := map[string]int{"alice": 0, "bob": 7}
	for _, k := range []string{"alice", "bob", "carol"} {
		v, ok := scores[k]
		fmt.Printf("  %-8s value=%-3d present=%-6v   `if scores[k] > 0` says %v\n",
			k, v, ok, scores[k] > 0)
	}
	fmt.Println()
	fmt.Println("  alice scored ZERO, which is not the same as not having played.")
	fmt.Println("  Any time the zero value is a MEANINGFUL value, you need comma-ok")
	fmt.Println("  -- and any time it is not, you do not. That is the whole rule,")
	fmt.Println("  and it is the same distinction as example 7's map[T]bool trap.")

	fmt.Println()
	fmt.Println("a map of slices needs no initialisation either -- append does it:")
	fmt.Println()
	byLen := map[int][]string{}
	for _, w := range words(text) {
		byLen[len(w)] = append(byLen[len(w)], w) // append to a nil slice is fine
	}
	for _, n := range slices.Sorted(maps.Keys(byLen)) {
		u := slices.Compact(slices.Sorted(slices.Values(byLen[n])))
		fmt.Printf("  %d letters: %v\n", n, u)
	}
	fmt.Println()
	fmt.Println("  `append(nil, x)` allocates and returns a slice, so a map of")
	fmt.Println("  slices is self-initialising. A map of MAPS is not:")
	fmt.Println()
	fmt.Println("      nested[a][b] = 1     // panics if nested[a] is nil")
	fmt.Println("      if nested[a] == nil { nested[a] = map[string]int{} }")
	fmt.Println()
	nested := map[string]map[string]int{}
	for _, pair := range [][2]string{{"x", "a"}, {"x", "b"}, {"y", "a"}} {
		if nested[pair[0]] == nil {
			nested[pair[0]] = map[string]int{}
		}
		nested[pair[0]][pair[1]]++
	}
	fmt.Printf("  nested: %v\n", nested)
	fmt.Println()
	fmt.Println("  the alternative is a COMPOSITE KEY, which is usually better:")
	flat := map[[2]string]int{}
	for _, pair := range [][2]string{{"x", "a"}, {"x", "b"}, {"y", "a"}} {
		flat[pair]++
	}
	fmt.Printf("  flat:   %v\n", flat)
	fmt.Println()
	fmt.Println("  one map, one allocation, no nil checks, and arrays are")
	fmt.Println("  comparable so they work as keys directly (example 9).")

	fmt.Println()
	fmt.Println("deleting while ranging is LEGAL, and adding is not:")
	fmt.Println()
	del := map[int]int{}
	for i := 0; i < 10; i++ {
		del[i] = i
	}
	for k := range del {
		if k%2 == 0 {
			delete(del, k)
		}
	}
	fmt.Printf("  deleted evens while ranging -> %v\n", slices.Sorted(maps.Keys(del)))
	fmt.Println()
	fmt.Println("  the spec guarantees this: an entry deleted during iteration will")
	fmt.Println("  not be produced. That makes filter-in-place a three-line loop.")
	fmt.Println()
	fmt.Println("  ADDING during iteration is different -- the spec says the new")
	fmt.Println("  entry MAY or MAY NOT be produced. It is not a panic and not a")
	fmt.Println("  race; it is simply unspecified, which is worse, because it will")
	fmt.Println("  work in testing.")
	added := map[int]int{1: 1, 2: 2, 3: 3}
	seen := 0
	for k := range added {
		if k < 100 && seen < 20 {
			added[k+100] = 0
		}
		seen++
	}
	fmt.Printf("  added during range: visited %d entries, final len %d\n", seen, len(added))
	fmt.Println("  (that number is not guaranteed to be the same on your machine)")

	fmt.Println()
	fmt.Println("clearing: three ways, and only one returns the memory:")
	fmt.Println()
	fmt.Printf("  %-34s %s\n", "m = map[K]V{}", "new map; the old one is garbage")
	fmt.Printf("  %-34s %s\n", "clear(m)", "keeps the capacity, drops the entries")
	fmt.Printf("  %-34s %s\n", "for k := range m { delete(m, k) }", "same, the pre-1.21 spelling")
	fmt.Println()
	fmt.Println("  `clear` is the one you want when the map is about to be refilled")
	fmt.Println("  to a similar size -- it keeps the buckets you already paid for.")
	fmt.Println("  Reassigning is the one you want when it is not, because a map")
	fmt.Println("  NEVER shrinks on its own. A map that peaked at 10 million entries")
	fmt.Println("  and now holds 3 is still 10 million entries wide.")
	fmt.Println()
	fmt.Println("  that is lesson 06's 'slices never shrink' and lesson 09's queue,")
	fmt.Println("  arriving a third time. Example 15 measures it.")

	fmt.Println()
	fmt.Println("and the alternative to a counter map, for the record:")
	fmt.Println()
	ws := words(strings.Repeat(text+" ", 200))
	tMap := nsPerOp(func() {
		c := map[string]int{}
		for _, w := range ws {
			c[w]++
		}
		sinkI = len(c)
	})
	tSort := nsPerOp(func() {
		cp := slices.Clone(ws)
		sort.Strings(cp)
		n := 0
		for i := 0; i < len(cp); {
			j := i
			for j < len(cp) && cp[j] == cp[i] {
				j++
			}
			n++
			i = j
		}
		sinkI = n
	})
	fmt.Printf("  %-34s %12.0f ns\n", "map[string]int counter", tMap)
	fmt.Printf("  %-34s %12.0f ns\n", "sort then count runs", tSort)
	fmt.Printf("  %d words, %d distinct\n", len(ws), len(count))
	fmt.Println()
	fmt.Println("  the map wins by 2x here and usually does. Sorting wins when you")
	fmt.Println("  needed the sorted order anyway, when the values are small")
	fmt.Println("  integers (use an array, lesson 14), or when the counts feed")
	fmt.Println("  straight into a ranking -- because then the sort was not extra.")
	sinkS = ws[:1]
}
```

**Sample output:**

```
the counter, in one line, with no initialisation:

      for _, w := range words { count[w]++ }

  13 distinct words from 29 total

  `count[w]++` on a key that has never been seen reads the zero
  value, adds one, and writes it back. No `if _, ok :=`, no
  initialisation, no branch. In a language where a missing key is
  an error, this is four lines and a nested block.

ranking it needs a slice, because a map has no order:

  the      ##### 5
  fox      #### 4
  dog      ### 3
  quick    ### 3
  a        ## 2
  brown    ## 2

  the tie-break on the key is REQUIRED, not tidiness. Map iteration
  is randomised (example 6), so `quick` and `dog` -- both 3 -- would
  swap places between runs and your test would flake.

  this is the general rule: any time a map's contents become an
  ordered output, the ordering must be TOTAL.

the comma-ok idiom, and when you actually need it:

  alice    value=0   present=true     `if scores[k] > 0` says false
  bob      value=7   present=true     `if scores[k] > 0` says true
  carol    value=0   present=false    `if scores[k] > 0` says false

  alice scored ZERO, which is not the same as not having played.
  Any time the zero value is a MEANINGFUL value, you need comma-ok
  -- and any time it is not, you do not. That is the whole rule,
  and it is the same distinction as example 7's map[T]bool trap.

a map of slices needs no initialisation either -- append does it:

  1 letters: [a]
  2 letters: [is]
  3 letters: [and dog fox the]
  4 letters: [lazy over runs]
  5 letters: [barks brown jumps quick]

  `append(nil, x)` allocates and returns a slice, so a map of
  slices is self-initialising. A map of MAPS is not:

      nested[a][b] = 1     // panics if nested[a] is nil
      if nested[a] == nil { nested[a] = map[string]int{} }

  nested: map[x:map[a:1 b:1] y:map[a:1]]

  the alternative is a COMPOSITE KEY, which is usually better:
  flat:   map[[x a]:1 [x b]:1 [y a]:1]

  one map, one allocation, no nil checks, and arrays are
  comparable so they work as keys directly (example 9).

deleting while ranging is LEGAL, and adding is not:

  deleted evens while ranging -> [1 3 5 7 9]

  the spec guarantees this: an entry deleted during iteration will
  not be produced. That makes filter-in-place a three-line loop.

  ADDING during iteration is different -- the spec says the new
  entry MAY or MAY NOT be produced. It is not a panic and not a
  race; it is simply unspecified, which is worse, because it will
  work in testing.
  added during range: visited 4 entries, final len 6
  (that number is not guaranteed to be the same on your machine)

clearing: three ways, and only one returns the memory:

  m = map[K]V{}                      new map; the old one is garbage
  clear(m)                           keeps the capacity, drops the entries
  for k := range m { delete(m, k) }  same, the pre-1.21 spelling

  `clear` is the one you want when the map is about to be refilled
  to a similar size -- it keeps the buckets you already paid for.
  Reassigning is the one you want when it is not, because a map
  NEVER shrinks on its own. A map that peaked at 10 million entries
  and now holds 3 is still 10 million entries wide.

  that is lesson 06's 'slices never shrink' and lesson 09's queue,
  arriving a third time. Example 15 measures it.

and the alternative to a counter map, for the record:

  map[string]int counter                    45802 ns
  sort then count runs                     102483 ns
  5800 words, 13 distinct

  the map wins by 2x here and usually does. Sorting wins when you
  needed the sorted order anyway, when the values are small
  integers (use an array, lesson 14), or when the counts feed
  straight into a ranking -- because then the sort was not extra.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::hash_map::DefaultHasher;
use std::collections::{BTreeMap, HashMap};
use std::fmt::Display;
use std::hash::{BuildHasherDefault, Hasher};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// The frequency counter is the single most useful thing a map does, and Go's
// map is designed around it. Three language decisions make it a one-liner in Go:
//
//	1. reading a missing key returns the ZERO VALUE, not an error
//	2. so m[k]++ works on a key that has never been seen
//	3. and `delete` on a missing key is a no-op
//
// Rust made different decisions (a missing key is an Option, not a default), and
// gave the same job a different tool: the ENTRY API. This example is the idioms,
// and the places they bite -- including the ones Rust refuses to compile.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Counting allocator: stand-in for testing.AllocsPerRun / runtime.MemStats.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events while running `f`, its result).
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

/// bytes currently live on the heap.
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
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

// ---- two extra hashers ----
//
// `Counted` is std's SipHash with a tally of how many times a key was hashed,
// so "how many lookups did that spelling cost?" is a number, not a feeling.
static HASHES: AtomicUsize = AtomicUsize::new(0);

#[derive(Default)]
struct Counted(DefaultHasher);
impl Hasher for Counted {
    fn write(&mut self, bytes: &[u8]) {
        self.0.write(bytes);
    }
    fn finish(&self) -> u64 {
        HASHES.fetch_add(1, Relaxed);
        self.0.finish()
    }
}

// `Fast` is example 2's finalized FNV-1a: FNV-1a over the bytes of the key
// (std's `str` hash also feeds it a 0xff terminator byte), then the splitmix64
// finalizer, bit-identical to Go's example 2. No random seed, so unlike std's
// default it is a hash an attacker can aim at (example 14).
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
    fn finish(&self) -> u64 {
        let mut h = self.0;
        h ^= h >> 30;
        h = h.wrapping_mul(0xbf58476d1ce4e5b9);
        h ^= h >> 27;
        h = h.wrapping_mul(0x94d049bb133111eb);
        h ^ (h >> 31)
    }
}

/// `[1 2 3]`, the way Go's %v prints a slice ({:?} would give `[1, 2, 3]`)
fn show<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Go's %v of map[string]map[string]int: keys sorted, `map[x:map[a:1 b:1] y:map[a:1]]`
fn show_nested(m: &HashMap<&str, HashMap<&str, i32>>) -> String {
    let sorted: BTreeMap<&str, BTreeMap<&str, i32>> = m
        .iter()
        .map(|(&k, inner)| (k, inner.iter().map(|(&k2, &n)| (k2, n)).collect()))
        .collect();
    let parts: Vec<String> = sorted
        .iter()
        .map(|(k, inner)| {
            let kv: Vec<String> = inner.iter().map(|(k2, n)| format!("{k2}:{n}")).collect();
            format!("{k}:map[{}]", kv.join(" "))
        })
        .collect();
    format!("map[{}]", parts.join(" "))
}

/// Go's %v of map[[2]string]int: `map[[x a]:1 [x b]:1 [y a]:1]`
fn show_flat(m: &HashMap<[&str; 2], i32>) -> String {
    let sorted: BTreeMap<&[&str; 2], &i32> = m.iter().collect();
    let parts: Vec<String> = sorted
        .iter()
        .map(|(k, n)| format!("[{} {}]:{n}", k[0], k[1]))
        .collect();
    format!("map[{}]", parts.join(" "))
}

/// How many times `count_one` hashes a key while counting `ws`, and how many
/// distinct words it found. The table is pre-sized: a resize rehashes every key
/// and would pollute the tally.
type CountedMap<'a> = HashMap<&'a str, usize, BuildHasherDefault<Counted>>;
fn tally<'a>(ws: &[&'a str], count_one: impl Fn(&mut CountedMap<'a>, &'a str)) -> (usize, usize) {
    let mut m = CountedMap::with_capacity_and_hasher(32, BuildHasherDefault::default());
    let before = HASHES.load(Relaxed);
    for &w in ws {
        count_one(&mut m, w);
    }
    (HASHES.load(Relaxed) - before, m.len())
}

fn words(s: &str) -> Vec<&str> {
    s.split_whitespace().collect()
}

const TEXT: &str = "the quick brown fox jumps over the lazy dog\n\
the dog barks and the fox runs the fox is quick\n\
a quick brown dog jumps over a lazy fox";

const PAIRS: [[&str; 2]; 3] = [["x", "a"], ["x", "b"], ["y", "a"]];

fn main() {
    // Go lowercases every word; the keys below are slices OF this string, so it
    // has to outlive every map that borrows from it.
    let text = TEXT.to_lowercase();
    let ws = words(&text);

    println!("the counter, in one line, with no initialisation:");
    println!();
    let mut count: HashMap<&str, usize> = HashMap::new();
    for &w in &ws {
        *count.entry(w).or_default() += 1;
    }
    println!("      for w in words {{ *count.entry(w).or_default() += 1 }}");
    println!();
    println!("  {} distinct words from {} total", count.len(), ws.len());
    println!();
    println!("  Go's `count[w]++` has no direct spelling here. `count[w]` panics on a key");
    println!("  that is missing, and HashMap has no IndexMut, so `count[w] += 1` is a compile");
    println!("  error (E0594: \"trait `IndexMut` is required to modify indexed content\"). The");
    println!(
        "  spelling is the Entry API: `entry(w)` finds the slot or reserves it, `or_default()`"
    );
    println!("  writes a 0 into a fresh one, and the `&mut usize` it hands back is incremented.");
    println!("  No `if let Some`, no branch, no second lookup. Is that last claim true? Count");
    println!("  how many times a key is hashed, 29 words and 13 distinct, against the spelling");
    println!("  that checks first:");
    println!();
    let (h_entry, d1) = tally(&ws, |m, w| *m.entry(w).or_default() += 1);
    let (h_check, d2) = tally(&ws, |m, w| match m.get_mut(w) {
        Some(c) => *c += 1,
        None => {
            m.insert(w, 1);
        }
    });
    assert_eq!(d1, d2);
    println!(
        "    {:<54} {:>3} hashes for {} words",
        "*m.entry(w).or_default() += 1",
        h_entry,
        ws.len()
    );
    println!(
        "    {:<54} {:>3} hashes for {} words",
        "match m.get_mut(w) { Some(c) => .., None => insert }",
        h_check,
        ws.len()
    );
    // a lookup on an EMPTY table returns before hashing (hashbrown's get_mut), so
    // the very first distinct word costs one hash even in the checking spelling
    assert_eq!(h_check - h_entry, d1 - 1);
    println!();
    println!(
        "  the check-then-insert spelling hashes every new key twice -- {} extra hashes for {} distinct",
        h_check - h_entry,
        d1
    );
    println!(
        "  words -- one fewer than the distinct words, because a lookup in an empty table skips the hash"
    );
    println!(
        "  (read in hashbrown's get_mut, not measured). entry() hashes each word once, hit or"
    );
    println!("  miss. That is what the Entry API is for: 'find or create' as ONE lookup that the");
    println!("  borrow checker accepts.");

    println!();
    println!("ranking it needs a slice, because a map has no order:");
    println!();
    let mut pairs: Vec<(&str, usize)> = count.iter().map(|(&k, &n)| (k, n)).collect();
    // Sort by count DESCENDING, then by key ASCENDING. The second key is not
    // decoration -- without it the output is nondeterministic, because the
    // input order came from a randomised map iteration.
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then_with(|| a.0.cmp(b.0)));
    for (k, n) in &pairs[..6] {
        println!("  {:<8} {} {}", k, "#".repeat(*n), n);
    }
    println!();
    // What happens without the tie-break? Rebuild the SAME counts in 32 fresh
    // maps (each gets its own random seed) and sort by count only, with the
    // stable sort. Where do the two words that tie on 3 land?
    let mut quick_first = 0;
    for _ in 0..32 {
        let mut fresh: HashMap<&str, usize> = HashMap::new();
        for &w in &ws {
            *fresh.entry(w).or_default() += 1;
        }
        let mut p: Vec<(&str, usize)> = fresh.into_iter().collect();
        p.sort_by_key(|&(_, n)| std::cmp::Reverse(n)); // count only: ties keep map order
        let at = |w: &str| {
            p.iter()
                .position(|&(k, _)| k == w)
                .expect("word was counted")
        };
        if at("quick") < at("dog") {
            quick_first += 1;
        }
    }
    println!("  the tie-break on the key is REQUIRED, not tidiness. Sorting by count alone,");
    println!(
        "  `quick` and `dog` -- both 3 -- came out quick-first in {quick_first} of 32 fresh maps built"
    );
    println!("  from the same text, because each map has its own random seed (example 1) and a");
    println!("  stable sort keeps ties in iteration order. Your test would flake.");
    println!();
    println!("  this is the general rule: any time a map's contents become an");
    println!("  ordered output, the ordering must be TOTAL.");

    println!();
    println!("the Option instead of the comma-ok idiom, and when you actually need it:");
    println!();
    let scores: HashMap<&str, i32> = HashMap::from([("alice", 0), ("bob", 7)]);
    for k in ["alice", "bob", "carol"] {
        let got = scores.get(k);
        println!(
            "  {:<8} get={:<9} present={:<6}   `unwrap_or(0) > 0` says {}",
            k,
            format!("{got:?}"),
            got.is_some(),
            got.copied().unwrap_or(0) > 0
        );
    }
    println!();
    println!("  alice scored ZERO, which is not the same as not having played. In Go the");
    println!("  zero value is what a missing key hands you, so you must remember comma-ok;");
    println!("  here `get` hands you an Option and the missing case is a different VALUE");
    println!("  (`None`) the compiler makes you handle. The trap is still there if you write");
    println!("  `unwrap_or(0)` or `unwrap_or_default()`, which is the zero-value default on");
    println!("  purpose -- and it is the same distinction as example 7's HashMap<T, bool> trap.");

    println!();
    println!("a map of vectors needs no initialisation either -- or_default does it:");
    println!();
    let mut by_len: HashMap<usize, Vec<&str>> = HashMap::new();
    for &w in &ws {
        by_len.entry(w.len()).or_default().push(w); // a fresh Vec::new() the first time
    }
    let mut lens: Vec<usize> = by_len.keys().copied().collect();
    lens.sort_unstable();
    for n in lens {
        let mut u = by_len[&n].clone();
        u.sort_unstable();
        u.dedup();
        println!("  {n} letters: {}", show(&u));
    }
    println!();
    println!(
        "  `or_default()` inserts `Vec::new()`, which owns no heap memory until the first push"
    );
    println!(
        "  (capacity {}), so a map of vectors is self-initialising. And so, here, is a map of MAPS:",
        Vec::<&str>::new().capacity()
    );
    println!();
    println!("      *nested.entry(a).or_default().entry(b).or_default() += 1");
    println!("      // Go: nested[a][b] = 1 panics if nested[a] is nil");
    println!();
    let (allocs_nested, nested) = allocs_of(|| {
        let mut nested: HashMap<&str, HashMap<&str, i32>> = HashMap::new();
        for [a, b] in PAIRS {
            *nested.entry(a).or_default().entry(b).or_default() += 1;
        }
        nested
    });
    println!("  nested: {}", show_nested(&nested));
    println!();
    println!("  the alternative is a COMPOSITE KEY, which is usually better:");
    let (allocs_flat, flat) = allocs_of(|| {
        let mut flat: HashMap<[&str; 2], i32> = HashMap::new();
        for pair in PAIRS {
            *flat.entry(pair).or_default() += 1;
        }
        flat
    });
    println!("  flat:   {}", show_flat(&flat));
    println!();
    println!(
        "  one map instead of {} -- {allocs_flat} allocation against {allocs_nested} for these three pairs -- and arrays",
        1 + nested.len()
    );
    println!("  are Hash + Eq so they work as keys directly (example 9). The Go program's");
    println!("  nil check is not needed in either shape: or_default() is the nil check.");

    println!();
    println!("deleting while looping is a COMPILE ERROR, and so is adding:");
    println!();
    // The loop Go allows, which Rust refuses (verified with rustc 1.98.1):
    //
    //     for k in del.keys() { if k % 2 == 0 { del.remove(k); } }
    //     error[E0502]: cannot borrow `del` as mutable because it is also
    //                   borrowed as immutable
    //
    // The spelling for "filter in place" is retain, which gets the closure and
    // the map to cooperate without a held borrow.
    let mut del: HashMap<i32, i32> = (0..10).map(|i| (i, i)).collect();
    del.retain(|k, _| k % 2 != 0);
    let mut kept: Vec<i32> = del.keys().copied().collect();
    kept.sort_unstable();
    println!("  retain(|k, _| k % 2 != 0) -> {}", show(&kept));
    println!();
    println!("  `for k in del.keys() {{ del.remove(k); }}` is E0502: the iterator holds a shared");
    println!(
        "  borrow of the map for the whole loop, and remove needs a mutable one. Go allows it"
    );
    println!("  and guarantees a deleted entry is not produced; Rust makes the question");
    println!("  unaskable, so retain (one pass, in place) is the way to filter.");
    println!();
    println!("  ADDING during iteration is the same error, E0502, where Go's spec says the new");
    println!("  entry MAY or MAY NOT be produced. There is no unspecified behaviour to be");
    println!("  surprised by; to grow a map from its own keys, loop over a snapshot of them:");
    let mut added: HashMap<i32, i32> = HashMap::from([(1, 1), (2, 2), (3, 3)]);
    let snapshot: Vec<i32> = added.keys().copied().collect();
    let mut seen = 0;
    for k in snapshot {
        added.insert(k + 100, 0); // the snapshot is a Vec, so no borrow of `added` is held
        seen += 1;
    }
    println!(
        "  added from a snapshot of the keys: visited {seen} entries, final len {}",
        added.len()
    );
    println!("  (visited is always the snapshot's length, and final len always 6, on every run)");

    println!();
    println!("clearing: four ways, and a map never shrinks on its own:");
    println!();
    let ways = [
        (
            "m = HashMap::new()",
            "new map; the old one is dropped (freed at once)",
        ),
        ("m.clear()", "keeps the capacity, drops the entries"),
        ("m.retain(|_, _| false)", "same effect as clear(), one pass"),
        (
            "m.shrink_to_fit()",
            "gives the unused capacity back (Go has no spelling)",
        ),
    ];
    for (how, what) in ways {
        println!("  {how:<34} {what}");
    }
    println!();
    println!(
        "  Measured, on a HashMap<i64, i64> that peaks at 1,000,000 entries (the Go text says"
    );
    println!(
        "  10 million; this is 10x smaller to keep the program light, the ratios are the point):"
    );
    println!();
    println!(
        "  {:<38} {:>9} {:>10} {:>9}",
        "step", "len", "capacity", "live KB"
    );
    let base = live_bytes();
    let peak = || {
        let mut m: HashMap<i64, i64> = HashMap::new(); // no hint: grows the hard way
        for i in 0..1_000_000 {
            m.insert(i, i);
        }
        m
    };
    let kb = |b: usize| (live_bytes() - b) as f64 / 1024.0;
    let row = |step: &str, m: &HashMap<i64, i64>, b: usize| {
        println!(
            "  {:<38} {:>9} {:>10} {:>9.1}",
            step,
            m.len(),
            m.capacity(),
            kb(b)
        );
    };
    let mut m = peak();
    let cap_peak = m.capacity();
    let kb_peak = kb(base);
    row("peaked at 1,000,000 inserts", &m, base);
    m.retain(|k, _| *k < 3);
    let cap_after_retain = m.capacity();
    let kb_after_retain = kb(base);
    row("retain(|k, _| *k < 3)  (holds 3)", &m, base);
    m.shrink_to_fit();
    row("  ... then shrink_to_fit()", &m, base);
    drop(m);
    let mut m = peak();
    m.clear();
    let cap_after_clear = m.capacity();
    row("fresh peak, then clear()", &m, base);
    drop(m);
    let mut m = peak();
    black_box(m.len()); // read the peak map once, then replace it
    m = HashMap::new();
    row("fresh peak, then m = HashMap::new()", &m, base);
    println!();
    assert_eq!(kb_after_retain, kb_peak);
    assert_eq!(cap_after_clear, cap_peak);
    assert!(cap_after_retain <= cap_peak);
    println!("  `retain` down to 3 entries leaves the table exactly as wide as at the peak: the");
    println!(
        "  same {kb_peak:.0} KB live for 3 entries. That is Go's \"a map NEVER shrinks on its own\","
    );
    println!(
        "  confirmed for Rust, and `clear()` keeps the table as well (same capacity, {cap_after_clear})."
    );
    println!(
        "  Reassigning frees it all, and where Go leaves you only that option, `shrink_to_fit()`"
    );
    println!(
        "  (or `shrink_to(n)`) shrinks in place. So `clear()` is the one for a map about to be"
    );
    println!("  refilled to a similar size, because it keeps the table you already paid for.");
    println!();
    println!(
        "  Note that `capacity()` is a poorer witness than the bytes: after `retain` it reads"
    );
    println!(
        "  {cap_after_retain} against the peak's {cap_peak}, though not one byte was given back. It"
    );
    println!("  is items + growth_left, and hashbrown erases a slot as a tombstone rather than as");
    println!("  empty when the slots around it were full, which gives back no headroom (read in");
    println!("  hashbrown's `erase`; this program does not count tombstones).");
    println!();
    println!("  that is lesson 06's 'slices never shrink' and lesson 09's queue,");
    println!("  arriving a third time -- Vec and VecDeque have shrink_to_fit as well.");

    println!();
    println!("and the alternative to a counter map, for the record:");
    println!();
    let big_text = format!("{text} ").repeat(200);
    let big = words(&big_text);
    let t_sip = ns_per_op(|| {
        let mut c: HashMap<&str, usize> = HashMap::new();
        for &w in black_box(&big) {
            *c.entry(w).or_default() += 1;
        }
        black_box(c.len());
    });
    let t_fast = ns_per_op(|| {
        let mut c: HashMap<&str, usize, BuildHasherDefault<Fast>> = HashMap::default();
        for &w in black_box(&big) {
            *c.entry(w).or_default() += 1;
        }
        black_box(c.len());
    });
    let t_sort = ns_per_op(|| {
        let mut cp = black_box(&big).clone();
        cp.sort_unstable();
        black_box(cp.chunk_by(|a, b| a == b).count());
    });
    println!(
        "  {:<44} {:>12.0} ns",
        "HashMap<&str, usize> counter (SipHash-1-3)", t_sip
    );
    println!(
        "  {:<44} {:>12.0} ns",
        "HashMap<&str, usize> counter (Fast: FNV+mix)", t_fast
    );
    println!("  {:<44} {:>12.0} ns", "sort then count runs", t_sort);
    println!("  {} words, {} distinct", big.len(), count.len());
    println!();
    println!(
        "  sorting costs {:.1}x the SipHash map and {:.1}x the Fast-hash map on this input.",
        t_sort / t_sip,
        t_sort / t_fast
    );
    println!(
        "  Go's advice, which this program does not measure: sorting wins when you needed the"
    );
    println!("  sorted order anyway, when");
    println!("  the values are small integers (use an array, lesson 14), or when the counts feed");
    println!("  straight into a ranking -- because then the sort was not extra.");
}
```

**Rust sample output:**

```
the counter, in one line, with no initialisation:

      for w in words { *count.entry(w).or_default() += 1 }

  13 distinct words from 29 total

  Go's `count[w]++` has no direct spelling here. `count[w]` panics on a key
  that is missing, and HashMap has no IndexMut, so `count[w] += 1` is a compile
  error (E0594: "trait `IndexMut` is required to modify indexed content"). The
  spelling is the Entry API: `entry(w)` finds the slot or reserves it, `or_default()`
  writes a 0 into a fresh one, and the `&mut usize` it hands back is incremented.
  No `if let Some`, no branch, no second lookup. Is that last claim true? Count
  how many times a key is hashed, 29 words and 13 distinct, against the spelling
  that checks first:

    *m.entry(w).or_default() += 1                           29 hashes for 29 words
    match m.get_mut(w) { Some(c) => .., None => insert }    41 hashes for 29 words

  the check-then-insert spelling hashes every new key twice -- 12 extra hashes for 13 distinct
  words -- one fewer than the distinct words, because a lookup in an empty table skips the hash
  (read in hashbrown's get_mut, not measured). entry() hashes each word once, hit or
  miss. That is what the Entry API is for: 'find or create' as ONE lookup that the
  borrow checker accepts.

ranking it needs a slice, because a map has no order:

  the      ##### 5
  fox      #### 4
  dog      ### 3
  quick    ### 3
  a        ## 2
  brown    ## 2

  the tie-break on the key is REQUIRED, not tidiness. Sorting by count alone,
  `quick` and `dog` -- both 3 -- came out quick-first in 16 of 32 fresh maps built
  from the same text, because each map has its own random seed (example 1) and a
  stable sort keeps ties in iteration order. Your test would flake.

  this is the general rule: any time a map's contents become an
  ordered output, the ordering must be TOTAL.

the Option instead of the comma-ok idiom, and when you actually need it:

  alice    get=Some(0)   present=true     `unwrap_or(0) > 0` says false
  bob      get=Some(7)   present=true     `unwrap_or(0) > 0` says true
  carol    get=None      present=false    `unwrap_or(0) > 0` says false

  alice scored ZERO, which is not the same as not having played. In Go the
  zero value is what a missing key hands you, so you must remember comma-ok;
  here `get` hands you an Option and the missing case is a different VALUE
  (`None`) the compiler makes you handle. The trap is still there if you write
  `unwrap_or(0)` or `unwrap_or_default()`, which is the zero-value default on
  purpose -- and it is the same distinction as example 7's HashMap<T, bool> trap.

a map of vectors needs no initialisation either -- or_default does it:

  1 letters: [a]
  2 letters: [is]
  3 letters: [and dog fox the]
  4 letters: [lazy over runs]
  5 letters: [barks brown jumps quick]

  `or_default()` inserts `Vec::new()`, which owns no heap memory until the first push
  (capacity 0), so a map of vectors is self-initialising. And so, here, is a map of MAPS:

      *nested.entry(a).or_default().entry(b).or_default() += 1
      // Go: nested[a][b] = 1 panics if nested[a] is nil

  nested: map[x:map[a:1 b:1] y:map[a:1]]

  the alternative is a COMPOSITE KEY, which is usually better:
  flat:   map[[x a]:1 [x b]:1 [y a]:1]

  one map instead of 3 -- 1 allocation against 3 for these three pairs -- and arrays
  are Hash + Eq so they work as keys directly (example 9). The Go program's
  nil check is not needed in either shape: or_default() is the nil check.

deleting while looping is a COMPILE ERROR, and so is adding:

  retain(|k, _| k % 2 != 0) -> [1 3 5 7 9]

  `for k in del.keys() { del.remove(k); }` is E0502: the iterator holds a shared
  borrow of the map for the whole loop, and remove needs a mutable one. Go allows it
  and guarantees a deleted entry is not produced; Rust makes the question
  unaskable, so retain (one pass, in place) is the way to filter.

  ADDING during iteration is the same error, E0502, where Go's spec says the new
  entry MAY or MAY NOT be produced. There is no unspecified behaviour to be
  surprised by; to grow a map from its own keys, loop over a snapshot of them:
  added from a snapshot of the keys: visited 3 entries, final len 6
  (visited is always the snapshot's length, and final len always 6, on every run)

clearing: four ways, and a map never shrinks on its own:

  m = HashMap::new()                 new map; the old one is dropped (freed at once)
  m.clear()                          keeps the capacity, drops the entries
  m.retain(|_, _| false)             same effect as clear(), one pass
  m.shrink_to_fit()                  gives the unused capacity back (Go has no spelling)

  Measured, on a HashMap<i64, i64> that peaks at 1,000,000 entries (the Go text says
  10 million; this is 10x smaller to keep the program light, the ratios are the point):

  step                                         len   capacity   live KB
  peaked at 1,000,000 inserts              1000000    1835008   34816.0
  retain(|k, _| *k < 3)  (holds 3)               3    1662087   34816.0
    ... then shrink_to_fit()                     3          3       0.1
  fresh peak, then clear()                       0    1835008   34816.0
  fresh peak, then m = HashMap::new()            0          0       0.0

  `retain` down to 3 entries leaves the table exactly as wide as at the peak: the
  same 34816 KB live for 3 entries. That is Go's "a map NEVER shrinks on its own",
  confirmed for Rust, and `clear()` keeps the table as well (same capacity, 1835008).
  Reassigning frees it all, and where Go leaves you only that option, `shrink_to_fit()`
  (or `shrink_to(n)`) shrinks in place. So `clear()` is the one for a map about to be
  refilled to a similar size, because it keeps the table you already paid for.

  Note that `capacity()` is a poorer witness than the bytes: after `retain` it reads
  1662087 against the peak's 1835008, though not one byte was given back. It
  is items + growth_left, and hashbrown erases a slot as a tombstone rather than as
  empty when the slots around it were full, which gives back no headroom (read in
  hashbrown's `erase`; this program does not count tombstones).

  that is lesson 06's 'slices never shrink' and lesson 09's queue,
  arriving a third time -- Vec and VecDeque have shrink_to_fit as well.

and the alternative to a counter map, for the record:

  HashMap<&str, usize> counter (SipHash-1-3)          56129 ns
  HashMap<&str, usize> counter (Fast: FNV+mix)        37029 ns
  sort then count runs                                97157 ns
  5800 words, 13 distinct

  sorting costs 1.7x the SipHash map and 2.6x the Fast-hash map on this input.
  Go's advice, which this program does not measure: sorting wins when you needed the
  sorted order anyway, when
  the values are small integers (use an array, lesson 14), or when the counts feed
  straight into a ranking -- because then the sort was not extra.
```

> **Go vs Rust:** Go's three language decisions do not exist in Rust: `count[w]++` is a compile error (E0594, `HashMap` has no `IndexMut`) and a missing key is `None`, so the counter is `*entry(w).or_default() += 1`, which hashes each word once (29 hashes for 29 words, against 41 for `get_mut` then `insert`) and also self-initialises the map of maps that Go's `nested[a][b]` cannot. Deleting or adding while looping is rejected at compile time (E0502) instead of guaranteed or unspecified, so the port uses `retain` and a snapshot of the keys (always 3 visited, final len 6; Go's visited count varies: 4 in its sample, 5-6 in six re-runs). Rust confirms that a map never shrinks (same 34,816 KB after `retain` down to 3 entries; the demo is 1,000,000 entries, 10x smaller than the Go text's 10 million) but has `shrink_to_fit()`, and in the benchmark (SipHash-1-3 against a `Fast` finalized-FNV hasher, seven runs) sorting costs 1.3-1.8x the default-hasher map and 2.4-3.0x the fast one, against 2.2-2.7x for Go's map.

**Complexity:** counting is Θ(n); ranking needs a **total** order or the output is nondeterministic ·
`append` to a nil slice is fine, so a map of slices self-initialises — a map of **maps** does not ·
deleting during `range` is guaranteed safe, **adding is explicitly unspecified**

---

## 9. What can be a map key

`🟡 medium` · *"comparable" is one word with sharp edges*

Structs and arrays are keys for free. Slices and maps are compile errors, which is a kindness. The
interesting cases are the two that compile cleanly and then betray you.

**Steps:**

1. Use a struct and an array as keys, and see why composite keys beat nested maps.
2. Put `NaN` in a map three times and fail to read any of it back.
3. Then put a slice in a `map[any]T` and watch it panic at run time.

```go
package main

import (
	"fmt"
	"math"
	"slices"
	"testing"
	"time"
)

// What can be a map key? Go's answer is one word -- COMPARABLE -- and the
// consequences of that word are where the surprises live.
//
//	comparable      booleans, numbers, strings, pointers, channels,
//	                interfaces, and structs/arrays of comparable things
//	NOT comparable  slices, maps, functions -- compile error as a key
//	comparable but treacherous  floats (NaN) and interfaces (runtime panic)
//
// The two treacherous cases are the point of this example. Both compile
// cleanly. One corrupts your map quietly; the other panics in production.

type Point struct{ X, Y int }

type WithSlice struct {
	Name string
	Tags []string // makes the whole struct non-comparable
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

func main() {
	fmt.Println("structs and arrays are comparable, so they are keys for free:")
	fmt.Println()
	grid := map[Point]string{
		{0, 0}:  "origin",
		{1, 2}:  "somewhere",
		{-1, 5}: "elsewhere",
	}
	fmt.Printf("  map[Point]string    %v -> %q\n", Point{1, 2}, grid[Point{1, 2}])
	fmt.Printf("  a missing point     %v -> %q (zero value)\n", Point{9, 9}, grid[Point{9, 9}])

	arr := map[[3]int]string{{1, 2, 3}: "a", {4, 5, 6}: "b"}
	fmt.Printf("  map[[3]int]string   %v -> %q\n", [3]int{4, 5, 6}, arr[[3]int{4, 5, 6}])

	type Version struct {
		Major, Minor, Patch int
		Pre                 string
	}
	vers := map[Version]bool{{1, 21, 0, ""}: true, {1, 22, 0, "rc1"}: true}
	fmt.Printf("  a 4-field struct    %v -> %v\n", Version{1, 21, 0, ""}, vers[Version{1, 21, 0, ""}])
	fmt.Println()
	fmt.Println("  this is why example 8's composite key works, and it is almost")
	fmt.Println("  always better than nesting maps: one lookup, one allocation, no")
	fmt.Println("  nil checks, and the key type documents itself.")

	fmt.Println()
	fmt.Println("what will NOT compile, and why that is a kindness:")
	fmt.Println()
	fmt.Println("      map[[]string]int      // slice: not comparable")
	fmt.Println("      map[map[K]V]int       // map: not comparable")
	fmt.Println("      map[func()]int        // func: not comparable")
	fmt.Println("      map[WithSlice]int     // struct CONTAINING a slice")
	fmt.Println()
	fmt.Println("  a slice is a header pointing at an array. Comparing two headers")
	fmt.Println("  would compare pointers, not contents -- so `==` would be")
	fmt.Println("  surprising for every reader. Go refuses rather than guess.")
	fmt.Println()
	fmt.Println("  the workaround is to make a comparable key out of it: a string,")
	fmt.Println("  an array, or a struct of scalars.")
	tags := []string{"go", "maps", "hash"}
	fmt.Printf("  []string{%v} -> key %q\n", tags, fmt.Sprint(tags))
	var fixed [3]string
	copy(fixed[:], tags)
	fmt.Printf("  or, with no allocation and no ambiguity: [3]string%v\n", fixed)

	fmt.Println()
	fmt.Println("FLOATS are comparable, and NaN is the reason you should hesitate:")
	fmt.Println()
	nan := math.NaN()
	fmt.Printf("  NaN == NaN            -> %v\n", nan == nan)
	m := map[float64]string{}
	m[nan] = "first"
	m[nan] = "second"
	m[math.NaN()] = "third"
	fmt.Printf("  after three inserts of NaN, len = %d\n", len(m))
	v, ok := m[nan]
	fmt.Printf("  m[nan] -> %q, present = %v\n", v, ok)
	fmt.Println()
	fmt.Println("  three entries, and NOT ONE of them can be read back. NaN is not")
	fmt.Println("  equal to itself, so every insert creates a new entry and every")
	fmt.Println("  lookup fails. `delete` cannot remove them either.")
	fmt.Println()
	fmt.Println("  they are not even invisible -- ranging finds them:")
	for k, val := range m {
		fmt.Printf("    key=%v value=%q  (k==k is %v)\n", k, val, k == k)
	}
	fmt.Println()
	fmt.Println("  so a map with a float key can grow without bound while `len`")
	fmt.Println("  climbs and every lookup misses. If floats must be keys, either")
	fmt.Println("  reject NaN at the door or quantise to an integer first.")
	fmt.Println()
	fmt.Printf("  also worth knowing: +0.0 == -0.0 is %v, so they are ONE key:\n", 0.0 == math.Copysign(0, -1))
	z := map[float64]string{}
	z[0.0] = "positive zero"
	z[math.Copysign(0, -1)] = "negative zero"
	fmt.Printf("  len = %d, m[0.0] = %q\n", len(z), z[0.0])

	fmt.Println()
	fmt.Println("INTERFACES are comparable, and that is a runtime panic waiting:")
	fmt.Println()
	any1 := map[any]string{}
	any1[42] = "int"
	any1["42"] = "string"
	any1[Point{1, 2}] = "struct"
	fmt.Printf("  map[any]string with mixed types: len=%d\n", len(any1))
	fmt.Printf("  any1[42]=%q  any1[\"42\"]=%q   <- 42 and \"42\" are different keys\n",
		any1[42], any1["42"])
	fmt.Println()
	fmt.Println("  now put a slice in it:")
	func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("  any1[[]int{1,2}] = \"x\"  ->  PANIC: %v\n", r)
			}
		}()
		any1[[]int{1, 2}] = "slice" //nolint
	}()
	fmt.Println()
	fmt.Println("  it compiled. `any` IS comparable as a static type, so the check")
	fmt.Println("  is deferred to run time -- and it fires on the insert, in")
	fmt.Println("  production, with whatever value a caller happened to pass.")
	fmt.Println()
	fmt.Println("  map[any]T is almost never what you want. If the key really is")
	fmt.Println("  open-ended, use a constrained interface or a tagged struct, and")
	fmt.Println("  let the compiler keep doing its job.")

	fmt.Println()
	fmt.Println("POINTERS are comparable by ADDRESS, which is rarely what is meant:")
	fmt.Println()
	a, b := &Point{1, 2}, &Point{1, 2}
	pm := map[*Point]string{a: "first"}
	fmt.Printf("  a and b hold equal values: *a == *b is %v\n", *a == *b)
	fmt.Printf("  but pm[a]=%q and pm[b]=%q\n", pm[a], pm[b])
	fmt.Println()
	fmt.Println("  a pointer key means IDENTITY, not equality. That is occasionally")
	fmt.Println("  exactly right -- an object registry, a visited-set of nodes -- and")
	fmt.Println("  usually a bug. Use the value type as the key when you mean value.")

	fmt.Println()
	fmt.Println("time.Time is the one from real life:")
	fmt.Println()
	t1 := time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)
	t2 := t1.In(time.FixedZone("same-instant", 0))
	fmt.Printf("  t1.Equal(t2)   -> %v   (same instant)\n", t1.Equal(t2))
	fmt.Printf("  t1 == t2       -> %v   (different monotonic/location fields)\n", t1 == t2)
	tm := map[time.Time]string{t1: "new year"}
	fmt.Printf("  tm[t1]=%q  tm[t2]=%q\n", tm[t1], tm[t2])
	fmt.Println()
	fmt.Println("  time.Time is a STRUCT with a wall clock, a monotonic reading and")
	fmt.Println("  a location pointer. Two Times can be the same instant and still")
	fmt.Println("  compare unequal, which is why the documentation says to use")
	fmt.Println("  Equal -- and why time.Time is a bad map key. Use UnixNano().")

	fmt.Println()
	fmt.Println("and what the key TYPE costs, since it is not free:")
	fmt.Println()
	const n = 200_000
	fmt.Printf("  %-26s %14s\n", "key type", "ns/lookup")
	ints := make([]int, n)
	strs := make([]string, n)
	pts := make([]Point, n)
	arrs := make([][4]int, n)
	for i := 0; i < n; i++ {
		ints[i] = i * 2654435761
		strs[i] = fmt.Sprintf("key_%d_padding", i)
		pts[i] = Point{i, i * 3}
		arrs[i] = [4]int{i, i * 2, i * 3, i * 4}
	}
	mi := make(map[int]int, n)
	ms := make(map[string]int, n)
	mp := make(map[Point]int, n)
	ma := make(map[[4]int]int, n)
	for i := 0; i < n; i++ {
		mi[ints[i]] = i
		ms[strs[i]] = i
		mp[pts[i]] = i
		ma[arrs[i]] = i
	}
	fmt.Printf("  %-26s %14.2f\n", "int", nsPerOp(func() {
		for _, k := range ints {
			_, sinkB = mi[k]
		}
	})/n)
	fmt.Printf("  %-26s %14.2f\n", "string (14 bytes)", nsPerOp(func() {
		for _, k := range strs {
			_, sinkB = ms[k]
		}
	})/n)
	fmt.Printf("  %-26s %14.2f\n", "struct{X,Y int}", nsPerOp(func() {
		for _, k := range pts {
			_, sinkB = mp[k]
		}
	})/n)
	fmt.Printf("  %-26s %14.2f\n", "[4]int", nsPerOp(func() {
		for _, k := range arrs {
			_, sinkB = ma[k]
		}
	})/n)
	fmt.Println()
	fmt.Println("  int keys get a specialised runtime path (runtime_fast64.go);")
	fmt.Println("  strings get another (runtime_faststr.go). Anything else goes")
	fmt.Println("  through the generic path, which hashes the bytes of the key and")
	fmt.Println("  compares them field by field.")
	fmt.Println()
	fmt.Println("  so prefer int and string keys on a hot path, and remember that a")
	fmt.Println("  wide struct key is hashed in full on EVERY lookup -- the cost")
	fmt.Println("  scales with the key, not with the map.")
	sinkI = len(slices.Clip(ints))
}
```

**Sample output:**

```
structs and arrays are comparable, so they are keys for free:

  map[Point]string    {1 2} -> "somewhere"
  a missing point     {9 9} -> "" (zero value)
  map[[3]int]string   [4 5 6] -> "b"
  a 4-field struct    {1 21 0 } -> true

  this is why example 8's composite key works, and it is almost
  always better than nesting maps: one lookup, one allocation, no
  nil checks, and the key type documents itself.

what will NOT compile, and why that is a kindness:

      map[[]string]int      // slice: not comparable
      map[map[K]V]int       // map: not comparable
      map[func()]int        // func: not comparable
      map[WithSlice]int     // struct CONTAINING a slice

  a slice is a header pointing at an array. Comparing two headers
  would compare pointers, not contents -- so `==` would be
  surprising for every reader. Go refuses rather than guess.

  the workaround is to make a comparable key out of it: a string,
  an array, or a struct of scalars.
  []string{[go maps hash]} -> key "[go maps hash]"
  or, with no allocation and no ambiguity: [3]string[go maps hash]

FLOATS are comparable, and NaN is the reason you should hesitate:

  NaN == NaN            -> false
  after three inserts of NaN, len = 3
  m[nan] -> "", present = false

  three entries, and NOT ONE of them can be read back. NaN is not
  equal to itself, so every insert creates a new entry and every
  lookup fails. `delete` cannot remove them either.

  they are not even invisible -- ranging finds them:
    key=NaN value="first"  (k==k is false)
    key=NaN value="second"  (k==k is false)
    key=NaN value="third"  (k==k is false)

  so a map with a float key can grow without bound while `len`
  climbs and every lookup misses. If floats must be keys, either
  reject NaN at the door or quantise to an integer first.

  also worth knowing: +0.0 == -0.0 is true, so they are ONE key:
  len = 1, m[0.0] = "negative zero"

INTERFACES are comparable, and that is a runtime panic waiting:

  map[any]string with mixed types: len=3
  any1[42]="int"  any1["42"]="string"   <- 42 and "42" are different keys

  now put a slice in it:
  any1[[]int{1,2}] = "x"  ->  PANIC: runtime error: hash of unhashable type []int

  it compiled. `any` IS comparable as a static type, so the check
  is deferred to run time -- and it fires on the insert, in
  production, with whatever value a caller happened to pass.

  map[any]T is almost never what you want. If the key really is
  open-ended, use a constrained interface or a tagged struct, and
  let the compiler keep doing its job.

POINTERS are comparable by ADDRESS, which is rarely what is meant:

  a and b hold equal values: *a == *b is true
  but pm[a]="first" and pm[b]=""

  a pointer key means IDENTITY, not equality. That is occasionally
  exactly right -- an object registry, a visited-set of nodes -- and
  usually a bug. Use the value type as the key when you mean value.

time.Time is the one from real life:

  t1.Equal(t2)   -> true   (same instant)
  t1 == t2       -> false   (different monotonic/location fields)
  tm[t1]="new year"  tm[t2]=""

  time.Time is a STRUCT with a wall clock, a monotonic reading and
  a location pointer. Two Times can be the same instant and still
  compare unequal, which is why the documentation says to use
  Equal -- and why time.Time is a bad map key. Use UnixNano().

and what the key TYPE costs, since it is not free:

  key type                        ns/lookup
  int                                  6.27
  string (14 bytes)                    9.05
  struct{X,Y int}                      8.76
  [4]int                              10.15

  int keys get a specialised runtime path (runtime_fast64.go);
  strings get another (runtime_faststr.go). Anything else goes
  through the generic path, which hashes the bytes of the key and
  compares them field by field.

  so prefer int and string keys on a hot path, and remember that a
  wide struct key is hashed in full on EVERY lookup -- the cost
  scales with the key, not with the map.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::hash_map::RandomState;
use std::collections::{BTreeMap, BTreeSet, HashMap, HashSet};
use std::hash::{BuildHasher, Hash, Hasher};
use std::hint::black_box;
use std::marker::PhantomData;
use std::rc::Rc;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};

// What can be a map key? Go's answer is one word -- COMPARABLE. Rust's answer
// is two traits -- `Hash + Eq` -- and the difference between one word and two
// traits is where this twin parts company with the Go example:
//
//     Go    comparable      booleans, numbers, strings, pointers, channels,
//                           interfaces, structs/arrays of comparable things
//           NOT comparable  slices, maps, functions
//           treacherous     floats (NaN) and interfaces (a run-time panic)
//
//     Rust  Hash + Eq       integers, bool, char, String, Vec<T>, [T; N],
//                           tuples, Option, BTreeMap, ... and anything that
//                           derives the two traits from keys
//           NOT keys        f32/f64, HashMap, HashSet, closures, dyn Any
//
// Rust checks the bound when you CALL insert, so most of what Go leaves
// to a run-time panic or to silent corruption is a compile error here. The
// program asks the compiler instead of asserting.

// ---- measuring kit (stand-ins for Go's testing.AllocsPerRun / Benchmark) ----

struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events

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
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
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

/// best of three `ns_per_op` runs: on a shared machine the minimum is the
/// least-disturbed sample.
fn best_ns(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// ---- asking the compiler "is T a key?" ----
//
// `Probe::<T>::IS_KEY` is `true` when `T: Hash + Eq` and `false` otherwise,
// decided at compile time. It works because an inherent associated const wins
// over a trait one when its bounds hold (a known trick; this version is std
// only).
struct Probe<T>(PhantomData<T>);
trait NotKey {
    const IS_KEY: bool = false;
}
impl<T> NotKey for Probe<T> {}
impl<T: Hash + Eq> Probe<T> {
    const IS_KEY: bool = true;
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Point {
    x: i64,
    y: i64,
}

// Go's WithSlice. In Go a slice field makes the whole struct non-comparable.
#[derive(Hash, PartialEq, Eq)]
struct WithVec {
    _name: String,
    _tags: Vec<String>,
}

// ...and the same struct with a float in it.
struct WithFloat {
    _name: String,
    _weight: f64,
}

// Go's open-ended `map[any]T`, closed: every key type the map accepts is a
// variant, so the compiler knows all of them (and checks each is Hash + Eq).
#[derive(Debug, PartialEq, Eq, Hash)]
enum Key {
    Int(i64),
    Str(String),
    Pt(Point),
    Seq(Vec<i64>),
}

// A float wrapper that PROMISES Eq it cannot keep: `impl Eq` is an empty
// marker, so nothing stops us -- and NaN != NaN breaks the promise. std calls
// this a logic error (unspecified results, but not undefined behaviour).
// Hashing +0.0 and -0.0 alike keeps equal values on equal hashes, as Go does.
#[derive(Clone, Copy, Debug)]
struct LyingF64(f64);
impl PartialEq for LyingF64 {
    fn eq(&self, o: &Self) -> bool {
        self.0 == o.0
    }
}
impl Eq for LyingF64 {}
impl Hash for LyingF64 {
    fn hash<H: Hasher>(&self, h: &mut H) {
        let x = if self.0 == 0.0 { 0.0 } else { self.0 };
        x.to_bits().hash(h);
    }
}

// The honest version: the constructor refuses NaN, so within this type
// `==` IS reflexive and `impl Eq` is true. (The `ordered-float` crate's
// `NotNan` is the same idea.) The field is private, so the check cannot be
// bypassed.
#[derive(Clone, Copy, Debug, PartialEq)]
struct NotNan(f64);
impl NotNan {
    fn new(x: f64) -> Option<NotNan> {
        (!x.is_nan()).then_some(NotNan(x))
    }
}
impl Eq for NotNan {}
impl Hash for NotNan {
    fn hash<H: Hasher>(&self, h: &mut H) {
        let x = if self.0 == 0.0 { 0.0 } else { self.0 };
        x.to_bits().hash(h);
    }
}

// `x == x`: the reflexivity that `Eq` promises and NaN breaks. Clippy flags the
// expression as a likely typo; here it is the experiment, so the lint is
// silenced on this one function.
#[allow(clippy::eq_op)]
fn is_reflexive<T: PartialEq>(x: &T) -> bool {
    x == x
}

macro_rules! probe_rows {
    ($($t:ty),* $(,)?) => {
        [$((stringify!($t), Probe::<$t>::IS_KEY)),*]
    };
}

fn main() {
    println!("structs and arrays are keys for free, once they derive Hash + Eq:");
    println!();
    let grid: HashMap<Point, &str> = HashMap::from([
        (Point { x: 0, y: 0 }, "origin"),
        (Point { x: 1, y: 2 }, "somewhere"),
        (Point { x: -1, y: 5 }, "elsewhere"),
    ]);
    println!(
        "  HashMap<Point, &str>    {:?} -> {:?}",
        Point { x: 1, y: 2 },
        grid.get(&Point { x: 1, y: 2 })
    );
    println!(
        "  a missing point         {:?} -> {:?} (no zero value: `get` returns an Option)",
        Point { x: 9, y: 9 },
        grid.get(&Point { x: 9, y: 9 })
    );

    let arr: HashMap<[i64; 3], &str> = HashMap::from([([1, 2, 3], "a"), ([4, 5, 6], "b")]);
    println!(
        "  HashMap<[i64; 3], &str> {:?} -> {:?}",
        [4, 5, 6],
        arr.get(&[4, 5, 6])
    );

    #[derive(Debug, PartialEq, Eq, Hash)]
    struct Version {
        major: u32,
        minor: u32,
        patch: u32,
        pre: &'static str,
    }
    let v = |major, minor, patch, pre| Version {
        major,
        minor,
        patch,
        pre,
    };
    let vers: HashMap<Version, bool> =
        HashMap::from([(v(1, 21, 0, ""), true), (v(1, 22, 0, "rc1"), true)]);
    println!(
        "  a 4-field struct        {:?} -> {:?}",
        v(1, 21, 0, ""),
        vers.get(&v(1, 21, 0, ""))
    );
    println!();
    println!("  this is why example 8's composite key works, and it is almost");
    println!("  always better than nesting maps: one lookup, one allocation, no");
    println!("  missing-inner-map case to handle, and the key type documents itself.");

    println!();
    println!("which types are keys? asked of the compiler, not asserted:");
    println!();
    let rows = probe_rows![
        i64,
        String,
        &'static str,
        (i64, i64),
        [u8; 3],
        Option<i64>,
        Vec<String>,
        &'static [i64],
        WithVec,
        BTreeMap<i64, i64>,
        BTreeSet<i64>,
        Rc<String>,
        *const Point,
        fn(),
        Duration,
        SystemTime,
        f64,
        f32,
        Option<f64>,
        [f64; 2],
        WithFloat,
        HashMap<i64, i64>,
        HashSet<i64>,
        Box<dyn Fn()>,
        Box<dyn std::any::Any>,
    ];
    println!("  {:<26} Hash + Eq?", "type");
    for (name, is_key) in rows {
        println!("  {:<26} {}", name, if is_key { "yes" } else { "NO" });
    }
    println!();
    println!("  Go refuses slices because a slice is a header pointing at an array, and");
    println!("  comparing two headers would compare pointers, not contents. Rust made the");
    println!("  other choice: `Vec<T> == Vec<T>` compares CONTENTS, so a Vec, a slice");
    println!("  and a struct containing one are all fine keys. What Rust refuses is the");
    println!("  float, and anything that holds one, plus the unordered std collections");
    println!("  (HashMap and HashSet have no `Hash` impl, plausibly because equal maps can");
    println!("  iterate in different orders) and types with no equality at all (closures,");
    println!(
        "  `dyn Any`). A bare `fn()` pointer IS a key, compared by address (Go: funcs are not comparable)."
    );
    println!();
    println!("  a Rust key that Go would not accept, used directly:");
    let tags: Vec<String> = ["go", "maps", "hash"].map(String::from).to_vec();
    let mut by_tags: HashMap<Vec<String>, i64> = HashMap::new();
    by_tags.insert(tags.clone(), 1);
    println!(
        "  HashMap<Vec<String>, i64>  {:?} -> {:?}",
        tags,
        by_tags.get(&tags)
    );
    println!(
        "  looked up through a borrowed slice, no clone: get(tags.as_slice()) -> {:?}",
        by_tags.get(tags.as_slice())
    );
    println!();
    println!("  what building each candidate key costs, counted by the allocator:");
    let (joined_allocs, joined) = allocs_of(|| tags.join(" "));
    let (array_allocs, array) =
        allocs_of(|| [tags[0].as_str(), tags[1].as_str(), tags[2].as_str()]);
    let (clone_allocs, cloned) = allocs_of(|| tags.clone());
    println!(
        "    tags.join(\" \")              {joined_allocs} allocation  -> {joined:?}   (Go's fmt.Sprint workaround)"
    );
    println!(
        "    [&str; 3] of borrows        {array_allocs} allocations -> {array:?}   (Go's [3]string)"
    );
    println!(
        "    tags.clone() as Vec<String> {clone_allocs} allocations -> {cloned:?}   (the owned key above)"
    );
    println!();
    println!("  and the string workaround has the ambiguity the array does not:");
    let a: [&str; 2] = ["a b", "c"];
    let b: [&str; 2] = ["a", "b c"];
    let joined_keys: HashSet<String> = [a.join(" "), b.join(" ")].into_iter().collect();
    let array_keys: HashSet<[&str; 2]> = [a, b].into_iter().collect();
    println!(
        "    [\"a b\", \"c\"] and [\"a\", \"b c\"]: joined with a space -> {} key, as arrays -> {} keys",
        joined_keys.len(),
        array_keys.len()
    );

    println!();
    println!("FLOATS: Rust refuses them as keys, so NaN has to be dealt with on purpose:");
    println!();
    println!("      let mut m: HashMap<f64, &str> = HashMap::new();");
    println!(
        "      m.insert(1.0, \"x\");   // E0599: insert exists, but its trait bounds were not satisfied"
    );
    println!(
        "      #[derive(Hash, Eq, PartialEq)] struct S {{ w: f64 }}   // E0277: f64: Hash, f64: Eq"
    );
    println!();
    println!("  (the two errors above are quoted from rustc, not provoked by this program.)");
    println!("  the type `HashMap<f64, _>` can still be NAMED; it is the call that fails.");
    println!("  Go's silent corruption is therefore not reachable by accident. There are");
    println!("  three ways to want a float key anyway, and they behave very differently:");
    println!();
    let nan = f64::NAN;
    println!("  NaN == NaN            -> {}", is_reflexive(&nan));
    println!();
    println!("  1. key on `to_bits()`, the integer with the same bits:");
    let nan_payload = f64::from_bits(nan.to_bits() | 1); // another NaN, one payload bit different
    let mut m: HashMap<u64, &str> = HashMap::new();
    m.insert(nan.to_bits(), "first");
    m.insert(nan.to_bits(), "second");
    m.insert((-nan).to_bits(), "third");
    m.insert(nan_payload.to_bits(), "fourth");
    println!(
        "     inserts of NaN, NaN, -NaN and a NaN with another payload: len = {}",
        m.len()
    );
    println!(
        "     m.get(&nan.to_bits()) -> {:?}   (found: the SAME bits are equal)",
        m.get(&nan.to_bits())
    );
    let mut z: HashMap<u64, &str> = HashMap::new();
    z.insert(0.0f64.to_bits(), "positive zero");
    z.insert((-0.0f64).to_bits(), "negative zero");
    println!(
        "     but +0.0 == -0.0 is {} and they are {} keys here (Go: one)",
        0.0 == -0.0f64,
        z.len()
    );
    println!("     NaN is found only because its bits happen to match, and there are many");
    println!("     NaN bit patterns: you have chosen bit-identity, not numeric equality.");
    println!();
    println!("  2. wrap the float in a type that claims `Eq` and keeps `==` -- Go's behaviour:");
    let mut m: HashMap<LyingF64, &str> = HashMap::new();
    m.insert(LyingF64(nan), "first");
    m.insert(LyingF64(nan), "second");
    m.insert(LyingF64(f64::NAN), "third");
    println!("     after three inserts of NaN, len = {}", m.len());
    let got = m.get(&LyingF64(nan)).copied();
    let removed = m.remove(&LyingF64(nan));
    println!("     m.get(&NaN) -> {got:?}, m.remove(&NaN) -> {removed:?}");
    println!();
    println!("     three entries, and NOT ONE of them can be read back or removed. They are");
    println!("     not invisible either -- iteration finds them:");
    let mut seen: Vec<(&str, bool)> = m.iter().map(|(k, v)| (*v, is_reflexive(k))).collect();
    seen.sort_unstable();
    for (val, reflexive) in seen {
        println!("       key=NaN value={val:?}  (k == k is {reflexive})");
    }
    let mut z: HashMap<LyingF64, &str> = HashMap::new();
    z.insert(LyingF64(0.0), "positive zero");
    z.insert(LyingF64(-0.0), "negative zero");
    println!(
        "     and +0.0 == -0.0 is {}, so they are ONE key: len = {}, m[0.0] = {:?}",
        0.0 == -0.0f64,
        z.len(),
        z.get(&LyingF64(0.0))
    );
    println!("     identical to Go, line for line -- except that here you had to write the");
    println!("     `impl Eq` that tells the compiler a lie. std calls it a logic error.");
    println!();
    println!("     one difference Go hides: every NaN above hashes to the SAME value (they");
    println!("     are bit-identical), so they pile into one probe sequence and each new");
    println!("     NaN is compared with all the earlier ones. Go hashes each NaN with a");
    println!("     fresh random number (runtime/alg.go) so they spread out. Inserting n NaNs:");
    println!(
        "       {:>8} {:>14} {:>16}",
        "n", "NaN keys ms", "distinct keys ms"
    );
    // best of three, so one preempted run on a shared machine cannot decide a row
    let best = |build: &dyn Fn() -> usize| -> f64 {
        (0..3)
            .map(|_| {
                let t = Instant::now();
                black_box(build());
                t.elapsed().as_secs_f64() * 1e3
            })
            .fold(f64::INFINITY, f64::min)
    };
    let mut ms_rows = Vec::new();
    for n in [4_000usize, 8_000, 16_000] {
        let nan_ms = best(&|| {
            let mut pile: HashMap<LyingF64, usize> = HashMap::new();
            for i in 0..n {
                pile.insert(LyingF64(f64::NAN), i);
            }
            pile.len()
        });
        let distinct_ms = best(&|| {
            let mut spread: HashMap<LyingF64, usize> = HashMap::new();
            for i in 0..n {
                spread.insert(LyingF64(i as f64), i);
            }
            spread.len()
        });
        println!("       {n:>8} {nan_ms:>14.2} {distinct_ms:>16.2}");
        ms_rows.push((nan_ms, distinct_ms));
    }
    println!(
        "     n grew 4x from the first row to the last; the NaN column grew {:.0}x (n squared",
        ms_rows[2].0 / ms_rows[0].0
    );
    println!(
        "     would be 16x) and the distinct column {:.0}x. A float-keyed map fed NaNs does not",
        ms_rows[2].1 / ms_rows[0].1
    );
    println!("     just leak, it slows down.");
    println!();
    println!("  3. refuse NaN at the door, so `Eq` is true:");
    let accepted = [1.5, f64::NAN, 0.0, -0.0];
    let mut clean: HashMap<NotNan, usize> = HashMap::new();
    let mut refused = 0;
    for (i, x) in accepted.into_iter().enumerate() {
        match NotNan::new(x) {
            Some(k) => {
                clean.insert(k, i);
            }
            None => refused += 1,
        }
    }
    println!(
        "     {} floats offered, {refused} refused, {} keys held (+0.0 and -0.0 are one key)",
        accepted.len(),
        clean.len()
    );
    println!("     the fix Go's example recommends, with the compiler holding you to it.");

    println!();
    println!("INTERFACES: `map[any]T` has no Rust spelling, and the panic goes with it:");
    println!();
    println!("      HashMap<Box<dyn Any>, &str>   // insert: E0599, `dyn Any` is not Hash + Eq");
    println!();
    println!("  an open-ended key becomes a CLOSED enum, one variant per accepted type:");
    let mut any1: HashMap<Key, &str> = HashMap::new();
    any1.insert(Key::Int(42), "int");
    any1.insert(Key::Str("42".to_string()), "string");
    any1.insert(Key::Pt(Point { x: 1, y: 2 }), "struct");
    println!("  HashMap<Key, &str> with mixed types: len={}", any1.len());
    println!(
        "  Key::Int(42)={:?}  Key::Str(\"42\")={:?}   <- 42 and \"42\" are different keys",
        any1[&Key::Int(42)],
        any1[&Key::Str("42".to_string())]
    );
    println!();
    println!("  now put a slice in it:");
    any1.insert(Key::Seq(vec![1, 2]), "slice");
    println!(
        "  insert(Key::Seq(vec![1, 2]), \"x\")  ->  fine, len={}, lookup -> {:?}",
        any1.len(),
        any1.get(&Key::Seq(vec![1, 2]))
    );
    println!();
    println!("  nothing panics: the question \"can this be a key?\" was answered when the");
    println!("  enum was declared. Give `Key` a variant holding an f64 and the derive");
    println!("  fails right there. The cost is that the set of key types is closed; a");
    println!("  genuinely open set needs a trait with its own hash method (not shown).");

    println!();
    println!("POINTERS: a reference hashes its REFERENT, the opposite of Go:");
    println!();
    let (pa, pb) = (
        Box::new(Point { x: 1, y: 2 }),
        Box::new(Point { x: 1, y: 2 }),
    );
    println!(
        "  a and b hold equal values: *a == *b is {}, but they are two allocations:",
        *pa == *pb
    );
    println!("  std::ptr::eq(&*a, &*b) is {}", std::ptr::eq(&*pa, &*pb));
    let by_value: HashMap<&Point, &str> = HashMap::from([(&*pa, "first")]);
    println!(
        "  HashMap<&Point, &str>:      pm[a]={:?} and pm[b]={:?}",
        by_value.get(&&*pa),
        by_value.get(&&*pb)
    );
    let addr = |p: &Point| p as *const Point;
    let by_addr: HashMap<*const Point, &str> = HashMap::from([(addr(&pa), "first")]);
    println!(
        "  HashMap<*const Point, &str>: pm[a]={:?} and pm[b]={:?}",
        by_addr.get(&addr(&pa)),
        by_addr.get(&addr(&pb))
    );
    println!();
    println!("  `&T`, `Box<T>` and `Rc<T>` all compare and hash what they point to, so");
    println!("  a Go-style map[*T] needs a raw `*const T` key (safe to make and hash, only");
    println!("  dereferencing is unsafe). A raw pointer key also does NOT keep the value");
    println!("  alive: Go's pointer key does, so its address cannot be reused while it is");
    println!("  in the map, and a Rust raw-pointer key can be left dangling.");

    println!();
    println!("time: the Go trap has no std counterpart:");
    println!();
    let t1 = UNIX_EPOCH + Duration::from_secs(1_767_268_800); // 2026-01-01 12:00:00 UTC
    let t2 = UNIX_EPOCH + Duration::from_millis(1_767_268_800_000);
    println!(
        "  t1 = UNIX_EPOCH + from_secs(1767268800), t2 = UNIX_EPOCH + from_millis(1767268800000)"
    );
    println!("  t1 == t2       -> {}   (same instant)", t1 == t2);
    let tm: HashMap<SystemTime, &str> = HashMap::from([(t1, "new year")]);
    println!("  tm[t1]={:?}  tm[t2]={:?}", tm.get(&t1), tm.get(&t2));
    println!(
        "  Duration::from_millis(1500) == Duration::new(1, 500_000_000) -> {}",
        Duration::from_millis(1500) == Duration::new(1, 500_000_000)
    );
    println!();
    println!("  `SystemTime` and `Duration` hold one normalised value, with no location and");
    println!("  no monotonic reading, and `Instant` (monotonic only) is a different type: two");
    println!("  std times cannot be the same instant yet compare unequal. Go's time.Time");
    println!("  needs `Equal` and `UnixNano()` because it bundles all three in one struct.");
    println!("  A date-time type that carries a zone would come from a crate (not checked here).");

    println!();
    println!("and what the key TYPE costs, since it is not free:");
    println!();
    const N: usize = 200_000;
    println!(
        "  {:<26} {:>12} {:>16}",
        "key type", "ns/lookup", "hash alone ns"
    );
    let ints: Vec<i64> = (0..N as i64).map(|i| i * 2654435761).collect();
    let strs: Vec<String> = (0..N).map(|i| format!("key_{i}_padding")).collect();
    let pts: Vec<Point> = (0..N as i64).map(|i| Point { x: i, y: i * 3 }).collect();
    let arrs: Vec<[i64; 4]> = (0..N as i64).map(|i| [i, i * 2, i * 3, i * 4]).collect();
    // not in the Go table: a wide key, to test "the cost scales with the key"
    let wide: Vec<[i64; 16]> = (0..N as i64).map(|i| [i; 16]).collect();
    let mi: HashMap<i64, usize> = ints.iter().copied().zip(0..).collect();
    let ms: HashMap<&str, usize> = strs.iter().map(String::as_str).zip(0..).collect();
    let mp: HashMap<Point, usize> = pts.iter().copied().zip(0..).collect();
    let ma: HashMap<[i64; 4], usize> = arrs.iter().copied().zip(0..).collect();
    let mw: HashMap<[i64; 16], usize> = wide.iter().copied().zip(0..).collect();
    let per = |total: f64| total / N as f64;
    let t_int = per(best_ns(|| {
        for k in &ints {
            black_box(mi.contains_key(black_box(k)));
        }
    }));
    let t_str = per(best_ns(|| {
        for k in &strs {
            black_box(ms.contains_key(black_box(k.as_str())));
        }
    }));
    let t_pt = per(best_ns(|| {
        for k in &pts {
            black_box(mp.contains_key(black_box(k)));
        }
    }));
    let t_arr = per(best_ns(|| {
        for k in &arrs {
            black_box(ma.contains_key(black_box(k)));
        }
    }));
    let t_wide = per(best_ns(|| {
        for k in &wide {
            black_box(mw.contains_key(black_box(k)));
        }
    }));
    // Hashing alone, with the same default hasher the maps use: `hash_one`
    // runs the key through SipHash-1-3 and nothing else (no table, no `==`).
    let sip = RandomState::new();
    fn hash_alone<K: Hash>(sip: &RandomState, keys: &[K]) -> f64 {
        best_ns(|| {
            for k in keys {
                black_box(sip.hash_one(black_box(k)));
            }
        }) / keys.len() as f64
    }
    let (h_int, h_str) = (hash_alone(&sip, &ints), hash_alone(&sip, &strs));
    let (h_pt, h_arr, h_wide) = (
        hash_alone(&sip, &pts),
        hash_alone(&sip, &arrs),
        hash_alone(&sip, &wide),
    );
    let table = [
        ("i64", t_int, h_int),
        ("&str (14 bytes)", t_str, h_str),
        ("struct {x, y: i64}", t_pt, h_pt),
        ("[i64; 4]", t_arr, h_arr),
        ("[i64; 16] (128 B, added)", t_wide, h_wide),
    ];
    for (name, lookup, hash) in table {
        println!("  {name:<26} {lookup:>12.2} {hash:>16.2}");
    }
    println!();
    println!("  every row is the best of three runs. The gaps BETWEEN the four rows Go compared");
    println!("  change from run to run on a shared machine; the wide row is the stable one:");
    println!(
        "  the 128-byte key costs {:.1}x the i64 one to look up but only {:.1}x to hash, so",
        t_wide / t_int,
        h_wide / h_int
    );
    println!("  hashing every byte is not the whole bill. Its table is also 8x bigger (the key is");
    println!("  stored inline), and a lookup compares 128 bytes; cache misses are a plausible");
    println!("  share of the rest, not separated here.");
    println!();
    println!("  there are no per-type runtime paths here (Go has runtime_fast64.go and");
    println!("  runtime_faststr.go): `HashMap<K, V, S>` is generic, so each key type gets");
    println!("  its own compiled lookup, and every one of them feeds its bytes through the");
    println!("  same `Hasher` (SipHash-1-3 by default) before comparing with `==`.");
    println!("  the key is hashed in full on EVERY lookup, so a wide key pays for every byte,");
    println!("  and the hash column is the part of the cost you can change: swap the hasher.");
    black_box((&ints, &strs, &pts, &arrs, &wide));
}
```

**Rust sample output:**

```
structs and arrays are keys for free, once they derive Hash + Eq:

  HashMap<Point, &str>    Point { x: 1, y: 2 } -> Some("somewhere")
  a missing point         Point { x: 9, y: 9 } -> None (no zero value: `get` returns an Option)
  HashMap<[i64; 3], &str> [4, 5, 6] -> Some("b")
  a 4-field struct        Version { major: 1, minor: 21, patch: 0, pre: "" } -> Some(true)

  this is why example 8's composite key works, and it is almost
  always better than nesting maps: one lookup, one allocation, no
  missing-inner-map case to handle, and the key type documents itself.

which types are keys? asked of the compiler, not asserted:

  type                       Hash + Eq?
  i64                        yes
  String                     yes
  &'static str               yes
  (i64, i64)                 yes
  [u8; 3]                    yes
  Option<i64>                yes
  Vec<String>                yes
  &'static [i64]             yes
  WithVec                    yes
  BTreeMap<i64, i64>         yes
  BTreeSet<i64>              yes
  Rc<String>                 yes
  *const Point               yes
  fn()                       yes
  Duration                   yes
  SystemTime                 yes
  f64                        NO
  f32                        NO
  Option<f64>                NO
  [f64; 2]                   NO
  WithFloat                  NO
  HashMap<i64, i64>          NO
  HashSet<i64>               NO
  Box<dyn Fn()>              NO
  Box<dyn std::any::Any>     NO

  Go refuses slices because a slice is a header pointing at an array, and
  comparing two headers would compare pointers, not contents. Rust made the
  other choice: `Vec<T> == Vec<T>` compares CONTENTS, so a Vec, a slice
  and a struct containing one are all fine keys. What Rust refuses is the
  float, and anything that holds one, plus the unordered std collections
  (HashMap and HashSet have no `Hash` impl, plausibly because equal maps can
  iterate in different orders) and types with no equality at all (closures,
  `dyn Any`). A bare `fn()` pointer IS a key, compared by address (Go: funcs are not comparable).

  a Rust key that Go would not accept, used directly:
  HashMap<Vec<String>, i64>  ["go", "maps", "hash"] -> Some(1)
  looked up through a borrowed slice, no clone: get(tags.as_slice()) -> Some(1)

  what building each candidate key costs, counted by the allocator:
    tags.join(" ")              1 allocation  -> "go maps hash"   (Go's fmt.Sprint workaround)
    [&str; 3] of borrows        0 allocations -> ["go", "maps", "hash"]   (Go's [3]string)
    tags.clone() as Vec<String> 4 allocations -> ["go", "maps", "hash"]   (the owned key above)

  and the string workaround has the ambiguity the array does not:
    ["a b", "c"] and ["a", "b c"]: joined with a space -> 1 key, as arrays -> 2 keys

FLOATS: Rust refuses them as keys, so NaN has to be dealt with on purpose:

      let mut m: HashMap<f64, &str> = HashMap::new();
      m.insert(1.0, "x");   // E0599: insert exists, but its trait bounds were not satisfied
      #[derive(Hash, Eq, PartialEq)] struct S { w: f64 }   // E0277: f64: Hash, f64: Eq

  (the two errors above are quoted from rustc, not provoked by this program.)
  the type `HashMap<f64, _>` can still be NAMED; it is the call that fails.
  Go's silent corruption is therefore not reachable by accident. There are
  three ways to want a float key anyway, and they behave very differently:

  NaN == NaN            -> false

  1. key on `to_bits()`, the integer with the same bits:
     inserts of NaN, NaN, -NaN and a NaN with another payload: len = 3
     m.get(&nan.to_bits()) -> Some("second")   (found: the SAME bits are equal)
     but +0.0 == -0.0 is true and they are 2 keys here (Go: one)
     NaN is found only because its bits happen to match, and there are many
     NaN bit patterns: you have chosen bit-identity, not numeric equality.

  2. wrap the float in a type that claims `Eq` and keeps `==` -- Go's behaviour:
     after three inserts of NaN, len = 3
     m.get(&NaN) -> None, m.remove(&NaN) -> None

     three entries, and NOT ONE of them can be read back or removed. They are
     not invisible either -- iteration finds them:
       key=NaN value="first"  (k == k is false)
       key=NaN value="second"  (k == k is false)
       key=NaN value="third"  (k == k is false)
     and +0.0 == -0.0 is true, so they are ONE key: len = 1, m[0.0] = Some("negative zero")
     identical to Go, line for line -- except that here you had to write the
     `impl Eq` that tells the compiler a lie. std calls it a logic error.

     one difference Go hides: every NaN above hashes to the SAME value (they
     are bit-identical), so they pile into one probe sequence and each new
     NaN is compared with all the earlier ones. Go hashes each NaN with a
     fresh random number (runtime/alg.go) so they spread out. Inserting n NaNs:
              n    NaN keys ms distinct keys ms
           4000           5.20             0.13
           8000          20.78             0.27
          16000          86.94             0.53
     n grew 4x from the first row to the last; the NaN column grew 17x (n squared
     would be 16x) and the distinct column 4x. A float-keyed map fed NaNs does not
     just leak, it slows down.

  3. refuse NaN at the door, so `Eq` is true:
     4 floats offered, 1 refused, 2 keys held (+0.0 and -0.0 are one key)
     the fix Go's example recommends, with the compiler holding you to it.

INTERFACES: `map[any]T` has no Rust spelling, and the panic goes with it:

      HashMap<Box<dyn Any>, &str>   // insert: E0599, `dyn Any` is not Hash + Eq

  an open-ended key becomes a CLOSED enum, one variant per accepted type:
  HashMap<Key, &str> with mixed types: len=3
  Key::Int(42)="int"  Key::Str("42")="string"   <- 42 and "42" are different keys

  now put a slice in it:
  insert(Key::Seq(vec![1, 2]), "x")  ->  fine, len=4, lookup -> Some("slice")

  nothing panics: the question "can this be a key?" was answered when the
  enum was declared. Give `Key` a variant holding an f64 and the derive
  fails right there. The cost is that the set of key types is closed; a
  genuinely open set needs a trait with its own hash method (not shown).

POINTERS: a reference hashes its REFERENT, the opposite of Go:

  a and b hold equal values: *a == *b is true, but they are two allocations:
  std::ptr::eq(&*a, &*b) is false
  HashMap<&Point, &str>:      pm[a]=Some("first") and pm[b]=Some("first")
  HashMap<*const Point, &str>: pm[a]=Some("first") and pm[b]=None

  `&T`, `Box<T>` and `Rc<T>` all compare and hash what they point to, so
  a Go-style map[*T] needs a raw `*const T` key (safe to make and hash, only
  dereferencing is unsafe). A raw pointer key also does NOT keep the value
  alive: Go's pointer key does, so its address cannot be reused while it is
  in the map, and a Rust raw-pointer key can be left dangling.

time: the Go trap has no std counterpart:

  t1 = UNIX_EPOCH + from_secs(1767268800), t2 = UNIX_EPOCH + from_millis(1767268800000)
  t1 == t2       -> true   (same instant)
  tm[t1]=Some("new year")  tm[t2]=Some("new year")
  Duration::from_millis(1500) == Duration::new(1, 500_000_000) -> true

  `SystemTime` and `Duration` hold one normalised value, with no location and
  no monotonic reading, and `Instant` (monotonic only) is a different type: two
  std times cannot be the same instant yet compare unequal. Go's time.Time
  needs `Equal` and `UnixNano()` because it bundles all three in one struct.
  A date-time type that carries a zone would come from a crate (not checked here).

and what the key TYPE costs, since it is not free:

  key type                      ns/lookup    hash alone ns
  i64                                8.04             4.44
  &str (14 bytes)                   13.64             6.20
  struct {x, y: i64}                11.99             6.21
  [i64; 4]                          16.44             8.71
  [i64; 16] (128 B, added)          79.51            23.82

  every row is the best of three runs. The gaps BETWEEN the four rows Go compared
  change from run to run on a shared machine; the wide row is the stable one:
  the 128-byte key costs 9.9x the i64 one to look up but only 5.4x to hash, so
  hashing every byte is not the whole bill. Its table is also 8x bigger (the key is
  stored inline), and a lookup compares 128 bytes; cache misses are a plausible
  share of the rest, not separated here.

  there are no per-type runtime paths here (Go has runtime_fast64.go and
  runtime_faststr.go): `HashMap<K, V, S>` is generic, so each key type gets
  its own compiled lookup, and every one of them feeds its bytes through the
  same `Hasher` (SipHash-1-3 by default) before comparing with `==`.
  the key is hashed in full on EVERY lookup, so a wide key pays for every byte,
  and the hash column is the part of the cost you can change: swap the hasher.
```

> **Go vs Rust:** Go's three traps move rather than disappear: a `Vec<String>` or a struct holding one is a fine key (Rust compares contents, so Go's "slices cannot be keys" has no Rust twin), `f64` and `HashMap` are refused at the `insert` call (E0599, quoted from rustc 1.98.1), and `map[any]` becomes a closed `enum Key`, so the run-time panic has no Rust spelling. Every NaN and ±0 line the two programs share is identical (three entries, none readable, +0.0 and -0.0 one key) but only through a hand-written `impl Eq` that lies, and references hash their referent, so `pm[b]` finds `a`'s entry where Go's pointer key finds nothing. Two measured Rust-only results: with every NaN hashing identically, 16,000 NaN inserts took 17-22x as long as 4,000 (Go hashes each NaN with a random value, read in runtime/alg.go), and a 128-byte key cost 9-17x an `i64` key per lookup (best of three, nine runs).

**Complexity:** `NaN != NaN`, so three inserts make **three entries, none retrievable, none
deletable** · `map[any]T` defers the comparability check to run time — `hash of unhashable type []int`
· pointer keys mean **identity, not equality**, and `time.Time` compares unequal for the same instant

---

## 10. What preallocation actually buys

`🟡 medium` · *not memory, and the size of the win runs backwards*

Example 1 showed the hint saves no memory. So what does it save, how much, and what are the three
related things people get wrong about `clear`, shrinking, and over-hinting?

**Steps:**

1. Measure hinted against unhinted across four orders of magnitude.
2. Measure the **worst single insert**, not the average.
3. Then find out what `clear` keeps, and what a map never gives back.

```go
package main

import (
	"fmt"
	"runtime"
	"testing"
	"time"
)

// Example 5 measured growth on my own table and found it cost 3.9x. Example 1
// found that a size hint does NOT save memory. So what does `make(map, n)`
// actually buy, and how much?
//
// This is the whole answer, on Go's own map, plus the three related questions
// people get wrong: what `clear` does, whether a map ever shrinks, and what a
// rehash does to your tail latency.

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

func main() {
	fmt.Println("what `make(map[K]V, n)` buys, across four orders of magnitude:")
	fmt.Println()
	fmt.Printf("  %12s %16s %16s %12s\n", "n", "no hint ns", "hinted ns", "speedup")
	for _, n := range []int{100, 10_000, 1_000_000, 5_000_000} {
		ks := make([]int, n)
		for i := range ks {
			ks[i] = i * 2654435761
		}
		tNo := nsPerOp(func() {
			m := make(map[int]int)
			for _, k := range ks {
				m[k] = 1
			}
			sinkI = len(m)
		})
		tYes := nsPerOp(func() {
			m := make(map[int]int, n)
			for _, k := range ks {
				m[k] = 1
			}
			sinkI = len(m)
		})
		fmt.Printf("  %12d %16.0f %16.0f %11.2fx\n", n, tNo, tYes, tNo/tYes)
	}
	fmt.Println()
	fmt.Println("  the hint always wins, and the size of the win runs BACKWARDS to")
	fmt.Println("  what I expected: 3.5x at ten thousand keys, 1.2x at five million.")
	fmt.Println()
	fmt.Println("  the saving itself is example 5's -- without a hint the map is")
	fmt.Println("  rebuilt at every doubling, rehashing every key already inserted.")
	fmt.Println("  That work is proportional to n either way.")
	fmt.Println()
	fmt.Println("  what changes is what it is a fraction OF. At 10,000 keys the")
	fmt.Println("  whole map is in cache and rehashing is most of the cost. At five")
	fmt.Println("  million every insert is a cache miss, the build is memory-bound,")
	fmt.Println("  and the rehashing disappears into the stalls. Lesson 04 again:")
	fmt.Println("  an optimisation's VALUE depends on what else is slow.")
	fmt.Println()
	fmt.Println("  it is still the cheapest optimisation in Go -- one extra argument")
	fmt.Println("  to a call you were making anyway. If you know the size, say so.")

	fmt.Println()
	fmt.Println("and here is what it does to TAIL LATENCY, which is the part that")
	fmt.Println("does not show up in an average:")
	fmt.Println()
	const n = 4_000_000
	measure := func(hint int) (worstNS int64, worstAt int) {
		var m map[int]int
		if hint > 0 {
			m = make(map[int]int, hint)
		} else {
			m = make(map[int]int)
		}
		for i := 0; i < n; i++ {
			start := time.Now()
			m[i*2654435761] = i
			if d := time.Since(start).Nanoseconds(); d > worstNS {
				worstNS, worstAt = d, i
			}
		}
		sinkI = len(m)
		return
	}
	wNo, atNo := measure(0)
	wYes, _ := measure(n)
	fmt.Printf("  %-30s worst single insert %9d ns (at insert %d)\n", "no hint", wNo, atNo)
	fmt.Printf("  %-30s worst single insert %9d ns\n", "hinted", wYes)
	fmt.Println()
	fmt.Printf("  one insert out of %d took %.2f ms. Every other insert in that\n",
		n, float64(wNo)/1e6)
	fmt.Println("  run took nanoseconds. If a request happens to be the one that")
	fmt.Println("  triggers the rehash, it pays for every key inserted before it.")
	fmt.Println()
	fmt.Printf("  note the hinted row is not zero either -- %.2f ms. A hint removes\n",
		float64(wYes)/1e6)
	fmt.Println("  the REHASHES, not the allocation: the backing memory is still")
	fmt.Println("  claimed and faulted in, and the GC still runs. Preallocation")
	fmt.Println("  shrinks the tail; it does not delete it.")
	fmt.Println()
	fmt.Println("  this is the same shape as lesson 06's append and lesson 09's")
	fmt.Println("  growable ring: amortized Theta(1) is a statement about totals,")
	fmt.Println("  and it says nothing at all about the worst case.")

	fmt.Println()
	fmt.Println("`clear` versus reassignment -- they are not the same thing:")
	fmt.Println()
	const big = 2_000_000
	kept := liveKB(func() any {
		m := make(map[int]int, big)
		for i := 0; i < big; i++ {
			m[i] = i
		}
		clear(m)
		m[1] = 1
		return m
	})
	fresh := liveKB(func() any {
		m := make(map[int]int, big)
		for i := 0; i < big; i++ {
			m[i] = i
		}
		m = make(map[int]int)
		m[1] = 1
		return m
	})
	fmt.Printf("  %-40s %10.0f KB held, len 1\n", "clear(m)", kept)
	fmt.Printf("  %-40s %10.0f KB held, len 1\n", "m = make(map[int]int)", fresh)
	fmt.Println()
	fmt.Println("  `clear` empties the map and KEEPS every slot it allocated. That")
	fmt.Println("  is the right call when you are about to refill it -- you keep the")
	fmt.Println("  capacity you already paid for and skip the regrow.")
	fmt.Println()
	fmt.Println("  it is the wrong call when you are not, because a Go map NEVER")
	fmt.Println("  SHRINKS. There is no operation that returns the memory except")
	fmt.Println("  dropping the map itself.")

	fmt.Println()
	fmt.Println("proof that a map never shrinks, since it is the trap that matters:")
	fmt.Println()
	shrunk := liveKB(func() any {
		m := make(map[int]int, big)
		for i := 0; i < big; i++ {
			m[i] = i
		}
		for i := 0; i < big-3; i++ {
			delete(m, i)
		}
		return m
	})
	rebuilt := liveKB(func() any {
		m := make(map[int]int, big)
		for i := 0; i < big; i++ {
			m[i] = i
		}
		for i := 0; i < big-3; i++ {
			delete(m, i)
		}
		out := make(map[int]int, len(m))
		for k, v := range m {
			out[k] = v
		}
		return out
	})
	fmt.Printf("  %-40s %10.0f KB held, len 3\n", "2M entries, delete all but 3", shrunk)
	fmt.Printf("  %-40s %10.0f KB held, len 3\n", "...then copy into a fresh map", rebuilt)
	fmt.Println()
	fmt.Printf("  a map holding THREE entries, keeping %.0f KB. The fix is the one\n", shrunk)
	fmt.Println("  the runtime will not do for you: build a new map and copy. That")
	fmt.Println("  is a deliberate policy -- shrinking would make deletes")
	fmt.Println("  unpredictable, so Go leaves the decision to you.")
	fmt.Println()
	fmt.Println("  the practical rule: a long-lived map with high churn needs a")
	fmt.Println("  periodic rebuild, or it holds its high-water mark forever. This")
	fmt.Println("  is lesson 06's slice, lesson 09's queue, and now the map --")
	fmt.Println("  three structures, one policy, and it is always the same fix.")

	fmt.Println()
	fmt.Println("one more: the hint is a HINT, not a reservation:")
	fmt.Println()
	over := liveKB(func() any {
		m := make(map[int]int, 1_000_000)
		for i := 0; i < 10; i++ {
			m[i] = i
		}
		return m
	})
	right := liveKB(func() any {
		m := make(map[int]int, 10)
		for i := 0; i < 10; i++ {
			m[i] = i
		}
		return m
	})
	fmt.Printf("  %-40s %10.1f KB\n", "make(map, 1000000), insert 10", over)
	fmt.Printf("  %-40s %10.1f KB\n", "make(map, 10), insert 10", right)
	fmt.Println()
	fmt.Println("  the hint allocates immediately, so an over-large hint is wasted")
	fmt.Println("  memory that will never be reclaimed. Hint with the number you")
	fmt.Println("  expect, not the number you fear.")

	fmt.Println()
	fmt.Println("the summary:")
	fmt.Println()
	fmt.Printf("  %-34s %s\n", "make(map, n) when n is known", "1.2-3.5x on build, no memory cost")
	fmt.Printf("  %-34s %s\n", "clear(m)", "reuse the capacity")
	fmt.Printf("  %-34s %s\n", "m = make(...)", "return the memory")
	fmt.Printf("  %-34s %s\n", "periodic rebuild", "the only way to shrink")
	fmt.Printf("  %-34s %s\n", "over-hinting", "wasted, permanently")
	sinkB = true
}
```

**Sample output:**

```
what `make(map[K]V, n)` buys, across four orders of magnitude:

             n       no hint ns        hinted ns      speedup
           100             2241              822        2.73x
         10000           295557            84015        3.52x
       1000000         52438303         37051743        1.42x
       5000000        371165514        301159062        1.23x

  the hint always wins, and the size of the win runs BACKWARDS to
  what I expected: 3.5x at ten thousand keys, 1.2x at five million.

  the saving itself is example 5's -- without a hint the map is
  rebuilt at every doubling, rehashing every key already inserted.
  That work is proportional to n either way.

  what changes is what it is a fraction OF. At 10,000 keys the
  whole map is in cache and rehashing is most of the cost. At five
  million every insert is a cache miss, the build is memory-bound,
  and the rehashing disappears into the stalls. Lesson 04 again:
  an optimisation's VALUE depends on what else is slow.

  it is still the cheapest optimisation in Go -- one extra argument
  to a call you were making anyway. If you know the size, say so.

and here is what it does to TAIL LATENCY, which is the part that
does not show up in an average:

  no hint                        worst single insert     55125 ns (at insert 3422086)
  hinted                         worst single insert     14834 ns

  one insert out of 4000000 took 0.06 ms. Every other insert in that
  run took nanoseconds. If a request happens to be the one that
  triggers the rehash, it pays for every key inserted before it.

  note the hinted row is not zero either -- 0.01 ms. A hint removes
  the REHASHES, not the allocation: the backing memory is still
  claimed and faulted in, and the GC still runs. Preallocation
  shrinks the tail; it does not delete it.

  this is the same shape as lesson 06's append and lesson 09's
  growable ring: amortized Theta(1) is a statement about totals,
  and it says nothing at all about the worst case.

`clear` versus reassignment -- they are not the same thing:

  clear(m)                                      73889 KB held, len 1
  m = make(map[int]int)                             0 KB held, len 1

  `clear` empties the map and KEEPS every slot it allocated. That
  is the right call when you are about to refill it -- you keep the
  capacity you already paid for and skip the regrow.

  it is the wrong call when you are not, because a Go map NEVER
  SHRINKS. There is no operation that returns the memory except
  dropping the map itself.

proof that a map never shrinks, since it is the trap that matters:

  2M entries, delete all but 3                  73888 KB held, len 3
  ...then copy into a fresh map                     0 KB held, len 3

  a map holding THREE entries, keeping 73888 KB. The fix is the one
  the runtime will not do for you: build a new map and copy. That
  is a deliberate policy -- shrinking would make deletes
  unpredictable, so Go leaves the decision to you.

  the practical rule: a long-lived map with high churn needs a
  periodic rebuild, or it holds its high-water mark forever. This
  is lesson 06's slice, lesson 09's queue, and now the map --
  three structures, one policy, and it is always the same fix.

one more: the hint is a HINT, not a reservation:

  make(map, 1000000), insert 10               36946.2 KB
  make(map, 10), insert 10                        0.4 KB

  the hint allocates immediately, so an over-large hint is wasted
  memory that will never be reclaimed. Hint with the number you
  expect, not the number you fear.

the summary:

  make(map, n) when n is known       1.2-3.5x on build, no memory cost
  clear(m)                           reuse the capacity
  m = make(...)                      return the memory
  periodic rebuild                   the only way to shrink
  over-hinting                       wasted, permanently
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::collections::hash_map::RandomState;
use std::hash::{BuildHasher, BuildHasherDefault, Hasher};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Example 5 measured growth on my own table and found it cost 3.9x. Example 1
// found that a size hint does NOT save memory. So what does
// `HashMap::with_capacity(n)` actually buy, and how much?
//
// This is the whole answer on std's HashMap (hashbrown's Swiss table, hashed
// with SipHash-1-3 by default), plus the three related questions people get
// wrong: what `clear` does, whether a map ever shrinks, and what a rehash does
// to your tail latency. Rust gives two things Go does not: `capacity()` shows
// what a hint turned into, and `shrink_to_fit()` exists.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Allocation counter: stand-in for runtime.MemStats. There is no GC, so the
// bytes a map holds are exact, and so is the high-water mark during a build.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live
static PEAK: AtomicUsize = AtomicUsize::new(0); // highest LIVE since reset_peak()

fn grew(by: usize) {
    let live = LIVE.fetch_add(by, Relaxed) + by;
    PEAK.fetch_max(live, Relaxed);
}

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        grew(l.size());
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        grew(new);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// bytes currently live on the heap.
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// What building `T` cost: allocation events, bytes it still holds, and the
/// most it held at any moment on the way (all relative to before the build).
struct Footprint {
    allocs: usize,
    held: usize,
    peak: usize,
}

fn footprint<T>(build: impl FnOnce() -> T) -> (Footprint, T) {
    let base = live_bytes();
    PEAK.store(base, Relaxed);
    let a0 = ALLOCS.load(Relaxed);
    let kept = build();
    let fp = Footprint {
        allocs: ALLOCS.load(Relaxed) - a0,
        held: live_bytes() - base,
        peak: PEAK.load(Relaxed) - base,
    };
    (fp, kept)
}

fn kb(bytes: usize) -> f64 {
    bytes as f64 / 1024.0
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

// ---- a cheap hasher, to ask what the hint is worth when rehashing is cheap ----

// std's default SipHash-1-3 costs about 5-10 ns per i64 key (example 1), and a
// rehash hashes every key it moves. The splitmix64 finalizer from example 2
// costs about a nanosecond, so a second set of columns shows how much of what
// the hint buys is the price of the hash.
fn mix(mut k: u64) -> u64 {
    k ^= k >> 30;
    k = k.wrapping_mul(0xbf58476d1ce4e5b9);
    k ^= k >> 27;
    k = k.wrapping_mul(0x94d049bb133111eb);
    k ^= k >> 31;
    k
}

#[derive(Default)]
struct MixHasher(u64);
impl Hasher for MixHasher {
    fn write(&mut self, _: &[u8]) {
        unreachable!("this example only hashes i64 keys");
    }
    fn write_u64(&mut self, k: u64) {
        self.0 = mix(k);
    }
    fn finish(&self) -> u64 {
        self.0
    }
}
type MixBuild = BuildHasherDefault<MixHasher>;

/// Go's `ks[i] = i * 2654435761`: the same spread of keys.
fn keys(n: usize) -> Vec<i64> {
    (0..n as i64).map(|i| i.wrapping_mul(2654435761)).collect()
}

/// Insert every key one at a time, `make(map[int]int, n)` or not. (Not
/// `.collect()`: FromIterator reserves from the iterator's size_hint, which
/// would quietly hand the "no hint" case a hint.)
fn build<S: BuildHasher + Default>(ks: &[i64], hinted: bool) -> HashMap<i64, i64, S> {
    let mut m = if hinted {
        HashMap::with_capacity_and_hasher(ks.len(), S::default())
    } else {
        HashMap::with_hasher(S::default())
    };
    for &k in ks {
        m.insert(k, 1);
    }
    m
}

/// How long one build takes, `drop` (the `free`) included: Go's benchmark
/// pays for the garbage it makes through the GC, this one pays for it here.
/// Best of three `ns_per_op` runs: on a shared machine the minimum is the
/// least-disturbed sample.
fn build_ns<S: BuildHasher + Default>(ks: &[i64], hinted: bool) -> f64 {
    (0..3)
        .map(|_| {
            ns_per_op(|| {
                black_box(build::<S>(black_box(ks), hinted).len());
            })
        })
        .fold(f64::INFINITY, f64::min)
}

/// "2.20x" for one value, "1.82-2.76x" for a range.
fn range(xs: &[f64]) -> String {
    let lo = xs.iter().copied().fold(f64::INFINITY, f64::min);
    let hi = xs.iter().copied().fold(0.0, f64::max);
    if hi - lo < 0.005 {
        format!("{lo:.2}x")
    } else {
        format!("{lo:.2}-{hi:.2}x")
    }
}

/// What one insert loop did to tail latency.
struct Tail {
    /// (insert index, capacity before, capacity after, ns): inserts that resized
    resizes: Vec<(usize, usize, usize, u64)>,
    /// the slowest insert that did NOT resize, and where
    worst_plain: (u64, usize),
    total_ns: u64,
}

fn tail(n: usize, hinted: bool) -> Tail {
    let mut m: HashMap<i64, i64> = if hinted {
        HashMap::with_capacity(n)
    } else {
        HashMap::new()
    };
    let mut cap = m.capacity();
    let mut t = Tail {
        resizes: Vec::new(),
        worst_plain: (0, 0),
        total_ns: 0,
    };
    for i in 0..n {
        let k = (i as i64).wrapping_mul(2654435761);
        let start = Instant::now();
        m.insert(k, i as i64);
        let ns = start.elapsed().as_nanos() as u64;
        t.total_ns += ns;
        let now = m.capacity(); // outside the timed region
        if now != cap {
            t.resizes.push((i, cap, now, ns));
            cap = now;
        } else if ns > t.worst_plain.0 {
            t.worst_plain = (ns, i);
        }
    }
    black_box(m.len());
    t
}

impl Tail {
    fn worst(&self) -> (u64, usize) {
        let resized = self.resizes.iter().map(|r| (r.3, r.0)).max();
        resized.max(Some(self.worst_plain)).unwrap_or_default()
    }
}

fn main() {
    const SIZES: [usize; 4] = [100, 10_000, 1_000_000, 5_000_000];
    println!("what `HashMap::with_capacity(n)` buys, across four orders of magnitude:");
    println!();
    println!(
        "  {:>12} {:>16} {:>16} {:>12}",
        "n", "no hint ns", "hinted ns", "speedup"
    );
    let mut sip = Vec::new(); // (no hint ns, hinted ns) per size, std's SipHash
    let mut cheap = Vec::new(); // the same with the cheap hash
    for n in SIZES {
        let ks = keys(n);
        sip.push((
            build_ns::<RandomState>(&ks, false),
            build_ns::<RandomState>(&ks, true),
        ));
        cheap.push((
            build_ns::<MixBuild>(&ks, false),
            build_ns::<MixBuild>(&ks, true),
        ));
    }
    for (n, (no, yes)) in SIZES.iter().zip(&sip) {
        println!("  {:>12} {:>16.0} {:>16.0} {:>11.2}x", n, no, yes, no / yes);
    }
    let sip_state = RandomState::new();
    let h_sip = ns_per_op(|| {
        black_box(sip_state.hash_one(black_box(12345i64)));
    });
    let h_mix = ns_per_op(|| {
        black_box(mix(black_box(12345u64)));
    });
    println!();
    println!(
        "  the same four rows with a cheap hash. Hashing one i64 costs {h_sip:.1} ns with SipHash and"
    );
    println!(
        "  {h_mix:.1} ns with the splitmix64 finalizer ({:.0}x cheaper):",
        h_sip / h_mix
    );
    println!();
    println!(
        "  {:>12} {:>16} {:>16} {:>12}",
        "n", "no hint ns", "hinted ns", "speedup"
    );
    for (n, (no, yes)) in SIZES.iter().zip(&cheap) {
        println!("  {:>12} {:>16.0} {:>16.0} {:>11.2}x", n, no, yes, no / yes);
    }
    let sip_x: Vec<f64> = sip.iter().map(|(no, yes)| no / yes).collect();
    let cheap_x: Vec<f64> = cheap.iter().map(|(no, yes)| no / yes).collect();
    let wins = |xs: &[f64]| xs.iter().filter(|&&x| x > 1.0).count();
    let cheap_bigger = cheap_x.iter().zip(&sip_x).filter(|(c, s)| c > s).count();
    println!();
    println!(
        "  with std's SipHash the hint wins on {} of the {} sizes, by {}; with the cheap",
        wins(&sip_x),
        SIZES.len(),
        range(&sip_x)
    );
    println!(
        "  hash on {}, by {} (a bigger win than SipHash's on {} of the {} sizes).",
        wins(&cheap_x),
        range(&cheap_x),
        cheap_bigger,
        SIZES.len()
    );
    println!(
        "  Go's sample fell from 3.52x at ten thousand keys to 1.23x at five million, 35% of the"
    );
    println!(
        "  first; here the five-million win is {:.0}% of the ten-thousand one. Go's explanation -- a",
        100.0 * sip_x[3] / sip_x[1]
    );
    println!("  build that is memory-bound at that size hides the rehashing -- is plausible for a");
    println!("  139 MB table (the memory table below), but it is not what was measured here.");
    println!();
    println!("  the saving itself is example 5's -- without a hint the table is rebuilt");
    println!("  at every doubling, rehashing every key already inserted, and the work is");
    println!("  proportional to n either way. A hash several times cheaper leaves the win in the");
    println!("  same range (usually bigger at the small sizes, about the same or a little smaller");
    println!("  at five million), so the price of hashing is not what the hint mostly saves;");
    println!("  plausibly it is the fresh table allocated at each doubling and the entries moved");
    println!("  into it (not separated here). What the hint does NOT change is the memory that is");
    println!("  left at the end; it changes what the build needs on the way there:");
    println!();
    println!(
        "  {:>12} {:>13} {:>13} {:>14} {:>14} {:>13} {:>13}",
        "n",
        "allocs grown",
        "allocs hint",
        "held KB grown",
        "held KB hint",
        "peak KB grown",
        "peak KB hint"
    );
    let mut same_held = true;
    let mut peak_ratio = Vec::new();
    for n in SIZES {
        let ks = keys(n);
        let (g, grown) = footprint(|| build::<RandomState>(&ks, false));
        drop(grown);
        let (h, hinted) = footprint(|| build::<RandomState>(&ks, true));
        drop(hinted);
        println!(
            "  {:>12} {:>13} {:>13} {:>14.1} {:>14.1} {:>13.1} {:>13.1}",
            n,
            g.allocs,
            h.allocs,
            kb(g.held),
            kb(h.held),
            kb(g.peak),
            kb(h.peak)
        );
        same_held &= g.held == h.held;
        peak_ratio.push(g.peak as f64 / h.peak as f64);
    }
    println!();
    println!(
        "  once the build is over the two maps hold {} bytes (example 1's finding, held).",
        if same_held {
            "the same number of"
        } else {
            "DIFFERENT numbers of"
        }
    );
    println!("  But at each doubling the unhinted build holds the old table AND the new one at");
    println!(
        "  once, half plus whole: its peak is {} the hinted one. The hint saves no",
        range(&peak_ratio)
    );
    println!("  memory at the end and does save the high-water mark, and it saves all but one of");
    println!("  the allocations.");
    println!();
    println!("  it is still the cheapest optimisation here -- one extra argument to a call");
    println!("  you were making anyway. If you know the size, say so.");

    println!();
    println!("and here is what it does to TAIL LATENCY, which is the part that");
    println!("does not show up in an average:");
    println!();
    const N: usize = 4_000_000;
    // Seven runs of each, interleaved, keeping the run whose slowest insert is
    // smallest: on a shared machine one preemption lands on a random insert and
    // looks like a table stall, so the least-disturbed run is the honest one.
    let mut runs_no = Vec::new();
    let mut runs_yes = Vec::new();
    for _ in 0..7 {
        runs_no.push(tail(N, false));
        runs_yes.push(tail(N, true));
    }
    // the quietest any run got for an insert that did NOT grow the table
    let quiet_no = runs_no.iter().map(|t| t.worst_plain.0).min().unwrap_or(0);
    let pick = |runs: Vec<Tail>| runs.into_iter().min_by_key(|t| t.worst().0).expect("runs");
    let (no, yes) = (pick(runs_no), pick(runs_yes));
    let (w_no, at_no) = no.worst();
    let (w_yes, at_yes) = yes.worst();
    println!(
        "  {:<30} worst single insert {:>10} ns (at insert {})",
        "no hint", w_no, at_no
    );
    println!(
        "  {:<30} worst single insert {:>10} ns (at insert {})",
        "hinted", w_yes, at_yes
    );
    println!("  (best of 7 runs each, N = {N}; the clock is read around every insert)");
    println!();
    println!(
        "  {} of the {} unhinted inserts grew the table. The five slowest of them:",
        no.resizes.len(),
        N
    );
    let mut by_cost = no.resizes.clone();
    by_cost.sort_by_key(|r| std::cmp::Reverse(r.3));
    println!(
        "    {:>10} {:>22} {:>12} {:>14}",
        "insert", "capacity", "took ms", "ns per key"
    );
    for &(at, before, after, ns) in by_cost.iter().take(5) {
        println!(
            "    {:>10} {:>10} -> {:>9} {:>12.2} {:>14.1}",
            at,
            before,
            after,
            ns as f64 / 1e6,
            ns as f64 / before as f64
        );
    }
    println!("  (\"ns per key\" divides by the capacity before the grow: a full table, every key");
    println!("  of which was rehashed into the new one.)");
    let resize_ns: u64 = no.resizes.iter().map(|r| r.3).sum();
    println!();
    println!(
        "  one insert out of {} took {:.2} ms, and it was the {}.",
        N,
        w_no as f64 / 1e6,
        if by_cost[0].0 == at_no {
            "LAST grow"
        } else {
            "grow before the last"
        }
    );
    println!(
        "  The {} that did not grow the table took at most {:.1} us in the quietest run.",
        N - no.resizes.len(),
        quiet_no as f64 / 1e3
    );
    println!("  The insert that triggers the rehash pays for every key inserted before it,");
    println!("  literally, and in one go.");
    println!();
    println!(
        "  together the {} growing inserts took {:.0} ms of the {:.0} ms the run spent inside",
        no.resizes.len(),
        resize_ns as f64 / 1e6,
        no.total_ns as f64 / 1e6
    );
    println!(
        "  `insert` ({:.0}%); the hinted run spent {:.0} ms there. Both totals include two",
        100.0 * resize_ns as f64 / no.total_ns as f64,
        yes.total_ns as f64 / 1e6
    );
    println!(
        "  clock reads per insert, which is why they are larger than the averages in the first table."
    );
    println!();
    println!(
        "  note the hinted row is not zero either -- {:.3} ms, {:.0}x smaller. A hint removes",
        w_yes as f64 / 1e6,
        w_no as f64 / w_yes as f64
    );
    println!("  the REHASHES, not the allocation: its worst insert is whatever else was slow --");
    println!("  plausibly the first touch of a fresh page, or the scheduler; not separated here.");
    println!("  Preallocation shrinks the tail; it does not delete it.");
    println!();
    println!("  Go's sample above saw 0.06 ms for the same loop, and the difference is the table,");
    println!(
        "  not the language: in the Go 1.27.1 runtime source a table never exceeds 1,024 slots"
    );
    println!("  (`maxTableCapacity` in internal/runtime/maps/table.go) and a full one is SPLIT in");
    println!("  two, so an insert rehashes at most one small table. hashbrown has a single table,");
    println!("  so one insert rehashes all of it (read in the source, not run).");
    println!();
    println!("  this is the same shape as lesson 06's append and lesson 09's");
    println!("  growable ring: amortized Theta(1) is a statement about totals,");
    println!("  and it says nothing at all about the worst case.");
    drop((no, yes));

    println!();
    println!("`clear` versus reassignment -- they are not the same thing:");
    println!();
    const BIG: usize = 2_000_000;
    let fill = |m: &mut HashMap<i64, i64>| {
        for i in 0..BIG as i64 {
            m.insert(i, i);
        }
    };
    let (kept, m) = footprint(|| {
        let mut m = HashMap::with_capacity(BIG);
        fill(&mut m);
        m.clear();
        m.insert(1, 1);
        m
    });
    let kept_cap = m.capacity();
    drop(m);
    let (fresh, m) = footprint(|| {
        let mut m: HashMap<i64, i64> = HashMap::with_capacity(BIG);
        fill(&mut m);
        m = HashMap::new();
        m.insert(1, 1);
        m
    });
    let fresh_cap = m.capacity();
    drop(m);
    println!(
        "  {:<40} {:>10.0} KB held, len 1, capacity() {}",
        "m.clear()",
        kb(kept.held),
        kept_cap
    );
    println!(
        "  {:<40} {:>10.0} KB held, len 1, capacity() {}",
        "m = HashMap::new()",
        kb(fresh.held),
        fresh_cap
    );
    println!();
    println!("  `clear` empties the map and KEEPS every slot it allocated. That");
    println!("  is the right call when you are about to refill it -- you keep the");
    println!("  capacity you already paid for and skip the regrow.");
    println!();
    println!("  it is the wrong call when you are not, because a Rust map does not");
    println!("  shrink ON ITS OWN either: nothing in `remove` or `clear` gives memory back.");
    println!("  Unlike Go there is a method that does, and the next section uses it.");

    println!();
    println!("proof that a map does not shrink by itself, and the two ways to make it:");
    println!();
    let churned = || {
        let mut m: HashMap<i64, i64> = HashMap::with_capacity(BIG);
        fill(&mut m);
        let full_cap = m.capacity();
        for i in 0..(BIG as i64 - 3) {
            m.remove(&i);
        }
        (m, full_cap)
    };
    let base = live_bytes();
    let (mut m, full_cap) = churned();
    let held_deleted = live_bytes() - base;
    let after_cap = m.capacity();
    println!(
        "  {:<40} {:>10.0} KB held, len {}, capacity() {}",
        "2M entries, delete all but 3",
        kb(held_deleted),
        m.len(),
        after_cap
    );
    let t = Instant::now();
    m.shrink_to_fit();
    let shrink_ms = t.elapsed().as_secs_f64() * 1e3;
    println!(
        "  {:<40} {:>10.1} KB held, len {}, capacity() {}   ({:.2} ms)",
        "...then m.shrink_to_fit()",
        kb(live_bytes() - base),
        m.len(),
        m.capacity(),
        shrink_ms
    );
    drop(m);
    let base = live_bytes();
    let (mut old, _) = churned();
    let t = Instant::now();
    let mut out: HashMap<i64, i64> = HashMap::with_capacity(old.len());
    out.extend(old.drain());
    drop(old);
    let copy_ms = t.elapsed().as_secs_f64() * 1e3;
    println!(
        "  {:<40} {:>10.1} KB held, len {}, capacity() {}   ({:.2} ms)",
        "...or copy into a fresh map",
        kb(live_bytes() - base),
        out.len(),
        out.capacity(),
        copy_ms
    );
    drop(out);
    println!();
    println!(
        "  a map holding THREE entries, keeping {:.0} KB. Go's fix -- build a new map and copy --",
        kb(held_deleted)
    );
    println!("  works here too, but Rust also has `shrink_to_fit()` (and `shrink_to(n)`): no copy");
    println!(
        "  loop to write, and the two timings above include freeing the old table. Neither is"
    );
    println!("  automatic: `remove` never reallocates, so it stays cheap and predictable, and the");
    println!("  decision is left to you -- the same policy as Go's, with a tool for it.");
    println!();
    println!(
        "  one oddity: capacity() was {full_cap} before the deletes and {after_cap} after them,"
    );
    if after_cap < full_cap {
        println!("  although the map holds three entries. capacity() is len plus the room left to");
        println!("  insert without growing, and a deleted slot that cannot be marked empty (a");
        println!(
            "  tombstone, example 4; hashbrown's `erase`, read in the vendored source) is not"
        );
        println!("  room. The memory is all still there; only the room to insert shrank.");
    } else {
        println!("  so deleting gave back none of the room: the memory is all still there.");
    }
    println!();
    println!("  the practical rule: a long-lived map with high churn needs a");
    println!("  periodic `shrink_to_fit` (or rebuild), or it holds its high-water mark");
    println!("  forever. This is lesson 06's slice, lesson 09's queue, and now the map --");
    println!("  three structures, one policy, and Vec has a `shrink_to_fit` too.");

    println!();
    println!("one more: the hint is a HINT, and it is rounded UP:");
    println!();
    let ten = |m: &mut HashMap<i64, i64>| {
        for i in 0..10 {
            m.insert(i, i);
        }
    };
    let base = live_bytes();
    let mut m: HashMap<i64, i64> = HashMap::with_capacity(1_000_000);
    ten(&mut m);
    let (over_kb, over_cap) = (kb(live_bytes() - base), m.capacity());
    m.shrink_to_fit();
    let (healed_kb, healed_cap) = (kb(live_bytes() - base), m.capacity());
    drop(m);
    let base = live_bytes();
    let mut m: HashMap<i64, i64> = HashMap::with_capacity(10);
    ten(&mut m);
    let (right_kb, right_cap) = (kb(live_bytes() - base), m.capacity());
    drop(m);
    println!(
        "  {:<44} {:>10.1} KB   capacity() {}",
        "with_capacity(1000000), insert 10", over_kb, over_cap
    );
    println!(
        "  {:<44} {:>10.1} KB   capacity() {}",
        "...then shrink_to_fit()", healed_kb, healed_cap
    );
    println!(
        "  {:<44} {:>10.1} KB   capacity() {}",
        "with_capacity(10), insert 10", right_kb, right_cap
    );
    println!();
    println!("  the hint allocates immediately, so an over-large hint is wasted memory --");
    println!("  but unlike Go's it is not wasted PERMANENTLY: `shrink_to_fit()` returns it.");
    println!("  (These are bytes the allocator handed out, not pages the OS made resident.)");
    println!();
    println!("  and what is allocated is not what was asked for: the table has a power-of-two");
    println!("  number of slots and is never filled past 7/8, so capacity() jumps:");
    println!();
    println!(
        "  {:>10} {:>12} {:>14} {:>10}",
        "hint", "capacity()", "KB allocated", "cap/hint"
    );
    for hint in [1usize, 10, 100, 917_504, 917_505, 1_000_000, 2_000_000] {
        let (fp, m) = footprint(|| HashMap::<i64, i64>::with_capacity(hint));
        println!(
            "  {:>10} {:>12} {:>14.1} {:>9.2}x",
            hint,
            m.capacity(),
            kb(fp.held),
            m.capacity() as f64 / hint as f64
        );
    }
    // the smallest and largest overshoot over every hint from 2 to 20,000
    let (mut lo, mut hi) = ((0usize, usize::MAX), (0usize, 0usize)); // (hint, capacity)
    for hint in 2usize..=20_000 {
        let cap = HashMap::<i64, i64>::with_capacity(hint).capacity();
        if (cap as f64 / hint as f64) < (lo.1 as f64 / lo.0.max(1) as f64) {
            lo = (hint, cap);
        }
        if (cap as f64 / hint as f64) > (hi.1 as f64 / hi.0.max(1) as f64) {
            hi = (hint, cap);
        }
    }
    println!();
    println!(
        "  917,504 and 917,505 differ by one key and by a factor of two in memory. Over every"
    );
    println!(
        "  hint from 2 to 20,000 the smallest capacity()/hint is {:.2}x (hint {}, capacity() {})",
        lo.1 as f64 / lo.0 as f64,
        lo.0,
        lo.1
    );
    println!(
        "  and the largest {:.4}x (hint {}, capacity() {}): you get at least what you asked for,",
        hi.1 as f64 / hi.0 as f64,
        hi.0,
        hi.1
    );
    println!("  and at most twice it. Hint with the number you expect, not the number you fear.");
    println!();
    println!("the summary:");
    println!();
    let build_win = format!(
        "{} on build, {} lower peak, same final size",
        range(&sip_x),
        range(&peak_ratio)
    );
    println!("  {:<34} {build_win}", "with_capacity(n) when n is known");
    println!("  {:<34} reuse the capacity", "clear()");
    println!("  {:<34} return the memory", "m = HashMap::new()");
    println!("  {:<34} return it without a copy loop", "shrink_to_fit()");
    println!(
        "  {:<34} wasted until shrunk; rounded up, at most 2x",
        "over-hinting"
    );
}
```

**Rust sample output:**

```
what `HashMap::with_capacity(n)` buys, across four orders of magnitude:

             n       no hint ns        hinted ns      speedup
           100             2026             1179        1.72x
         10000           266710            83489        3.19x
       1000000         34190780         12321362        2.77x
       5000000        178470042         85951306        2.08x

  the same four rows with a cheap hash. Hashing one i64 costs 5.7 ns with SipHash and
  0.6 ns with the splitmix64 finalizer (10x cheaper):

             n       no hint ns        hinted ns      speedup
           100             1134              370        3.07x
         10000            99329            26864        3.70x
       1000000         19198469          6395597        3.00x
       5000000        103335792         48500733        2.13x

  with std's SipHash the hint wins on 4 of the 4 sizes, by 1.72-3.19x; with the cheap
  hash on 4, by 2.13-3.70x (a bigger win than SipHash's on 4 of the 4 sizes).
  Go's sample fell from 3.52x at ten thousand keys to 1.23x at five million, 35% of the
  first; here the five-million win is 65% of the ten-thousand one. Go's explanation -- a
  build that is memory-bound at that size hides the rehashing -- is plausible for a
  139 MB table (the memory table below), but it is not what was measured here.

  the saving itself is example 5's -- without a hint the table is rebuilt
  at every doubling, rehashing every key already inserted, and the work is
  proportional to n either way. A hash several times cheaper leaves the win in the
  same range (usually bigger at the small sizes, about the same or a little smaller
  at five million), so the price of hashing is not what the hint mostly saves;
  plausibly it is the fresh table allocated at each doubling and the entries moved
  into it (not separated here). What the hint does NOT change is the memory that is
  left at the end; it changes what the build needs on the way there:

             n  allocs grown   allocs hint  held KB grown   held KB hint peak KB grown  peak KB hint
           100             6             1            2.1            2.1           3.2           2.1
         10000            13             1          272.0          272.0         408.0         272.0
       1000000            20             1        34816.0        34816.0       52224.0       34816.0
       5000000            22             1       139264.0       139264.0      208896.0      139264.0

  once the build is over the two maps hold the same number of bytes (example 1's finding, held).
  But at each doubling the unhinted build holds the old table AND the new one at
  once, half plus whole: its peak is 1.50x the hinted one. The hint saves no
  memory at the end and does save the high-water mark, and it saves all but one of
  the allocations.

  it is still the cheapest optimisation here -- one extra argument to a call
  you were making anyway. If you know the size, say so.

and here is what it does to TAIL LATENCY, which is the part that
does not show up in an average:

  no hint                        worst single insert   38303375 ns (at insert 3670016)
  hinted                         worst single insert      14417 ns (at insert 2307)
  (best of 7 runs each, N = 4000000; the clock is read around every insert)

  22 of the 4000000 unhinted inserts grew the table. The five slowest of them:
        insert               capacity      took ms     ns per key
       3670016    3670016 ->   7340032        38.30           10.4
       1835008    1835008 ->   3670016        19.44           10.6
        917504     917504 ->   1835008         9.31           10.1
        458752     458752 ->    917504         4.55            9.9
        229376     229376 ->    458752         2.50           10.9
  ("ns per key" divides by the capacity before the grow: a full table, every key
  of which was rehashed into the new one.)

  one insert out of 4000000 took 38.30 ms, and it was the LAST grow.
  The 3999978 that did not grow the table took at most 6.5 us in the quietest run.
  The insert that triggers the rehash pays for every key inserted before it,
  literally, and in one go.

  together the 22 growing inserts took 76 ms of the 177 ms the run spent inside
  `insert` (43%); the hinted run spent 106 ms there. Both totals include two
  clock reads per insert, which is why they are larger than the averages in the first table.

  note the hinted row is not zero either -- 0.014 ms, 2657x smaller. A hint removes
  the REHASHES, not the allocation: its worst insert is whatever else was slow --
  plausibly the first touch of a fresh page, or the scheduler; not separated here.
  Preallocation shrinks the tail; it does not delete it.

  Go's sample above saw 0.06 ms for the same loop, and the difference is the table,
  not the language: in the Go 1.27.1 runtime source a table never exceeds 1,024 slots
  (`maxTableCapacity` in internal/runtime/maps/table.go) and a full one is SPLIT in
  two, so an insert rehashes at most one small table. hashbrown has a single table,
  so one insert rehashes all of it (read in the source, not run).

  this is the same shape as lesson 06's append and lesson 09's
  growable ring: amortized Theta(1) is a statement about totals,
  and it says nothing at all about the worst case.

`clear` versus reassignment -- they are not the same thing:

  m.clear()                                     69632 KB held, len 1, capacity() 3670016
  m = HashMap::new()                                0 KB held, len 1, capacity() 3

  `clear` empties the map and KEEPS every slot it allocated. That
  is the right call when you are about to refill it -- you keep the
  capacity you already paid for and skip the regrow.

  it is the wrong call when you are not, because a Rust map does not
  shrink ON ITS OWN either: nothing in `remove` or `clear` gives memory back.
  Unlike Go there is a method that does, and the next section uses it.

proof that a map does not shrink by itself, and the two ways to make it:

  2M entries, delete all but 3                  69632 KB held, len 3, capacity() 3325105
  ...then m.shrink_to_fit()                       0.1 KB held, len 3, capacity() 3   (0.70 ms)
  ...or copy into a fresh map                     0.1 KB held, len 3, capacity() 3   (0.72 ms)

  a map holding THREE entries, keeping 69632 KB. Go's fix -- build a new map and copy --
  works here too, but Rust also has `shrink_to_fit()` (and `shrink_to(n)`): no copy
  loop to write, and the two timings above include freeing the old table. Neither is
  automatic: `remove` never reallocates, so it stays cheap and predictable, and the
  decision is left to you -- the same policy as Go's, with a tool for it.

  one oddity: capacity() was 3670016 before the deletes and 3325105 after them,
  although the map holds three entries. capacity() is len plus the room left to
  insert without growing, and a deleted slot that cannot be marked empty (a
  tombstone, example 4; hashbrown's `erase`, read in the vendored source) is not
  room. The memory is all still there; only the room to insert shrank.

  the practical rule: a long-lived map with high churn needs a
  periodic `shrink_to_fit` (or rebuild), or it holds its high-water mark
  forever. This is lesson 06's slice, lesson 09's queue, and now the map --
  three structures, one policy, and Vec has a `shrink_to_fit` too.

one more: the hint is a HINT, and it is rounded UP:

  with_capacity(1000000), insert 10               34816.0 KB   capacity() 1835008
  ...then shrink_to_fit()                             0.3 KB   capacity() 14
  with_capacity(10), insert 10                        0.3 KB   capacity() 14

  the hint allocates immediately, so an over-large hint is wasted memory --
  but unlike Go's it is not wasted PERMANENTLY: `shrink_to_fit()` returns it.
  (These are bytes the allocator handed out, not pages the OS made resident.)

  and what is allocated is not what was asked for: the table has a power-of-two
  number of slots and is never filled past 7/8, so capacity() jumps:

        hint   capacity()   KB allocated   cap/hint
           1            3            0.1      3.00x
          10           14            0.3      1.40x
         100          112            2.1      1.12x
      917504       917504        17408.0      1.00x
      917505      1835008        34816.0      2.00x
     1000000      1835008        34816.0      1.84x
     2000000      3670016        69632.0      1.84x

  917,504 and 917,505 differ by one key and by a factor of two in memory. Over every
  hint from 2 to 20,000 the smallest capacity()/hint is 1.00x (hint 3, capacity() 3)
  and the largest 1.9999x (hint 14337, capacity() 28672): you get at least what you asked for,
  and at most twice it. Hint with the number you expect, not the number you fear.

the summary:

  with_capacity(n) when n is known   1.72-3.19x on build, 1.50x lower peak, same final size
  clear()                            reuse the capacity
  m = HashMap::new()                 return the memory
  shrink_to_fit()                    return it without a copy loop
  over-hinting                       wasted until shrunk; rounded up, at most 2x
```

> **Go vs Rust:** Rust's numbers differ from Go's in three places. The hint wins at every size (1.3-3.3x over ten runs; 2.6-3.3x at 10,000 keys, 1.3-2.1x at 5,000,000, against Go's 3.5x and 1.2x) and it lowers the peak, which the Go example did not measure: an unhinted grow holds the old and the new table together, exactly 1.50x the hinted high-water mark, though the final size is equal. The worst unhinted insert is one whole-table rehash, 28-52 ms at insert 3,670,016 (Go's sample: 0.06 ms, because in the Go 1.27.1 source a table never exceeds 1,024 slots and is split, which I read and did not run), against 12 us-1.7 ms hinted. And Go's "no operation returns the memory except dropping the map" is false here: `shrink_to_fit()` took a map that held 3 of its 2M entries from 69,632 KB to 0.1 KB in about 1 ms, and `capacity()` shows what a hint became (`with_capacity(917_505)` costs twice `with_capacity(917_504)`).

**Complexity:** the hint is worth **3.5× at 10,000 keys and only 1.2× at 5,000,000** — at scale the
build is memory-bound and the rehashing hides in the stalls · a map **never shrinks**: 2,000,000
entries deleted down to 3 still holds **73,888 KB**

---

## 11. Four implementations, five workloads, and a column I forgot

`🟡 medium` · *chaining wins three of four, and is still the wrong answer*

Chaining, linear probing, Go's map and a sorted slice, all behind one interface. The result contradicts
the received wisdom — until you measure the axis a lookup benchmark structurally cannot see.

**Steps:**

1. Build, hit, miss, delete and memory for all four.
2. Notice that chaining wins most of it.
3. Then run the garbage collector with each one live.

```go
package main

import (
	"fmt"
	"runtime"
	"slices"
	"testing"
	"time"
)

// Four implementations of "map from int to int", five workloads, one table.
// The workloads are chosen because they DISAGREE -- a benchmark that only
// measures lookups will tell you the sorted slice is fine, and it will be
// lying by omission.
//
// All four go behind one interface so the call overhead is identical. Lesson
// 09 example 11 is why that matters: the same code measured 1.82 ns/op inline
// and 7.99 behind an interface, and the harness changed the ranking.

type Dict interface {
	Put(k, v int)
	Get(k int) (int, bool)
	Delete(k int)
	Len() int
}

// ---- 1. separate chaining ---------------------------------------------------

type node struct {
	k, v int
	next *node
}

type Chained struct {
	buckets []*node
	n       int
}

func NewChained(hint int) *Chained {
	size := 1
	for size < hint {
		size <<= 1
	}
	return &Chained{buckets: make([]*node, max(size, 8))}
}

func mix(k int) uint64 {
	h := uint64(k)
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

func (c *Chained) idx(k int) int { return int(mix(k) & uint64(len(c.buckets)-1)) }

func (c *Chained) Put(k, v int) {
	if c.n >= len(c.buckets) { // load factor 1.0, the classic choice for chaining
		c.grow()
	}
	i := c.idx(k)
	for e := c.buckets[i]; e != nil; e = e.next {
		if e.k == k {
			e.v = v
			return
		}
	}
	c.buckets[i] = &node{k: k, v: v, next: c.buckets[i]}
	c.n++
}

func (c *Chained) grow() {
	old := c.buckets
	c.buckets = make([]*node, len(old)*2)
	for _, e := range old {
		for e != nil {
			next := e.next
			i := c.idx(e.k)
			e.next = c.buckets[i]
			c.buckets[i] = e
			e = next
		}
	}
}

func (c *Chained) Get(k int) (int, bool) {
	for e := c.buckets[c.idx(k)]; e != nil; e = e.next {
		if e.k == k {
			return e.v, true
		}
	}
	return 0, false
}

func (c *Chained) Delete(k int) {
	pp := &c.buckets[c.idx(k)]
	for *pp != nil {
		if (*pp).k == k {
			*pp = (*pp).next
			c.n--
			return
		}
		pp = &(*pp).next
	}
}

func (c *Chained) Len() int { return c.n }

// ---- 2. open addressing, linear probing -------------------------------------

type slotState uint8

const (
	free slotState = iota
	used
	tomb
)

type Probed struct {
	keys   []int
	vals   []int
	states []slotState
	n      int
	tombs  int
}

func NewProbed(hint int) *Probed {
	size := 8
	for float64(hint) > 0.875*float64(size) {
		size <<= 1
	}
	return &Probed{
		keys:   make([]int, size),
		vals:   make([]int, size),
		states: make([]slotState, size),
	}
}

func (p *Probed) mask() uint64 { return uint64(len(p.keys) - 1) }

func (p *Probed) Put(k, v int) {
	if float64(p.n+p.tombs+1) > 0.875*float64(len(p.keys)) {
		p.rehash()
	}
	p.put(k, v)
}

func (p *Probed) put(k, v int) {
	i := mix(k) & p.mask()
	firstTomb := -1
	for {
		switch p.states[i] {
		case free:
			if firstTomb >= 0 {
				i, p.tombs = uint64(firstTomb), p.tombs-1
			}
			p.keys[i], p.vals[i], p.states[i] = k, v, used
			p.n++
			return
		case tomb:
			if firstTomb < 0 {
				firstTomb = int(i)
			}
		case used:
			if p.keys[i] == k {
				p.vals[i] = v
				return
			}
		}
		i = (i + 1) & p.mask()
	}
}

func (p *Probed) rehash() {
	size := len(p.keys)
	if float64(p.n+1) > 0.875*float64(size)/2 {
		size *= 2
	}
	ok, ov, os := p.keys, p.vals, p.states
	p.keys, p.vals, p.states = make([]int, size), make([]int, size), make([]slotState, size)
	p.n, p.tombs = 0, 0
	for i, s := range os {
		if s == used {
			p.put(ok[i], ov[i])
		}
	}
}

func (p *Probed) Get(k int) (int, bool) {
	i := mix(k) & p.mask()
	for {
		switch p.states[i] {
		case free:
			return 0, false
		case used:
			if p.keys[i] == k {
				return p.vals[i], true
			}
		}
		i = (i + 1) & p.mask()
	}
}

func (p *Probed) Delete(k int) {
	i := mix(k) & p.mask()
	for {
		switch p.states[i] {
		case free:
			return
		case used:
			if p.keys[i] == k {
				p.states[i] = tomb
				p.n--
				p.tombs++
				return
			}
		}
		i = (i + 1) & p.mask()
	}
}

func (p *Probed) Len() int { return p.n }

// ---- 3. Go's own map --------------------------------------------------------

type GoMap struct{ m map[int]int }

func NewGoMap(hint int) *GoMap         { return &GoMap{m: make(map[int]int, hint)} }
func (g *GoMap) Put(k, v int)          { g.m[k] = v }
func (g *GoMap) Get(k int) (int, bool) { v, ok := g.m[k]; return v, ok }
func (g *GoMap) Delete(k int)          { delete(g.m, k) }
func (g *GoMap) Len() int              { return len(g.m) }

// ---- 4. sorted slice + binary search ----------------------------------------

type pair struct{ k, v int }

type Sorted struct{ s []pair }

func NewSorted(hint int) *Sorted { return &Sorted{s: make([]pair, 0, hint)} }

func (s *Sorted) find(k int) (int, bool) {
	return slices.BinarySearchFunc(s.s, k, func(p pair, k int) int { return p.k - k })
}

func (s *Sorted) Put(k, v int) {
	i, ok := s.find(k)
	if ok {
		s.s[i].v = v
		return
	}
	s.s = slices.Insert(s.s, i, pair{k, v}) // Theta(n) -- this is the point
}

func (s *Sorted) Get(k int) (int, bool) {
	if i, ok := s.find(k); ok {
		return s.s[i].v, true
	}
	return 0, false
}

func (s *Sorted) Delete(k int) {
	if i, ok := s.find(k); ok {
		s.s = slices.Delete(s.s, i, i+1)
	}
}

func (s *Sorted) Len() int { return len(s.s) }

// ---- harness ----------------------------------------------------------------

type impl struct {
	name string
	new  func(hint int) Dict
}

var impls = []impl{
	{"chaining", func(h int) Dict { return NewChained(h) }},
	{"linear probing", func(h int) Dict { return NewProbed(h) }},
	{"Go map (swiss)", func(h int) Dict { return NewGoMap(h) }},
	{"sorted slice", func(h int) Dict { return NewSorted(h) }},
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

// gcCost measures what a live structure costs the COLLECTOR, which is the axis
// a lookup benchmark cannot see.
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

func main() {
	const n = 200_000
	present := make([]int, n)
	absent := make([]int, n)
	for i := 0; i < n; i++ {
		present[i] = i*2654435761 + 1
		absent[i] = -(i*2654435761 + 1)
	}
	// The sorted slice is Theta(n) per insert, so building it 200,000 times
	// would dominate the whole program. Build it once at a smaller size.
	const sortedN = 20_000

	build := func(im impl, keys []int) Dict {
		d := im.new(len(keys))
		for i, k := range keys {
			d.Put(k, i)
		}
		return d
	}

	fmt.Println("four dictionaries, 200,000 int keys, all behind one interface.")
	fmt.Println()
	fmt.Printf("  %-18s %13s %11s %11s %12s %11s\n",
		"implementation", "build ns/op", "hit ns", "miss ns", "delete ns", "KB")
	for _, im := range impls {
		keys, m := present, n
		note := ""
		if im.name == "sorted slice" {
			keys, m = present[:sortedN], sortedN
			note = "*"
		}
		miss := absent[:m]

		tBuild := nsPerOp(func() { sinkI = build(im, keys).Len() }) / float64(m)
		d := build(im, keys)
		tHit := nsPerOp(func() {
			for _, k := range keys {
				_, sinkB = d.Get(k)
			}
		}) / float64(m)
		tMiss := nsPerOp(func() {
			for _, k := range miss {
				_, sinkB = d.Get(k)
			}
		}) / float64(m)
		tDel := nsPerOp(func() {
			dd := build(im, keys)
			for _, k := range keys {
				dd.Delete(k)
			}
			sinkI = dd.Len()
		})/float64(m) - tBuild
		kb := liveKB(func() any { return build(im, keys) })

		fmt.Printf("  %-18s %13.1f %11.1f %11.1f %12.1f %10.0f%s\n",
			im.name, tBuild, tHit, tMiss, tDel, kb, note)
	}
	fmt.Printf("\n  * the sorted slice was built at %d keys, not %d -- its insert is\n", sortedN, n)
	fmt.Println("    Theta(n), so building it at 200,000 takes minutes. That is not")
	fmt.Println("    a footnote, it is the result: the row cannot be run.")

	fmt.Println()
	fmt.Println("reading the table, and it does not say what I expected:")
	fmt.Println()
	fmt.Println("  CHAINING WINS three of the four speed columns -- hit, miss and")
	fmt.Println("  delete. I had written the opposite into this file before running")
	fmt.Println("  it, on the strength of 'everyone moved to open addressing'.")
	fmt.Println()
	fmt.Println("  the reason it wins MISS is example 3's arithmetic: at load factor")
	fmt.Println("  1.0 about 37% of buckets are empty, so a miss usually loads one")
	fmt.Println("  nil pointer and stops. Linear probing at 87.5% load has to walk a")
	fmt.Println("  cluster before it finds a free slot -- that is example 4's table.")
	fmt.Println()
	fmt.Println("  so on this workload -- int keys, a good hash, no churn, and a")
	fmt.Println("  benchmark loop that never collects -- chaining is genuinely fine.")
	fmt.Println("  The table is not wrong. It is incomplete.")

	fmt.Println()
	fmt.Println("here is the column it was missing:")
	fmt.Println()
	fmt.Printf("  %-18s %16s %20s\n", "implementation", "heap objects", "ms per GC cycle")
	for _, im := range impls {
		keys := present
		if im.name == "sorted slice" {
			keys = present[:sortedN]
		}
		objs, msc := gcCost(func() any { return build(im, keys) })
		fmt.Printf("  %-18s %16d %20.3f\n", im.name, objs, msc)
	}
	fmt.Println()
	fmt.Println("  chaining holds one heap object PER ENTRY, and the garbage")
	fmt.Println("  collector must trace every one of them on every cycle, forever,")
	fmt.Println("  whether or not the map is being used. The other three hold a")
	fmt.Println("  handful of arrays.")
	fmt.Println()
	fmt.Println("  that is the answer to 'why did everyone move to open addressing',")
	fmt.Println("  and a lookup benchmark structurally cannot show it -- the cost is")
	fmt.Println("  not paid by the lookup. It is paid by every OTHER part of your")
	fmt.Println("  program, at a time you do not control.")
	fmt.Println()
	fmt.Println("  lesson 04 measured this as 59x GC work for []*T over []T, and")
	fmt.Println("  lesson 07 as 149,997 heap objects for a 100,000-element list.")
	fmt.Println("  Same finding, third structure.")
	fmt.Println()
	fmt.Println("  the rest of the table, briefly: linear probing uses the least")
	fmt.Println("  memory and has the worst miss (clustering). Go's map has the best")
	fmt.Println("  hit and near-zero GC cost, which is the combination you want.")

	fmt.Println("the workload that reverses the ranking -- ITERATION IN ORDER:")
	fmt.Println()
	const it = 20_000
	sortedD := NewSorted(it)
	goD := NewGoMap(it)
	for i := 0; i < it; i++ {
		sortedD.Put(present[i], i)
		goD.Put(present[i], i)
	}
	tSortedIter := nsPerOp(func() {
		for _, p := range sortedD.s {
			sinkI = p.k
		}
	})
	tMapIter := nsPerOp(func() {
		ks := make([]int, 0, len(goD.m))
		for k := range goD.m {
			ks = append(ks, k)
		}
		slices.Sort(ks)
		sinkI = len(ks)
	})
	fmt.Printf("  %-40s %14.0f ns\n", "sorted slice: just range it", tSortedIter)
	fmt.Printf("  %-40s %14.0f ns   %.0fx slower\n",
		"map: collect keys, then sort", tMapIter, tMapIter/tSortedIter)
	fmt.Println()
	fmt.Println("  the sorted slice is ALREADY in order, so this workload is a scan")
	fmt.Println("  of contiguous memory. The map has to materialise every key and")
	fmt.Println("  sort them, every single time.")
	fmt.Println()
	fmt.Println("  and a sorted slice answers questions a map cannot answer at all:")
	fmt.Println("  the smallest key above x, the 100 keys in a range, the median.")
	fmt.Println("  Those are Theta(log n) on a slice and Theta(n) on a map.")

	fmt.Println()
	fmt.Println("the verdict:")
	fmt.Println()
	fmt.Printf("  %-18s %-32s %s\n", "implementation", "use it when", "never when")
	for _, r := range [][3]string{
		{"Go map", "essentially always", "you need ordering or bounds"},
		{"chaining", "you are writing one to learn", "the GC has to trace it"},
		{"linear probing", "you need control over layout", "deletes churn heavily"},
		{"sorted slice", "small, or read-mostly and ordered", "inserts are frequent"},
	} {
		fmt.Printf("  %-18s %-32s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  the honest summary of this whole lesson: WRITE `map[K]V`. The")
	fmt.Println("  three hand-written tables above exist so you can say why the")
	fmt.Println("  built-in one is shaped the way it is -- not because you should")
	fmt.Println("  ship one.")
	fmt.Println()
	fmt.Println("  the exception that is worth the trouble is a specialised open-")
	fmt.Println("  addressed table for a hot inner loop with a known key type and")
	fmt.Println("  size. Example 16 builds one and measures whether it was worth it.")
}
```

**Sample output:**

```
four dictionaries, 200,000 int keys, all behind one interface.

  implementation       build ns/op      hit ns     miss ns    delete ns          KB
  chaining                    20.9         7.2        10.2          9.4       6736
  linear probing              11.7        10.1        19.4          9.1       4352
  Go map (swiss)              12.6         6.9        15.4         22.5       4618
  sorted slice                20.4        41.2        21.0       2109.8        320*

  * the sorted slice was built at 20000 keys, not 200000 -- its insert is
    Theta(n), so building it at 200,000 takes minutes. That is not
    a footnote, it is the result: the row cannot be run.

reading the table, and it does not say what I expected:

  CHAINING WINS three of the four speed columns -- hit, miss and
  delete. I had written the opposite into this file before running
  it, on the strength of 'everyone moved to open addressing'.

  the reason it wins MISS is example 3's arithmetic: at load factor
  1.0 about 37% of buckets are empty, so a miss usually loads one
  nil pointer and stops. Linear probing at 87.5% load has to walk a
  cluster before it finds a free slot -- that is example 4's table.

  so on this workload -- int keys, a good hash, no churn, and a
  benchmark loop that never collects -- chaining is genuinely fine.
  The table is not wrong. It is incomplete.

here is the column it was missing:

  implementation         heap objects      ms per GC cycle
  chaining                     200404                2.371
  linear probing                  409                0.079
  Go map (swiss)                  923                0.088
  sorted slice                    412                0.083

  chaining holds one heap object PER ENTRY, and the garbage
  collector must trace every one of them on every cycle, forever,
  whether or not the map is being used. The other three hold a
  handful of arrays.

  that is the answer to 'why did everyone move to open addressing',
  and a lookup benchmark structurally cannot show it -- the cost is
  not paid by the lookup. It is paid by every OTHER part of your
  program, at a time you do not control.

  lesson 04 measured this as 59x GC work for []*T over []T, and
  lesson 07 as 149,997 heap objects for a 100,000-element list.
  Same finding, third structure.

  the rest of the table, briefly: linear probing uses the least
  memory and has the worst miss (clustering). Go's map has the best
  hit and near-zero GC cost, which is the combination you want.
the workload that reverses the ranking -- ITERATION IN ORDER:

  sorted slice: just range it                        5062 ns
  map: collect keys, then sort                     948179 ns   187x slower

  the sorted slice is ALREADY in order, so this workload is a scan
  of contiguous memory. The map has to materialise every key and
  sort them, every single time.

  and a sorted slice answers questions a map cannot answer at all:
  the smallest key above x, the 100 keys in a range, the median.
  Those are Theta(log n) on a slice and Theta(n) on a map.

the verdict:

  implementation     use it when                      never when
  Go map             essentially always               you need ordering or bounds
  chaining           you are writing one to learn     the GC has to trace it
  linear probing     you need control over layout     deletes churn heavily
  sorted slice       small, or read-mostly and ordered inserts are frequent

  the honest summary of this whole lesson: WRITE `map[K]V`. The
  three hand-written tables above exist so you can say why the
  built-in one is shaped the way it is -- not because you should
  ship one.

  the exception that is worth the trouble is a specialised open-
  addressed table for a hot inner loop with a known key type and
  size. Example 16 builds one and measures whether it was worth it.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{BTreeMap, HashMap};
use std::hash::{BuildHasher, BuildHasherDefault, Hasher, RandomState};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Four implementations of "map from int to int", five workloads, one table.
// The workloads are chosen because they DISAGREE -- a benchmark that only
// measures lookups will tell you the sorted slice is fine, and it will be
// lying by omission.
//
// Every row goes behind one trait object so the call overhead is identical.
// Lesson 09 example 11 is why that matters: the same code measured 1.82 ns/op
// inline and 7.99 behind an interface in Go, and the harness changed the
// ranking. `Box<dyn Dict>` is Rust's interface value: a pointer plus a vtable.
//
// Rust's std `HashMap` is a Swiss table (hashbrown) like Go's map, but it
// appears TWICE below, because "the built-in map" is two things in Rust:
//   - with its default hasher, SipHash-1-3 keyed by a random per-map seed
//     (built to resist the attack of example 14, and slower for it), and
//   - with the same `mix` hash the two hand-written tables use, so that the
//     only difference between those rows is the table.
// Go has no such choice: its map's hash is built in.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Allocation counter: stand-in for runtime.MemStats. Rust has no GC, so "how
// many heap blocks, how many bytes" is an exact number. Go reports
// HeapObjects, which includes the runtime's own ~400; this counts only ours.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live
static BLOCKS: AtomicUsize = AtomicUsize::new(0); // heap blocks currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BLOCKS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        BLOCKS.fetch_sub(1, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events while running `f`, its result).
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}
/// bytes currently live on the heap (a deterministic MemStats.HeapAlloc).
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}
/// heap blocks currently live (a deterministic MemStats.HeapObjects).
fn live_blocks() -> usize {
    BLOCKS.load(Relaxed)
}

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`, as the FASTEST of
/// three 100 ms windows. This machine is shared and loaded, and interference
/// only ever adds time, so the minimum is the least-contaminated reading. Go's
/// `testing.Benchmark` reports one run.
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(100);
    let mut best = f64::MAX;
    for _ in 0..3 {
        let mut n: u64 = 1;
        let ns = loop {
            let t = Instant::now();
            for _ in 0..n {
                f();
            }
            let el = t.elapsed();
            if el >= target || n >= 1 << 40 {
                break el.as_nanos() as f64 / n as f64;
            }
            let per = el.as_nanos().max(1) as f64 / n as f64;
            n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
        };
        best = best.min(ns);
    }
    best
}

// ns per run of `run`, timing ONLY `run`, as the fastest of 5 to 30 runs (it stops
// at 150 ms of timed work, or at 30 runs when `run` is much cheaper than `setup`,
// as a single `drop` is). `setup` builds its input outside the stopwatch and the
// value `run` returns is dropped outside it too. Go's harness times
// `build + delete` and subtracts the build, and its collector frees the garbage
// somewhere else, on its own time. Here freeing is deterministic and happens at
// a known line, so the harness can leave it out (or, for the drop column below,
// time nothing else).
fn timed_ns<S, R>(mut setup: impl FnMut() -> S, mut run: impl FnMut(S) -> R) -> f64 {
    let target = Duration::from_millis(150);
    let (mut total, mut runs, mut best) = (Duration::ZERO, 0u32, Duration::MAX);
    while runs < 5 || (total < target && runs < 30) {
        let input = black_box(setup());
        let t = Instant::now();
        let out = run(input);
        let el = t.elapsed();
        total += el;
        best = best.min(el);
        drop(black_box(out));
        runs += 1;
    }
    best.as_nanos() as f64
}

// ---- the hash: the splitmix64 finalizer from example 2 ----

// Bit-identical to Go's `mix`: every shift and multiply is the same, and the
// multiplies wrap explicitly (Go wraps silently).
fn mix(k: i64) -> u64 {
    let mut h = k as u64;
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58476d1ce4e5b9);
    h ^= h >> 27;
    h = h.wrapping_mul(0x94d049bb133111eb);
    h ^= h >> 31;
    h
}

// The same function as a `std::hash::Hasher`, so std's HashMap can use the
// very hash the hand-written tables use (as in example 5). `i64::hash` calls
// `write_i64`, whose default implementation calls `write_u64`.
#[derive(Default)]
struct MixHasher(u64);
impl Hasher for MixHasher {
    fn write(&mut self, _: &[u8]) {
        unreachable!("this example only hashes integer keys");
    }
    fn write_u64(&mut self, k: u64) {
        self.0 = mix(k as i64);
    }
    fn finish(&self) -> u64 {
        self.0
    }
}

trait Dict {
    fn put(&mut self, k: i64, v: i64);
    // Go returns (int, bool); Rust returns the pair as one type.
    fn get(&self, k: i64) -> Option<i64>;
    fn delete(&mut self, k: i64);
    fn len(&self) -> usize;
    // Not in the Go interface: visit every entry once and sum it. It stands in
    // for "everything that has to touch all of the table", which in Go includes
    // the collector's mark phase, and in Rust includes iteration and drop.
    fn scan(&self) -> i64;
}

// ---- 1. separate chaining ---------------------------------------------------

struct Node {
    k: i64,
    v: i64,
    next: Link,
}
type Link = Option<Box<Node>>;

struct Chained {
    buckets: Vec<Link>,
    n: usize,
}

impl Chained {
    fn new(hint: usize) -> Self {
        // Go: grow `size` by doubling until it reaches the hint; at least 8.
        let size = hint.next_power_of_two().max(8);
        Chained {
            buckets: (0..size).map(|_| None).collect(),
            n: 0,
        }
    }

    fn idx(&self, k: i64) -> usize {
        (mix(k) & (self.buckets.len() as u64 - 1)) as usize
    }

    fn grow(&mut self) {
        let size = self.buckets.len() * 2;
        let old = std::mem::replace(&mut self.buckets, (0..size).map(|_| None).collect());
        // Relink every node into the new array: no allocation, no key copied.
        // Go's loop is `next := e.next; e.next = new[i]; new[i] = e; e = next`;
        // `take()` is how Rust detaches the tail before the node moves.
        for mut head in old {
            while let Some(mut e) = head {
                head = e.next.take();
                let i = self.idx(e.k);
                e.next = self.buckets[i].take();
                self.buckets[i] = Some(e);
            }
        }
    }

    // Nodes a lookup of `k` visits: the chain up to the match, or all of it.
    // Not in Go: the counted quantity behind the timings.
    fn links(&self, k: i64) -> usize {
        let mut e = self.buckets[self.idx(k)].as_deref();
        let mut visited = 0;
        while let Some(node) = e {
            visited += 1;
            if node.k == k {
                break;
            }
            e = node.next.as_deref();
        }
        visited
    }

    // (empty buckets, longest chain). Not in Go.
    fn shape(&self) -> (usize, usize) {
        let lens: Vec<usize> = self
            .buckets
            .iter()
            .map(|b| std::iter::successors(b.as_deref(), |e| e.next.as_deref()).count())
            .collect();
        let empty = lens.iter().filter(|&&l| l == 0).count();
        (empty, lens.iter().copied().max().unwrap_or(0))
    }
}

// No `Drop` impl here. Example 3 needed an iterative one because its worst case
// is one chain a million boxes long, and dropping that recursively overflowed
// the 8 MB stack. A good hash at load <= 1 gives chains a handful of nodes
// long (`shape()` prints the longest), so the default recursive drop is at most
// that many frames deep.
impl Dict for Chained {
    fn put(&mut self, k: i64, v: i64) {
        if self.n >= self.buckets.len() {
            self.grow(); // load factor 1.0, the classic choice for chaining
        }
        let i = self.idx(k);
        let mut cur = self.buckets[i].as_deref_mut();
        while let Some(e) = cur {
            if e.k == k {
                e.v = v;
                return;
            }
            cur = e.next.as_deref_mut();
        }
        // prepend: the old head becomes the new node's `next`
        let next = self.buckets[i].take();
        self.buckets[i] = Some(Box::new(Node { k, v, next }));
        self.n += 1;
    }

    fn get(&self, k: i64) -> Option<i64> {
        let mut e = self.buckets[self.idx(k)].as_deref();
        while let Some(node) = e {
            if node.k == k {
                return Some(node.v);
            }
            e = node.next.as_deref();
        }
        None
    }

    // Go walks a pointer to the LINK (`pp := &c.buckets[i]`). The Rust twin is
    // a `&mut Option<Box<Node>>` cursor, and the direct transliteration of Go's
    // loop is rejected by the borrow checker (example 3 shows the error), so
    // the loop condition only peeks, and the node is `take()`n before the link
    // is rewritten.
    fn delete(&mut self, k: i64) {
        let i = self.idx(k);
        let mut pp: &mut Link = &mut self.buckets[i];
        while pp.as_ref().is_some_and(|e| e.k != k) {
            pp = &mut pp.as_mut().unwrap().next;
        }
        if let Some(doomed) = pp.take() {
            // The node is freed HERE, as `doomed` goes out of scope: delete
            // pays for the free, where Go leaves it to the collector.
            *pp = doomed.next;
            self.n -= 1;
        }
    }

    fn len(&self) -> usize {
        self.n
    }

    fn scan(&self) -> i64 {
        let mut sum = 0i64;
        for b in &self.buckets {
            let mut e = b.as_deref();
            while let Some(node) = e {
                sum = sum.wrapping_add(node.k).wrapping_add(node.v);
                e = node.next.as_deref();
            }
        }
        sum
    }
}

// ---- 2. open addressing, linear probing -------------------------------------

#[derive(Clone, Copy, PartialEq)]
enum State {
    Free,
    Used,
    Tomb,
}

// Go keeps THREE PARALLEL ARRAYS (8-byte key, 8-byte value, 1-byte state) and
// so does this (as in example 5): 17 bytes a slot. One `Vec` of an enum would
// be tidier and `size_of::<Slot>()` bytes, which would make this row's KB
// column about 40% worse for a reason that has nothing to do with probing.
struct Probed {
    keys: Vec<i64>,
    vals: Vec<i64>,
    states: Vec<State>,
    n: usize,
    tombs: usize,
}

impl Probed {
    fn new(hint: usize) -> Self {
        let mut size = 8;
        while hint as f64 > 0.875 * size as f64 {
            size <<= 1;
        }
        Probed {
            keys: vec![0; size],
            vals: vec![0; size],
            states: vec![State::Free; size],
            n: 0,
            tombs: 0,
        }
    }

    fn mask(&self) -> usize {
        self.keys.len() - 1
    }

    fn home(&self, k: i64) -> usize {
        (mix(k) & self.mask() as u64) as usize
    }

    fn place(&mut self, k: i64, v: i64) {
        let mut i = self.home(k);
        let mut first_tomb: Option<usize> = None; // Go: -1 means "none yet"
        loop {
            match self.states[i] {
                State::Free => {
                    if let Some(t) = first_tomb {
                        i = t;
                        self.tombs -= 1;
                    }
                    self.keys[i] = k;
                    self.vals[i] = v;
                    self.states[i] = State::Used;
                    self.n += 1;
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

    fn rehash(&mut self) {
        let mut size = self.keys.len();
        if (self.n + 1) as f64 > 0.875 * size as f64 / 2.0 {
            size *= 2;
        }
        let old_k = std::mem::replace(&mut self.keys, vec![0; size]);
        let old_v = std::mem::replace(&mut self.vals, vec![0; size]);
        let old_s = std::mem::replace(&mut self.states, vec![State::Free; size]);
        self.n = 0;
        self.tombs = 0;
        for ((&k, &v), &s) in old_k.iter().zip(&old_v).zip(&old_s) {
            if s == State::Used {
                self.place(k, v);
            }
        }
    }

    // Slots a lookup of `k` examines: up to and including the match, or the
    // FREE slot that ends a miss. Not in Go.
    fn probes(&self, k: i64) -> usize {
        let mut i = self.home(k);
        let mut examined = 0;
        loop {
            examined += 1;
            match self.states[i] {
                State::Free => return examined,
                State::Used if self.keys[i] == k => return examined,
                State::Used | State::Tomb => {}
            }
            i = (i + 1) & self.mask();
        }
    }
}

impl Dict for Probed {
    fn put(&mut self, k: i64, v: i64) {
        if (self.n + self.tombs + 1) as f64 > 0.875 * self.keys.len() as f64 {
            self.rehash();
        }
        self.place(k, v);
    }

    fn get(&self, k: i64) -> Option<i64> {
        let mut i = self.home(k);
        loop {
            match self.states[i] {
                State::Free => return None,
                State::Used if self.keys[i] == k => return Some(self.vals[i]),
                State::Used | State::Tomb => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    fn delete(&mut self, k: i64) {
        let mut i = self.home(k);
        loop {
            match self.states[i] {
                State::Free => return,
                State::Used if self.keys[i] == k => {
                    self.states[i] = State::Tomb;
                    self.n -= 1;
                    self.tombs += 1;
                    return;
                }
                State::Used | State::Tomb => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    fn len(&self) -> usize {
        self.n
    }

    fn scan(&self) -> i64 {
        let mut sum = 0i64;
        for ((&k, &v), &s) in self.keys.iter().zip(&self.vals).zip(&self.states) {
            if s == State::Used {
                sum = sum.wrapping_add(k).wrapping_add(v);
            }
        }
        sum
    }
}

// ---- 3. std's own map -------------------------------------------------------

// Generic over the hasher, so the same code is both std rows: `RandomState`
// (SipHash-1-3, random seed) and `BuildHasherDefault<MixHasher>` (the hash the
// hand-written tables use).
struct Std<S> {
    m: HashMap<i64, i64, S>,
}

impl<S: BuildHasher + Default> Std<S> {
    fn new(hint: usize) -> Self {
        Std {
            m: HashMap::with_capacity_and_hasher(hint, S::default()),
        }
    }
}

impl<S: BuildHasher> Dict for Std<S> {
    fn put(&mut self, k: i64, v: i64) {
        self.m.insert(k, v);
    }
    fn get(&self, k: i64) -> Option<i64> {
        self.m.get(&k).copied()
    }
    fn delete(&mut self, k: i64) {
        self.m.remove(&k);
    }
    fn len(&self) -> usize {
        self.m.len()
    }
    fn scan(&self) -> i64 {
        self.m
            .iter()
            .fold(0i64, |s, (&k, &v)| s.wrapping_add(k).wrapping_add(v))
    }
}

// ---- 4. sorted Vec + binary search ------------------------------------------

type Pair = (i64, i64);

struct Sorted {
    s: Vec<Pair>,
}

impl Sorted {
    fn new(hint: usize) -> Self {
        Sorted {
            s: Vec::with_capacity(hint),
        }
    }

    // Go returns (index, found). Rust's `binary_search` returns the same
    // information as a `Result`: Ok(index of the key) or Err(where it would go).
    fn find(&self, k: i64) -> Result<usize, usize> {
        self.s.binary_search_by_key(&k, |p| p.0)
    }
}

impl Dict for Sorted {
    fn put(&mut self, k: i64, v: i64) {
        match self.find(k) {
            Ok(i) => self.s[i].1 = v,
            Err(i) => self.s.insert(i, (k, v)), // Theta(n): shifts everything after i
        }
    }
    fn get(&self, k: i64) -> Option<i64> {
        self.find(k).ok().map(|i| self.s[i].1)
    }
    fn delete(&mut self, k: i64) {
        if let Ok(i) = self.find(k) {
            self.s.remove(i);
        }
    }
    fn len(&self) -> usize {
        self.s.len()
    }
    fn scan(&self) -> i64 {
        self.s
            .iter()
            .fold(0i64, |s, p| s.wrapping_add(p.0).wrapping_add(p.1))
    }
}

// ---- harness ----------------------------------------------------------------

struct Impl {
    name: &'static str,
    tag: &'static str, // short label for the ranking lines
    new: fn(usize) -> Box<dyn Dict>,
}

const IMPLS: [Impl; 5] = [
    Impl {
        name: "chaining",
        tag: "chain",
        new: |h| Box::new(Chained::new(h)),
    },
    Impl {
        name: "linear probing",
        tag: "probe",
        new: |h| Box::new(Probed::new(h)),
    },
    Impl {
        name: "HashMap (SipHash)",
        tag: "sip",
        new: |h| Box::new(Std::<RandomState>::new(h)),
    },
    Impl {
        name: "HashMap (mix hash)",
        tag: "mix",
        new: |h| Box::new(Std::<BuildHasherDefault<MixHasher>>::new(h)),
    },
    Impl {
        name: "sorted Vec",
        tag: "vec",
        new: |h| Box::new(Sorted::new(h)),
    },
];

fn build(im: &Impl, keys: &[i64]) -> Box<dyn Dict> {
    let mut d = (im.new)(keys.len());
    for (i, &k) in keys.iter().enumerate() {
        d.put(k, i as i64);
    }
    d
}

// ---- measurements, one per column of the table ----

fn build_ns(im: &Impl, keys: &[i64]) -> f64 {
    timed_ns(|| (), |()| build(im, keys)) / keys.len() as f64
}

// ns per lookup of every key in `keys`, through the trait object
fn lookup_ns(d: &dyn Dict, keys: &[i64]) -> f64 {
    let each = ns_per_op(|| {
        for &k in keys {
            black_box(d.get(black_box(k)));
        }
    });
    each / keys.len() as f64
}

// Go times build+delete and subtracts the build. With the stopwatch around the
// deletes alone there is nothing to subtract, and the table that `run` hands
// back (empty) is dropped outside it. Deleted in the order given.
fn delete_ns(im: &Impl, keys: &[i64], order: impl Iterator<Item = i64> + Clone) -> f64 {
    let ns = timed_ns(
        || build(im, keys),
        |mut d| {
            for k in order.clone() {
                d.delete(black_box(k));
            }
            black_box(d.len());
            d
        },
    );
    ns / keys.len() as f64
}

fn kb_of(im: &Impl, keys: &[i64]) -> f64 {
    let before = live_bytes();
    let d = build(im, keys);
    let kb = (live_bytes() - before) as f64 / 1024.0;
    drop(d);
    kb
}

// one measured row of the main table
struct Row {
    tag: &'static str,
    build: f64,
    hit: f64,
    miss: f64,
    del: Option<f64>,
    kb: f64,
}

// "a 3.9 < b 7.9 ~ c 8.1": names in ascending order of ns, with `~` between
// neighbours that are within 10% of each other. The ranking is computed from
// the measurements, so the sentence is true of whatever this run printed.
fn ranking(mut xs: Vec<(&str, f64)>) -> String {
    xs.sort_by(|a, b| a.1.total_cmp(&b.1));
    let mut s = format!("{} {:.1}", xs[0].0, xs[0].1);
    for w in xs.windows(2) {
        let sep = if w[1].1 < w[0].1 * 1.10 { "~" } else { "<" };
        s += &format!(" {sep} {} {:.1}", w[1].0, w[1].1);
    }
    s
}

fn avg(it: impl Iterator<Item = usize>) -> f64 {
    let (sum, n) = it.fold((0usize, 0usize), |(s, n), x| (s + x, n + 1));
    sum as f64 / n as f64
}

fn main() {
    const N: usize = 200_000;
    const MAX: usize = 262_144; // the largest key set used anywhere below
    let present: Vec<i64> = (0..MAX as i64).map(|i| i * 2654435761 + 1).collect();
    let absent: Vec<i64> = present.iter().map(|&k| -k).collect();
    // The sorted slice is Theta(n) per insert, so building it 200,000 times
    // would dominate the whole program. Build it once at a smaller size.
    const SORTED_N: usize = 20_000;
    let [_, _, _, _, sorted_impl] = &IMPLS;

    println!("four dictionaries, 200,000 int keys, all behind one trait object.");
    println!("Go's map is one row here, std's HashMap two: its default hasher, and the");
    println!("hash the two hand-written tables use. Every timing is the fastest of several");
    println!("runs, because this machine is shared (Go's harness reports one).");
    println!();
    println!(
        "  {:<18} {:>13} {:>11} {:>11} {:>12} {:>11}",
        "implementation", "build ns/op", "hit ns", "miss ns", "delete ns", "KB"
    );
    let mut rows: Vec<Row> = Vec::new();
    for im in &IMPLS {
        let (keys, note) = match im.tag {
            "vec" => (&present[..SORTED_N], "*"),
            _ => (&present[..N], ""),
        };
        // The build and delete columns time the operation alone. Freeing the
        // table is left out of both, except where it is part of the operation
        // (a chained delete frees its node). Dropping gets its own column below.
        let t_build = build_ns(im, keys);
        let d = build(im, keys);
        let (t_hit, t_miss) = (lookup_ns(&*d, keys), lookup_ns(&*d, &absent[..keys.len()]));
        drop(d);
        let t_del = delete_ns(im, keys, keys.iter().copied());
        let kb = kb_of(im, keys);
        println!(
            "  {:<18} {:>13.1} {:>11.1} {:>11.1} {:>12.1} {:>10.0}{note}",
            im.name, t_build, t_hit, t_miss, t_del, kb
        );
        rows.push(Row {
            tag: im.tag,
            build: t_build,
            hit: t_hit,
            miss: t_miss,
            del: Some(t_del),
            kb,
        });
    }
    // The same sorted Vec at the SAME size as the others, so that the lookup
    // columns compare like with like. Not in the Go program.
    let keys = &present[..N];
    let d = build(sorted_impl, keys);
    let (t_hit, t_miss) = (lookup_ns(&*d, keys), lookup_ns(&*d, &absent[..N]));
    drop(d);
    let (t_build, kb) = (build_ns(sorted_impl, keys), kb_of(sorted_impl, keys));
    println!(
        "  {:<18} {:>13.1} {:>11.1} {:>11.1} {:>12} {:>11.0}",
        "sorted Vec, 200k", t_build, t_hit, t_miss, "not run", kb
    );
    rows.push(Row {
        tag: "vec200k",
        build: t_build,
        hit: t_hit,
        miss: t_miss,
        del: None,
        kb,
    });
    println!("\n  * the sorted Vec was built at {SORTED_N} keys, not {N}, as in the Go program.");
    println!("    The last row is the same Vec at {N}, which the Go footnote says cannot be");
    println!("    run. It is not in the Go program.");

    // ---- the footnote, tested ----
    println!();
    println!("the footnote, tested. Go says the insert is Theta(n) 'so building it at");
    println!("200,000 takes minutes'. The keys above go in ASCENDING order, so every");
    println!("insert lands at the END and shifts nothing. Same keys, other orders:");
    println!();
    let asc = &present[..SORTED_N];
    // A deterministic shuffle: order the keys by their hash. Same order every run.
    let mut shuf20 = asc.to_vec();
    shuf20.sort_by_key(|&k| mix(k ^ 7));
    let mut shuf40 = present[..2 * SORTED_N].to_vec();
    shuf40.sort_by_key(|&k| mix(k ^ 7));
    let t_asc = build_ns(sorted_impl, asc);
    let (t_s20, t_s40) = (
        build_ns(sorted_impl, &shuf20),
        build_ns(sorted_impl, &shuf40),
    );
    println!(
        "  {:<44} {:>10} {:>14}",
        "sorted Vec build", "keys", "ns per insert"
    );
    println!(
        "  {:<44} {:>10} {:>14.1}",
        "ascending (the table's order)", SORTED_N, t_asc
    );
    println!("  {:<44} {:>10} {:>14.1}", "shuffled", SORTED_N, t_s20);
    println!("  {:<44} {:>10} {:>14.1}", "shuffled", 2 * SORTED_N, t_s40);
    let doubling = t_s40 / t_s20;
    let predicted = t_s40 * (N / (2 * SORTED_N)) as f64 * N as f64 / 1e9;
    let verdict = if predicted < 60.0 {
        " -- seconds, not minutes"
    } else {
        ""
    };
    println!();
    println!(
        "  doubling the keys multiplies the cost per insert by {doubling:.1} (Theta(n) predicts 2,"
    );
    println!("  and it wanders, plausibly because a 640 KB array is harder on the cache than a");
    println!("  320 KB one; not measured).");
    println!(
        "  Extrapolated linearly, not run: {N} shuffled keys, about {predicted:.0} s{verdict}."
    );
    let t_dfront = rows[4].del.unwrap_or(0.0);
    let t_dback = delete_ns(sorted_impl, asc, asc.iter().rev().copied());
    println!();
    println!(
        "  the delete column's {t_dfront:.0} ns is the same effect from the other side: that loop"
    );
    println!("  deletes in ascending order, always index 0, so every delete shifts every key");
    println!("  after it. Deleted from the back instead: {t_dback:.1} ns each. The Theta(n) is");
    println!("  real, but it is a memmove, and it depends on WHERE the insert or delete lands.");

    // ---- reading the table ----
    println!();
    println!("reading the table. Every ranking is computed from the rows above, `~` joins");
    println!("neighbours within 10% of each other, and the sorted Vec is the 200k row:");
    println!();
    let pick = |f: fn(&Row) -> Option<f64>, tags: &[&str]| {
        let xs = rows.iter().filter(|r| tags.contains(&r.tag));
        ranking(xs.filter_map(|r| f(r).map(|v| (r.tag, v))).collect())
    };
    let all = ["chain", "probe", "sip", "mix", "vec200k"];
    println!("  build  {}", pick(|r| Some(r.build), &all));
    println!("  hit    {}", pick(|r| Some(r.hit), &all));
    println!("  miss   {}", pick(|r| Some(r.miss), &all));
    let hash_rows = ["chain", "probe", "sip", "mix", "vec"];
    println!(
        "  delete {}   (sorted Vec at 20k)",
        pick(|r| r.del, &hash_rows)
    );

    // ---- what the lookups did, counted ----
    println!();
    println!("what the lookups did, counted. These do not depend on the machine. A chained");
    println!("lookup reads one bucket and then visits `links` nodes; a probed lookup examines");
    println!("`probes` slots. `longest` is the longest chain / longest probe sequence of a hit.");
    println!();
    println!(
        "  {:<16} {:>8} {:>8} {:>6} {:>7} {:>8} {:>9} {:>8} {:>8}",
        "table", "keys", "slots", "load", "empty", "per hit", "per miss", "longest", "miss ns"
    );
    // (chained?, keys). The hint is the key count, so the slot count is the
    // next power of two (chaining) or the first power of two that keeps the
    // load factor at or under 7/8 (probing).
    let configs = [
        (true, 200_000),
        (false, 200_000),
        (true, 229_376),
        (false, 229_376),
        (true, 262_144),
    ];
    // (load, per miss, empty %, miss ns) per row, for the commentary
    let mut counted = Vec::new();
    for (chained, n) in configs {
        let (hit_keys, miss_keys) = (&present[..n], &absent[..n]);
        let (d, name, slots, empty, hit, miss, longest): (Box<dyn Dict>, _, _, _, _, _, _) =
            if chained {
                let mut c = Chained::new(n);
                for (i, &k) in hit_keys.iter().enumerate() {
                    c.put(k, i as i64);
                }
                let (empty, longest) = c.shape();
                let hit = avg(hit_keys.iter().map(|&k| c.links(k)));
                let miss = avg(miss_keys.iter().map(|&k| c.links(k)));
                let slots = c.buckets.len();
                (Box::new(c), "chaining", slots, empty, hit, miss, longest)
            } else {
                let mut p = Probed::new(n);
                for (i, &k) in hit_keys.iter().enumerate() {
                    p.put(k, i as i64);
                }
                let hit = avg(hit_keys.iter().map(|&k| p.probes(k)));
                let miss = avg(miss_keys.iter().map(|&k| p.probes(k)));
                let longest = hit_keys.iter().map(|&k| p.probes(k)).max().unwrap_or(0);
                let (slots, free) = (p.keys.len(), p.keys.len() - p.n);
                (
                    Box::new(p),
                    "linear probing",
                    slots,
                    free,
                    hit,
                    miss,
                    longest,
                )
            };
        let (load, empty) = (n as f64 / slots as f64, 100.0 * empty as f64 / slots as f64);
        let t_miss = lookup_ns(&*d, miss_keys);
        println!(
            "  {name:<16} {n:>8} {slots:>8} {load:>6.3} {empty:>6.1}% {hit:>8.2} {miss:>9.2} {longest:>8} {t_miss:>8.1}"
        );
        counted.push((load, miss, empty, t_miss));
    }

    // ---- the commentary the numbers support ----
    let theory = |a: f64| (1.0 + 1.0 / ((1.0 - a) * (1.0 - a))) / 2.0;
    let (chain, probe, sip, mix_row) = (&rows[0], &rows[1], &rows[2], &rows[3]);
    let [
        (load, lk_miss, lk_empty, lk_ns),
        (_, pr_miss, _, pr_ns),
        (load2, lk2_miss, _, lk2_ns),
        (load3, pr3_miss, _, pr3_ns),
        (load4, lk4_miss, lk4_empty, _),
    ] = counted[..]
    else {
        unreachable!("five rows");
    };
    let (t_a, t_b) = (theory(load), theory(load3));
    let (r1, r2) = (pr_ns / lk_ns, pr3_ns / lk2_ns);
    println!();
    println!(
        "  the Go text explains its miss column with load factors of 1.0 (chaining) and 0.875"
    );
    println!(
        "  (probing). Neither is this table's: the hint rounds both hand-written tables up to"
    );
    println!(
        "  262,144 slots, so the first table ran BOTH at load factor {load:.3}. The mechanism"
    );
    println!("  the text gives survives anyway, and it is counted here, not assumed:");
    println!(
        "    chaining  {lk_empty:.1}% of buckets are empty, so a miss visits {lk_miss:.2} links -- the load"
    );
    println!(
        "              factor, which example 3 showed. At {load4:.3} (last row): {lk4_empty:.1}% empty,"
    );
    println!("              {lk4_miss:.2} links. (At {load2:.3}: {lk2_miss:.2}.)");
    println!(
        "    probing   a miss examines {pr_miss:.2} slots at {load:.3} and {pr3_miss:.2} at {load3:.3}; the formula"
    );
    println!("              from example 4, (1 + 1/(1-a)^2)/2, gives {t_a:.2} and {t_b:.2}.");
    println!(
        "  in nanoseconds a probed miss costs {r1:.1}x a chained miss at {load:.3}, {r2:.1}x at {load3:.3}."
    );
    println!();
    let (h_lp, m_lp) = (chain.hit / probe.hit, chain.miss / probe.miss);
    let (h_sip, m_sip) = (chain.hit / sip.hit, chain.miss / sip.miss);
    let (d_lp, b_lp) = (
        chain.del.unwrap_or(0.0) / probe.del.unwrap_or(1.0),
        chain.build / probe.build,
    );
    println!("  Go reads the first table as 'chaining wins hit, miss and delete'. In this port:");
    println!(
        "    hit, miss  chaining takes {h_lp:.2}x and {m_lp:.2}x linear probing's time, and {h_sip:.2}x and"
    );
    println!("               {m_sip:.2}x the time of the std HashMap with its default hasher.");
    println!(
        "    delete     chaining's is {d_lp:.1}x linear probing's. It FREES a node per delete; Go's"
    );
    println!("               leaves the freeing to the collector, on someone else's clock.");
    println!("    build      chaining's is {b_lp:.1}x linear probing's: one malloc per entry.");
    println!(
        "  and the hash matters as much as the table. The SAME std table costs {:.1} ns a",
        sip.hit
    );
    println!(
        "  hit with its default SipHash-1-3 and {:.1} with `mix`: {:.1}x, from the hash alone.",
        mix_row.hit,
        sip.hit / mix_row.hit
    );
    println!("  SipHash buys resistance to the attack example 14 builds; Go's map has its own");
    println!("  randomised hash built in, so this choice does not exist there.");

    // ---- the column it was missing ----
    println!();
    println!("here is the column it was missing, Rust edition. Go's was 'heap objects' and");
    println!("'ms per GC cycle'. There is no collector here, so the second has no counterpart;");
    println!("what a chained table costs a program WITHOUT one is the allocator, at both ends,");
    println!("and the walk. Blocks and events are exact counts from a counting allocator:");
    println!();
    println!(
        "  {:<18} {:>12} {:>13} {:>14} {:>14}",
        "implementation", "heap blocks", "alloc events", "drop ns/entry", "scan ns/entry"
    );
    // (blocks, alloc events, entries, drop ns/entry, scan ns/entry) per row
    let mut costs = Vec::new();
    for im in &IMPLS {
        let keys = if im.tag == "vec" {
            &present[..SORTED_N]
        } else {
            &present[..N]
        };
        let m = keys.len();
        let before = live_blocks();
        let (events, d) = allocs_of(|| build(im, keys));
        // minus the one block that holds the `Box<dyn Dict>` itself
        let (blocks, events) = (live_blocks() - before - 1, events - 1);
        let scan = ns_per_op(|| {
            black_box(d.scan());
        }) / m as f64;
        drop(d);
        let t_drop = timed_ns(|| build(im, keys), drop) / m as f64;
        println!(
            "  {:<18} {blocks:>12} {events:>13} {t_drop:>14.3} {scan:>14.2}",
            im.name
        );
        costs.push((blocks, events, m, t_drop, scan));
    }
    let [
        (c_blocks, c_events, c_n, c_drop, c_scan),
        (p_blocks, _, _, p_drop, p_scan),
        (s_blocks, ..),
        _,
        (v_blocks, ..),
    ] = costs[..]
    else {
        unreachable!("five rows");
    };
    let (drop_ms, drop_pct, scan_x) = (
        c_drop * c_n as f64 / 1e6,
        100.0 * p_drop / c_drop,
        c_scan / p_scan,
    );
    println!();
    println!(
        "  chaining holds one heap block PER ENTRY: {c_blocks} blocks for {c_n} entries (the nodes, and"
    );
    println!(
        "  the bucket array). linear probing holds {p_blocks} (three arrays), the HashMap {s_blocks}, and the"
    );
    println!("  sorted Vec {v_blocks}.");
    println!();
    println!(
        "  nothing traces those blocks while the table lives, so Go's 'forever, whether or not"
    );
    println!("  the map is being used' has no Rust counterpart. The cost moves to the two ends of");
    println!("  the table's life instead:");
    println!("    building  {c_events} allocation events (a node each, and the bucket array): the");
    println!("              build column's {b_lp:.1}x for chaining against linear probing.");
    println!(
        "    dropping  one free per entry: {c_drop:.1} ns/entry, {drop_ms:.1} ms for the whole table, once,"
    );
    println!(
        "              where linear probing's three arrays cost {p_drop:.2} ns/entry ({drop_pct:.1}% of that)."
    );
    println!("  the part of Go's cost that DOES carry over is the walk: a marker touches every");
    println!("  object, and the scan column is that access pattern with no collector attached:");
    println!(
        "  {c_scan:.1} ns/entry through chaining's nodes, which it meets in bucket order and not"
    );
    println!(
        "  allocation order, against {p_scan:.1} for linear probing's arrays ({scan_x:.1}x). That the nodes"
    );
    println!("  are scattered is the plausible mechanism for the gap; no cache counter was read.");
    println!();
    println!("  memory, the other thing a block per entry costs: chaining's KB column above is");
    println!(
        "  {:.2}x the HashMap's: an 8-byte `next` per node and a bucket array, on top of",
        chain.kb / mix_row.kb
    );
    println!("  the 16-byte key-value pair both hold. Both are bytes REQUESTED; the system");
    println!("  allocator's own rounding and headers are not counted, so each is a lower bound.");
    // hashbrown allows 7/8 of its buckets to be filled, so capacity() * 8/7 is
    // the bucket count (an implementation detail; the byte count below checks it).
    let cap = Std::<RandomState>::new(N).m.capacity();
    let buckets = cap * 8 / 7;
    let kb_17 = (buckets * 17) as f64 / 1024.0;
    println!();
    println!("  linear probing and both HashMaps print the same KB, and it is arithmetic: the");
    println!("  HashMap reports capacity() = {cap}, which at hashbrown's 7/8 maximum load is");
    println!(
        "  {buckets} buckets, and {buckets} x 17 B (a 16-byte pair and one control byte) is {kb_17:.0} KB,"
    );
    println!("  the 17 bytes a slot that linear probing's three arrays also cost. Against std's");
    println!("  table, 'linear probing uses the least memory' is a tie.");

    // ---- iteration in order ----
    println!();
    println!("the workload that reverses the ranking -- ITERATION IN ORDER:");
    println!();
    const IT: usize = 20_000;
    let mut sorted_d = Sorted::new(IT);
    let mut std_d: HashMap<i64, i64> = HashMap::with_capacity(IT);
    let mut tree: BTreeMap<i64, i64> = BTreeMap::new();
    for (i, &k) in present[..IT].iter().enumerate() {
        sorted_d.put(k, i as i64);
        std_d.insert(k, i as i64);
        tree.insert(k, i as i64);
    }
    let t_sorted_iter = ns_per_op(|| {
        for p in &sorted_d.s {
            black_box(p.0);
        }
    });
    let t_map_iter = ns_per_op(|| {
        let mut ks: Vec<i64> = Vec::with_capacity(std_d.len());
        ks.extend(std_d.keys());
        ks.sort_unstable();
        black_box(&ks);
    });
    let t_tree_iter = ns_per_op(|| {
        for k in tree.keys() {
            black_box(*k);
        }
    });
    let (map_x, tree_x) = (t_map_iter / t_sorted_iter, t_tree_iter / t_sorted_iter);
    println!(
        "  {:<40} {t_sorted_iter:>14.0} ns",
        "sorted Vec: just iterate it"
    );
    println!(
        "  {:<40} {t_map_iter:>14.0} ns   {map_x:.0}x slower",
        "HashMap: collect keys, then sort"
    );
    println!(
        "  {:<40} {t_tree_iter:>14.0} ns   {tree_x:.1}x (not in the Go program)",
        "BTreeMap: just iterate it"
    );
    println!();
    println!("  the sorted Vec is ALREADY in order, so this workload is a scan of contiguous");
    println!("  memory. The HashMap has to materialise every key and sort them, every single");
    println!("  time. std ships a structure that is in order AND takes inserts, BTreeMap; the");
    println!("  walk over it is the third row.");
    println!();
    println!("  and a sorted Vec answers questions a hash map cannot answer at all: the smallest");
    println!("  key above x (`partition_point`), the keys in a range (a subslice), the median");
    println!("  (an index). Those are Theta(log n) on a Vec, or on a BTreeMap's `range`, and");
    println!("  Theta(n) on a HashMap, which has to look at every key.");

    // ---- the verdict ----
    println!();
    println!("the verdict:");
    println!();
    for r in [
        ["implementation", "use it when", "never when"],
        [
            "HashMap",
            "essentially always",
            "you need ordering or bounds",
        ],
        [
            "chaining",
            "you are writing one to learn",
            "inserts and frees are on a hot path",
        ],
        [
            "linear probing",
            "you need control over layout",
            "deletes churn, or load nears 7/8",
        ],
        [
            "sorted Vec",
            "small, or read-mostly and ordered",
            "inserts arrive out of order",
        ],
        [
            "BTreeMap",
            "you need order and inserts",
            "you only ever look keys up",
        ],
    ] {
        println!("  {:<18} {:<34} {}", r[0], r[1], r[2]);
    }
    println!();
    println!("  the honest summary of this whole lesson: WRITE `HashMap<K, V>`. The");
    println!("  three hand-written tables above exist so you can say why the");
    println!("  built-in one is shaped the way it is -- not because you should");
    println!("  ship one. And know which hasher you got: the default one cost");
    println!(
        "  {:.1}x the same table with `mix` on a hit, and swapping it is one `BuildHasher`;",
        sip.hit / mix_row.hit
    );
    println!("  whether that is safe depends on who chooses your keys (example 14).");
    println!();
    println!("  the exception that is worth the trouble is a specialised open-");
    println!("  addressed table for a hot inner loop with a known key type and");
    println!("  size. Example 16 builds one and measures whether it was worth it.");
}
```

**Rust sample output:**

```
four dictionaries, 200,000 int keys, all behind one trait object.
Go's map is one row here, std's HashMap two: its default hasher, and the
hash the two hand-written tables use. Every timing is the fastest of several
runs, because this machine is shared (Go's harness reports one).

  implementation       build ns/op      hit ns     miss ns    delete ns          KB
  chaining                    26.7        13.1        18.0         44.4       6736
  linear probing              10.6        14.0        21.6          8.8       4352
  HashMap (SipHash)            8.8        16.9        17.2         14.3       4352
  HashMap (mix hash)           4.8         4.2         8.7          9.8       4352
  sorted Vec                  10.3        12.0        10.9       2591.1        313*
  sorted Vec, 200k            14.9        17.4        16.0      not run        3125

  * the sorted Vec was built at 20000 keys, not 200000, as in the Go program.
    The last row is the same Vec at 200000, which the Go footnote says cannot be
    run. It is not in the Go program.

the footnote, tested. Go says the insert is Theta(n) 'so building it at
200,000 takes minutes'. The keys above go in ASCENDING order, so every
insert lands at the END and shifts nothing. Same keys, other orders:

  sorted Vec build                                   keys  ns per insert
  ascending (the table's order)                     20000           10.3
  shuffled                                          20000          938.3
  shuffled                                          40000         2364.4

  doubling the keys multiplies the cost per insert by 2.5 (Theta(n) predicts 2,
  and it wanders, plausibly because a 640 KB array is harder on the cache than a
  320 KB one; not measured).
  Extrapolated linearly, not run: 200000 shuffled keys, about 2 s -- seconds, not minutes.

  the delete column's 2591 ns is the same effect from the other side: that loop
  deletes in ascending order, always index 0, so every delete shifts every key
  after it. Deleted from the back instead: 10.0 ns each. The Theta(n) is
  real, but it is a memmove, and it depends on WHERE the insert or delete lands.

reading the table. Every ranking is computed from the rows above, `~` joins
neighbours within 10% of each other, and the sorted Vec is the 200k row:

  build  mix 4.8 < sip 8.8 < probe 10.6 < vec200k 14.9 < chain 26.7
  hit    mix 4.2 < chain 13.1 ~ probe 14.0 < sip 16.9 ~ vec200k 17.4
  miss   mix 8.7 < vec200k 16.0 ~ sip 17.2 ~ chain 18.0 < probe 21.6
  delete probe 8.8 < mix 9.8 < sip 14.3 < chain 44.4 < vec 2591.1   (sorted Vec at 20k)

what the lookups did, counted. These do not depend on the machine. A chained
lookup reads one bucket and then visits `links` nodes; a probed lookup examines
`probes` slots. `longest` is the longest chain / longest probe sequence of a hit.

  table                keys    slots   load   empty  per hit  per miss  longest  miss ns
  chaining           200000   262144  0.763   46.7%     1.38      0.76        8     12.3
  linear probing     200000   262144  0.763   23.7%     2.59      9.11      122     18.8
  chaining           229376   262144  0.875   41.7%     1.44      0.87        8     13.0
  linear probing     229376   262144  0.875   12.5%     4.41     31.70      480     32.2
  chaining           262144   262144  1.000   36.8%     1.50      1.00        9     14.5

  the Go text explains its miss column with load factors of 1.0 (chaining) and 0.875
  (probing). Neither is this table's: the hint rounds both hand-written tables up to
  262,144 slots, so the first table ran BOTH at load factor 0.763. The mechanism
  the text gives survives anyway, and it is counted here, not assumed:
    chaining  46.7% of buckets are empty, so a miss visits 0.76 links -- the load
              factor, which example 3 showed. At 1.000 (last row): 36.8% empty,
              1.00 links. (At 0.875: 0.87.)
    probing   a miss examines 9.11 slots at 0.763 and 31.70 at 0.875; the formula
              from example 4, (1 + 1/(1-a)^2)/2, gives 9.40 and 32.50.
  in nanoseconds a probed miss costs 1.5x a chained miss at 0.763, 2.5x at 0.875.

  Go reads the first table as 'chaining wins hit, miss and delete'. In this port:
    hit, miss  chaining takes 0.94x and 0.83x linear probing's time, and 0.78x and
               1.05x the time of the std HashMap with its default hasher.
    delete     chaining's is 5.0x linear probing's. It FREES a node per delete; Go's
               leaves the freeing to the collector, on someone else's clock.
    build      chaining's is 2.5x linear probing's: one malloc per entry.
  and the hash matters as much as the table. The SAME std table costs 16.9 ns a
  hit with its default SipHash-1-3 and 4.2 with `mix`: 4.0x, from the hash alone.
  SipHash buys resistance to the attack example 14 builds; Go's map has its own
  randomised hash built in, so this choice does not exist there.

here is the column it was missing, Rust edition. Go's was 'heap objects' and
'ms per GC cycle'. There is no collector here, so the second has no counterpart;
what a chained table costs a program WITHOUT one is the allocator, at both ends,
and the walk. Blocks and events are exact counts from a counting allocator:

  implementation      heap blocks  alloc events  drop ns/entry  scan ns/entry
  chaining                 200001        200001         57.849           6.89
  linear probing                3             3          0.166           1.07
  HashMap (SipHash)             1             1          0.061           1.35
  HashMap (mix hash)            1             1          0.061           1.37
  sorted Vec                    1             1          0.000           0.17

  chaining holds one heap block PER ENTRY: 200001 blocks for 200000 entries (the nodes, and
  the bucket array). linear probing holds 3 (three arrays), the HashMap 1, and the
  sorted Vec 1.

  nothing traces those blocks while the table lives, so Go's 'forever, whether or not
  the map is being used' has no Rust counterpart. The cost moves to the two ends of
  the table's life instead:
    building  200001 allocation events (a node each, and the bucket array): the
              build column's 2.5x for chaining against linear probing.
    dropping  one free per entry: 57.8 ns/entry, 11.6 ms for the whole table, once,
              where linear probing's three arrays cost 0.17 ns/entry (0.3% of that).
  the part of Go's cost that DOES carry over is the walk: a marker touches every
  object, and the scan column is that access pattern with no collector attached:
  6.9 ns/entry through chaining's nodes, which it meets in bucket order and not
  allocation order, against 1.1 for linear probing's arrays (6.4x). That the nodes
  are scattered is the plausible mechanism for the gap; no cache counter was read.

  memory, the other thing a block per entry costs: chaining's KB column above is
  1.55x the HashMap's: an 8-byte `next` per node and a bucket array, on top of
  the 16-byte key-value pair both hold. Both are bytes REQUESTED; the system
  allocator's own rounding and headers are not counted, so each is a lower bound.

  linear probing and both HashMaps print the same KB, and it is arithmetic: the
  HashMap reports capacity() = 229376, which at hashbrown's 7/8 maximum load is
  262144 buckets, and 262144 x 17 B (a 16-byte pair and one control byte) is 4352 KB,
  the 17 bytes a slot that linear probing's three arrays also cost. Against std's
  table, 'linear probing uses the least memory' is a tie.

the workload that reverses the ranking -- ITERATION IN ORDER:

  sorted Vec: just iterate it                        5182 ns
  HashMap: collect keys, then sort                 149650 ns   29x slower
  BTreeMap: just iterate it                         17659 ns   3.4x (not in the Go program)

  the sorted Vec is ALREADY in order, so this workload is a scan of contiguous
  memory. The HashMap has to materialise every key and sort them, every single
  time. std ships a structure that is in order AND takes inserts, BTreeMap; the
  walk over it is the third row.

  and a sorted Vec answers questions a hash map cannot answer at all: the smallest
  key above x (`partition_point`), the keys in a range (a subslice), the median
  (an index). Those are Theta(log n) on a Vec, or on a BTreeMap's `range`, and
  Theta(n) on a HashMap, which has to look at every key.

the verdict:

  implementation     use it when                        never when
  HashMap            essentially always                 you need ordering or bounds
  chaining           you are writing one to learn       inserts and frees are on a hot path
  linear probing     you need control over layout       deletes churn, or load nears 7/8
  sorted Vec         small, or read-mostly and ordered  inserts arrive out of order
  BTreeMap           you need order and inserts         you only ever look keys up

  the honest summary of this whole lesson: WRITE `HashMap<K, V>`. The
  three hand-written tables above exist so you can say why the
  built-in one is shaped the way it is -- not because you should
  ship one. And know which hasher you got: the default one cost
  4.0x the same table with `mix` on a hit, and swapping it is one `BuildHasher`;
  whether that is safe depends on who chooses your keys (example 14).

  the exception that is worth the trouble is a specialised open-
  addressed table for a hot inner loop with a known key type and
  size. Example 16 builds one and measures whether it was worth it.
```

> **Go vs Rust:** Counted quantities match Go (6,736 KB for chaining, 4,352 KB for linear probing), but Go's `map` row becomes two `HashMap` rows: std's default SipHash-1-3 with a random per-map seed, and the example's own `mix` hasher, both 4,352 KB (the same 17 bytes a slot as the probed table); the hasher alone is 2.4-2.7x on a hit, a choice Go's built-in runtime hash does not offer. Three Go claims fail here: both tables ran at load factor 0.763, not the 1.0 and 0.875 the text uses (a miss is 0.76 links against 9.11 probes, so the mechanism holds and the loads are wrong); "building the sorted slice at 200,000 takes minutes" is wrong because the keys ascend and append (Go's own row shows 20 ns a build; here 10-15 ns, and 200,000 shuffled keys extrapolate to 2-3 s); and "chaining wins delete" reverses to 3.8-5.0x slower than linear probing over ten runs, because Rust frees the node inside `delete` where Go's collector does it later (its build is also 2.0-2.5x, one malloc per entry), though chaining does beat linear probing on hit and miss (0.64-0.71x and 0.51-0.58x of its time) and is level with the SipHash `HashMap` on hits (0.95-1.08x). With no collector there is no "ms per GC cycle": the forgotten column becomes 200,001 heap blocks against 1-3, and the `Drop` of the chained table, 45-63 ns an entry (9-13 ms, once) against 0.13-0.17 for linear probing, plus a walk of every entry at 5-9 ns against 0.7-1.6; ordered iteration through a `HashMap` sort is 28-33x slower than the sorted `Vec` (Go: 187x).

**Complexity:** chaining wins **hit, miss and delete** on int keys with a good hash — its miss is
fastest because at load 1.0 about 37% of buckets are empty · and it holds **200,404 heap objects to
linear probing's 409**, costing **~30× the GC time per cycle**, forever

---

> Next tier: [🔴 hard](3-hard.md) — the problems maps own, the attack that made every language
> randomise its hash, when a map is the wrong answer, and the capstone.
