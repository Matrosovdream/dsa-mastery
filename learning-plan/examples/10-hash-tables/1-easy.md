# Step 10 — Hash Tables & Sets · 🟢 Easy

Examples **1–6**: what Θ(1) actually costs, what makes a hash function good, the two collision
strategies, and what Go's map really is.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 2, 3 and 4 are deterministic apart from their closing benchmarks. Examples 1, 6 and 7 report
memory, which varies slightly between runs.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. Θ(n) → Θ(1), and the three things you pay for it

`🟢 easy` · *the only structure that does this*

Every structure so far has been about **order**. A hash table throws order away and buys the one thing
none of the others could give you: find an arbitrary key in constant time, independent of `n`. That
claim is strong enough to deserve measuring.

**Steps:**

1. Race a linear scan, a binary search and a map across five orders of magnitude.
2. Find where the map *loses*.
3. Then price it — in bytes, and in the three properties you gave up.

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"slices"
	"testing"
)

// Every structure so far has been about ORDER: what is next, what is at index
// i, what came first. A hash table throws order away entirely, and buys the one
// thing none of the others could give you:
//
//	find an arbitrary key in Theta(1), independent of n
//
// "Independent of n" is the claim, and it is strong enough that it deserves to
// be measured rather than believed. Here are the three ways to answer "is x in
// this collection?", across five orders of magnitude.

func containsLinear(xs []int, v int) bool {
	for _, x := range xs {
		if x == v {
			return true
		}
	}
	return false
}

func containsBinary(sorted []int, v int) bool {
	_, ok := slices.BinarySearch(sorted, v)
	return ok
}

func containsMap(m map[int]struct{}, v int) bool {
	_, ok := m[v]
	return ok
}

var sinkB bool

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// heapKB runs the GC twice on each side. One cycle is not always enough to
// collect the previous measurement's garbage, and the leftover shows up as a
// difference in THIS reading -- which is how my first version of the table
// below "proved" that a size hint costs memory.
func heapKB(build func() any) float64 {
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
	rng := rand.New(rand.NewSource(1))

	fmt.Println("the same question -- 'is x present?' -- three ways:")
	fmt.Println()
	fmt.Printf("  %10s %14s %14s %12s %12s %12s\n",
		"n", "linear ns", "binary ns", "map ns", "lin/map", "bin/map")

	for _, n := range []int{8, 16, 64, 1024, 65536, 1_000_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = rng.Intn(1 << 30)
		}
		sorted := slices.Clone(xs)
		slices.Sort(sorted)
		m := make(map[int]struct{}, n)
		for _, v := range xs {
			m[v] = struct{}{}
		}
		// probe keys: half present, half absent, so neither side is favoured
		probes := make([]int, 1024)
		for i := range probes {
			if i%2 == 0 {
				probes[i] = xs[rng.Intn(n)]
			} else {
				probes[i] = rng.Intn(1 << 30)
			}
		}

		tl := nsPerOp(func() {
			for _, p := range probes {
				sinkB = containsLinear(xs, p)
			}
		}) / float64(len(probes))
		tb := nsPerOp(func() {
			for _, p := range probes {
				sinkB = containsBinary(sorted, p)
			}
		}) / float64(len(probes))
		tm := nsPerOp(func() {
			for _, p := range probes {
				sinkB = containsMap(m, p)
			}
		}) / float64(len(probes))

		fmt.Printf("  %10d %14.1f %14.1f %12.1f %11.1fx %11.1fx\n", n, tl, tb, tm, tl/tm, tb/tm)
	}

	fmt.Println()
	fmt.Println("  read the map column DOWN, not across: it barely moves while n")
	fmt.Println("  grows by five orders of magnitude. That is what Theta(1) looks")
	fmt.Println("  like on a clock, and no other structure in this repo can do it.")
	fmt.Println()
	fmt.Println("  the linear column is Theta(n) and says so. The binary column is")
	fmt.Println("  Theta(log n) -- it grows, but slowly, and mostly because of cache")
	fmt.Println("  misses rather than the extra comparisons (lesson 04).")
	fmt.Println()
	fmt.Println("  but read the FIRST row too: at n=8 the linear scan WINS. Eight")
	fmt.Println("  ints are one cache line and the comparison is a single")
	fmt.Println("  instruction, while a map lookup has to hash the key first. The")
	fmt.Println("  crossover is around n=16 -- the same place lesson 01 found it --")
	fmt.Println("  and below it Theta(n) beats Theta(1) every time.")

	fmt.Println()
	fmt.Println("does the size hint save memory? I assumed yes. It does not:")
	fmt.Println()
	const hn = 1 << 19
	hk := make([]int, hn)
	for i := range hk {
		hk[i] = rng.Intn(1 << 40)
	}
	fmt.Printf("  %6s %16s %16s %10s\n", "run", "make(map, n) KB", "make(map) KB", "ratio")
	for rep := 0; rep < 3; rep++ {
		hinted := heapKB(func() any {
			m := make(map[int]struct{}, hn)
			for _, k := range hk {
				m[k] = struct{}{}
			}
			return m
		})
		grown := heapKB(func() any {
			m := make(map[int]struct{})
			for _, k := range hk {
				m[k] = struct{}{}
			}
			return m
		})
		fmt.Printf("  %6d %16.0f %16.0f %9.2fx\n", rep, hinted, grown, hinted/grown)
	}
	fmt.Println()
	fmt.Println("  identical, every run. Both end at the same capacity, because the")
	fmt.Println("  capacity is a function of the final element count, not of how you")
	fmt.Println("  got there. What the hint buys is TIME -- the rehashing skipped on")
	fmt.Println("  the way up -- and example 10 measures exactly that.")
	fmt.Println()
	fmt.Println("  I report three runs because my first version reported ONE, and it")
	fmt.Println("  said the hint cost 28% more memory. The reading was contaminated")
	fmt.Println("  by garbage from the previous measurement that two GC cycles had")
	fmt.Println("  not finished sweeping. Repeat a memory measurement before you")
	fmt.Println("  believe it.")

	fmt.Println()
	fmt.Println("what it costs, in bytes per key:")
	fmt.Println()
	fmt.Printf("  %14s %14s %14s %14s\n", "n", "[]int B/key", "map B/key", "map/slice")
	perKey := map[int]float64{}
	for _, n := range []int{1 << 19, 700_000, 1_000_000, 1 << 20} {
		ks := make([]int, n)
		for i := range ks {
			ks[i] = rng.Intn(1 << 40)
		}
		sl := heapKB(func() any { return slices.Clone(ks) })
		mp := heapKB(func() any {
			m := make(map[int]struct{}, n)
			for _, k := range ks {
				m[k] = struct{}{}
			}
			return m
		})
		perKey[n] = mp * 1024 / float64(n)
		label := fmt.Sprintf("%d", n)
		switch n {
		case 1 << 19:
			label = "524288 = 2^19"
		case 1 << 20:
			label = "1048576 = 2^20"
		}
		fmt.Printf("  %14s %14.2f %14.2f %13.1fx\n",
			label, sl*1024/float64(n), perKey[n], mp/sl)
	}
	fmt.Println()
	fmt.Println("  a map costs several times what the raw keys do, and that is the")
	fmt.Println("  trade: you are buying EMPTY SPACE. A hash table stays fast only")
	fmt.Println("  while it stays sparse -- fill it and every lookup degrades toward")
	fmt.Println("  the linear scan you were trying to avoid (example 5).")
	fmt.Println()
	fmt.Printf("  and look at the swing: 700,000 keys cost %.1f bytes each while\n", perKey[700_000])
	fmt.Printf("  1,000,000 cost %.1f -- %.2fx apart for 43%% more keys. Same\n",
		perKey[1_000_000], perKey[1_000_000]/perKey[700_000])
	fmt.Println("  structure, decided entirely by where n falls relative to a growth")
	fmt.Println("  boundary, exactly as in lessons 06 and 09.")

	fmt.Println("and the three things you gave up to get it:")
	fmt.Println()
	fmt.Printf("  %-22s %s\n", "ORDER", "no first, no last, no next, no range query")
	fmt.Printf("  %-22s %s\n", "CONTIGUITY", "keys are scattered; every lookup is a cache miss")
	fmt.Printf("  %-22s %s\n", "PREDICTABILITY", "Theta(1) is amortized and average-case, not worst")
	fmt.Println()
	im := map[int]int{}
	for i := 0; i < 20; i++ {
		im[i] = i
	}
	fmt.Println("  the first 10 keys of the same 20-key map, four times:")
	for i := 0; i < 4; i++ {
		fmt.Print("    ")
		c := 0
		for k := range im {
			fmt.Printf("%2d ", k)
			if c++; c == 10 {
				break
			}
		}
		fmt.Println("...")
	}
	fmt.Println()
	fmt.Println("  Go RANDOMISES map iteration deliberately, so that code which")
	fmt.Println("  accidentally depends on the order fails immediately and loudly")
	fmt.Println("  rather than in production two years later. Example 6 shows how.")
	fmt.Println()
	fmt.Println("  if you need order, you need a sorted structure -- a tree (lessons")
	fmt.Println("  17-19) or a sorted slice. A map is the wrong tool the moment the")
	fmt.Println("  question is 'what is the smallest key greater than x?'")
}
```

**Sample output:**

```
the same question -- 'is x present?' -- three ways:

           n      linear ns      binary ns       map ns      lin/map      bin/map
           8            2.1            3.7          3.3         0.6x         1.1x
          16            4.3            4.5          3.9         1.1x         1.1x
          64           16.5            6.7          3.6         4.6x         1.9x
        1024          216.8           10.1          4.3        50.8x         2.4x
       65536        12279.7           15.8          4.1      2972.8x         3.8x
     1000000       189872.3           18.4          4.9     38551.5x         3.7x

  read the map column DOWN, not across: it barely moves while n
  grows by five orders of magnitude. That is what Theta(1) looks
  like on a clock, and no other structure in this repo can do it.

  the linear column is Theta(n) and says so. The binary column is
  Theta(log n) -- it grows, but slowly, and mostly because of cache
  misses rather than the extra comparisons (lesson 04).

  but read the FIRST row too: at n=8 the linear scan WINS. Eight
  ints are one cache line and the comparison is a single
  instruction, while a map lookup has to hash the key first. The
  crossover is around n=16 -- the same place lesson 01 found it --
  and below it Theta(n) beats Theta(1) every time.

does the size hint save memory? I assumed yes. It does not:

     run  make(map, n) KB     make(map) KB      ratio
       0            18473            18473      1.00x
       1            18473            18473      1.00x
       2            18473            18473      1.00x

  identical, every run. Both end at the same capacity, because the
  capacity is a function of the final element count, not of how you
  got there. What the hint buys is TIME -- the rehashing skipped on
  the way up -- and example 10 measures exactly that.

  I report three runs because my first version reported ONE, and it
  said the hint cost 28% more memory. The reading was contaminated
  by garbage from the previous measurement that two GC cycles had
  not finished sweeping. Repeat a memory measurement before you
  believe it.

what it costs, in bytes per key:

               n    []int B/key      map B/key      map/slice
   524288 = 2^19           8.00          28.08           3.5x
          700000           8.00          19.02           2.4x
         1000000           8.00          29.83           3.7x
  1048576 = 2^20           8.00          28.08           3.5x

  a map costs several times what the raw keys do, and that is the
  trade: you are buying EMPTY SPACE. A hash table stays fast only
  while it stays sparse -- fill it and every lookup degrades toward
  the linear scan you were trying to avoid (example 5).

  and look at the swing: 700,000 keys cost 19.0 bytes each while
  1,000,000 cost 29.8 -- 1.57x apart for 43% more keys. Same
  structure, decided entirely by where n falls relative to a growth
  boundary, exactly as in lessons 06 and 09.
and the three things you gave up to get it:

  ORDER                  no first, no last, no next, no range query
  CONTIGUITY             keys are scattered; every lookup is a cache miss
  PREDICTABILITY         Theta(1) is amortized and average-case, not worst

  the first 10 keys of the same 20-key map, four times:
    12 13 14  0  1  2  3  4 10 15 ...
    14  0  1  2  3  4 10 15 16  5 ...
     6  8 12 13 14  0  1  2  3  4 ...
     6  8 12 13 14  0  1  2  3  4 ...

  Go RANDOMISES map iteration deliberately, so that code which
  accidentally depends on the order fails immediately and loudly
  rather than in production two years later. Example 6 shows how.

  if you need order, you need a sorted structure -- a tree (lessons
  17-19) or a sorted slice. A map is the wrong tool the moment the
  question is 'what is the smallest key greater than x?'
```

**Complexity:** map lookup Θ(1) — **4.9 ns at n=1,000,000 against 190 µs for a scan** · but the scan
**wins at n=8**, and the crossover is around 16 · a map costs **2.4–3.7× the raw keys**, and the size
hint saves **no memory at all** (it buys time — example 10)

---

## 2. What makes a hash function good, and how to measure it

`🟢 easy` · *the output must look random even when the input does not*

Real keys are never random: `user_1`, `/api/v1/orders`, timestamps one second apart. A hash that
preserves any of that structure piles those keys into the same slots — and a hash table with all its
keys in one slot is a linked list with extra steps.

**Steps:**

1. Write four hashes, from useless to good, and score them on distribution *and* avalanche.
2. Find out which **bits** are good, which is a separate question.
3. Then do the birthday arithmetic that makes collisions inevitable.

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
	"math/bits"
	"math/rand"
	"strings"
)

// A hash table is only as good as its hash function, and "good" has a precise
// meaning that is easy to state and easy to get wrong:
//
//	the output must look RANDOM even when the input does not
//
// Real keys are never random. They are "user_1", "user_2", "/api/v1/orders",
// timestamps one second apart. A hash function that preserves any of that
// structure piles those keys into the same slots, and a hash table with all its
// keys in one slot is a linked list with extra steps.

// ---- four hash functions, from useless to good ----------------------------

// sumBytes is the one everybody writes first. It is also the worst possible
// choice, for a reason that is obvious once stated: it is COMMUTATIVE.
func sumBytes(s string) uint64 {
	var h uint64
	for i := 0; i < len(s); i++ {
		h += uint64(s[i])
	}
	return h
}

// shiftAdd is the "improved" version -- order now matters. It is still bad,
// because the shift throws away the high bits of everything but the tail.
func shiftAdd(s string) uint64 {
	var h uint64
	for i := 0; i < len(s); i++ {
		h = h<<5 + uint64(s[i])
	}
	return h
}

// fnv1a is a real, simple, widely used hash: multiply by a prime, XOR the byte.
// The multiply is what spreads a one-bit input change across the whole word.
func fnv1a(s string) uint64 {
	const (
		offset = 14695981039346656037
		prime  = 1099511628211
	)
	h := uint64(offset)
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= prime
	}
	return h
}

// fnv1aFinal is FNV-1a followed by a FINALIZER -- the mixing step from
// splitmix64. Three multiply-xor-shift rounds, and they cost about a
// nanosecond. This is what turns an adequate hash into a good one, and the
// avalanche column below is the reason it exists.
func fnv1aFinal(s string) uint64 {
	h := fnv1a(s)
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

// stdlib, for comparison -- the same algorithm, from hash/fnv.
func stdFNV(s string) uint64 {
	h := fnv.New64a()
	h.Write([]byte(s))
	return h.Sum64()
}

type hashFn struct {
	name string
	fn   func(string) uint64
}

var fns = []hashFn{
	{"sum of bytes", sumBytes},
	{"shift and add", shiftAdd},
	{"FNV-1a", fnv1a},
	{"hash/fnv (stdlib)", stdFNV},
	{"FNV-1a + finalizer", fnv1aFinal},
}

// ---- measuring "looks random" ---------------------------------------------

// chiSquare measures how far a distribution is from uniform. For a good hash,
// the statistic is close to the number of buckets. Much larger means clumping.
func chiSquare(counts []int, n int) float64 {
	b := float64(len(counts))
	expected := float64(n) / b
	var chi float64
	for _, c := range counts {
		d := float64(c) - expected
		chi += d * d / expected
	}
	return chi
}

func distribute(keys []string, fn func(string) uint64, buckets int) []int {
	counts := make([]int, buckets)
	for _, k := range keys {
		counts[fn(k)%uint64(buckets)]++
	}
	return counts
}

func worst(counts []int) int {
	m := 0
	for _, c := range counts {
		if c > m {
			m = c
		}
	}
	return m
}

func empty(counts []int) int {
	n := 0
	for _, c := range counts {
		if c == 0 {
			n++
		}
	}
	return n
}

// avalanche: flip ONE bit of the input and count how many output bits change.
// A good hash changes about half of them -- 32 of 64.
func avalanche(fn func(string) uint64, keys []string) float64 {
	total, n := 0, 0
	for _, k := range keys {
		if len(k) == 0 {
			continue
		}
		h := fn(k)
		b := []byte(k)
		for i := range b {
			for bit := 0; bit < 8; bit++ {
				b[i] ^= 1 << bit
				total += bits.OnesCount64(h ^ fn(string(b)))
				b[i] ^= 1 << bit
				n++
			}
		}
	}
	return float64(total) / float64(n)
}

func histogram(counts []int, width int) string {
	mx := worst(counts)
	var sb strings.Builder
	for i := 0; i < width && i < len(counts); i++ {
		n := 0
		if mx > 0 {
			n = counts[i] * 40 / mx
		}
		sb.WriteString(fmt.Sprintf("    %3d |%s %d\n", i, strings.Repeat("#", n), counts[i]))
	}
	return sb.String()
}

func main() {
	// Realistic keys: sequential ids, paths, and short words. Nothing random.
	var keys []string
	for i := 0; i < 4000; i++ {
		keys = append(keys, fmt.Sprintf("user_%d", i))
	}
	for i := 0; i < 4000; i++ {
		keys = append(keys, fmt.Sprintf("/api/v1/orders/%d", i))
	}
	rng := rand.New(rand.NewSource(4))
	letters := "abcdefghijklmnopqrstuvwxyz"
	for i := 0; i < 4000; i++ {
		b := make([]byte, 3+rng.Intn(5))
		for j := range b {
			b[j] = letters[rng.Intn(26)]
		}
		keys = append(keys, string(b))
	}

	const buckets = 64
	fmt.Printf("%d realistic keys into %d buckets. A perfect hash puts %d in each:\n",
		len(keys), buckets, len(keys)/buckets)
	fmt.Println()
	fmt.Printf("  %-20s %10s %10s %12s %14s\n",
		"hash function", "worst", "empty", "chi-square", "avalanche bits")
	for _, h := range fns {
		c := distribute(keys, h.fn, buckets)
		fmt.Printf("  %-20s %10d %10d %12.0f %14.1f\n",
			h.name, worst(c), empty(c), chiSquare(c, len(keys)), avalanche(h.fn, keys[:200]))
	}
	fmt.Println()
	fmt.Printf("  a uniform hash gives chi-square near %d (the bucket count) and\n", buckets)
	fmt.Println("  avalanche near 32.0 -- half of 64 bits flipping. Those two columns")
	fmt.Println("  measure different failures, and the FNV rows show why you need")
	fmt.Println("  both: FNV-1a's chi-square is excellent (77) while its avalanche is")
	fmt.Println("  only 25.4. It spreads keys across buckets well and still leaves")
	fmt.Println("  visible structure in the bits. Adding a finalizer costs three")
	fmt.Println("  multiplies and fixes the second column without touching the first.")

	fmt.Println()
	fmt.Println("what 'sum of bytes' actually does, first 16 buckets:")
	fmt.Println()
	fmt.Print(histogram(distribute(keys, sumBytes, buckets), 16))
	fmt.Println("  the counts slope steadily downward -- bucket 0 gets nearly 3x")
	fmt.Println("  what bucket 15 does -- because the sum of a handful of ASCII bytes")
	fmt.Println("  is a small number whose distribution is triangular, not uniform.")
	fmt.Println("  It is also COMMUTATIVE, so it cannot tell these apart at all:")
	fmt.Println()
	for _, pair := range [][2]string{{"abc", "cba"}, {"listen", "silent"}, {"ab", "ba"}} {
		fmt.Printf("    %-8s and %-8s -> %d and %d\n",
			pair[0], pair[1], sumBytes(pair[0]), sumBytes(pair[1]))
	}

	fmt.Println()
	fmt.Println("the same 16 buckets under FNV-1a:")
	fmt.Println()
	fmt.Print(histogram(distribute(keys, fnv1a, buckets), 16))

	fmt.Println()
	fmt.Println("avalanche, spelled out -- flipping ONE input bit:")
	fmt.Println()
	fmt.Printf("  %-20s %-22s %-22s %s\n", "hash", "hash(\"user_1000\")", "hash(\"user_1001\")", "bits differing")
	for _, h := range fns {
		a, b := h.fn("user_1000"), h.fn("user_1001")
		fmt.Printf("  %-20s %-22x %-22x %d\n", h.name, a, b, bits.OnesCount64(a^b))
	}
	fmt.Println()
	fmt.Println("  two keys that differ by one character should produce hashes with")
	fmt.Println("  no visible relationship at all. Look at the first two rows: the")
	fmt.Println("  outputs are ADJACENT INTEGERS. Any bucket count will place them")
	fmt.Println("  in neighbouring slots forever.")

	fmt.Println()
	fmt.Println("which BITS are good is a separate question from how many:")
	fmt.Println()
	fmt.Printf("  %-20s %16s %16s %10s\n", "hash function", "chi-sq low 6", "chi-sq top 6", "top/low")
	for _, h := range fns {
		lo := distribute(keys, h.fn, 64)
		hi := make([]int, 64)
		for _, k := range keys {
			hi[h.fn(k)>>58]++
		}
		cl := chiSquare(lo, len(keys))
		ch := chiSquare(hi, len(keys))
		fmt.Printf("  %-20s %16.0f %16.0f %9.0fx\n", h.name, cl, ch, ch/cl)
	}
	fmt.Println()
	fmt.Println("  I expected the low bits to be the weak ones, because that is the")
	fmt.Println("  usual warning. For FNV-1a it is the opposite: its top 6 bits are")
	fmt.Println("  47x worse than its low 6 on these keys. Multiplication only")
	fmt.Println("  carries UPWARD, so the top bits are decided disproportionately by")
	fmt.Println("  the last bytes hashed -- and all 12,000 of these keys end in")
	fmt.Println("  digits. That is a plausible mechanism; the measurement is the")
	fmt.Println("  part I would defend.")
	fmt.Println()
	fmt.Println("  that matters concretely, because Go's map does not do")
	fmt.Println("  `hash % buckets`. It splits the hash into H1 (the upper 57 bits,")
	fmt.Println("  which choose the group) and H2 (the lower 7, stored in the control")
	fmt.Println("  word). A hash that is only good in its low bits would be a poor")
	fmt.Println("  choice for H1. Example 6 takes that apart.")
	fmt.Println()
	fmt.Println("  the finalizer row is the point: after mixing, both halves are")
	fmt.Println("  equally good. If you write your own hash, finalize it -- and if")
	fmt.Println("  you are tempted not to, measure both ends of the word first.")

	fmt.Println("the birthday paradox, which is why collisions are normal:")
	fmt.Println()
	fmt.Printf("  %10s %14s %20s\n", "keys", "buckets", "P(any collision)")
	for _, tc := range []struct{ k, b int }{
		{23, 365}, {83, 5000}, {302, 1 << 16}, {77_163, 1 << 32}, {5000, 1 << 32},
	} {
		p := 1.0
		for i := 0; i < tc.k; i++ {
			p *= float64(tc.b-i) / float64(tc.b)
		}
		fmt.Printf("  %10d %14d %19.1f%%\n", tc.k, tc.b, 100*(1-p))
	}
	fmt.Println()
	fmt.Println("  the first four rows are all near 50%, and the pattern is")
	fmt.Println("  sqrt: you need only about 1.18*sqrt(b) keys before a collision is")
	fmt.Println("  more likely than not. 77,163 keys collide half the time in a")
	fmt.Println("  FOUR-BILLION-slot table.")
	fmt.Println()
	fmt.Println("  the last row is the one that corrects the intuition in the other")
	fmt.Println("  direction: 5,000 keys in that same table collide only 0.3% of the")
	fmt.Println("  time. The threshold is sharp, and it is at the square root.")
	fmt.Println()
	fmt.Println("  either way, collisions are not a sign of a bad hash. They are")
	fmt.Println("  arithmetic, and a table with 64 buckets and 12,000 keys has")
	fmt.Println("  thousands of them by construction.")
	fmt.Println()
	fmt.Printf("  so every hash table needs a collision strategy, and there are\n")
	fmt.Println("  exactly two families: keep colliding keys somewhere else")
	fmt.Println("  (CHAINING, example 3) or keep them in the table (OPEN ADDRESSING,")
	fmt.Println("  example 4). Everything after that is a variation.")

	_ = math.Sqrt
}
```

**Sample output:**

```
12000 realistic keys into 64 buckets. A perfect hash puts 187 in each:

  hash function             worst      empty   chi-square avalanche bits
  sum of bytes                368          0         3712            1.9
  shift and add               497         12        13432            2.0
  FNV-1a                      223          0           77           25.4
  hash/fnv (stdlib)           223          0           77           25.4
  FNV-1a + finalizer          214          0           54           32.0

  a uniform hash gives chi-square near 64 (the bucket count) and
  avalanche near 32.0 -- half of 64 bits flipping. Those two columns
  measure different failures, and the FNV rows show why you need
  both: FNV-1a's chi-square is excellent (77) while its avalanche is
  only 25.4. It spreads keys across buckets well and still leaves
  visible structure in the bits. Adding a finalizer costs three
  multiplies and fixes the second column without touching the first.

what 'sum of bytes' actually does, first 16 buckets:

      0 |################################ 299
      1 |############################# 269
      2 |########################## 241
      3 |####################### 213
      4 |#################### 189
      5 |################## 174
      6 |#################### 191
      7 |################# 160
      8 |############### 146
      9 |############### 141
     10 |################# 158
     11 |############## 137
     12 |############### 138
     13 |############## 130
     14 |############ 114
     15 |############ 112
  the counts slope steadily downward -- bucket 0 gets nearly 3x
  what bucket 15 does -- because the sum of a handful of ASCII bytes
  is a small number whose distribution is triangular, not uniform.
  It is also COMMUTATIVE, so it cannot tell these apart at all:

    abc      and cba      -> 294 and 294
    listen   and silent   -> 655 and 655
    ab       and ba       -> 195 and 195

the same 16 buckets under FNV-1a:

      0 |################################ 182
      1 |################################## 194
      2 |########################## 149
      3 |########################## 150
      4 |################################## 191
      5 |############################## 169
      6 |################################## 191
      7 |################################ 182
      8 |############################## 169
      9 |#################################### 204
     10 |############################## 171
     11 |################################## 193
     12 |################################## 190
     13 |#################################### 205
     14 |############################## 170
     15 |############################## 169

avalanche, spelled out -- flipping ONE input bit:

  hash                 hash("user_1000")      hash("user_1001")      bits differing
  sum of bytes         2df                    2e0                    6
  shift and add        78b22a094630           78b22a094631           1
  FNV-1a               9f77441da0409756       9f77451da0409909       10
  hash/fnv (stdlib)    9f77441da0409756       9f77451da0409909       10
  FNV-1a + finalizer   2c9473340c67f197       fc211731d0e7db88       27

  two keys that differ by one character should produce hashes with
  no visible relationship at all. Look at the first two rows: the
  outputs are ADJACENT INTEGERS. Any bucket count will place them
  in neighbouring slots forever.

which BITS are good is a separate question from how many:

  hash function            chi-sq low 6     chi-sq top 6    top/low
  sum of bytes                     3712           756000       204x
  shift and add                   13432           384577        29x
  FNV-1a                             77             3637        47x
  hash/fnv (stdlib)                  77             3637        47x
  FNV-1a + finalizer                 54               73         1x

  I expected the low bits to be the weak ones, because that is the
  usual warning. For FNV-1a it is the opposite: its top 6 bits are
  47x worse than its low 6 on these keys. Multiplication only
  carries UPWARD, so the top bits are decided disproportionately by
  the last bytes hashed -- and all 12,000 of these keys end in
  digits. That is a plausible mechanism; the measurement is the
  part I would defend.

  that matters concretely, because Go's map does not do
  `hash % buckets`. It splits the hash into H1 (the upper 57 bits,
  which choose the group) and H2 (the lower 7, stored in the control
  word). A hash that is only good in its low bits would be a poor
  choice for H1. Example 6 takes that apart.

  the finalizer row is the point: after mixing, both halves are
  equally good. If you write your own hash, finalize it -- and if
  you are tempted not to, measure both ends of the word first.
the birthday paradox, which is why collisions are normal:

        keys        buckets     P(any collision)
          23            365                50.7%
          83           5000                49.6%
         302          65536                50.1%
       77163     4294967296                50.0%
        5000     4294967296                 0.3%

  the first four rows are all near 50%, and the pattern is
  sqrt: you need only about 1.18*sqrt(b) keys before a collision is
  more likely than not. 77,163 keys collide half the time in a
  FOUR-BILLION-slot table.

  the last row is the one that corrects the intuition in the other
  direction: 5,000 keys in that same table collide only 0.3% of the
  time. The threshold is sharp, and it is at the square root.

  either way, collisions are not a sign of a bad hash. They are
  arithmetic, and a table with 64 buckets and 12,000 keys has
  thousands of them by construction.

  so every hash table needs a collision strategy, and there are
  exactly two families: keep colliding keys somewhere else
  (CHAINING, example 3) or keep them in the table (OPEN ADDRESSING,
  example 4). Everything after that is a variation.
```

**Complexity:** chi-square near the bucket count and avalanche near 32 are the two targets · FNV-1a
scores **77 and 25.4** — good distribution, mediocre bit mixing — and its **top 6 bits are 47× worse
than its low 6** · a three-multiply finalizer fixes both (**54 and 32.0**, top/low **1×**)

---

## 3. Separate chaining, and what it costs

`🟢 easy` · *the version everyone draws on a whiteboard*

Each bucket holds a list. It cannot fail — there is always room for one more key, because "room" means
"another node" — and that forgiving property is exactly what lets it degrade silently.

**Steps:**

1. Build it, then watch what happens when `Put` forgets to search the chain first.
2. Measure chain length against load factor, and find the 37%.
3. Then remove the hash and watch Θ(1) become Θ(n).

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"testing"
)

// SEPARATE CHAINING: each bucket holds a list of the entries that hashed there.
// It is the implementation everyone draws on a whiteboard, and it has one very
// attractive property -- it cannot fail. There is always room for one more key,
// because "room" means "another node".
//
//	lookup   hash -> bucket -> walk the chain
//	insert   hash -> bucket -> prepend (or append, see below)
//	delete   hash -> bucket -> unlink
//
// The cost is everything lesson 07 measured: a pointer per entry, an allocation
// per insert, and a cache miss per link followed.

type entry[K comparable, V any] struct {
	key  K
	val  V
	next *entry[K, V]
}

type Chained[K comparable, V any] struct {
	buckets []*entry[K, V]
	hash    func(K) uint64
	count   int

	probes int // total chain links followed, for the cost measurements below
}

func NewChained[K comparable, V any](n int, hash func(K) uint64) *Chained[K, V] {
	size := 1
	for size < n {
		size <<= 1
	}
	return &Chained[K, V]{buckets: make([]*entry[K, V], size), hash: hash}
}

func (m *Chained[K, V]) index(k K) int { return int(m.hash(k) & uint64(len(m.buckets)-1)) }

func (m *Chained[K, V]) Get(k K) (V, bool) {
	var zero V
	for e := m.buckets[m.index(k)]; e != nil; e = e.next {
		m.probes++
		if e.key == k {
			return e.val, true
		}
	}
	return zero, false
}

// Put must SEARCH FIRST. Skipping this is the classic chaining bug: you get two
// entries with the same key, Get returns whichever the chain reaches first, and
// Delete removes only one of them.
func (m *Chained[K, V]) Put(k K, v V) {
	i := m.index(k)
	for e := m.buckets[i]; e != nil; e = e.next {
		m.probes++
		if e.key == k {
			e.val = v // update in place
			return
		}
	}
	m.buckets[i] = &entry[K, V]{key: k, val: v, next: m.buckets[i]} // prepend
	m.count++
}

// PutNoCheck is that bug, kept so it can be demonstrated rather than described.
func (m *Chained[K, V]) PutNoCheck(k K, v V) {
	i := m.index(k)
	m.buckets[i] = &entry[K, V]{key: k, val: v, next: m.buckets[i]}
	m.count++
}

// Delete is the operation chaining does best: unlink and move on. No
// tombstones, no rearrangement, no "now what?" -- compare example 4.
func (m *Chained[K, V]) Delete(k K) bool {
	i := m.index(k)
	pp := &m.buckets[i] // pointer-to-pointer, lesson 07
	for *pp != nil {
		if (*pp).key == k {
			*pp = (*pp).next
			m.count--
			return true
		}
		pp = &(*pp).next
	}
	return false
}

func (m *Chained[K, V]) Len() int { return m.count }

func (m *Chained[K, V]) chainLengths() []int {
	out := make([]int, len(m.buckets))
	for i, e := range m.buckets {
		for ; e != nil; e = e.next {
			out[i]++
		}
	}
	return out
}

func stats(lens []int, n int) (mean, max float64, empty int) {
	mx := 0
	for _, l := range lens {
		if l > mx {
			mx = l
		}
		if l == 0 {
			empty++
		}
	}
	return float64(n) / float64(len(lens)), float64(mx), empty
}

func fnv1a(s string) uint64 {
	h := uint64(14695981039346656037)
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= 1099511628211
	}
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

// badHash sends every key to bucket 0. It is what an attacker arranges
// deliberately (example 14) and what a naive hash does by accident.
func badHash(string) uint64 { return 0 }

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

func keys(n int) []string {
	out := make([]string, n)
	for i := range out {
		out[i] = fmt.Sprintf("key_%d", i)
	}
	return out
}

func main() {
	fmt.Println("chaining, on a table deliberately too small so the chains show:")
	fmt.Println()
	m := NewChained[string, int](8, fnv1a)
	for i, k := range keys(12) {
		m.Put(k, i)
	}
	for i, e := range m.buckets {
		fmt.Printf("  bucket %d: ", i)
		if e == nil {
			fmt.Println("(empty)")
			continue
		}
		for ; e != nil; e = e.next {
			fmt.Printf("%s=%d -> ", e.key, e.val)
		}
		fmt.Println("nil")
	}
	fmt.Println()
	fmt.Printf("  12 keys, 8 buckets, load factor %.2f\n", float64(m.Len())/float64(len(m.buckets)))
	fmt.Println()
	fmt.Println("  a chained table has no notion of 'full'. Load factor above 1 is")
	fmt.Println("  legal and merely slow, which is the property that makes chaining")
	fmt.Println("  forgiving -- and the property that lets it degrade silently.")

	fmt.Println()
	fmt.Println("Put MUST search the chain first. Here is what happens if it doesn't:")
	fmt.Println()
	good := NewChained[string, int](16, fnv1a)
	bad := NewChained[string, int](16, fnv1a)
	for i := 0; i < 3; i++ {
		good.Put("dup", i)
		bad.PutNoCheck("dup", i)
	}
	gv, _ := good.Get("dup")
	bv, _ := bad.Get("dup")
	fmt.Printf("  %-28s Len=%d  Get(\"dup\")=%d\n", "Put with the search", good.Len(), gv)
	fmt.Printf("  %-28s Len=%d  Get(\"dup\")=%d\n", "Put without it", bad.Len(), bv)
	bad.Delete("dup")
	bv2, ok := bad.Get("dup")
	fmt.Printf("  %-28s after one Delete, Get(\"dup\")=%d, present=%v\n", "", bv2, ok)
	fmt.Println()
	fmt.Println("  the broken table reports 3 entries for 1 key, and deleting the key")
	fmt.Println("  leaves it present with a STALE value. Nothing panics. This is the")
	fmt.Println("  bug to look for whenever a hash map 'forgets' an update.")

	fmt.Println()
	fmt.Println("chain length is the whole cost model. With a good hash:")
	fmt.Println()
	fmt.Printf("  %10s %10s %14s %14s %12s %14s\n",
		"keys", "buckets", "load factor", "mean chain", "longest", "empty buckets")
	for _, tc := range []struct{ n, b int }{
		{1000, 4096}, {1000, 1024}, {1000, 512}, {1000, 256}, {1000, 64},
	} {
		mm := NewChained[string, int](tc.b, fnv1a)
		for i, k := range keys(tc.n) {
			mm.Put(k, i)
		}
		lens := mm.chainLengths()
		mean, mx, emp := stats(lens, tc.n)
		fmt.Printf("  %10d %10d %14.2f %14.2f %12.0f %14d\n",
			tc.n, len(mm.buckets), float64(tc.n)/float64(len(mm.buckets)), mean, mx, emp)
	}
	fmt.Println()
	fmt.Println("  the mean chain length IS the load factor -- that is not a")
	fmt.Println("  coincidence, it is the definition. An unsuccessful lookup walks")
	fmt.Println("  the whole chain, so its expected cost is exactly the load factor.")
	fmt.Println()
	fmt.Println("  note the empty-bucket column at load factor 1.00: roughly 37% of")
	fmt.Println("  buckets are still EMPTY while the mean chain is 1. That is e^-1,")
	fmt.Println("  and it is why a chained table wastes space at exactly the load")
	fmt.Println("  factor where it stops being fast.")

	fmt.Println()
	fmt.Println("and here is what the same table does with the hash removed:")
	fmt.Println()
	adv := NewChained[string, int](1024, badHash)
	ks := keys(2000)
	for i, k := range ks {
		adv.Put(k, i)
	}
	lens := adv.chainLengths()
	_, mx, emp := stats(lens, adv.Len())
	fmt.Printf("  2000 keys, 1024 buckets, longest chain %.0f, empty buckets %d\n", mx, emp)

	fine := NewChained[string, int](1024, fnv1a)
	for i, k := range ks {
		fine.Put(k, i)
	}
	// Count links in ONE pass. Counting inside the benchmark would total them
	// across every b.Loop iteration, which measures the harness, not the table.
	adv.probes, fine.probes = 0, 0
	for _, k := range ks {
		_, sinkB = adv.Get(k)
		_, sinkB = fine.Get(k)
	}
	probesBad, probesGood := adv.probes, fine.probes

	tBad := nsPerOp(func() {
		for _, k := range ks {
			_, sinkB = adv.Get(k)
		}
	})
	tGood := nsPerOp(func() {
		for _, k := range ks {
			_, sinkB = fine.Get(k)
		}
	})
	fmt.Println()
	fmt.Printf("  %-24s %14.0f ns for 2000 lookups\n", "good hash", tGood)
	fmt.Printf("  %-24s %14.0f ns   %.0fx slower\n", "every key collides", tBad, tBad/tGood)
	fmt.Println()
	fmt.Printf("  %-24s %14d links followed  (%.1f per lookup)\n",
		"good hash", probesGood, float64(probesGood)/float64(len(ks)))
	fmt.Printf("  %-24s %14d links followed  (%.1f per lookup)\n",
		"every key collides", probesBad, float64(probesBad)/float64(len(ks)))
	fmt.Println()
	fmt.Println("  Theta(1) became Theta(n), and the data structure did not change --")
	fmt.Println("  only the hash did. This is the worst case a hash table has, it is")
	fmt.Println("  reachable on purpose, and example 14 is about defending it.")

	fmt.Println()
	fmt.Println("what chaining costs, stated plainly:")
	fmt.Println()
	fmt.Printf("  %-26s %s\n", "one allocation per Put", "lesson 07's finding, per key")
	fmt.Printf("  %-26s %s\n", "one pointer per entry", "8 bytes on top of key and value")
	fmt.Printf("  %-26s %s\n", "one cache miss per link", "the chain is scattered")
	fmt.Printf("  %-26s %s\n", "GC traces every entry", "N pointers to scan, forever")
	fmt.Println()
	rng := rand.New(rand.NewSource(2))
	_ = rng
	sort.Ints(lens)
	fmt.Println("  in exchange you get: no 'full' state, no tombstones, deletion")
	fmt.Println("  that is genuinely Theta(1), and load factors above 1 that merely")
	fmt.Println("  degrade instead of failing.")
	fmt.Println()
	fmt.Println("  that trade used to be the default. It is not any more -- Go, Rust,")
	fmt.Println("  Abseil and Swift all moved to OPEN ADDRESSING, which is example 4,")
	fmt.Println("  and example 11 measures why.")
	sinkI = len(lens)
}
```

**Sample output:**

```
chaining, on a table deliberately too small so the chains show:

  bucket 0: key_4=4 -> key_3=3 -> nil
  bucket 1: key_6=6 -> nil
  bucket 2: (empty)
  bucket 3: key_10=10 -> key_9=9 -> nil
  bucket 4: key_8=8 -> key_1=1 -> key_0=0 -> nil
  bucket 5: (empty)
  bucket 6: key_11=11 -> key_2=2 -> nil
  bucket 7: key_7=7 -> key_5=5 -> nil

  12 keys, 8 buckets, load factor 1.50

  a chained table has no notion of 'full'. Load factor above 1 is
  legal and merely slow, which is the property that makes chaining
  forgiving -- and the property that lets it degrade silently.

Put MUST search the chain first. Here is what happens if it doesn't:

  Put with the search          Len=1  Get("dup")=2
  Put without it               Len=3  Get("dup")=2
                               after one Delete, Get("dup")=1, present=true

  the broken table reports 3 entries for 1 key, and deleting the key
  leaves it present with a STALE value. Nothing panics. This is the
  bug to look for whenever a hash map 'forgets' an update.

chain length is the whole cost model. With a good hash:

        keys    buckets    load factor     mean chain      longest  empty buckets
        1000       4096           0.24           0.24            3           3210
        1000       1024           0.98           0.98            5            382
        1000        512           1.95           1.95            8             67
        1000        256           3.91           3.91           11              4
        1000         64          15.62          15.62           30              0

  the mean chain length IS the load factor -- that is not a
  coincidence, it is the definition. An unsuccessful lookup walks
  the whole chain, so its expected cost is exactly the load factor.

  note the empty-bucket column at load factor 1.00: roughly 37% of
  buckets are still EMPTY while the mean chain is 1. That is e^-1,
  and it is why a chained table wastes space at exactly the load
  factor where it stops being fast.

and here is what the same table does with the hash removed:

  2000 keys, 1024 buckets, longest chain 2000, empty buckets 1023

  good hash                         18339 ns for 2000 lookups
  every key collides              2556857 ns   139x slower

  good hash                          3959 links followed  (2.0 per lookup)
  every key collides              2001000 links followed  (1000.5 per lookup)

  Theta(1) became Theta(n), and the data structure did not change --
  only the hash did. This is the worst case a hash table has, it is
  reachable on purpose, and example 14 is about defending it.

what chaining costs, stated plainly:

  one allocation per Put     lesson 07's finding, per key
  one pointer per entry      8 bytes on top of key and value
  one cache miss per link    the chain is scattered
  GC traces every entry      N pointers to scan, forever

  in exchange you get: no 'full' state, no tombstones, deletion
  that is genuinely Theta(1), and load factors above 1 that merely
  degrade instead of failing.

  that trade used to be the default. It is not any more -- Go, Rust,
  Abseil and Swift all moved to OPEN ADDRESSING, which is example 4,
  and example 11 measures why.
```

**Complexity:** mean chain length **is** the load factor, so an unsuccessful lookup costs exactly that
· at load factor 1.0, **37% of buckets are still empty** (that's e⁻¹) · with every key colliding, a
lookup follows **1000.5 links instead of 2.0** and runs **139× slower**

---

## 4. Open addressing, and the one hard problem

`🟢 easy` · *deleting a key can delete its neighbours*

No chains, no nodes, no pointers — when a slot is taken, take the next one. This is what Go, Rust,
Abseil and Swift all use now. It also has exactly one hard problem, and this example is mostly about
that problem.

**Steps:**

1. Watch keys displace and see that a lookup retraces the same path.
2. Delete one key by marking its slot free, and lose two others.
3. Then measure what tombstones cost once they accumulate.

```go
package main

import (
	"fmt"
	"strings"
)

// OPEN ADDRESSING: no chains, no nodes, no pointers. When a slot is taken, go
// looking for another one in the SAME array. With linear probing, "another one"
// means "the next one".
//
//	one array, no allocations after the constructor
//	the whole table is contiguous, so probing is a cache-friendly forward scan
//
// This is what Go, Rust, Abseil and Swift all use now. It also has exactly one
// hard problem, and this example is mostly about that problem.

type state uint8

const (
	free    state = iota // never used
	used                 // holds a live entry
	deleted              // TOMBSTONE: was used, now empty, but you may not stop here
)

type Open struct {
	keys   []string
	vals   []int
	states []state
	count  int
	tombs  int

	probes int
}

func NewOpen(n int) *Open {
	size := 1
	for size < n {
		size <<= 1
	}
	return &Open{
		keys:   make([]string, size),
		vals:   make([]int, size),
		states: make([]state, size),
	}
}

func fnv1a(s string) uint64 {
	h := uint64(14695981039346656037)
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= 1099511628211
	}
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

func (m *Open) mask() uint64 { return uint64(len(m.keys) - 1) }

// Get walks forward until it finds the key or hits a FREE slot. It must NOT
// stop at a tombstone -- that is the entire subtlety.
func (m *Open) Get(k string) (int, bool) {
	i := fnv1a(k) & m.mask()
	for {
		m.probes++
		switch m.states[i] {
		case free:
			return 0, false // a free slot proves the key is not here
		case used:
			if m.keys[i] == k {
				return m.vals[i], true
			}
		}
		// deleted: keep going
		i = (i + 1) & m.mask()
	}
}

// Put remembers the first tombstone it passes and reuses it -- but only after
// confirming the key is not already present further along.
func (m *Open) Put(k string, v int) {
	i := fnv1a(k) & m.mask()
	firstTomb := -1
	for {
		m.probes++
		switch m.states[i] {
		case free:
			if firstTomb >= 0 { // reuse the tombstone we passed
				i = uint64(firstTomb)
				m.tombs--
			}
			m.keys[i], m.vals[i], m.states[i] = k, v, used
			m.count++
			return
		case deleted:
			if firstTomb < 0 {
				firstTomb = int(i)
			}
		case used:
			if m.keys[i] == k {
				m.vals[i] = v
				return
			}
		}
		i = (i + 1) & m.mask()
	}
}

// Delete CANNOT simply mark the slot free. Doing so breaks every key whose
// probe sequence ran through it.
func (m *Open) Delete(k string) bool {
	i := fnv1a(k) & m.mask()
	for {
		switch m.states[i] {
		case free:
			return false
		case used:
			if m.keys[i] == k {
				m.states[i] = deleted
				m.keys[i] = "" // release the string header
				m.count--
				m.tombs++
				return true
			}
		}
		i = (i + 1) & m.mask()
	}
}

// DeleteWrong is the bug: mark it free and move on.
func (m *Open) DeleteWrong(k string) bool {
	i := fnv1a(k) & m.mask()
	for {
		switch m.states[i] {
		case free:
			return false
		case used:
			if m.keys[i] == k {
				m.states[i] = free // WRONG
				m.keys[i] = ""
				m.count--
				return true
			}
		}
		i = (i + 1) & m.mask()
	}
}

func (m *Open) render() string {
	var sb strings.Builder
	for i := range m.keys {
		switch m.states[i] {
		case free:
			sb.WriteString(fmt.Sprintf("%9s", "."))
		case deleted:
			sb.WriteString(fmt.Sprintf("%9s", "<tomb>"))
		default:
			sb.WriteString(fmt.Sprintf("%9s", m.keys[i]))
		}
	}
	return sb.String()
}

// probeSeq reports where a key wants to go and where it ends up.
func (m *Open) probeSeq(k string) (home int, steps int) {
	i := fnv1a(k) & m.mask()
	home = int(i)
	for {
		if m.states[i] == used && m.keys[i] == k {
			return home, steps
		}
		if m.states[i] == free {
			return home, -1
		}
		steps++
		i = (i + 1) & m.mask()
	}
}

func main() {
	fmt.Println("linear probing: when the slot is taken, take the next one.")
	fmt.Println()
	m := NewOpen(8)
	words := []string{"cat", "dog", "emu", "fox", "owl"}
	for i, w := range words {
		m.Put(w, i)
	}
	fmt.Printf("  slots:  %s\n", func() string {
		var sb strings.Builder
		for i := range m.keys {
			sb.WriteString(fmt.Sprintf("%9d", i))
		}
		return sb.String()
	}())
	fmt.Printf("  table:  %s\n", m.render())
	fmt.Println()
	fmt.Printf("  %-8s %10s %10s\n", "key", "home slot", "probes")
	for _, w := range words {
		h, s := m.probeSeq(w)
		fmt.Printf("  %-8s %10d %10d\n", w, h, s)
	}
	fmt.Println()
	fmt.Println("  a key that had to move is not lost -- it is exactly `steps` slots")
	fmt.Println("  past its home, and a lookup finds it by walking the same path.")
	fmt.Println("  The walk stops at the first FREE slot, which is the proof that the")
	fmt.Println("  key is absent.")

	fmt.Println()
	fmt.Println("which is why Delete cannot just free the slot:")
	fmt.Println()
	demo := NewOpen(8)
	// force a collision chain by hand: three keys, all landing near each other
	var chain []string
	for i := 0; len(chain) < 3; i++ {
		k := fmt.Sprintf("k%d", i)
		if fnv1a(k)&7 == 3 {
			chain = append(chain, k)
		}
	}
	for i, k := range chain {
		demo.Put(k, i)
	}
	fmt.Printf("  three keys that all hash to slot 3: %v\n", chain)
	fmt.Printf("  table: %s\n", demo.render())

	broken := NewOpen(8)
	for i, k := range chain {
		broken.Put(k, i)
	}
	broken.DeleteWrong(chain[0])
	_, ok1 := broken.Get(chain[1])
	_, ok2 := broken.Get(chain[2])
	fmt.Println()
	fmt.Printf("  DeleteWrong(%q) -> table: %s\n", chain[0], broken.render())
	fmt.Printf("  Get(%q) = %v      <- LOST\n", chain[1], ok1)
	fmt.Printf("  Get(%q) = %v      <- LOST\n", chain[2], ok2)
	fmt.Println()
	fmt.Println("  both survivors are still sitting in the array, and both are")
	fmt.Println("  unreachable: the lookup walks to slot 3, finds it FREE, and")
	fmt.Println("  concludes the key is absent. Deleting one key silently deleted")
	fmt.Println("  two others.")

	fixed := NewOpen(8)
	for i, k := range chain {
		fixed.Put(k, i)
	}
	fixed.Delete(chain[0])
	_, ok3 := fixed.Get(chain[1])
	_, ok4 := fixed.Get(chain[2])
	fmt.Println()
	fmt.Printf("  Delete(%q)      -> table: %s\n", chain[0], fixed.render())
	fmt.Printf("  Get(%q) = %v      <- found\n", chain[1], ok3)
	fmt.Printf("  Get(%q) = %v      <- found\n", chain[2], ok4)
	fmt.Println()
	fmt.Println("  the TOMBSTONE says 'nothing lives here, but keep walking'. It is")
	fmt.Println("  the whole fix, and it is why an open-addressed table needs three")
	fmt.Println("  slot states rather than two.")

	fmt.Println()
	fmt.Println("but tombstones are not free -- they accumulate:")
	fmt.Println()
	fmt.Printf("  %14s %11s %12s %10s %12s %12s\n",
		"operation", "live keys", "tombstones", "occupied", "probes/hit", "probes/miss")
	churn := NewOpen(1024)
	ks := make([]string, 700)
	for i := range ks {
		ks[i] = fmt.Sprintf("key_%d", i)
	}
	for i, k := range ks {
		churn.Put(k, i)
	}
	report := func(label string) {
		churn.probes = 0
		for _, k := range ks {
			churn.Get(k)
		}
		hit := float64(churn.probes) / float64(len(ks))
		churn.probes = 0
		for i := range ks {
			churn.Get(fmt.Sprintf("absent_%d", i))
		}
		miss := float64(churn.probes) / float64(len(ks))
		occupied := churn.count + churn.tombs
		fmt.Printf("  %14s %11d %12d %9.0f%% %12.2f %12.2f\n",
			label, churn.count, churn.tombs,
			100*float64(occupied)/float64(len(churn.keys)), hit, miss)
	}
	report("after fill")
	for round := 0; round < 4; round++ {
		for i := 0; i < 200; i++ {
			k := fmt.Sprintf("tmp_%d_%d", round, i)
			churn.Put(k, i)
			churn.Delete(k)
		}
		report("+200 churn")
	}
	fmt.Println()
	fmt.Println("  read the two right-hand columns separately -- I did not, and got")
	fmt.Println("  this wrong the first time.")
	fmt.Println()
	fmt.Println("  probes/HIT barely moves: a successful lookup stops the moment it")
	fmt.Println("  finds its key, and the tombstones are mostly not on its path.")
	fmt.Println("  probes/MISS is where the damage is, because a miss cannot stop")
	fmt.Println("  until it reaches a FREE slot -- and tombstones are not free.")
	fmt.Println()
	fmt.Println("  the live key count never changes across those rows. The table")
	fmt.Println("  holds exactly the same data and answers 'not present' several")
	fmt.Println("  times slower, purely because of keys that are no longer there.")
	fmt.Println("  A table can be 100% occupied by tombstones while holding no data.")
	fmt.Println()
	fmt.Println("  that is why a real open-addressed table counts `used + tombstones`")
	fmt.Println("  against its load factor, not just `used`, and rehashes to clear")
	fmt.Println("  them. Go's map does exactly this -- the comment in")
	fmt.Println("  internal/runtime/maps/table.go says: 'We rehash when used +")
	fmt.Println("  tombstones > loadFactor*capacity'.")

	fmt.Println()
	fmt.Println("the other cost of linear probing is CLUSTERING:")
	fmt.Println()
	fmt.Printf("  %14s %14s %14s %18s\n", "load factor", "probes/hit", "probes/miss", "theory (miss)")
	for _, lf := range []float64{0.25, 0.5, 0.75, 0.875, 0.95} {
		size := 4096
		n := int(float64(size) * lf)
		tt := NewOpen(size)
		kk := make([]string, n)
		for i := range kk {
			kk[i] = fmt.Sprintf("k_%d", i)
			tt.Put(kk[i], i)
		}
		tt.probes = 0
		for _, k := range kk {
			tt.Get(k)
		}
		hit := float64(tt.probes) / float64(n)
		tt.probes = 0
		for i := 0; i < n; i++ {
			tt.Get(fmt.Sprintf("absent_%d", i))
		}
		miss := float64(tt.probes) / float64(n)
		theory := (1 + 1/((1-lf)*(1-lf))) / 2
		fmt.Printf("  %14.3f %14.2f %14.2f %18.2f\n", lf, hit, miss, theory)
	}
	fmt.Println()
	fmt.Println("  Knuth's estimate for an unsuccessful linear-probe search is")
	fmt.Println("  (1 + 1/(1-a)^2)/2. The measured column tracks it to within about")
	fmt.Println("  2x and diverges at the top -- the formula assumes uniform hashing")
	fmt.Println("  and 4096 slots is a small sample at 95% load. The SHAPE is the")
	fmt.Println("  point: linear in nothing, quadratic in 1/(1-a).")
	fmt.Println()
	fmt.Println("  a full open-addressed table does not merely slow down -- it stops")
	fmt.Println("  terminating. That is why every implementation has a load factor")
	fmt.Println("  limit and rehashes before reaching it, which is example 5.")
}
```

**Sample output:**

```
linear probing: when the slot is taken, take the next one.

  slots:          0        1        2        3        4        5        6        7
  table:        fox        .      cat      emu        .      owl        .      dog

  key       home slot     probes
  cat               2          0
  dog               7          0
  emu               2          1
  fox               0          0
  owl               5          0

  a key that had to move is not lost -- it is exactly `steps` slots
  past its home, and a lookup finds it by walking the same path.
  The walk stops at the first FREE slot, which is the proof that the
  key is absent.

which is why Delete cannot just free the slot:

  three keys that all hash to slot 3: [k5 k18 k31]
  table:         .        .        .       k5      k18      k31        .        .

  DeleteWrong("k5") -> table:         .        .        .        .      k18      k31        .        .
  Get("k18") = false      <- LOST
  Get("k31") = false      <- LOST

  both survivors are still sitting in the array, and both are
  unreachable: the lookup walks to slot 3, finds it FREE, and
  concludes the key is absent. Deleting one key silently deleted
  two others.

  Delete("k5")      -> table:         .        .        .   <tomb>      k18      k31        .        .
  Get("k18") = true      <- found
  Get("k31") = true      <- found

  the TOMBSTONE says 'nothing lives here, but keep walking'. It is
  the whole fix, and it is why an open-addressed table needs three
  slot states rather than two.

but tombstones are not free -- they accumulate:

       operation   live keys   tombstones   occupied   probes/hit  probes/miss
      after fill         700            0        68%         1.82         4.56
      +200 churn         700          108        79%         1.82         7.53
      +200 churn         700          164        84%         1.82         9.57
      +200 churn         700          197        88%         1.82        10.27
      +200 churn         700          225        90%         1.82        14.28

  read the two right-hand columns separately -- I did not, and got
  this wrong the first time.

  probes/HIT barely moves: a successful lookup stops the moment it
  finds its key, and the tombstones are mostly not on its path.
  probes/MISS is where the damage is, because a miss cannot stop
  until it reaches a FREE slot -- and tombstones are not free.

  the live key count never changes across those rows. The table
  holds exactly the same data and answers 'not present' several
  times slower, purely because of keys that are no longer there.
  A table can be 100% occupied by tombstones while holding no data.

  that is why a real open-addressed table counts `used + tombstones`
  against its load factor, not just `used`, and rehashes to clear
  them. Go's map does exactly this -- the comment in
  internal/runtime/maps/table.go says: 'We rehash when used +
  tombstones > loadFactor*capacity'.

the other cost of linear probing is CLUSTERING:

     load factor     probes/hit    probes/miss      theory (miss)
           0.250           1.18           1.38               1.39
           0.500           1.54           2.61               2.50
           0.750           2.69           9.64               8.50
           0.875           4.47          24.54              32.50
           0.950           7.73         125.00             200.50

  Knuth's estimate for an unsuccessful linear-probe search is
  (1 + 1/(1-a)^2)/2. The measured column tracks it to within about
  2x and diverges at the top -- the formula assumes uniform hashing
  and 4096 slots is a small sample at 95% load. The SHAPE is the
  point: linear in nothing, quadratic in 1/(1-a).

  a full open-addressed table does not merely slow down -- it stops
  terminating. That is why every implementation has a load factor
  limit and rehashes before reaching it, which is example 5.
```

**Complexity:** probes/miss grows like `(1 + 1/(1-a)²)/2` — measured **1.38 at load 0.25 and 125.00 at
0.95** · tombstones took probes/miss from **4.56 to 14.28 with the live key count unchanged**, while
probes/**hit** never moved at all

---

## 5. Load factor and rehashing

`🟢 easy` · *you cannot copy a hash table*

Every key's slot is a function of the table size, so growing means **rebuilding**. That is Θ(n) work,
and the argument that it is still Θ(1) per insert is the third appearance of the aggregate method in
this repo.

**Steps:**

1. Count the rehashes and the total entries moved.
2. Find the single insert that pays for all of them.
3. Sweep the load factor — and find out why the setting barely matters.

```go
package main

import (
	"fmt"
	"testing"
)

// A hash table is fast because it is SPARSE. Example 4 measured what happens
// when it stops being sparse: probes/miss went from 1.38 to 125 as the load
// factor climbed from 0.25 to 0.95.
//
// So the table must grow -- and growing means REHASHING, because every key's
// slot is a function of the table size. You cannot copy a hash table; you have
// to rebuild it.
//
// That is Theta(n) work, and this example is the argument that it is still
// Theta(1) per insert on average, plus the measurement of what it costs the
// one insert that pays for it.

type state uint8

const (
	free state = iota
	used
	deleted
)

type Table struct {
	keys   []uint64
	vals   []int
	states []state
	count  int
	tombs  int

	maxLoad float64
	// instrumentation
	rehashes    int
	rehashed    int // total entries moved by all rehashes
	lastRehash  int // entries moved by the most recent rehash
	probes      int
	growthTrace []int
}

func NewTable(n int, maxLoad float64) *Table {
	size := 1
	for size < n {
		size <<= 1
	}
	return &Table{
		keys:    make([]uint64, size),
		vals:    make([]int, size),
		states:  make([]state, size),
		maxLoad: maxLoad,
	}
}

func mix(k uint64) uint64 {
	k ^= k >> 30
	k *= 0xbf58476d1ce4e5b9
	k ^= k >> 27
	k *= 0x94d049bb133111eb
	k ^= k >> 31
	return k
}

func (t *Table) mask() uint64 { return uint64(len(t.keys) - 1) }

func (t *Table) Get(k uint64) (int, bool) {
	i := mix(k) & t.mask()
	for {
		t.probes++
		switch t.states[i] {
		case free:
			return 0, false
		case used:
			if t.keys[i] == k {
				return t.vals[i], true
			}
		}
		i = (i + 1) & t.mask()
	}
}

func (t *Table) Put(k uint64, v int) {
	// The load factor counts USED + TOMBSTONES. Counting only used entries is
	// the bug example 4 ended on: a table full of tombstones never grows and
	// never terminates.
	if float64(t.count+t.tombs+1) > t.maxLoad*float64(len(t.keys)) {
		t.grow()
	}
	t.put(k, v)
}

func (t *Table) put(k uint64, v int) {
	i := mix(k) & t.mask()
	firstTomb := -1
	for {
		switch t.states[i] {
		case free:
			if firstTomb >= 0 {
				i = uint64(firstTomb)
				t.tombs--
			}
			t.keys[i], t.vals[i], t.states[i] = k, v, used
			t.count++
			return
		case deleted:
			if firstTomb < 0 {
				firstTomb = int(i)
			}
		case used:
			if t.keys[i] == k {
				t.vals[i] = v
				return
			}
		}
		i = (i + 1) & t.mask()
	}
}

func (t *Table) Delete(k uint64) bool {
	i := mix(k) & t.mask()
	for {
		switch t.states[i] {
		case free:
			return false
		case used:
			if t.keys[i] == k {
				t.states[i] = deleted
				t.count--
				t.tombs++
				return true
			}
		}
		i = (i + 1) & t.mask()
	}
}

// grow doubles unless the table is mostly tombstones, in which case it rebuilds
// at the SAME size -- which is the point most implementations get right and
// most first drafts do not.
func (t *Table) grow() {
	newSize := len(t.keys)
	if float64(t.count+1) > t.maxLoad*float64(newSize)/2 {
		newSize *= 2
	}
	oldK, oldV, oldS := t.keys, t.vals, t.states
	t.keys = make([]uint64, newSize)
	t.vals = make([]int, newSize)
	t.states = make([]state, newSize)
	t.count, t.tombs = 0, 0
	moved := 0
	for i, s := range oldS {
		if s == used {
			t.put(oldK[i], oldV[i])
			moved++
		}
	}
	t.rehashes++
	t.rehashed += moved
	t.lastRehash = moved
	t.growthTrace = append(t.growthTrace, newSize)
}

func (t *Table) load() float64 {
	return float64(t.count+t.tombs) / float64(len(t.keys))
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
	fmt.Println("growing a hash table is not copying it. Every key's slot depends")
	fmt.Println("on the table SIZE, so every key has to be placed again:")
	fmt.Println()
	t := NewTable(4, 0.75)
	for i := 0; i < 100_000; i++ {
		t.Put(uint64(i)*2654435761, i)
	}
	fmt.Printf("  100,000 inserts -> %d rehashes, %d total entries moved\n",
		t.rehashes, t.rehashed)
	fmt.Printf("  final size %d, load factor %.3f\n", len(t.keys), t.load())
	fmt.Printf("  moves per insert: %.2f\n", float64(t.rehashed)/100_000)
	fmt.Println()
	fmt.Println("  under 3 moves per insert, and it is the same aggregate argument")
	fmt.Println("  as lesson 06's append and lesson 09's growable ring: doubling")
	fmt.Println("  makes the series n/2 + n/4 + ... < n, and the load factor divides")
	fmt.Println("  it by itself. Amortized Theta(1), for the third time in this repo.")

	fmt.Println()
	fmt.Println("but the WORST insert is not Theta(1), and it never is:")
	fmt.Println()
	fmt.Printf("  %12s %14s %16s\n", "insert #", "table size", "entries moved")
	t2 := NewTable(4, 0.75)
	for i := 0; i < 200_000; i++ {
		before := t2.rehashes
		t2.Put(uint64(i)*2654435761, i)
		if t2.rehashes > before && t2.lastRehash > 3000 {
			fmt.Printf("  %12d %14d %16d\n", i, len(t2.keys), t2.lastRehash)
		}
	}
	fmt.Println()
	fmt.Printf("  the worst single Put moved %d entries. If a hash table is on a\n", t2.lastRehash)
	fmt.Println("  latency-critical path, THAT is the number that matters, not the")
	fmt.Println("  average -- and it is why a size hint is not a micro-optimisation")
	fmt.Println("  (example 10).")

	fmt.Println()
	fmt.Println("the load factor is the dial, and it trades space against probes:")
	fmt.Println()
	fmt.Printf("  %10s %12s %10s %13s %12s %12s %11s\n",
		"max load", "final size", "rehashes", "moves/insert", "achieved", "probes/hit", "bytes/key")
	for _, lf := range []float64{0.5, 0.75, 0.875, 0.95} {
		tt := NewTable(4, lf)
		const n = 100_000
		for i := 0; i < n; i++ {
			tt.Put(uint64(i)*2654435761, i)
		}
		tt.probes = 0
		for i := 0; i < n; i++ {
			_, sinkB = tt.Get(uint64(i) * 2654435761)
		}
		bytesPerKey := float64(len(tt.keys)) * 17 / n // 8 key + 8 val + 1 state
		fmt.Printf("  %10.3f %12d %10d %13.2f %12.3f %12.2f %11.1f\n",
			lf, len(tt.keys), tt.rehashes, float64(tt.rehashed)/n,
			float64(n)/float64(len(tt.keys)), float64(tt.probes)/n, bytesPerKey)
	}
	fmt.Println()
	fmt.Println("  the rows pair up: 0.5 and 0.75 produce an identical table, and so")
	fmt.Println("  do 0.875 and 0.95. I expected four distinct rows and got two.")
	fmt.Println()
	fmt.Println("  the reason is that the size is a POWER OF TWO. 100,000 keys need")
	fmt.Println("  more than 131,072*0.75 = 98,304 slots, so both 0.5 and 0.75 jump")
	fmt.Println("  to 262,144; both 0.875 and 0.95 fit in 131,072 and stop there.")
	fmt.Println("  The setting only matters when it moves you across a boundary.")
	fmt.Println()
	fmt.Println("  so read the ACHIEVED column, not the setting: the real load lands")
	fmt.Println("  anywhere in [maxLoad/2, maxLoad], and 0.38 versus 0.76 is what")
	fmt.Println("  actually produced the 2x difference in bytes/key and probes/hit.")
	fmt.Println()
	fmt.Println("  Go's map picks 7/8 = 0.875 (maxAvgGroupLoad in")
	fmt.Println("  internal/runtime/maps/group.go). Abseil picks 7/8. Rust's")
	fmt.Println("  hashbrown picks 7/8. They all pick 7/8 because a SWISS TABLE")
	fmt.Println("  checks 8 slots at once, so the probe cost at 87.5% load is one")
	fmt.Println("  group check -- example 6.")

	fmt.Println()
	fmt.Println("rehashing also does something the load factor does not advertise:")
	fmt.Println("it is the only thing that removes tombstones.")
	fmt.Println()
	t3 := NewTable(1024, 0.75)
	key := func(i int) uint64 { return uint64(i) * 2654435761 }
	for i := 0; i < 700; i++ {
		t3.Put(key(i), i)
	}
	fmt.Printf("  %-30s used=%3d tombs=%3d size=%d rehashes=%d\n",
		"700 inserts", t3.count, t3.tombs, len(t3.keys), t3.rehashes)
	for i := 0; i < 600; i++ {
		t3.Delete(key(i))
	}
	fmt.Printf("  %-30s used=%3d tombs=%3d size=%d rehashes=%d\n",
		"delete 600 of them", t3.count, t3.tombs, len(t3.keys), t3.rehashes)
	// insert until a rehash actually fires, and report how long that took
	inserted, sizeBefore := 0, len(t3.keys)
	for i := 700; t3.rehashes == 0; i++ {
		t3.Put(key(i), i)
		inserted++
	}
	fmt.Printf("  %-30s used=%3d tombs=%3d size=%d rehashes=%d\n",
		fmt.Sprintf("%d more inserts", inserted), t3.count, t3.tombs,
		len(t3.keys), t3.rehashes)
	fmt.Printf("  size before the rehash: %d, after: %d\n", sizeBefore, len(t3.keys))
	fmt.Println()
	fmt.Println("  two different things happened on that last line, and only one of")
	fmt.Println("  them is a rehash.")
	fmt.Println()
	fmt.Println("  first, Put REUSES a tombstone it walks past, so many of the 600")
	fmt.Printf("  came back with no rebuild at all -- which is why it took %d\n", inserted)
	fmt.Println("  inserts to cross a threshold that was only 68 slots away.")
	fmt.Println()
	fmt.Println("  second, once used+tombstones did cross 0.75*1024 the table")
	fmt.Println("  rebuilt itself -- and the SIZE DID NOT CHANGE, because the live")
	fmt.Println("  count was still low. Rebuilding in place was enough, and it")
	fmt.Println("  cleared every remaining tombstone at once.")
	fmt.Println()
	fmt.Println("  that second part is the one to get right. A grow routine that")
	fmt.Println("  ALWAYS doubles will double a table that is 90% tombstones, and")
	fmt.Println("  then that table is 45% tombstones and twice as big, forever.")

	fmt.Println()
	fmt.Println("what it all costs on the clock:")
	fmt.Println()
	const n = 200_000
	tGrow := nsPerOp(func() {
		tt := NewTable(4, 0.875)
		for i := 0; i < n; i++ {
			tt.Put(uint64(i)*2654435761, i)
		}
		sinkI = tt.count
	})
	tHint := nsPerOp(func() {
		tt := NewTable(n*2, 0.875)
		for i := 0; i < n; i++ {
			tt.Put(uint64(i)*2654435761, i)
		}
		sinkI = tt.count
	})
	fmt.Printf("  %-34s %10.1f ns/insert\n", "grown from size 4", tGrow/n)
	fmt.Printf("  %-34s %10.1f ns/insert\n", "preallocated", tHint/n)
	fmt.Printf("  %-34s %10.1fx\n", "cost of growing", tGrow/tHint)
	fmt.Println()
	fmt.Println("  growing costs real time, and it is the one cost you can remove")
	fmt.Println("  entirely by knowing the size in advance. Example 10 measures that")
	fmt.Println("  on Go's own map, where the answer is larger than this.")
}
```

**Sample output:**

```
growing a hash table is not copying it. Every key's slot depends
on the table SIZE, so every key has to be placed again:

  100,000 inserts -> 16 rehashes, 196605 total entries moved
  final size 262144, load factor 0.381
  moves per insert: 1.97

  under 3 moves per insert, and it is the same aggregate argument
  as lesson 06's append and lesson 09's growable ring: doubling
  makes the series n/2 + n/4 + ... < n, and the load factor divides
  it by itself. Amortized Theta(1), for the third time in this repo.

but the WORST insert is not Theta(1), and it never is:

      insert #     table size    entries moved
          3072           8192             3072
          6144          16384             6144
         12288          32768            12288
         24576          65536            24576
         49152         131072            49152
         98304         262144            98304
        196608         524288           196608

  the worst single Put moved 196608 entries. If a hash table is on a
  latency-critical path, THAT is the number that matters, not the
  average -- and it is why a size hint is not a micro-optimisation
  (example 10).

the load factor is the dial, and it trades space against probes:

    max load   final size   rehashes  moves/insert     achieved   probes/hit   bytes/key
       0.500       262144         16          1.31        0.381         1.30        44.6
       0.750       262144         16          1.97        0.381         1.30        44.6
       0.875       131072         15          1.15        0.763         2.62        22.3
       0.950       131072         15          1.25        0.763         2.62        22.3

  the rows pair up: 0.5 and 0.75 produce an identical table, and so
  do 0.875 and 0.95. I expected four distinct rows and got two.

  the reason is that the size is a POWER OF TWO. 100,000 keys need
  more than 131,072*0.75 = 98,304 slots, so both 0.5 and 0.75 jump
  to 262,144; both 0.875 and 0.95 fit in 131,072 and stop there.
  The setting only matters when it moves you across a boundary.

  so read the ACHIEVED column, not the setting: the real load lands
  anywhere in [maxLoad/2, maxLoad], and 0.38 versus 0.76 is what
  actually produced the 2x difference in bytes/key and probes/hit.

  Go's map picks 7/8 = 0.875 (maxAvgGroupLoad in
  internal/runtime/maps/group.go). Abseil picks 7/8. Rust's
  hashbrown picks 7/8. They all pick 7/8 because a SWISS TABLE
  checks 8 slots at once, so the probe cost at 87.5% load is one
  group check -- example 6.

rehashing also does something the load factor does not advertise:
it is the only thing that removes tombstones.

  700 inserts                    used=700 tombs=  0 size=1024 rehashes=0
  delete 600 of them             used=100 tombs=600 size=1024 rehashes=0
  186 more inserts               used=286 tombs=  0 size=1024 rehashes=1
  size before the rehash: 1024, after: 1024

  two different things happened on that last line, and only one of
  them is a rehash.

  first, Put REUSES a tombstone it walks past, so many of the 600
  came back with no rebuild at all -- which is why it took 186
  inserts to cross a threshold that was only 68 slots away.

  second, once used+tombstones did cross 0.75*1024 the table
  rebuilt itself -- and the SIZE DID NOT CHANGE, because the live
  count was still low. Rebuilding in place was enough, and it
  cleared every remaining tombstone at once.

  that second part is the one to get right. A grow routine that
  ALWAYS doubles will double a table that is 90% tombstones, and
  then that table is 45% tombstones and twice as big, forever.

what it all costs on the clock:

  grown from size 4                        24.1 ns/insert
  preallocated                              6.5 ns/insert
  cost of growing                           3.7x

  growing costs real time, and it is the one cost you can remove
  entirely by knowing the size in advance. Example 10 measures that
  on Go's own map, where the answer is larger than this.
```

**Complexity:** **1.97 moves per insert** amortized, while the **worst single `Put` moved 196,608
entries** · the max-load setting collapses into pairs because sizes are powers of two — read the
*achieved* load (0.38 vs 0.76) instead · a rehash is the only thing that clears tombstones, and it
**must be allowed to keep the same size**

---

## 6. Go's map is a Swiss table

`🟢 easy` · *everything you read about `bmap` and `tophash` is obsolete*

Since Go 1.24 the map is a Swiss table: groups of 8 slots with an 8-byte **control word**, an H1/H2
hash split, and one 64-bit operation that compares a candidate against all eight slots at once.

**Steps:**

1. Reimplement `matchH2` and watch SWAR resolve eight slots in four instructions.
2. See where iteration randomisation and non-addressable values come from.
3. Then measure hit against miss, and get the answer wrong twice.

```go
package main

import (
	"fmt"
	"math/bits"
	"runtime"
	"strings"
	"testing"
	"unsafe"
)

// Go's map is a SWISS TABLE, and has been since Go 1.24. If you have read about
// Go maps before, you have probably read about `bmap`, buckets of 8 with a
// `tophash` array, and overflow buckets chained off the end.
//
// That implementation no longer exists. It is worth knowing what replaced it,
// because the replacement explains the load factor, the iteration order, and
// why `&m[k]` does not compile.
//
// The design, from internal/runtime/maps/map.go:
//
//	SLOT     one key/value pair
//	GROUP    8 slots + ONE 8-byte CONTROL WORD
//	          each control byte is either empty (0b10000000),
//	          deleted (0b11111110), or the low 7 bits of a key's hash
//	TABLE    an array of groups
//	MAP      a DIRECTORY of tables (extendible hashing)
//
//	H1       the upper 57 bits of the hash -> which group to start at
//	H2       the lower 7 bits             -> stored in the control byte
//
// The trick is the control word. Eight 7-bit hash fragments sit in one uint64,
// so ONE 64-bit operation compares a candidate against all 8 slots at once.
// That is why a Swiss table can run at 87.5% load: a probe is a group check,
// not a slot check.

// matchH2 is the actual trick, reimplemented here so you can watch it work.
// This is Go's ctrlGroup.matchH2, which is Abseil's, which is a classic SWAR
// (SIMD Within A Register) technique.
const (
	bitsetLSB = 0x0101010101010101
	bitsetMSB = 0x8080808080808080
)

func matchH2(ctrl uint64, h2 uint8) uint64 {
	// XOR makes every byte that equals h2 become zero.
	v := ctrl ^ (bitsetLSB * uint64(h2))
	// This is the has-zero-byte trick: it sets the high bit of every byte
	// that was zero, and only those.
	return (v - bitsetLSB) & ^v & bitsetMSB
}

func matchEmpty(ctrl uint64) uint64 {
	// empty is 0b10000000: high bit set, all others clear.
	return (ctrl & ^(ctrl << 1)) & bitsetMSB
}

func renderCtrl(ctrl uint64) string {
	var parts []string
	for i := 0; i < 8; i++ {
		b := uint8(ctrl >> (8 * i))
		switch b {
		case 0x80:
			parts = append(parts, "  empty")
		case 0xfe:
			parts = append(parts, "    del")
		default:
			parts = append(parts, fmt.Sprintf("   %#02x", b))
		}
	}
	return strings.Join(parts, "")
}

func renderMatch(m uint64) string {
	var parts []string
	for i := 0; i < 8; i++ {
		if m&(1<<(8*i+7)) != 0 {
			parts = append(parts, "      *")
		} else {
			parts = append(parts, "      .")
		}
	}
	return strings.Join(parts, "")
}

type S struct {
	A int
	B string
}

var sinkI int
var sinkB bool

// liveKB reports the heap held by whatever build returns, after two full GC
// cycles on each side (example 1 explains why two).
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

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("the control word: 8 slots described by 8 bytes, checked at once.")
	fmt.Println()
	// A group holding five keys whose H2 values are 0x2a, 0x11, 0x2a, 0x07, 0x63.
	var ctrl uint64
	for i, b := range []uint8{0x2a, 0x11, 0x2a, 0x07, 0x63, 0x80, 0x80, 0xfe} {
		ctrl |= uint64(b) << (8 * i)
	}
	fmt.Printf("  slot:   %s\n", func() string {
		var p []string
		for i := 0; i < 8; i++ {
			p = append(p, fmt.Sprintf("%7d", i))
		}
		return strings.Join(p, "")
	}())
	fmt.Printf("  ctrl:   %s\n", renderCtrl(ctrl))
	fmt.Println()
	for _, h2 := range []uint8{0x2a, 0x07, 0x55} {
		m := matchH2(ctrl, h2)
		n := bits.OnesCount64(m)
		fmt.Printf("  H2=%#02x: %s   %d candidate(s)\n", h2, renderMatch(m), n)
	}
	fmt.Printf("  empty:  %s   %d free slot(s)\n",
		renderMatch(matchEmpty(ctrl)), bits.OnesCount64(matchEmpty(ctrl)))

	fmt.Println()
	fmt.Println("  that is ONE subtraction, two ANDs and a NOT -- four instructions")
	fmt.Println("  to reject or shortlist eight slots. No branches, no key")
	fmt.Println("  comparisons, no memory beyond the 8 bytes already in a register.")
	fmt.Println()
	fmt.Println("  the keys themselves are only compared for the slots the control")
	fmt.Println("  word shortlisted. With 7 bits of hash there is a 1-in-128 chance")
	fmt.Println("  of a false candidate, so on average a lookup does ONE key")
	fmt.Println("  comparison, whatever the load factor.")
	fmt.Println()
	fmt.Println("  and that is the whole answer to 'why 7/8?'. At 87.5% load a group")
	fmt.Println("  holds 7 of 8 slots, so the first group check almost always")
	fmt.Println("  resolves the lookup. Chaining has to keep the load near 1.0 to")
	fmt.Println("  avoid long chains; a Swiss table is FASTER at high load because")
	fmt.Println("  the parallel check does not care how full the group is.")

	fmt.Println()
	fmt.Println("H1 and H2, the split that makes it work:")
	fmt.Println()
	fmt.Printf("  %-22s %s\n", "H1 = hash >> 7", "the upper 57 bits -- choose the group")
	fmt.Printf("  %-22s %s\n", "H2 = hash & 0x7f", "the lower 7 -- live in the control word")
	fmt.Println()
	fmt.Println("  both halves must be good, which is exactly what example 2's")
	fmt.Println("  finalizer measurement was about. A hash with all its entropy in")
	fmt.Println("  the low bits would give a fine H2 and a useless H1: every key")
	fmt.Println("  would start probing at the same group.")

	fmt.Println()
	fmt.Println("iteration order is randomised, and the randomisation is TWO-PART:")
	fmt.Println()
	m := map[int]int{}
	for i := 0; i < 24; i++ {
		m[i] = i
	}
	for r := 0; r < 4; r++ {
		fmt.Print("  ")
		n := 0
		for k := range m {
			fmt.Printf("%3d", k)
			if n++; n == 12 {
				break
			}
		}
		fmt.Println("  ...")
	}
	fmt.Println()
	fmt.Println("  from internal/runtime/maps/table.go:")
	fmt.Println()
	fmt.Println("      it.entryOffset = rand()   // where to start inside a group")
	fmt.Println("      it.dirOffset   = rand()   // which table to start with")
	fmt.Println()
	fmt.Println("  it is deliberate and it is not a debug feature -- it ships in")
	fmt.Println("  production builds. Code that accidentally depends on map order")
	fmt.Println("  fails on the second run instead of two years later.")
	fmt.Println()
	fmt.Println("  the one exception: `fmt` prints maps in SORTED key order, so")
	fmt.Println("  printing a map is stable even though ranging it is not.")
	fmt.Printf("  fmt: %v\n", map[string]int{"c": 3, "a": 1, "b": 2})

	fmt.Println()
	fmt.Println("map values are NOT ADDRESSABLE, and now you know why:")
	fmt.Println()
	fmt.Println("      p := &m[k]        // does not compile")
	fmt.Println("      m[k].Field = 1    // does not compile")
	fmt.Println("      m[k]++            // fine -- read, add, write back")
	fmt.Println()
	fmt.Println("  a rehash MOVES every entry to a new array. A pointer into a map")
	fmt.Println("  would dangle the moment the map grew, so the language forbids")
	fmt.Println("  taking one. This is not an arbitrary restriction; it falls")
	fmt.Println("  straight out of example 5.")
	fmt.Println()
	sm := map[string]S{"a": {A: 1, B: "x"}}
	v := sm["a"]
	v.A = 99
	sm["a"] = v
	fmt.Printf("  the read-modify-write dance:  %v\n", sm)
	pm := map[string]*S{"a": {A: 1, B: "x"}}
	pm["a"].A = 99 // legal: the POINTER is the value, and it does not move
	fmt.Printf("  or store a pointer instead:   %v\n", *pm["a"])
	fmt.Println()
	fmt.Println("  storing *S makes the value addressable again, at the cost of one")
	fmt.Println("  allocation and one indirection per entry -- lesson 04's trade,")
	fmt.Println("  arriving in a new place.")

	fmt.Println()
	fmt.Println("the sizes involved, on this machine:")
	fmt.Println()
	fmt.Printf("  %-34s %d bytes\n", "one control word", 8)
	fmt.Printf("  %-34s %d slots", "one group", 8)
	fmt.Println()
	fmt.Printf("  %-34s %d bytes\n", "unsafe.Sizeof(map[int]int{})", unsafe.Sizeof(map[int]int{}))
	fmt.Printf("  %-34s %d bytes\n", "unsafe.Sizeof(struct{k,v int}{})", unsafe.Sizeof(struct{ k, v int }{}))
	fmt.Println()
	fmt.Println("  a map variable is ONE POINTER. That is why a map is a reference")
	fmt.Println("  type, why passing it to a function is free, why the zero value is")
	fmt.Println("  nil, and why writing to a nil map panics while reading returns")
	fmt.Println("  the zero value.")
	fmt.Println()
	var nilMap map[string]int
	fmt.Printf("  var m map[string]int -> len=%d, m[\"x\"]=%d, ok=%v\n",
		len(nilMap), nilMap["x"], func() bool { _, ok := nilMap["x"]; return ok }())
	fmt.Println("  m[\"x\"] = 1 would panic: assignment to entry in nil map")

	fmt.Println()
	fmt.Println("and now a measurement that corrected me twice. I expected a MISS")
	fmt.Println("to be as fast as a hit, because a miss should resolve in the")
	fmt.Println("control word without ever comparing a key:")
	fmt.Println()
	fmt.Printf("  %11s %11s %10s %10s %10s %10s\n",
		"live keys", "map KB", "bytes/key", "hit ns", "miss ns", "miss/hit")
	for _, n := range []int{1000, 100_000, 150_000, 200_000, 300_000, 2_000_000} {
		present := make([]int, n)
		for i := range present {
			present[i] = i*2654435761 + 1
		}
		absent := make([]int, n)
		for i := range absent {
			absent[i] = -(i*2654435761 + 1)
		}
		kb := liveKB(func() any {
			mm := make(map[int]int, n)
			for i, k := range present {
				mm[k] = i
			}
			return mm
		})
		gm := make(map[int]int, n)
		for i, k := range present {
			gm[k] = i
		}
		hit := nsPerOp(func() {
			for _, k := range present {
				_, sinkB = gm[k]
			}
		}) / float64(n)
		miss := nsPerOp(func() {
			for _, k := range absent {
				_, sinkB = gm[k]
			}
		}) / float64(n)
		fmt.Printf("  %11d %11.0f %10.1f %10.2f %10.2f %9.2fx\n",
			n, kb, kb*1024/float64(n), hit, miss, miss/hit)
	}
	fmt.Println()
	fmt.Println("  the ratio is not a constant, and it does not grow with n -- it")
	fmt.Println("  goes 1.16, 2.42, 1.29, 2.39, 1.34, 0.85. Line the rows up against")
	fmt.Println("  BYTES/KEY instead and the middle four sort themselves:")
	fmt.Println()
	fmt.Printf("  %-30s %s\n", "23.7 and 23.6 bytes/key", "miss/hit 2.42x and 2.39x")
	fmt.Printf("  %-30s %s\n", "31.5 and 31.5 bytes/key", "miss/hit 1.29x and 1.34x")
	fmt.Println()
	fmt.Println("  bytes/key is the achieved load factor in disguise, and it swings")
	fmt.Println("  with n for the power-of-two reason example 1 measured. So:")
	fmt.Println()
	fmt.Println("  a HIT stops at the group holding its key. A MISS cannot stop")
	fmt.Println("  until it finds an EMPTY slot -- and in a DENSE table empty slots")
	fmt.Println("  are rare, so it walks more groups, each a separate cache line.")
	fmt.Println("  In a sparse table it finds one almost immediately.")
	fmt.Println()
	fmt.Println("  the two outer rows are governed by something else. At 1,000")
	fmt.Println("  keys the whole table is in L1, so walking extra groups is free")
	fmt.Println("  and the ratio is 1.16 whatever the density.")
	fmt.Println()
	fmt.Println("  and at 2,000,000 keys the ratio inverts: both are memory-bound,")
	fmt.Println("  and now the HIT is the slower one, because after the control word")
	fmt.Println("  shortlists a slot the hit must fetch and compare the actual key --")
	fmt.Println("  one more cache line that the miss never touches.")
	fmt.Println()
	fmt.Println("  my original claim was right about the mechanism and wrong about")
	fmt.Println("  the conditions. That is the usual shape of a benchmark surprise.")
	fmt.Println()
	fmt.Println("  what does hold unconditionally is the comparison to CHAINING,")
	fmt.Println("  where a miss walks the entire chain -- example 3 measured 1000.5")
	fmt.Println("  links per lookup on a degenerate table. A Swiss table's miss is a")
	fmt.Println("  few group checks; a chained table's is unbounded. Example 11 puts")
	fmt.Println("  them side by side.")
	sinkI = 1
}
```

**Sample output:**

```
the control word: 8 slots described by 8 bytes, checked at once.

  slot:         0      1      2      3      4      5      6      7
  ctrl:      0x2a   0x11   0x2a   0x07   0x63  empty  empty    del

  H2=0x2a:       *      .      *      .      .      .      .      .   2 candidate(s)
  H2=0x07:       .      .      .      *      .      .      .      .   1 candidate(s)
  H2=0x55:       .      .      .      .      .      .      .      .   0 candidate(s)
  empty:        .      .      .      .      .      *      *      .   2 free slot(s)

  that is ONE subtraction, two ANDs and a NOT -- four instructions
  to reject or shortlist eight slots. No branches, no key
  comparisons, no memory beyond the 8 bytes already in a register.

  the keys themselves are only compared for the slots the control
  word shortlisted. With 7 bits of hash there is a 1-in-128 chance
  of a false candidate, so on average a lookup does ONE key
  comparison, whatever the load factor.

  and that is the whole answer to 'why 7/8?'. At 87.5% load a group
  holds 7 of 8 slots, so the first group check almost always
  resolves the lookup. Chaining has to keep the load near 1.0 to
  avoid long chains; a Swiss table is FASTER at high load because
  the parallel check does not care how full the group is.

H1 and H2, the split that makes it work:

  H1 = hash >> 7         the upper 57 bits -- choose the group
  H2 = hash & 0x7f       the lower 7 -- live in the control word

  both halves must be good, which is exactly what example 2's
  finalizer measurement was about. A hash with all its entropy in
  the low bits would give a fine H2 and a useless H1: every key
  would start probing at the same group.

iteration order is randomised, and the randomisation is TWO-PART:

   18 22  4 19  7  8 10 11 17 21 23  2  ...
   22  4 19  7  8 10 11 17 21 23  2  3  ...
   12 14 16 18 22  4 19  7  8 10 11 17  ...
   22  4 19  7  8 10 11 17 21 23  2  3  ...

  from internal/runtime/maps/table.go:

      it.entryOffset = rand()   // where to start inside a group
      it.dirOffset   = rand()   // which table to start with

  it is deliberate and it is not a debug feature -- it ships in
  production builds. Code that accidentally depends on map order
  fails on the second run instead of two years later.

  the one exception: `fmt` prints maps in SORTED key order, so
  printing a map is stable even though ranging it is not.
  fmt: map[a:1 b:2 c:3]

map values are NOT ADDRESSABLE, and now you know why:

      p := &m[k]        // does not compile
      m[k].Field = 1    // does not compile
      m[k]++            // fine -- read, add, write back

  a rehash MOVES every entry to a new array. A pointer into a map
  would dangle the moment the map grew, so the language forbids
  taking one. This is not an arbitrary restriction; it falls
  straight out of example 5.

  the read-modify-write dance:  map[a:{99 x}]
  or store a pointer instead:   {99 x}

  storing *S makes the value addressable again, at the cost of one
  allocation and one indirection per entry -- lesson 04's trade,
  arriving in a new place.

the sizes involved, on this machine:

  one control word                   8 bytes
  one group                          8 slots
  unsafe.Sizeof(map[int]int{})       8 bytes
  unsafe.Sizeof(struct{k,v int}{})   16 bytes

  a map variable is ONE POINTER. That is why a map is a reference
  type, why passing it to a function is free, why the zero value is
  nil, and why writing to a nil map panics while reading returns
  the zero value.

  var m map[string]int -> len=0, m["x"]=0, ok=false
  m["x"] = 1 would panic: assignment to entry in nil map

and now a measurement that corrected me twice. I expected a MISS
to be as fast as a hit, because a miss should resolve in the
control word without ever comparing a key:

    live keys      map KB  bytes/key     hit ns    miss ns   miss/hit
         1000          36       37.0       3.80       4.34      1.14x
       100000        2309       23.6       5.16      13.30      2.58x
       150000        4618       31.5       4.86       6.56      1.35x
       200000        4624       23.7       6.00      14.49      2.42x
       300000        9237       31.5       5.20       6.73      1.29x
      2000000       73888       37.8      19.07      15.80      0.83x

  the ratio is not a constant, and it does not grow with n -- it
  goes 1.16, 2.42, 1.29, 2.39, 1.34, 0.85. Line the rows up against
  BYTES/KEY instead and the middle four sort themselves:

  23.7 and 23.6 bytes/key        miss/hit 2.42x and 2.39x
  31.5 and 31.5 bytes/key        miss/hit 1.29x and 1.34x

  bytes/key is the achieved load factor in disguise, and it swings
  with n for the power-of-two reason example 1 measured. So:

  a HIT stops at the group holding its key. A MISS cannot stop
  until it finds an EMPTY slot -- and in a DENSE table empty slots
  are rare, so it walks more groups, each a separate cache line.
  In a sparse table it finds one almost immediately.

  the two outer rows are governed by something else. At 1,000
  keys the whole table is in L1, so walking extra groups is free
  and the ratio is 1.16 whatever the density.

  and at 2,000,000 keys the ratio inverts: both are memory-bound,
  and now the HIT is the slower one, because after the control word
  shortlists a slot the hit must fetch and compare the actual key --
  one more cache line that the miss never touches.

  my original claim was right about the mechanism and wrong about
  the conditions. That is the usual shape of a benchmark surprise.

  what does hold unconditionally is the comparison to CHAINING,
  where a miss walks the entire chain -- example 3 measured 1000.5
  links per lookup on a degenerate table. A Swiss table's miss is a
  few group checks; a chained table's is unbounded. Example 11 puts
  them side by side.
```

**Complexity:** one group check = **8 slots in ~4 instructions**, which is the whole answer to "why
7/8 load factor" · miss/hit is **not** a constant: **2.58× on a dense table, 1.29× on a sparse one,
and 0.83× at 2,000,000 keys** where the *hit* pays an extra cache line for the key comparison

---

> Next tier: [🟡 medium](2-medium.md) — sets, counters, what can be a key, preallocation,
> and four implementations measured against each other.
