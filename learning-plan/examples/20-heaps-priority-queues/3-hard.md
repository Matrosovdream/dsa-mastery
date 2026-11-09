# Step 20 — Heaps & Priority Queues · 🔴 Hard

Examples **12–16**: Huffman coding checked against the entropy bound and an exhaustive search, the Dijkstra
frontier under lazy deletion and `Fix`, the allocation and call cost of `container/heap`, bottom-up and d-ary
pops, and the capstone.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match character for character; **Sample
output:** ones print timings.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Huffman coding, against the entropy bound

`🔴 hard` · *Greedy merge / optimal prefix code*

Huffman's algorithm is a heap loop: pop the two lightest trees, push their merge, and repeat until one tree is left, so `n-1` merges cost exactly `2(n-1)` pops and `n-1` pushes and the whole build is Θ(n log n). Nodes live in an arena (a slice indexed by id) and the heap orders `(weight, tie)` with a unique tie, a strict total order, so the tree does not depend on how the heap sifts. The resulting prefix code is optimal and its average length `L` obeys `H ≤ L < H + 1` for the entropy `H`; *why* the greedy choice is optimal is an exchange argument (lesson 27), so here it is checked instead by an exhaustive search over merging any two trees, with a deliberately wrong greedy as a control, and the code is checked by encoding and decoding a real text.

**Steps:**

1. Build the tree with `container/heap` over `(weight, tie, id)`, trace the merges of `a:5 b:2 c:1 d:1`, then code a 164-byte text and compare `L` with `H`, the Kraft sum, and each symbol's length with `⌈-lg p⌉`.
2. Encode the text to a `'0'/'1'` string and decode it back through the tree; then run 2,000 random alphabets against the exhaustive minimum (memoised on the sorted multiset of weights), under two tie-breaks and with "merge the two heaviest" as a control.
3. Read the extremes: 100 equal weights, Fibonacci weights (a chain of depth `n-1`), a single symbol, and the same weights `1 1 2 2` coded as two different trees of the same cost.

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
)

// A Huffman tree lives in an arena: nodes[id]. Leaves take ids 0..n-1 in symbol
// order; every merge appends ONE internal node, so internal ids are creation
// order. There are no pointers, only indices.
type node struct {
	weight      int
	left, right int // child ids; -1 for a leaf
	sym         byte
}

// entry is what the heap orders: (weight, tie), and tie is unique, so the order
// is strict and total. The pop sequence, and so the tree, does not depend on how
// the heap sifts. The standard tie-break is tie = id (older first); the other
// one, tie = -id, pops the newest node first.
type entry struct{ weight, tie, id int }

type pq struct {
	items        []entry
	pushes, pops int
}

func (q *pq) Len() int { return len(q.items) }
func (q *pq) Less(i, j int) bool {
	a, b := q.items[i], q.items[j]
	if a.weight != b.weight {
		return a.weight < b.weight
	}
	return a.tie < b.tie
}
func (q *pq) Swap(i, j int) { q.items[i], q.items[j] = q.items[j], q.items[i] }
func (q *pq) Push(x any)    { q.items = append(q.items, x.(entry)) }
func (q *pq) Pop() any {
	old := q.items
	x := old[len(old)-1]
	q.items = old[:len(old)-1]
	return x
}

func (q *pq) push(e entry) { q.pushes++; heap.Push(q, e) }
func (q *pq) pop() entry   { q.pops++; return heap.Pop(q).(entry) }

type tree struct {
	nodes        []node
	root         int
	pushes, pops int
}

// build is Huffman's loop: pop the two lightest trees, push their merge. The
// FIRST one popped becomes the left child (bit 0).
func build(weights []int, syms []byte, newestFirst bool) tree {
	n := len(weights)
	t := tree{nodes: make([]node, n, 2*n-1)}
	q := &pq{}
	tie := func(id int) int {
		if newestFirst {
			return -id
		}
		return id
	}
	for i, w := range weights {
		t.nodes[i] = node{weight: w, left: -1, right: -1, sym: syms[i]}
		q.items = append(q.items, entry{w, tie(i), i}) // initial fill: no sift yet
	}
	heap.Init(q)
	for q.Len() > 1 {
		a, b := q.pop(), q.pop()
		id := len(t.nodes)
		t.nodes = append(t.nodes, node{weight: a.weight + b.weight, left: a.id, right: b.id})
		q.push(entry{a.weight + b.weight, tie(id), id})
	}
	t.root = q.items[0].id // the one tree left: peek, not a pop
	t.pushes, t.pops = q.pushes, q.pops
	return t
}

// codes walks the tree: left is '0', right is '1'. A lone leaf gets "0".
func (t tree) codes(n int) []string {
	out := make([]string, n)
	var walk func(id int, prefix string)
	walk = func(id int, prefix string) {
		nd := t.nodes[id]
		if nd.left < 0 {
			if prefix == "" {
				prefix = "0"
			}
			out[id] = prefix
			return
		}
		walk(nd.left, prefix+"0")
		walk(nd.right, prefix+"1")
	}
	walk(t.root, "")
	return out
}

func lengths(codes []string) []int {
	out := make([]int, len(codes))
	for i, c := range codes {
		out[i] = len(c)
	}
	return out
}

func cost(w, lens []int) int {
	c := 0
	for i := range w {
		c += w[i] * lens[i]
	}
	return c
}

func entropy(w []int) float64 {
	total := 0
	for _, c := range w {
		total += c
	}
	h := 0.0
	for _, c := range w {
		p := float64(c) / float64(total)
		h -= p * math.Log2(p)
	}
	return h
}

// kraft returns sum 2^(maxLen-len) and 2^maxLen: the code is complete and
// prefix-free-capable exactly when they are equal. Integers only.
func kraft(lens []int) (sum, full int) {
	maxLen := slices.Max(lens)
	for _, l := range lens {
		sum += 1 << (maxLen - l)
	}
	return sum, 1 << maxLen
}

// ceilLg returns the smallest k with count*2^k >= total, i.e. ceil(-lg p), exactly.
func ceilLg(total, count int) int {
	k := 0
	for count<<k < total {
		k++
	}
	return k
}

// versus counts symbols whose code is shorter than, equal to, and longer than
// ceil(-lg p), where p = count/total.
func versus(w, lens []int) (below, equal, above int) {
	total := sum(w)
	for i := range w {
		switch cl := ceilLg(total, w[i]); {
		case lens[i] < cl:
			below++
		case lens[i] == cl:
			equal++
		default:
			above++
		}
	}
	return
}

func decode(t tree, bits string) []byte {
	var out []byte
	cur := t.root
	for _, b := range []byte(bits) {
		if nd := t.nodes[cur]; nd.left >= 0 {
			cur = nd.left
			if b == '1' {
				cur = nd.right
			}
		}
		if t.nodes[cur].left < 0 {
			out = append(out, t.nodes[cur].sym)
			cur = t.root
		}
	}
	return out
}

// The exhaustive oracle: the cost of a code tree is the sum of its internal
// nodes' weights, and every full binary tree is some sequence of merges of ANY
// two trees. So try every pair, recursively, and keep the cheapest. The future
// depends only on the multiset of weights left, so memoise on the sorted multiset.
type key [7]uint8

func minCost(ws []int, memo map[key]int, pairs *int) int {
	if len(ws) == 1 {
		return 0
	}
	var k key
	for i, w := range ws {
		k[i] = uint8(w)
	}
	if c, ok := memo[k]; ok {
		return c
	}
	best := math.MaxInt
	for i := range ws {
		for j := i + 1; j < len(ws); j++ {
			*pairs++
			rest := make([]int, 0, len(ws)-1)
			for t, w := range ws {
				if t != i && t != j {
					rest = append(rest, w)
				}
			}
			rest = append(rest, ws[i]+ws[j])
			slices.Sort(rest)
			best = min(best, ws[i]+ws[j]+minCost(rest, memo, pairs))
		}
	}
	memo[k] = best
	return best
}

// greedyLargest is a deliberately wrong greedy: merge the two HEAVIEST trees.
func greedyLargest(ws []int) int {
	a := slices.Clone(ws)
	total := 0
	for len(a) > 1 {
		slices.Sort(a)
		m := a[len(a)-1] + a[len(a)-2]
		total += m
		a = append(a[:len(a)-2], m)
	}
	return total
}

func symbolsFor(n int) []byte {
	s := make([]byte, n)
	for i := range s {
		s[i] = byte('a' + i)
	}
	return s
}

func lens2s(l []int) string {
	parts := make([]string, len(l))
	for i, x := range l {
		parts[i] = fmt.Sprint(x)
	}
	return strings.Join(parts, " ")
}

func main() {
	// (a) Worked example: weights a:5 b:2 c:1 d:1, every merge shown.
	fmt.Println("(a) weights a:5 b:2 c:1 d:1, heap key (weight, id), first popped = left = bit 0:")
	w4, s4 := []int{5, 2, 1, 1}, []byte("abcd")
	t4 := build(w4, s4, false)
	label := func(t tree, id int) string {
		if t.nodes[id].left < 0 {
			return fmt.Sprintf("(%d,#%d '%c')", t.nodes[id].weight, id, t.nodes[id].sym)
		}
		return fmt.Sprintf("(%d,#%d)", t.nodes[id].weight, id)
	}
	for id := len(w4); id < len(t4.nodes); id++ {
		nd := t4.nodes[id]
		fmt.Printf("  merge %d: pop %-10s + pop %-10s -> push (%d,#%d)\n",
			id-len(w4)+1, label(t4, nd.left), label(t4, nd.right), nd.weight, id)
	}
	c4 := t4.codes(4)
	for i := range c4 {
		fmt.Printf("  %c  weight %d  code %-4s length %d\n", s4[i], w4[i], c4[i], len(c4[i]))
	}
	fmt.Printf("  cost = sum of weight*length = %d bits (fixed 2-bit codes: %d)\n", cost(w4, lengths(c4)), 2*sum(w4))

	// (b) A real text.
	text := "it was the best of times it was the worst of times it was the age of wisdom " +
		"it was the age of foolishness it was the epoch of belief it was the epoch of incredulity"
	var freq [256]int
	for i := 0; i < len(text); i++ {
		freq[text[i]]++
	}
	var syms []byte
	var weights []int
	for c := 0; c < 256; c++ {
		if freq[c] > 0 {
			syms = append(syms, byte(c))
			weights = append(weights, freq[c])
		}
	}
	n := len(syms)
	t := build(weights, syms, false)
	codes := t.codes(n)
	lens := lengths(codes)
	total := len(text)
	fmt.Printf("\n(b) the text: %d bytes, %d distinct symbols (ids in byte order)\n", total, n)
	order := make([]int, n)
	for i := range order {
		order[i] = i
	}
	slices.SortFunc(order, func(a, b int) int {
		if weights[a] != weights[b] {
			return weights[b] - weights[a]
		}
		return a - b
	})
	fmt.Println("  sym  count  code      len  ceil(-lg p)")
	for _, i := range order {
		fmt.Printf("  %-4q %5d  %-8s %4d  %6d\n", syms[i], weights[i], codes[i], lens[i], ceilLg(total, weights[i]))
	}
	below, equal, above := versus(weights, lens)
	bits := cost(weights, lens)
	L := float64(bits) / float64(total)
	H := entropy(weights)
	ks, kf := kraft(lens)
	fmt.Printf("  total %d bits against %d bits at 8 bits per byte (%.1f%%)\n", bits, 8*total, 100*float64(bits)/float64(8*total))
	fmt.Printf("  L = %.4f bits/symbol, H = %.4f, H*len = %.1f bits;  H <= L < H+1: %v\n", L, H, H*float64(total), H <= L && L < H+1)
	fmt.Printf("  Kraft sum: %d == %d: %v\n", ks, kf, ks == kf)
	fmt.Printf("  heap operations for n = %d: pops %d (2(n-1) = %d), pushes %d (n-1 = %d)\n", n, t.pops, 2*(n-1), t.pushes, n-1)
	fmt.Printf("  code length against ceil(-lg p), per symbol: %d shorter, %d equal, %d longer\n", below, equal, above)

	// (c) Encode to a '0'/'1' string and decode it back through the tree.
	var sb strings.Builder
	for i := 0; i < len(text); i++ {
		sb.WriteString(codes[slices.Index(syms, text[i])])
	}
	enc := sb.String()
	fmt.Printf("\n(c) encoded: %d bits, starts %s...\n", len(enc), enc[:40])
	fmt.Printf("  decoded back through the tree equals the text: %v\n", string(decode(t, enc)) == text)

	// (d) Optimality by exhaustive search: 2,000 random small alphabets.
	r := rand.New(rand.NewPCG(1, 2))
	memo := map[key]int{}
	pairs := 0
	mismatches, wrongGreedy, boundFails, kraftFails, lenDiffer, costDiffer, maxLenOK := 0, 0, 0, 0, 0, 0, true
	fmt.Printf("\n(d) the oracle on the worked example 5 2 1 1: %d bits (the Huffman code above got %d)\n",
		minCost([]int{1, 1, 2, 5}, map[key]int{}, new(int)), cost(w4, lengths(c4)))
	var first string
	var symShorter, symLonger, alphaLonger int
	for range 2000 {
		m := 2 + r.IntN(6)
		ws := make([]int, m)
		for i := range ws {
			ws[i] = 1 + r.IntN(20)
		}
		sy := symbolsFor(m)
		ta, tb := build(ws, sy, false), build(ws, sy, true)
		la, lb := lengths(ta.codes(m)), lengths(tb.codes(m))
		ca, cb := cost(ws, la), cost(ws, lb)
		sorted := slices.Clone(ws)
		slices.Sort(sorted)
		best := minCost(sorted, memo, &pairs)
		if ca != best || cb != best {
			mismatches++
		}
		if greedyLargest(ws) != best {
			wrongGreedy++
		}
		Lc, Hc := float64(ca)/float64(sum(ws)), entropy(ws)
		if Lc < Hc-1e-9 || Lc >= Hc+1 {
			boundFails++
		}
		if s, f := kraft(la); s != f {
			kraftFails++
		}
		if !slices.Equal(la, lb) {
			lenDiffer++
			if first == "" {
				first = fmt.Sprintf("weights %s: lengths %s (oldest first) vs %s (newest first), cost %d and %d",
					lens2s(ws), lens2s(la), lens2s(lb), ca, cb)
			}
		}
		if ca != cb {
			costDiffer++
		}
		bl, _, ab := versus(ws, la)
		symShorter += bl
		symLonger += ab
		if ab > 0 {
			alphaLonger++
		}
		maxLenOK = maxLenOK && slices.Max(la) <= m-1
	}
	fmt.Println("  2,000 random alphabets (n in 2..7, weights 1..20) against the exhaustive minimum over merging ANY two trees:")
	fmt.Printf("  exhaustive search: %d distinct weight multisets solved, %d merge choices tried\n", len(memo), pairs)
	fmt.Printf("  Huffman cost != minimum: %d of 2000 (both tie-breaks)\n", mismatches)
	fmt.Printf("  control, merge the two heaviest instead: %d of 2000 are not optimal\n", wrongGreedy)
	fmt.Printf("  H <= L < H+1 violated: %d;  Kraft sum != 2^maxLen: %d;  max length <= n-1 always: %v\n", boundFails, kraftFails, maxLenOK)
	fmt.Printf("  the two tie-breaks gave different code lengths in %d cases and different costs in %d\n", lenDiffer, costDiffer)
	fmt.Printf("  first such case: %s\n", first)
	fmt.Printf("  per symbol against ceil(-lg p): %d symbols shorter; %d longer, in %d of the 2000 alphabets\n", symShorter, symLonger, alphaLonger)

	// (e) Extremes.
	fmt.Println("\n(e) extremes:")
	show := func(name string, ws []int) {
		tt := build(ws, symbolsFor(len(ws)), false)
		l := lengths(tt.codes(len(ws)))
		hist := map[int]int{}
		for _, x := range l {
			hist[x]++
		}
		var parts []string
		for ln := 1; ln <= slices.Max(l); ln++ {
			if hist[ln] > 0 {
				parts = append(parts, fmt.Sprintf("%d x%d", ln, hist[ln]))
			}
		}
		Lx, Hx := float64(cost(ws, l))/float64(sum(ws)), entropy(ws)
		bl, eq, ab := versus(ws, l)
		fmt.Printf("  %-18s n=%-3d cost %-4d L-H %.4f  H<=L<H+1: %-5v  vs ceil(-lg p): %d shorter %d equal %d longer\n",
			name, len(ws), cost(ws, l), Lx-Hx, Hx <= Lx+1e-9 && Lx < Hx+1, bl, eq, ab)
		fmt.Printf("  %-18s lengths (length x count): %s\n", "", strings.Join(parts, "  "))
	}
	equalW := slices.Repeat([]int{1}, 100)
	fib := []int{1, 1, 2, 3, 5, 8, 13, 21, 34, 55}
	show("100 equal weights", equalW)
	show("Fibonacci 1..55", fib)
	show("one symbol", []int{7})
	fmt.Printf("  Fibonacci lengths in symbol order: %s (depth n-1 = %d)\n", lens2s(lengths(build(fib, symbolsFor(10), false).codes(10))), len(fib)-1)
	t1 := build([]int{7}, []byte("x"), false)
	fmt.Printf("  one symbol: %d merges, %d pops, %d pushes; the code is %q\n", len(t1.nodes)-1, t1.pops, t1.pushes, t1.codes(1)[0])
	fmt.Println("  same weights, two tie-breaks:")
	for _, nf := range []bool{false, true} {
		ws := []int{1, 1, 2, 2}
		tt := build(ws, symbolsFor(4), nf)
		l := lengths(tt.codes(4))
		fmt.Printf("    weights 1 1 2 2, newest first = %-5v lengths %s, cost %d\n", nf, lens2s(l), cost(ws, l))
	}
}

func sum(xs []int) int {
	t := 0
	for _, x := range xs {
		t += x
	}
	return t
}
```

**Output:**

```
(a) weights a:5 b:2 c:1 d:1, heap key (weight, id), first popped = left = bit 0:
  merge 1: pop (1,#2 'c') + pop (1,#3 'd') -> push (2,#4)
  merge 2: pop (2,#1 'b') + pop (2,#4)     -> push (4,#5)
  merge 3: pop (4,#5)     + pop (5,#0 'a') -> push (9,#6)
  a  weight 5  code 1    length 1
  b  weight 2  code 00   length 2
  c  weight 1  code 010  length 3
  d  weight 1  code 011  length 3
  cost = sum of weight*length = 15 bits (fixed 2-bit codes: 18)

(b) the text: 164 bytes, 21 distinct symbols (ids in byte order)
  sym  count  code      len  ceil(-lg p)
  ' '     35  01          2       3
  'e'     17  000         3       4
  't'     17  001         3       4
  's'     14  1101        4       4
  'i'     13  1100        4       4
  'o'     12  1011        4       4
  'h'      9  1001        4       5
  'a'      8  11101       5       5
  'f'      8  11110       5       5
  'w'      8  11111       5       5
  'c'      3  101011      6       6
  'l'      3  111000      6       6
  'm'      3  111001      6       6
  'b'      2  100000      6       7
  'd'      2  100001      6       7
  'g'      2  100010      6       7
  'n'      2  100011      6       7
  'p'      2  101000      6       7
  'r'      2  101001      6       7
  'u'      1  1010100     7       8
  'y'      1  1010101     7       8
  total 624 bits against 1312 bits at 8 bits per byte (47.6%)
  L = 3.8049 bits/symbol, H = 3.7617, H*len = 616.9 bits;  H <= L < H+1: true
  Kraft sum: 128 == 128: true
  heap operations for n = 21: pops 40 (2(n-1) = 40), pushes 20 (n-1 = 20)
  code length against ceil(-lg p), per symbol: 12 shorter, 9 equal, 0 longer

(c) encoded: 624 bits, starts 1100001011111111101110101001100100001100...
  decoded back through the tree equals the text: true

(d) the oracle on the worked example 5 2 1 1: 15 bits (the Huffman code above got 15)
  2,000 random alphabets (n in 2..7, weights 1..20) against the exhaustive minimum over merging ANY two trees:
  exhaustive search: 119774 distinct weight multisets solved, 825931 merge choices tried
  Huffman cost != minimum: 0 of 2000 (both tie-breaks)
  control, merge the two heaviest instead: 1655 of 2000 are not optimal
  H <= L < H+1 violated: 0;  Kraft sum != 2^maxLen: 0;  max length <= n-1 always: true
  the two tie-breaks gave different code lengths in 298 cases and different costs in 0
  first such case: weights 3 8 19 17 8 14: lengths 4 4 2 2 3 2 (oldest first) vs 4 3 2 2 4 2 (newest first), cost 168 and 168
  per symbol against ceil(-lg p): 4315 symbols shorter; 68 longer, in 68 of the 2000 alphabets

(e) extremes:
  100 equal weights  n=100 cost 672  L-H 0.0761  H<=L<H+1: true   vs ceil(-lg p): 28 shorter 72 equal 0 longer
                     lengths (length x count): 6 x28  7 x72
  Fibonacci 1..55    n=10  cost 363  L-H 0.0895  H<=L<H+1: true   vs ceil(-lg p): 2 shorter 3 equal 5 longer
                     lengths (length x count): 1 x1  2 x1  3 x1  4 x1  5 x1  6 x1  7 x1  8 x1  9 x2
  one symbol         n=1   cost 7    L-H 1.0000  H<=L<H+1: false  vs ceil(-lg p): 0 shorter 0 equal 1 longer
                     lengths (length x count): 1 x1
  Fibonacci lengths in symbol order: 9 9 8 7 6 5 4 3 2 1 (depth n-1 = 9)
  one symbol: 0 merges, 0 pops, 0 pushes; the code is "0"
  same weights, two tie-breaks:
    weights 1 1 2 2, newest first = false lengths 2 2 2 2, cost 12
    weights 1 1 2 2, newest first = true  lengths 3 3 1 2, cost 12
```

**Rust:**

```rust
use std::cmp::Reverse;
use std::collections::{BinaryHeap, HashMap};

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the
// 2,000 random alphabets in (d) are the ones the Go program sees.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

// A Huffman tree lives in an arena: nodes[id]. Leaves take ids 0..n-1 in symbol
// order; every merge appends ONE internal node, so internal ids are creation
// order. Go's version already used indices; in Rust this is also how you avoid
// a tree of `Box<Node>` (or `Rc<RefCell<Node>>`) when nothing needs to own a
// child on its own. Children are `Option<(usize, usize)>`, not a pair of -1s.
struct Node {
    weight: usize,
    children: Option<(usize, usize)>, // (left, right); None for a leaf
    sym: u8,
}

struct Tree {
    nodes: Vec<Node>,
    root: usize,
    pushes: usize,
    pops: usize,
}

/// build is Huffman's loop: pop the two lightest trees, push their merge. The
/// FIRST one popped becomes the left child (bit 0).
///
/// The heap holds `Reverse((weight, tie, id))`: std's BinaryHeap is a max-heap,
/// so `Reverse` makes it a min-heap. `tie` is unique (the id, or its negation
/// for newest-first), so the key is a strict total order and the pop sequence
/// is the same as Go's container/heap, whatever the two sifts do.
fn build(weights: &[usize], syms: &[u8], newest_first: bool) -> Tree {
    let n = weights.len();
    let tie = |id: usize| -> i64 {
        if newest_first {
            -(id as i64)
        } else {
            id as i64
        }
    };
    let mut nodes: Vec<Node> = Vec::with_capacity(2 * n - 1);
    let mut items = Vec::with_capacity(n);
    for (i, &w) in weights.iter().enumerate() {
        nodes.push(Node {
            weight: w,
            children: None,
            sym: syms[i],
        });
        items.push(Reverse((w, tie(i), i)));
    }
    // heapify: the n leaves enter in place, Θ(n), and are not counted as pushes
    let mut heap = BinaryHeap::from(items);
    let (mut pushes, mut pops) = (0, 0);
    while heap.len() > 1 {
        let Reverse((wa, _, a)) = heap.pop().expect("len > 1");
        let Reverse((wb, _, b)) = heap.pop().expect("len > 1");
        pops += 2;
        let id = nodes.len();
        nodes.push(Node {
            weight: wa + wb,
            children: Some((a, b)),
            sym: 0,
        });
        heap.push(Reverse((wa + wb, tie(id), id)));
        pushes += 1;
    }
    let root = heap.peek().expect("one tree left").0.2; // peek, not a pop
    Tree {
        nodes,
        root,
        pushes,
        pops,
    }
}

impl Tree {
    /// codes walks the tree: left is '0', right is '1'. A lone leaf gets "0".
    fn codes(&self, n: usize) -> Vec<String> {
        let mut out = vec![String::new(); n];
        self.walk(self.root, String::new(), &mut out);
        out
    }

    fn walk(&self, id: usize, prefix: String, out: &mut [String]) {
        match self.nodes[id].children {
            None => {
                out[id] = if prefix.is_empty() {
                    "0".into()
                } else {
                    prefix
                }
            }
            Some((l, r)) => {
                self.walk(l, format!("{prefix}0"), out);
                self.walk(r, format!("{prefix}1"), out);
            }
        }
    }

    fn decode(&self, bits: &str) -> Vec<u8> {
        let mut out = Vec::new();
        let mut cur = self.root;
        for b in bits.bytes() {
            if let Some((l, r)) = self.nodes[cur].children {
                cur = if b == b'1' { r } else { l };
            }
            if self.nodes[cur].children.is_none() {
                out.push(self.nodes[cur].sym);
                cur = self.root;
            }
        }
        out
    }
}

fn lengths(codes: &[String]) -> Vec<usize> {
    codes.iter().map(String::len).collect()
}

fn cost(w: &[usize], lens: &[usize]) -> usize {
    w.iter().zip(lens).map(|(w, l)| w * l).sum()
}

fn entropy(w: &[usize]) -> f64 {
    let total: usize = w.iter().sum();
    let mut h = 0.0;
    for &c in w {
        let p = c as f64 / total as f64;
        h -= p * p.log2();
    }
    h
}

/// kraft returns sum 2^(maxLen-len) and 2^maxLen: the code is complete exactly
/// when they are equal. Integers only.
fn kraft(lens: &[usize]) -> (usize, usize) {
    let max_len = *lens.iter().max().expect("non-empty");
    (lens.iter().map(|&l| 1 << (max_len - l)).sum(), 1 << max_len)
}

/// ceil_lg returns the smallest k with count*2^k >= total, i.e. ceil(-lg p), exactly.
fn ceil_lg(total: usize, count: usize) -> usize {
    let mut k = 0;
    while count << k < total {
        k += 1;
    }
    k
}

/// versus counts symbols whose code is shorter than, equal to, and longer than
/// ceil(-lg p), where p = count/total.
fn versus(w: &[usize], lens: &[usize]) -> (usize, usize, usize) {
    let total: usize = w.iter().sum();
    let (mut below, mut equal, mut above) = (0, 0, 0);
    for (&c, &l) in w.iter().zip(lens) {
        let cl = ceil_lg(total, c);
        if l < cl {
            below += 1;
        } else if l == cl {
            equal += 1;
        } else {
            above += 1;
        }
    }
    (below, equal, above)
}

// The exhaustive oracle: the cost of a code tree is the sum of its internal
// nodes' weights, and every full binary tree is some sequence of merges of ANY
// two trees. So try every pair, recursively, and keep the cheapest. The future
// depends only on the multiset of weights left, so memoise on the sorted multiset.
type Key = [u8; 7];

fn min_cost(ws: &[usize], memo: &mut HashMap<Key, usize>, pairs: &mut usize) -> usize {
    if ws.len() == 1 {
        return 0;
    }
    let mut k: Key = [0; 7];
    for (slot, &w) in k.iter_mut().zip(ws) {
        *slot = w as u8;
    }
    if let Some(&c) = memo.get(&k) {
        return c;
    }
    let mut best = usize::MAX;
    for i in 0..ws.len() {
        for j in i + 1..ws.len() {
            *pairs += 1;
            let mut rest: Vec<usize> = ws
                .iter()
                .enumerate()
                .filter(|&(t, _)| t != i && t != j)
                .map(|(_, &w)| w)
                .collect();
            rest.push(ws[i] + ws[j]);
            rest.sort_unstable();
            best = best.min(ws[i] + ws[j] + min_cost(&rest, memo, pairs));
        }
    }
    memo.insert(k, best);
    best
}

/// greedy_largest is a deliberately wrong greedy: merge the two HEAVIEST trees.
fn greedy_largest(ws: &[usize]) -> usize {
    let mut a = ws.to_vec();
    let mut total = 0;
    while a.len() > 1 {
        a.sort_unstable();
        let m = a[a.len() - 1] + a[a.len() - 2];
        total += m;
        a.truncate(a.len() - 2);
        a.push(m);
    }
    total
}

fn symbols_for(n: usize) -> Vec<u8> {
    (0..n).map(|i| b'a' + i as u8).collect()
}

/// Go prints a slice as [1 2 3]; here only the space-joined numbers are needed.
fn join(l: &[usize]) -> String {
    l.iter().map(usize::to_string).collect::<Vec<_>>().join(" ")
}

fn main() {
    // (a) Worked example: weights a:5 b:2 c:1 d:1, every merge shown.
    println!("(a) weights a:5 b:2 c:1 d:1, heap key (weight, id), first popped = left = bit 0:");
    let (w4, s4) = ([5usize, 2, 1, 1], *b"abcd");
    let t4 = build(&w4, &s4, false);
    let label = |t: &Tree, id: usize| -> String {
        let nd = &t.nodes[id];
        if nd.children.is_none() {
            format!("({},#{id} '{}')", nd.weight, nd.sym as char)
        } else {
            format!("({},#{id})", nd.weight)
        }
    };
    for id in w4.len()..t4.nodes.len() {
        let nd = &t4.nodes[id];
        let (l, r) = nd.children.expect("internal node");
        println!(
            "  merge {}: pop {:<10} + pop {:<10} -> push ({},#{id})",
            id - w4.len() + 1,
            label(&t4, l),
            label(&t4, r),
            nd.weight
        );
    }
    let c4 = t4.codes(4);
    for i in 0..4 {
        println!(
            "  {}  weight {}  code {:<4} length {}",
            s4[i] as char,
            w4[i],
            c4[i],
            c4[i].len()
        );
    }
    println!(
        "  cost = sum of weight*length = {} bits (fixed 2-bit codes: {})",
        cost(&w4, &lengths(&c4)),
        2 * w4.iter().sum::<usize>()
    );

    // (b) A real text.
    let text = "it was the best of times it was the worst of times it was the age of wisdom \
                it was the age of foolishness it was the epoch of belief it was the epoch of incredulity";
    let mut freq = [0usize; 256];
    for b in text.bytes() {
        freq[b as usize] += 1;
    }
    let (mut syms, mut weights) = (Vec::new(), Vec::new());
    for (c, &f) in freq.iter().enumerate() {
        if f > 0 {
            syms.push(c as u8);
            weights.push(f);
        }
    }
    let n = syms.len();
    let t = build(&weights, &syms, false);
    let codes = t.codes(n);
    let lens = lengths(&codes);
    let total = text.len();
    println!("\n(b) the text: {total} bytes, {n} distinct symbols (ids in byte order)");
    let mut order: Vec<usize> = (0..n).collect();
    order.sort_by(|&a, &b| weights[b].cmp(&weights[a]).then(a.cmp(&b)));
    println!("  sym  count  code      len  ceil(-lg p)");
    for &i in &order {
        println!(
            "  {:<4} {:>5}  {:<8} {:>4}  {:>6}",
            format!("{:?}", syms[i] as char),
            weights[i],
            codes[i],
            lens[i],
            ceil_lg(total, weights[i])
        );
    }
    let (below, equal, above) = versus(&weights, &lens);
    let bits = cost(&weights, &lens);
    let big_l = bits as f64 / total as f64;
    let big_h = entropy(&weights);
    let (ks, kf) = kraft(&lens);
    println!(
        "  total {bits} bits against {} bits at 8 bits per byte ({:.1}%)",
        8 * total,
        100.0 * bits as f64 / (8 * total) as f64
    );
    println!(
        "  L = {big_l:.4} bits/symbol, H = {big_h:.4}, H*len = {:.1} bits;  H <= L < H+1: {}",
        big_h * total as f64,
        big_h <= big_l && big_l < big_h + 1.0
    );
    println!("  Kraft sum: {ks} == {kf}: {}", ks == kf);
    println!(
        "  heap operations for n = {n}: pops {} (2(n-1) = {}), pushes {} (n-1 = {})",
        t.pops,
        2 * (n - 1),
        t.pushes,
        n - 1
    );
    println!(
        "  code length against ceil(-lg p), per symbol: {below} shorter, {equal} equal, {above} longer"
    );

    // (c) Encode to a '0'/'1' string and decode it back through the tree.
    let mut enc = String::new();
    for b in text.bytes() {
        let i = syms.iter().position(|&s| s == b).expect("symbol in table");
        enc.push_str(&codes[i]);
    }
    println!(
        "\n(c) encoded: {} bits, starts {}...",
        enc.len(),
        &enc[..40]
    );
    println!(
        "  decoded back through the tree equals the text: {}",
        t.decode(&enc) == text.as_bytes()
    );

    // (d) Optimality by exhaustive search: 2,000 random small alphabets.
    let mut r = Pcg::new(1, 2);
    let mut memo: HashMap<Key, usize> = HashMap::new();
    let mut pairs = 0;
    let (mut mismatches, mut wrong_greedy, mut bound_fails, mut kraft_fails) = (0, 0, 0, 0);
    let (mut len_differ, mut cost_differ, mut max_len_ok) = (0, 0, true);
    println!(
        "\n(d) the oracle on the worked example 5 2 1 1: {} bits (the Huffman code above got {})",
        min_cost(&[1, 1, 2, 5], &mut HashMap::new(), &mut 0),
        cost(&w4, &lengths(&c4))
    );
    let mut first = String::new();
    let (mut sym_shorter, mut sym_longer, mut alpha_longer) = (0, 0, 0);
    for _ in 0..2000 {
        let m = 2 + r.int_n(6) as usize;
        let ws: Vec<usize> = (0..m).map(|_| 1 + r.int_n(20) as usize).collect();
        let sy = symbols_for(m);
        let (ta, tb) = (build(&ws, &sy, false), build(&ws, &sy, true));
        let (la, lb) = (lengths(&ta.codes(m)), lengths(&tb.codes(m)));
        let (ca, cb) = (cost(&ws, &la), cost(&ws, &lb));
        let mut sorted = ws.clone();
        sorted.sort_unstable();
        let best = min_cost(&sorted, &mut memo, &mut pairs);
        if ca != best || cb != best {
            mismatches += 1;
        }
        if greedy_largest(&ws) != best {
            wrong_greedy += 1;
        }
        let sum: usize = ws.iter().sum();
        let (lc, hc) = (ca as f64 / sum as f64, entropy(&ws));
        if lc < hc - 1e-9 || lc >= hc + 1.0 {
            bound_fails += 1;
        }
        let (s, f) = kraft(&la);
        if s != f {
            kraft_fails += 1;
        }
        if la != lb {
            len_differ += 1;
            if first.is_empty() {
                first = format!(
                    "weights {}: lengths {} (oldest first) vs {} (newest first), cost {ca} and {cb}",
                    join(&ws),
                    join(&la),
                    join(&lb)
                );
            }
        }
        if ca != cb {
            cost_differ += 1;
        }
        let (bl, _, ab) = versus(&ws, &la);
        sym_shorter += bl;
        sym_longer += ab;
        if ab > 0 {
            alpha_longer += 1;
        }
        max_len_ok = max_len_ok && *la.iter().max().expect("non-empty") < m;
    }
    println!(
        "  2,000 random alphabets (n in 2..7, weights 1..20) against the exhaustive minimum over merging ANY two trees:"
    );
    println!(
        "  exhaustive search: {} distinct weight multisets solved, {pairs} merge choices tried",
        memo.len()
    );
    println!("  Huffman cost != minimum: {mismatches} of 2000 (both tie-breaks)");
    println!("  control, merge the two heaviest instead: {wrong_greedy} of 2000 are not optimal");
    println!(
        "  H <= L < H+1 violated: {bound_fails};  Kraft sum != 2^maxLen: {kraft_fails};  max length <= n-1 always: {max_len_ok}"
    );
    println!(
        "  the two tie-breaks gave different code lengths in {len_differ} cases and different costs in {cost_differ}"
    );
    println!("  first such case: {first}");
    println!(
        "  per symbol against ceil(-lg p): {sym_shorter} symbols shorter; {sym_longer} longer, in {alpha_longer} of the 2000 alphabets"
    );

    // (e) Extremes.
    println!("\n(e) extremes:");
    let show = |name: &str, ws: &[usize]| {
        let tt = build(ws, &symbols_for(ws.len()), false);
        let l = lengths(&tt.codes(ws.len()));
        let max = *l.iter().max().expect("non-empty");
        let parts: Vec<String> = (1..=max)
            .filter_map(|ln| {
                let c = l.iter().filter(|&&x| x == ln).count();
                (c > 0).then(|| format!("{ln} x{c}"))
            })
            .collect();
        let sum: usize = ws.iter().sum();
        let (lx, hx) = (cost(ws, &l) as f64 / sum as f64, entropy(ws));
        let (bl, eq, ab) = versus(ws, &l);
        println!(
            "  {name:<18} n={:<3} cost {:<4} L-H {:.4}  H<=L<H+1: {:<5}  vs ceil(-lg p): {bl} shorter {eq} equal {ab} longer",
            ws.len(),
            cost(ws, &l),
            lx - hx,
            hx <= lx + 1e-9 && lx < hx + 1.0
        );
        println!(
            "  {:<18} lengths (length x count): {}",
            "",
            parts.join("  ")
        );
    };
    let fib = [1usize, 1, 2, 3, 5, 8, 13, 21, 34, 55];
    show("100 equal weights", &[1; 100]);
    show("Fibonacci 1..55", &fib);
    show("one symbol", &[7]);
    println!(
        "  Fibonacci lengths in symbol order: {} (depth n-1 = {})",
        join(&lengths(&build(&fib, &symbols_for(10), false).codes(10))),
        fib.len() - 1
    );
    let t1 = build(&[7], b"x", false);
    println!(
        "  one symbol: {} merges, {} pops, {} pushes; the code is {:?}",
        t1.nodes.len() - 1,
        t1.pops,
        t1.pushes,
        t1.codes(1)[0]
    );
    println!("  same weights, two tie-breaks:");
    for nf in [false, true] {
        let ws = [1usize, 1, 2, 2];
        let tt = build(&ws, &symbols_for(4), nf);
        let l = lengths(&tt.codes(4));
        println!(
            "    weights 1 1 2 2, newest first = {nf:<5} lengths {}, cost {}",
            join(&l),
            cost(&ws, &l)
        );
    }
}
```

**Rust output:**

```
(a) weights a:5 b:2 c:1 d:1, heap key (weight, id), first popped = left = bit 0:
  merge 1: pop (1,#2 'c') + pop (1,#3 'd') -> push (2,#4)
  merge 2: pop (2,#1 'b') + pop (2,#4)     -> push (4,#5)
  merge 3: pop (4,#5)     + pop (5,#0 'a') -> push (9,#6)
  a  weight 5  code 1    length 1
  b  weight 2  code 00   length 2
  c  weight 1  code 010  length 3
  d  weight 1  code 011  length 3
  cost = sum of weight*length = 15 bits (fixed 2-bit codes: 18)

(b) the text: 164 bytes, 21 distinct symbols (ids in byte order)
  sym  count  code      len  ceil(-lg p)
  ' '     35  01          2       3
  'e'     17  000         3       4
  't'     17  001         3       4
  's'     14  1101        4       4
  'i'     13  1100        4       4
  'o'     12  1011        4       4
  'h'      9  1001        4       5
  'a'      8  11101       5       5
  'f'      8  11110       5       5
  'w'      8  11111       5       5
  'c'      3  101011      6       6
  'l'      3  111000      6       6
  'm'      3  111001      6       6
  'b'      2  100000      6       7
  'd'      2  100001      6       7
  'g'      2  100010      6       7
  'n'      2  100011      6       7
  'p'      2  101000      6       7
  'r'      2  101001      6       7
  'u'      1  1010100     7       8
  'y'      1  1010101     7       8
  total 624 bits against 1312 bits at 8 bits per byte (47.6%)
  L = 3.8049 bits/symbol, H = 3.7617, H*len = 616.9 bits;  H <= L < H+1: true
  Kraft sum: 128 == 128: true
  heap operations for n = 21: pops 40 (2(n-1) = 40), pushes 20 (n-1 = 20)
  code length against ceil(-lg p), per symbol: 12 shorter, 9 equal, 0 longer

(c) encoded: 624 bits, starts 1100001011111111101110101001100100001100...
  decoded back through the tree equals the text: true

(d) the oracle on the worked example 5 2 1 1: 15 bits (the Huffman code above got 15)
  2,000 random alphabets (n in 2..7, weights 1..20) against the exhaustive minimum over merging ANY two trees:
  exhaustive search: 119774 distinct weight multisets solved, 825931 merge choices tried
  Huffman cost != minimum: 0 of 2000 (both tie-breaks)
  control, merge the two heaviest instead: 1655 of 2000 are not optimal
  H <= L < H+1 violated: 0;  Kraft sum != 2^maxLen: 0;  max length <= n-1 always: true
  the two tie-breaks gave different code lengths in 298 cases and different costs in 0
  first such case: weights 3 8 19 17 8 14: lengths 4 4 2 2 3 2 (oldest first) vs 4 3 2 2 4 2 (newest first), cost 168 and 168
  per symbol against ceil(-lg p): 4315 symbols shorter; 68 longer, in 68 of the 2000 alphabets

(e) extremes:
  100 equal weights  n=100 cost 672  L-H 0.0761  H<=L<H+1: true   vs ceil(-lg p): 28 shorter 72 equal 0 longer
                     lengths (length x count): 6 x28  7 x72
  Fibonacci 1..55    n=10  cost 363  L-H 0.0895  H<=L<H+1: true   vs ceil(-lg p): 2 shorter 3 equal 5 longer
                     lengths (length x count): 1 x1  2 x1  3 x1  4 x1  5 x1  6 x1  7 x1  8 x1  9 x2
  one symbol         n=1   cost 7    L-H 1.0000  H<=L<H+1: false  vs ceil(-lg p): 0 shorter 0 equal 1 longer
                     lengths (length x count): 1 x1
  Fibonacci lengths in symbol order: 9 9 8 7 6 5 4 3 2 1 (depth n-1 = 9)
  one symbol: 0 merges, 0 pops, 0 pushes; the code is "0"
  same weights, two tie-breaks:
    weights 1 1 2 2, newest first = false lengths 2 2 2 2, cost 12
    weights 1 1 2 2, newest first = true  lengths 3 3 1 2, cost 12
```

> **Go vs Rust:** Identical output, character for character, including the 4-decimal `L` and `H` (`f64::log2` may differ from `math.Log2` in the last ulp; it does not show at 4 places). Both use the same tie-break, **older node first** (`tie = id`; the demo flips it to `-id`), and because `(weight, tie)` is a strict total order the pop sequence and the tree cannot depend on the heap: Go's `container/heap` swaps down with two comparisons per level, Rust's `BinaryHeap<Reverse<(usize, i64, usize)>>` moves a hole to the bottom, and they still agree. The arena's `children: Option<(usize, usize)>` replaces Go's pair of `-1`s, and `BinaryHeap::from(items)` is `heap.Init`.

**Complexity:** Θ(n log n) time, Θ(n) space · `n-1` merges, `2(n-1)` pops and `n-1` pushes (40 and 20 for the 21 symbols of the text) · 624 bits for the 164-byte text against 1,312 at 8 bits per byte, `L = 3.8049` against `H = 3.7617` · cost equal to the exhaustive minimum in all 2,000 random alphabets, while "merge the two heaviest" was wrong in 1,655 · tie-breaks change code lengths (298 of 2,000) but never the cost · a symbol's length is not tied to `⌈-lg p⌉`: shorter for 12 of the 21 symbols of the text (never longer there), but longer in 68 of the 2,000 random alphabets and for 5 of the 10 Fibonacci symbols · one symbol alone gets 1 bit, so `L - H = 1` and the strict bound `L < H + 1` needs n of at least 2

---

## 13. Dijkstra's frontier, and the order it settles

`🔴 hard` · *Frontier / lazy deletion vs Fix*

Dijkstra's heap *is* the frontier: the vertices discovered but not yet settled, keyed by tentative distance. With non-negative weights the minimum is always final, so vertices **settle in non-decreasing distance order**. There is no decrease-key in `container/heap` without an index, so there are two ways to improve an entry: **lazy deletion** pushes a second `(dist, v)` entry and skips the stale one when it surfaces (the heap can hold up to `E+1` entries), and an **indexed** heap keeps one entry per vertex with its slot in a `pos` array and calls `heap.Fix` (at most `V` entries). Both are Θ((V+E) log V). Keying on `(dist, v)` makes the pop order strict, so both variants settle the same vertices in the same order and every count except the heap's own is a property of the graph. (Negative edges, Bellman–Ford proper, path reconstruction and A* are lesson 24; lesson 09 example 13 showed a deque replacing the heap for 0/1 weights.)

**Steps:**

1. Generate random directed graphs (`V*deg` edges, weights 1..100, source 0) and run both variants on three sizes, printing pushes, pops, stale pops, `Fix` calls, relaxations, the peak heap size and the `Less` calls, and check the identities that connect them.
2. Check every distance against a relaxation-to-a-fixed-point oracle (and on 2,000 tiny graphs with unreachable vertices), then check that the settle sequence is the vertices sorted by `(true distance, id)` and that stopping at a target settles exactly its rank.
3. Take snapshots of the frontier while the run is in progress: the indexed heap is exactly the frontier, and the lazy heap is the frontier plus the stale entries.

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

const inf = math.MaxInt

type edge struct{ to, w int }
type graph struct {
	adj   [][]edge
	edges int
}

// randomGraph: V*deg directed edges (u, v, w) with w in 1..100. Self-loops and
// parallel edges are allowed. Draw order: u, v, w.
func randomGraph(r *rand.Rand, V, deg int) graph {
	g := graph{adj: make([][]edge, V), edges: V * deg}
	for range V * deg {
		u := r.IntN(V)
		v := r.IntN(V)
		w := 1 + r.IntN(100)
		g.adj[u] = append(g.adj[u], edge{v, w})
	}
	return g
}

// result carries the dist array and every count the table prints.
type result struct {
	dist                              []int
	order                             []int // vertices in the order they were settled
	pushes, pops, stale, fixes, relax int
	scanned, maxHeap, less            int
	snaps                             map[int]snap // heap state after the k-th settle, for k in marks
}

// snap is the frontier at one moment: the heap's length and how many of its
// entries are live (d == dist[v]); only the lazy heap can hold stale ones.
type snap struct{ heapLen, live int }

func newResult(V int) result {
	d := make([]int, V)
	for i := range d {
		d[i] = inf
	}
	return result{dist: d, snaps: map[int]snap{}}
}

// ---- Variant 1: lazy deletion. Every improvement PUSHES a new (dist, v) entry;
// the old one stays inside. A pop whose d is worse than dist[v] is stale: skip it.
// Key (d, v) is a strict total order: two entries for one vertex never share a d.
type item struct{ d, v int }
type lazyPQ struct {
	a    []item
	less int
}

func (h *lazyPQ) Len() int { return len(h.a) }
func (h *lazyPQ) Less(i, j int) bool {
	h.less++
	if h.a[i].d != h.a[j].d {
		return h.a[i].d < h.a[j].d
	}
	return h.a[i].v < h.a[j].v
}
func (h *lazyPQ) Swap(i, j int) { h.a[i], h.a[j] = h.a[j], h.a[i] }
func (h *lazyPQ) Push(x any)    { h.a = append(h.a, x.(item)) }
func (h *lazyPQ) Pop() any {
	x := h.a[len(h.a)-1]
	h.a = h.a[:len(h.a)-1]
	return x
}

// lazy runs Dijkstra from src; target >= 0 stops the moment target is settled.
func lazy(g graph, src, target int, marks map[int]bool) result {
	res := newResult(len(g.adj))
	h := &lazyPQ{}
	res.dist[src] = 0
	heap.Push(h, item{0, src})
	res.pushes, res.maxHeap = 1, 1
	for h.Len() > 0 {
		it := heap.Pop(h).(item)
		res.pops++
		if it.d > res.dist[it.v] {
			res.stale++
			continue
		}
		res.order = append(res.order, it.v) // settled: dist[it.v] is final
		if it.v == target {
			break
		}
		for _, e := range g.adj[it.v] {
			res.scanned++
			if nd := it.d + e.w; nd < res.dist[e.to] {
				res.dist[e.to] = nd
				res.relax++
				heap.Push(h, item{nd, e.to})
				res.pushes++
				res.maxHeap = max(res.maxHeap, h.Len())
			}
		}
		if k := len(res.order); marks[k] {
			live := 0
			for _, x := range h.a {
				if x.d == res.dist[x.v] {
					live++
				}
			}
			res.snaps[k] = snap{h.Len(), live}
		}
	}
	res.less = h.less
	return res
}

// ---- Variant 2: indexed. One entry per vertex; pos[v] is its slot, kept up to
// date by Swap/Push/Pop, so an improvement is heap.Fix(h, pos[v]).
type idxPQ struct {
	vs        []int
	pos, dist []int
	less      int
}

func (h *idxPQ) Len() int { return len(h.vs) }
func (h *idxPQ) Less(i, j int) bool {
	h.less++
	a, b := h.vs[i], h.vs[j]
	if h.dist[a] != h.dist[b] {
		return h.dist[a] < h.dist[b]
	}
	return a < b
}
func (h *idxPQ) Swap(i, j int) {
	h.vs[i], h.vs[j] = h.vs[j], h.vs[i]
	h.pos[h.vs[i]], h.pos[h.vs[j]] = i, j
}
func (h *idxPQ) Push(x any) {
	v := x.(int)
	h.pos[v] = len(h.vs)
	h.vs = append(h.vs, v)
}
func (h *idxPQ) Pop() any {
	v := h.vs[len(h.vs)-1]
	h.vs = h.vs[:len(h.vs)-1]
	h.pos[v] = -1
	return v
}

func indexed(g graph, src int, marks map[int]bool) result {
	res := newResult(len(g.adj))
	h := &idxPQ{pos: make([]int, len(g.adj)), dist: res.dist}
	res.dist[src] = 0
	heap.Push(h, src)
	res.pushes, res.maxHeap = 1, 1
	for h.Len() > 0 {
		u := heap.Pop(h).(int)
		res.pops++
		res.order = append(res.order, u)
		for _, e := range g.adj[u] {
			res.scanned++
			nd := res.dist[u] + e.w
			if nd >= res.dist[e.to] {
				continue
			}
			res.relax++
			first := res.dist[e.to] == inf // never discovered: not in the heap yet
			res.dist[e.to] = nd
			if first {
				heap.Push(h, e.to)
				res.pushes++
				res.maxHeap = max(res.maxHeap, h.Len())
			} else {
				heap.Fix(h, h.pos[e.to]) // key decreased: sift up
				res.fixes++
			}
		}
		if k := len(res.order); marks[k] {
			res.snaps[k] = snap{h.Len(), h.Len()}
		}
	}
	res.less = h.less
	return res
}

// fixedPoint is the oracle: relax every edge, again and again, until a whole
// sweep changes nothing (lesson 24 explains why this converges: Bellman-Ford).
func fixedPoint(g graph, src int) (dist []int, sweeps int) {
	dist = make([]int, len(g.adj))
	for i := range dist {
		dist[i] = inf
	}
	dist[src] = 0
	for changed := true; changed; {
		changed = false
		sweeps++
		for u, es := range g.adj {
			if dist[u] == inf {
				continue
			}
			for _, e := range es {
				if dist[u]+e.w < dist[e.to] {
					dist[e.to] = dist[u] + e.w
					changed = true
				}
			}
		}
	}
	return dist, sweeps
}

func reached(dist []int) int {
	n := 0
	for _, d := range dist {
		if d != inf {
			n++
		}
	}
	return n
}

func nonDecreasing(dist, order []int) bool {
	for i := 1; i < len(order); i++ {
		if dist[order[i]] < dist[order[i-1]] {
			return false
		}
	}
	return true
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	// (a) The frontier, two ways, on three random graphs.
	type config struct{ V, deg int }
	configs := []config{{2000, 8}, {2000, 40}, {20000, 8}}
	var graphs []graph
	var lazies, indexeds []result
	for _, c := range configs {
		marks := map[int]bool{}
		for _, p := range []int{1, 10, 25, 50, 75, 90, 99} {
			marks[c.V*p/100] = true
		}
		g := randomGraph(r, c.V, c.deg)
		lz, ix := lazy(g, 0, -1, marks), indexed(g, 0, marks)
		graphs, lazies, indexeds = append(graphs, g), append(lazies, lz), append(indexeds, ix)
		reach := len(lz.order)
		fmt.Printf("(a) V=%d, %d edges (degree %d): %d vertices reachable from 0, %d edge scans\n", c.V, g.edges, c.deg, reach, lz.scanned)
		fmt.Printf("  %-8s %7s %7s %7s %6s %12s %9s %10s\n", "variant", "pushes", "pops", "stale", "fixes", "relaxations", "max heap", "Less calls")
		for _, row := range []struct {
			name string
			x    result
		}{{"lazy", lz}, {"indexed", ix}} {
			fmt.Printf("  %-8s %7d %7d %7d %6d %12d %9d %10d\n", row.name, row.x.pushes, row.x.pops, row.x.stale, row.x.fixes, row.x.relax, row.x.maxHeap, row.x.less)
		}
		fmt.Printf("  same settle sequence: %v;  lazy pushes == 1 + relaxations: %v;  lazy stale == pops - reachable: %v\n",
			slices.Equal(lz.order, ix.order), lz.pushes == 1+lz.relax, lz.stale == lz.pops-reach)
		fmt.Printf("  lazy max heap %d <= E+1 = %d: %v;  indexed max heap %d <= V = %d: %v;  indexed fixes == relaxations - (reachable-1): %v\n",
			lz.maxHeap, g.edges+1, lz.maxHeap <= g.edges+1, ix.maxHeap, c.V, ix.maxHeap <= c.V, ix.fixes == ix.relax-(reach-1))
	}

	// (b) Oracle: distances against the relaxation-to-a-fixed-point loop.
	fmt.Println("\n(b) distances against the fixed-point oracle:")
	for i, c := range configs {
		want, sweeps := fixedPoint(graphs[i], 0)
		fmt.Printf("  V=%-6d deg=%-3d oracle needed %2d sweeps;  mismatches: lazy %d, indexed %d, lazy vs indexed %d\n", c.V, c.deg, sweeps,
			countDiff(lazies[i].dist, want), countDiff(indexeds[i].dist, want), countDiff(lazies[i].dist, indexeds[i].dist))
	}
	tiny, tinyBad, tinyUnreach := 2000, 0, 0
	for range tiny {
		V := 1 + r.IntN(8)
		g := graph{adj: make([][]edge, V)}
		for range r.IntN(21) {
			u := r.IntN(V)
			v := r.IntN(V)
			w := 1 + r.IntN(10)
			g.adj[u] = append(g.adj[u], edge{v, w})
		}
		want, _ := fixedPoint(g, 0)
		if countDiff(lazy(g, 0, -1, nil).dist, want)+countDiff(indexed(g, 0, nil).dist, want) > 0 {
			tinyBad++
		}
		if reached(want) < V {
			tinyUnreach++
		}
	}
	fmt.Printf("  %d tiny random graphs (V 1..8, up to 20 edges, weights 1..10): mismatches %d (%d of them had unreachable vertices)\n", tiny, tinyBad, tinyUnreach)

	// (c) Settle order, and what early exit buys.
	g, full := graphs[2], lazies[2]
	want, _ := fixedPoint(g, 0)
	ranked := slices.Clone(full.order)
	slices.SortFunc(ranked, func(a, b int) int {
		if want[a] != want[b] {
			return want[a] - want[b]
		}
		return a - b
	})
	reach := len(full.order)
	fmt.Printf("\n(c) settle order on V=20000, degree 8 (%d reachable):\n", reach)
	fmt.Printf("  settled distances non-decreasing: lazy %v, indexed %v\n", nonDecreasing(full.dist, full.order), nonDecreasing(indexeds[2].dist, indexeds[2].order))
	fmt.Printf("  settle sequence == reachable vertices sorted by (true distance, id): %v\n", slices.Equal(full.order, ranked))
	fmt.Printf("  stop as soon as the target is settled (full run: %d pops, %d pushes):\n", full.pops, full.pushes)
	fmt.Printf("  %6s %6s %7s %6s %8s %7s %7s %7s\n", "pctile", "rank", "target", "dist", "settled", "equal?", "pops", "pushes")
	for _, p := range []int{10, 50, 90, 100} {
		rank := (p*reach + 99) / 100
		target := ranked[rank-1]
		x := lazy(g, 0, target, nil)
		fmt.Printf("  %5d%% %6d %7d %6d %8d %7v %7d %7d\n", p, rank, target, want[target], len(x.order), len(x.order) == rank && slices.Equal(x.order, ranked[:rank]), x.pops, x.pushes)
	}

	// (d) The frontier while it runs: indexed heap = frontier; lazy heap = frontier + stale.
	fmt.Println("\n(d) the frontier on V=20000, degree 8, after k vertices are settled:")
	fmt.Printf("  %7s %9s %10s %12s %7s %10s\n", "settled", "frontier", "lazy heap", "lazy live", "stale", "live==front")
	ks := make([]int, 0, len(lazies[2].snaps))
	for k := range lazies[2].snaps {
		ks = append(ks, k)
	}
	slices.Sort(ks)
	for _, k := range ks {
		lz, ix := lazies[2].snaps[k], indexeds[2].snaps[k]
		fmt.Printf("  %7d %9d %10d %12d %7d %10v\n", k, ix.heapLen, lz.heapLen, lz.live, lz.heapLen-lz.live, lz.live == ix.heapLen)
	}
}

func countDiff(a, b []int) int {
	n := 0
	for i := range a {
		if a[i] != b[i] {
			n++
		}
	}
	return n
}
```

**Output:**

```
(a) V=2000, 16000 edges (degree 8): 1999 vertices reachable from 0, 15993 edge scans
  variant   pushes    pops   stale  fixes  relaxations  max heap Less calls
  lazy        3572    3572    1573      0         3571      2086      70806
  indexed     1999    1999       0   1573         3571      1192      39709
  same settle sequence: true;  lazy pushes == 1 + relaxations: true;  lazy stale == pops - reachable: true
  lazy max heap 2086 <= E+1 = 16001: true;  indexed max heap 1192 <= V = 2000: true;  indexed fixes == relaxations - (reachable-1): true
(a) V=2000, 80000 edges (degree 40): 2000 vertices reachable from 0, 80000 edge scans
  variant   pushes    pops   stale  fixes  relaxations  max heap Less calls
  lazy        6297    6297    4297      0         6296      4955     141578
  indexed     2000    2000       0   4297         6296      1764      46821
  same settle sequence: true;  lazy pushes == 1 + relaxations: true;  lazy stale == pops - reachable: true
  lazy max heap 4955 <= E+1 = 80001: true;  indexed max heap 1764 <= V = 2000: true;  indexed fixes == relaxations - (reachable-1): true
(a) V=20000, 160000 edges (degree 8): 19993 vertices reachable from 0, 159947 edge scans
  variant   pushes    pops   stale  fixes  relaxations  max heap Less calls
  lazy       35485   35485   15492      0        35484     21056     943326
  indexed    19993   19993       0  15492        35484     12340     530793
  same settle sequence: true;  lazy pushes == 1 + relaxations: true;  lazy stale == pops - reachable: true
  lazy max heap 21056 <= E+1 = 160001: true;  indexed max heap 12340 <= V = 20000: true;  indexed fixes == relaxations - (reachable-1): true

(b) distances against the fixed-point oracle:
  V=2000   deg=8   oracle needed  8 sweeps;  mismatches: lazy 0, indexed 0, lazy vs indexed 0
  V=2000   deg=40  oracle needed  8 sweeps;  mismatches: lazy 0, indexed 0, lazy vs indexed 0
  V=20000  deg=8   oracle needed 16 sweeps;  mismatches: lazy 0, indexed 0, lazy vs indexed 0
  2000 tiny random graphs (V 1..8, up to 20 edges, weights 1..10): mismatches 0 (1084 of them had unreachable vertices)

(c) settle order on V=20000, degree 8 (19993 reachable):
  settled distances non-decreasing: lazy true, indexed true
  settle sequence == reachable vertices sorted by (true distance, id): true
  stop as soon as the target is settled (full run: 35485 pops, 35485 pushes):
  pctile   rank  target   dist  settled  equal?    pops  pushes
     10%   2000   10747    114     2000    true    2023   12590
     50%   9997    4143    143     9997    true   10837   31782
     90%  17994    8677    171    17994    true   23909   35391
    100%  19993   10724    283    19993    true   35485   35485

(d) the frontier on V=20000, degree 8, after k vertices are settled:
  settled  frontier  lazy heap    lazy live   stale live==front
      200      1285       1310         1285      25       true
     2000      8912      10570         8912    1658       true
     5000     12316      18418        12316    6102       true
    10000      9629      20943         9629   11314       true
    15000      4957      16953         4957   11996       true
    18000      1987      11468         1987    9481       true
    19800       192       2969          192    2777       true
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::Ordering;
use std::collections::{BTreeMap, BinaryHeap};

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the
// random graphs here are the ones the Go program sees.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

const INF: usize = usize::MAX;

struct Graph {
    adj: Vec<Vec<(usize, usize)>>, // (to, weight)
    edges: usize,
}

/// random_graph: V*deg directed edges (u, v, w) with w in 1..100. Self-loops
/// and parallel edges are allowed. Draw order: u, v, w.
fn random_graph(r: &mut Pcg, v_count: usize, deg: usize) -> Graph {
    let mut adj = vec![Vec::new(); v_count];
    for _ in 0..v_count * deg {
        let u = r.int_n(v_count as u64) as usize;
        let v = r.int_n(v_count as u64) as usize;
        let w = 1 + r.int_n(100) as usize;
        adj[u].push((v, w));
    }
    Graph {
        adj,
        edges: v_count * deg,
    }
}

/// Snap is the frontier at one moment: the heap's length and how many of its
/// entries are live (d == dist[v]); only the lazy heap can hold stale ones.
struct Snap {
    heap_len: usize,
    live: usize,
}

/// Res carries the dist array and every count the table prints.
struct Res {
    dist: Vec<usize>,
    order: Vec<usize>, // vertices in the order they were settled
    pushes: usize,
    pops: usize,
    stale: usize,
    fixes: usize,
    relax: usize,
    scanned: usize,
    max_heap: usize,
    less: usize,
    snaps: BTreeMap<usize, Snap>, // heap state after the k-th settle, for k in marks
}

impl Res {
    fn new(v_count: usize) -> Self {
        Res {
            dist: vec![INF; v_count],
            order: Vec::new(),
            pushes: 0,
            pops: 0,
            stale: 0,
            fixes: 0,
            relax: 0,
            scanned: 0,
            max_heap: 0,
            less: 0,
            snaps: BTreeMap::new(),
        }
    }
}

// ---- Variant 1: lazy deletion on std's BinaryHeap. Every improvement PUSHES a
// new (dist, v) entry; the old one stays inside. A pop whose d is worse than
// dist[v] is stale: skip it. The key (d, v) is a strict total order: two
// entries for one vertex never share a d.
//
// BinaryHeap is a max-heap, so `cmp` is written reversed to make it a
// min-heap (the same effect as `Reverse((d, v))`), and it counts its calls in
// a thread-local so the table can print "Less calls" like Go's Less counter.
thread_local!(static LESS: Cell<usize> = const { Cell::new(0) });

#[derive(PartialEq, Eq)]
struct Entry {
    d: usize,
    v: usize,
}

impl Ord for Entry {
    fn cmp(&self, other: &Self) -> Ordering {
        LESS.set(LESS.get() + 1);
        other.d.cmp(&self.d).then(other.v.cmp(&self.v))
    }
}
impl PartialOrd for Entry {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// lazy runs Dijkstra from src; a target stops it the moment it is settled.
fn lazy(g: &Graph, src: usize, target: Option<usize>, marks: &[usize]) -> Res {
    let mut res = Res::new(g.adj.len());
    let mut h = BinaryHeap::new();
    LESS.set(0);
    res.dist[src] = 0;
    h.push(Entry { d: 0, v: src });
    res.pushes = 1;
    res.max_heap = 1;
    while let Some(Entry { d, v }) = h.pop() {
        res.pops += 1;
        if d > res.dist[v] {
            res.stale += 1;
            continue;
        }
        res.order.push(v); // settled: dist[v] is final
        if Some(v) == target {
            break;
        }
        for &(to, w) in &g.adj[v] {
            res.scanned += 1;
            let nd = d + w;
            if nd < res.dist[to] {
                res.dist[to] = nd;
                res.relax += 1;
                h.push(Entry { d: nd, v: to });
                res.pushes += 1;
                res.max_heap = res.max_heap.max(h.len());
            }
        }
        let k = res.order.len();
        if marks.contains(&k) {
            let live = h.iter().filter(|x| x.d == res.dist[x.v]).count();
            res.snaps.insert(
                k,
                Snap {
                    heap_len: h.len(),
                    live,
                },
            );
        }
    }
    res.less = LESS.get();
    res
}

// ---- Variant 2: indexed. std has no `Fix`, so this is Go's container/heap
// (`up`, `down`, `Fix`) ported by hand: one entry per vertex, `pos[v]` is its
// slot, kept up to date by `swap`. The distances live outside (Dijkstra writes
// them between heap calls), so every method borrows `dist` for the call.
struct IdxPQ {
    vs: Vec<usize>,
    pos: Vec<usize>,
    less: usize,
}

impl IdxPQ {
    fn less(&mut self, i: usize, j: usize, dist: &[usize]) -> bool {
        self.less += 1;
        let (a, b) = (self.vs[i], self.vs[j]);
        if dist[a] != dist[b] {
            return dist[a] < dist[b];
        }
        a < b
    }

    fn swap(&mut self, i: usize, j: usize) {
        self.vs.swap(i, j);
        self.pos[self.vs[i]] = i;
        self.pos[self.vs[j]] = j;
    }

    fn up(&mut self, mut j: usize, dist: &[usize]) {
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less(j, i, dist) {
                break;
            }
            self.swap(i, j);
            j = i;
        }
    }

    /// down sifts slot i0 down inside the first n slots; true if it moved.
    fn down(&mut self, i0: usize, n: usize, dist: &[usize]) -> bool {
        let mut i = i0;
        loop {
            let j1 = 2 * i + 1;
            if j1 >= n {
                break;
            }
            let mut j = j1;
            if j1 + 1 < n && self.less(j1 + 1, j1, dist) {
                j = j1 + 1;
            }
            if !self.less(j, i, dist) {
                break;
            }
            self.swap(i, j);
            i = j;
        }
        i > i0
    }

    fn push(&mut self, v: usize, dist: &[usize]) {
        self.pos[v] = self.vs.len();
        self.vs.push(v);
        self.up(self.vs.len() - 1, dist);
    }

    fn pop(&mut self, dist: &[usize]) -> Option<usize> {
        let n = self.vs.len().checked_sub(1)?;
        self.swap(0, n);
        self.down(0, n, dist);
        let v = self.vs.pop()?;
        self.pos[v] = INF; // Go's -1: "not in the heap"
        Some(v)
    }

    fn fix(&mut self, i: usize, dist: &[usize]) {
        if !self.down(i, self.vs.len(), dist) {
            self.up(i, dist);
        }
    }
}

fn indexed(g: &Graph, src: usize, marks: &[usize]) -> Res {
    let mut res = Res::new(g.adj.len());
    let mut h = IdxPQ {
        vs: Vec::new(),
        pos: vec![INF; g.adj.len()],
        less: 0,
    };
    res.dist[src] = 0;
    h.push(src, &res.dist);
    res.pushes = 1;
    res.max_heap = 1;
    while let Some(u) = h.pop(&res.dist) {
        res.pops += 1;
        res.order.push(u);
        for &(to, w) in &g.adj[u] {
            res.scanned += 1;
            let nd = res.dist[u] + w;
            if nd >= res.dist[to] {
                continue;
            }
            res.relax += 1;
            let first = res.dist[to] == INF; // never discovered: not in the heap yet
            res.dist[to] = nd;
            if first {
                h.push(to, &res.dist);
                res.pushes += 1;
                res.max_heap = res.max_heap.max(h.vs.len());
            } else {
                h.fix(h.pos[to], &res.dist); // key decreased: sift up
                res.fixes += 1;
            }
        }
        let k = res.order.len();
        if marks.contains(&k) {
            res.snaps.insert(
                k,
                Snap {
                    heap_len: h.vs.len(),
                    live: h.vs.len(),
                },
            );
        }
    }
    res.less = h.less;
    res
}

/// fixed_point is the oracle: relax every edge, again and again, until a whole
/// sweep changes nothing (lesson 24 explains why this converges: Bellman-Ford).
fn fixed_point(g: &Graph, src: usize) -> (Vec<usize>, usize) {
    let mut dist = vec![INF; g.adj.len()];
    dist[src] = 0;
    let mut sweeps = 0;
    let mut changed = true;
    while changed {
        changed = false;
        sweeps += 1;
        for (u, es) in g.adj.iter().enumerate() {
            if dist[u] == INF {
                continue;
            }
            for &(to, w) in es {
                if dist[u] + w < dist[to] {
                    dist[to] = dist[u] + w;
                    changed = true;
                }
            }
        }
    }
    (dist, sweeps)
}

fn reached(dist: &[usize]) -> usize {
    dist.iter().filter(|&&d| d != INF).count()
}

fn non_decreasing(dist: &[usize], order: &[usize]) -> bool {
    order.windows(2).all(|w| dist[w[0]] <= dist[w[1]])
}

fn count_diff(a: &[usize], b: &[usize]) -> usize {
    a.iter().zip(b).filter(|(x, y)| x != y).count()
}

fn main() {
    let mut r = Pcg::new(1, 2);

    // (a) The frontier, two ways, on three random graphs.
    let configs = [(2000usize, 8usize), (2000, 40), (20000, 8)];
    let (mut graphs, mut lazies, mut indexeds) = (Vec::new(), Vec::new(), Vec::new());
    for &(v_count, deg) in &configs {
        let marks: Vec<usize> = [1, 10, 25, 50, 75, 90, 99]
            .iter()
            .map(|p| v_count * p / 100)
            .collect();
        let g = random_graph(&mut r, v_count, deg);
        let (lz, ix) = (lazy(&g, 0, None, &marks), indexed(&g, 0, &marks));
        let reach = lz.order.len();
        println!(
            "(a) V={v_count}, {} edges (degree {deg}): {reach} vertices reachable from 0, {} edge scans",
            g.edges, lz.scanned
        );
        println!(
            "  {:<8} {:>7} {:>7} {:>7} {:>6} {:>12} {:>9} {:>10}",
            "variant", "pushes", "pops", "stale", "fixes", "relaxations", "max heap", "Less calls"
        );
        for (name, x) in [("lazy", &lz), ("indexed", &ix)] {
            println!(
                "  {:<8} {:>7} {:>7} {:>7} {:>6} {:>12} {:>9} {:>10}",
                name, x.pushes, x.pops, x.stale, x.fixes, x.relax, x.max_heap, x.less
            );
        }
        println!(
            "  same settle sequence: {};  lazy pushes == 1 + relaxations: {};  lazy stale == pops - reachable: {}",
            lz.order == ix.order,
            lz.pushes == 1 + lz.relax,
            lz.stale == lz.pops - reach
        );
        println!(
            "  lazy max heap {} <= E+1 = {}: {};  indexed max heap {} <= V = {v_count}: {};  indexed fixes == relaxations - (reachable-1): {}",
            lz.max_heap,
            g.edges + 1,
            lz.max_heap <= g.edges + 1,
            ix.max_heap,
            ix.max_heap <= v_count,
            ix.fixes == ix.relax - (reach - 1)
        );
        graphs.push(g);
        lazies.push(lz);
        indexeds.push(ix);
    }

    // (b) Oracle: distances against the relaxation-to-a-fixed-point loop.
    println!("\n(b) distances against the fixed-point oracle:");
    for (i, &(v_count, deg)) in configs.iter().enumerate() {
        let (want, sweeps) = fixed_point(&graphs[i], 0);
        println!(
            "  V={v_count:<6} deg={deg:<3} oracle needed {sweeps:>2} sweeps;  mismatches: lazy {}, indexed {}, lazy vs indexed {}",
            count_diff(&lazies[i].dist, &want),
            count_diff(&indexeds[i].dist, &want),
            count_diff(&lazies[i].dist, &indexeds[i].dist)
        );
    }
    let (tiny, mut tiny_bad, mut tiny_unreach) = (2000, 0, 0);
    for _ in 0..tiny {
        let v_count = 1 + r.int_n(8) as usize;
        let mut adj = vec![Vec::new(); v_count];
        for _ in 0..r.int_n(21) {
            let u = r.int_n(v_count as u64) as usize;
            let v = r.int_n(v_count as u64) as usize;
            let w = 1 + r.int_n(10) as usize;
            adj[u].push((v, w));
        }
        let g = Graph { adj, edges: 0 };
        let (want, _) = fixed_point(&g, 0);
        if count_diff(&lazy(&g, 0, None, &[]).dist, &want)
            + count_diff(&indexed(&g, 0, &[]).dist, &want)
            > 0
        {
            tiny_bad += 1;
        }
        if reached(&want) < v_count {
            tiny_unreach += 1;
        }
    }
    println!(
        "  {tiny} tiny random graphs (V 1..8, up to 20 edges, weights 1..10): mismatches {tiny_bad} ({tiny_unreach} of them had unreachable vertices)"
    );

    // (c) Settle order, and what early exit buys.
    let (g, full) = (&graphs[2], &lazies[2]);
    let (want, _) = fixed_point(g, 0);
    let mut ranked = full.order.clone();
    ranked.sort_by(|&a, &b| want[a].cmp(&want[b]).then(a.cmp(&b)));
    let reach = full.order.len();
    println!("\n(c) settle order on V=20000, degree 8 ({reach} reachable):");
    println!(
        "  settled distances non-decreasing: lazy {}, indexed {}",
        non_decreasing(&full.dist, &full.order),
        non_decreasing(&indexeds[2].dist, &indexeds[2].order)
    );
    println!(
        "  settle sequence == reachable vertices sorted by (true distance, id): {}",
        full.order == ranked
    );
    println!(
        "  stop as soon as the target is settled (full run: {} pops, {} pushes):",
        full.pops, full.pushes
    );
    println!(
        "  {:>6} {:>6} {:>7} {:>6} {:>8} {:>7} {:>7} {:>7}",
        "pctile", "rank", "target", "dist", "settled", "equal?", "pops", "pushes"
    );
    for p in [10, 50, 90, 100] {
        let rank = (p * reach).div_ceil(100);
        let target = ranked[rank - 1];
        let x = lazy(g, 0, Some(target), &[]);
        println!(
            "  {p:>5}% {rank:>6} {target:>7} {:>6} {:>8} {:>7} {:>7} {:>7}",
            want[target],
            x.order.len(),
            x.order.len() == rank && x.order == ranked[..rank],
            x.pops,
            x.pushes
        );
    }

    // (d) The frontier while it runs: indexed heap = frontier; lazy heap = frontier + stale.
    println!("\n(d) the frontier on V=20000, degree 8, after k vertices are settled:");
    println!(
        "  {:>7} {:>9} {:>10} {:>12} {:>7} {:>10}",
        "settled", "frontier", "lazy heap", "lazy live", "stale", "live==front"
    );
    for (k, lz) in &lazies[2].snaps {
        let ix = &indexeds[2].snaps[k];
        println!(
            "  {k:>7} {:>9} {:>10} {:>12} {:>7} {:>10}",
            ix.heap_len,
            lz.heap_len,
            lz.live,
            lz.heap_len - lz.live,
            lz.live == ix.heap_len
        );
    }
}
```

**Rust output:**

```
(a) V=2000, 16000 edges (degree 8): 1999 vertices reachable from 0, 15993 edge scans
  variant   pushes    pops   stale  fixes  relaxations  max heap Less calls
  lazy        3572    3572    1573      0         3571      2086      44016
  indexed     1999    1999       0   1573         3571      1192      39709
  same settle sequence: true;  lazy pushes == 1 + relaxations: true;  lazy stale == pops - reachable: true
  lazy max heap 2086 <= E+1 = 16001: true;  indexed max heap 1192 <= V = 2000: true;  indexed fixes == relaxations - (reachable-1): true
(a) V=2000, 80000 edges (degree 40): 2000 vertices reachable from 0, 80000 edge scans
  variant   pushes    pops   stale  fixes  relaxations  max heap Less calls
  lazy        6297    6297    4297      0         6296      4955      87789
  indexed     2000    2000       0   4297         6296      1764      46821
  same settle sequence: true;  lazy pushes == 1 + relaxations: true;  lazy stale == pops - reachable: true
  lazy max heap 4955 <= E+1 = 80001: true;  indexed max heap 1764 <= V = 2000: true;  indexed fixes == relaxations - (reachable-1): true
(a) V=20000, 160000 edges (degree 8): 19993 vertices reachable from 0, 159947 edge scans
  variant   pushes    pops   stale  fixes  relaxations  max heap Less calls
  lazy       35485   35485   15492      0        35484     21056     558016
  indexed    19993   19993       0  15492        35484     12340     530793
  same settle sequence: true;  lazy pushes == 1 + relaxations: true;  lazy stale == pops - reachable: true
  lazy max heap 21056 <= E+1 = 160001: true;  indexed max heap 12340 <= V = 20000: true;  indexed fixes == relaxations - (reachable-1): true

(b) distances against the fixed-point oracle:
  V=2000   deg=8   oracle needed  8 sweeps;  mismatches: lazy 0, indexed 0, lazy vs indexed 0
  V=2000   deg=40  oracle needed  8 sweeps;  mismatches: lazy 0, indexed 0, lazy vs indexed 0
  V=20000  deg=8   oracle needed 16 sweeps;  mismatches: lazy 0, indexed 0, lazy vs indexed 0
  2000 tiny random graphs (V 1..8, up to 20 edges, weights 1..10): mismatches 0 (1084 of them had unreachable vertices)

(c) settle order on V=20000, degree 8 (19993 reachable):
  settled distances non-decreasing: lazy true, indexed true
  settle sequence == reachable vertices sorted by (true distance, id): true
  stop as soon as the target is settled (full run: 35485 pops, 35485 pushes):
  pctile   rank  target   dist  settled  equal?    pops  pushes
     10%   2000   10747    114     2000    true    2023   12590
     50%   9997    4143    143     9997    true   10837   31782
     90%  17994    8677    171    17994    true   23909   35391
    100%  19993   10724    283    19993    true   35485   35485

(d) the frontier on V=20000, degree 8, after k vertices are settled:
  settled  frontier  lazy heap    lazy live   stale live==front
      200      1285       1310         1285      25       true
     2000      8912      10570         8912    1658       true
     5000     12316      18418        12316    6102       true
    10000      9629      20943         9629   11314       true
    15000      4957      16953         4957   11996       true
    18000      1987      11468         1987    9481       true
    19800       192       2969          192    2777       true
```

> **Go vs Rust:** Identical to the Go output except one column: the **lazy** row's `Less calls` (V=20000: 558,016 here against Go's 943,326), because those rows run on std's `BinaryHeap`, whose `pop` sifts the hole to the bottom with one comparison per level (read in the source, not isolated here; example 15 counts it) while `container/heap` spends two. Every push, pop, stale, fix and relaxation count, the peak sizes, the snapshots and the whole indexed row match, since that heap is Go's `up`/`down`/`Fix` ported by hand (std has no `Fix` or index) and its methods borrow `&[usize]` for the call because `dist` changes between heap calls. `Entry`'s reversed `cmp` is a `Reverse((d, v))` that also counts its calls.

**Complexity:** Θ((V+E) log V) · lazy: one pop per push, the heap peaked at 21,056 entries (bound E+1 = 160,001) and 15,492 of the 35,485 pops were stale · indexed: 19,993 pushes, 15,492 `Fix` calls, the heap peaked at 12,340 (bound V = 20,000) · both settle the same vertices in the same order (distances non-decreasing, equal to the sort by `(distance, id)`), and stopping at the target ranked r settles exactly r vertices · `container/heap` made 943,326 `Less` calls for the lazy run and 530,793 for the indexed one

---

## 14. What `container/heap` costs

`🔴 hard` · *Interface boxing / `any`*

`container/heap` predates generics: `Push(x any)` and `Pop() any` move every element through an interface, and `Len`, `Less` and `Swap` are interface method calls. This example prices that in the order of how reliable the numbers are. First **allocations per operation** with `testing.AllocsPerRun` (capacity reserved so slice growth is excluded, and a *varying* pushed value, because the runtime boxes constants and values below 256 for free), then **calls per operation** (counters inside the methods, against a generic heap with the same algorithm, so the gap cannot be comparisons), and last one timing table. Every variant is Θ(log n) per operation; what differs is the constant. (Lesson 04 example 11 measured boxing for stacks, queues and trees; this is the heap's own bill.)

**Steps:**

1. Write the three `container/heap` element types (an `int`, a 16-byte `Item`, a `*Item`) and a generic `Heap[T]` with a `less` func, then count allocations per `heap.Push` and `heap.Pop` for each.
2. Count every method call (`Len`, `Less`, `Swap`, `Push`, `Pop`) over n = 2^16 random pushes and pops, against the generic heap's `Less` calls.
3. Time push-n-then-pop-n for `container/heap`, the generic heap, a specialised `[]int` heap with `<` inline and `slices.Sort`, interleaved and best-of-rounds, and check every variant drained the same sorted order.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
	"time"
)

// ---- container/heap needs five methods per element type. These are the three
// element types the allocation table compares: an int, a 16-byte struct, a pointer.
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x // an int becomes an `any` here
}

type Item struct{ pri, id int }
type ItemHeap []Item

func (h ItemHeap) Len() int           { return len(h) }
func (h ItemHeap) Less(i, j int) bool { return h[i].pri < h[j].pri }
func (h ItemHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *ItemHeap) Push(x any)        { *h = append(*h, x.(Item)) }
func (h *ItemHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}

type PtrHeap []*Item

func (h PtrHeap) Len() int           { return len(h) }
func (h PtrHeap) Less(i, j int) bool { return h[i].pri < h[j].pri }
func (h PtrHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *PtrHeap) Push(x any)        { *h = append(*h, x.(*Item)) }
func (h *PtrHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	old[len(old)-1] = nil
	*h = old[:len(old)-1]
	return x
}

// Heap[T] is the generic alternative: the same up/down as container/heap, but the
// elements never leave their type and Less is a func value, not an interface method.
type Heap[T any] struct {
	a    []T
	less func(a, b T) bool
}

func (h *Heap[T]) Push(x T) {
	h.a = append(h.a, x)
	for j := len(h.a) - 1; ; {
		i := (j - 1) / 2
		if i == j || !h.less(h.a[j], h.a[i]) {
			break
		}
		h.a[i], h.a[j] = h.a[j], h.a[i]
		j = i
	}
}

func (h *Heap[T]) Pop() T {
	n := len(h.a) - 1
	h.a[0], h.a[n] = h.a[n], h.a[0]
	h.down(0, n)
	x := h.a[n]
	h.a = h.a[:n]
	return x
}

func (h *Heap[T]) down(i, n int) {
	for {
		j := 2*i + 1
		if j >= n {
			break
		}
		if j+1 < n && h.less(h.a[j+1], h.a[j]) {
			j++
		}
		if !h.less(h.a[j], h.a[i]) {
			break
		}
		h.a[i], h.a[j] = h.a[j], h.a[i]
		i = j
	}
}

// ---- (a) allocations per operation
const runs = 1000 // AllocsPerRun calls f runs+1 times (one warm-up): reserve runs+2 slots

func pushAllocs(h heap.Interface, mk func(i int) any) float64 {
	i := 0
	return testing.AllocsPerRun(runs, func() { heap.Push(h, mk(i)); i++ })
}

func popAllocs(h heap.Interface) float64 {
	return testing.AllocsPerRun(runs, func() { heap.Pop(h) })
}

func row(name string, push, pop float64) {
	fmt.Printf("  %-34s %10.0f %10.0f\n", name, push, pop)
}

// ---- (b) calls per operation: every method of the container/heap type counts itself
type countedHeap struct {
	a                   []int
	lens, less, swaps   int
	pushCalls, popCalls int
}

func (h *countedHeap) Len() int           { h.lens++; return len(h.a) }
func (h *countedHeap) Less(i, j int) bool { h.less++; return h.a[i] < h.a[j] }
func (h *countedHeap) Swap(i, j int)      { h.swaps++; h.a[i], h.a[j] = h.a[j], h.a[i] }
func (h *countedHeap) Push(x any)         { h.pushCalls++; h.a = append(h.a, x.(int)) }
func (h *countedHeap) Pop() any {
	h.popCalls++
	x := h.a[len(h.a)-1]
	h.a = h.a[:len(h.a)-1]
	return x
}

// ---- (c) timing: three heaps and a sort, same keys
// minHeap is the specialised heap: []int and `<` written inline, no func value.
func pushInt(a *[]int, x int) {
	*a = append(*a, x)
	for j := len(*a) - 1; ; {
		i := (j - 1) / 2
		if i == j || (*a)[j] >= (*a)[i] {
			break
		}
		(*a)[i], (*a)[j] = (*a)[j], (*a)[i]
		j = i
	}
}

func popInt(a *[]int) int {
	s := *a
	n := len(s) - 1
	s[0], s[n] = s[n], s[0]
	for i := 0; ; {
		j := 2*i + 1
		if j >= n {
			break
		}
		if j+1 < n && s[j+1] < s[j] {
			j++
		}
		if s[j] >= s[i] {
			break
		}
		s[i], s[j] = s[j], s[i]
		i = j
	}
	x := s[n]
	*a = s[:n]
	return x
}

type variant struct {
	name string
	// run pushes all of in, then pops everything, reps times; it returns the time
	// spent in each phase and a checksum of the popped order.
	run func(in []int, reps int) (push, pop time.Duration, sum uint64)
}

func fold(sum uint64, x int) uint64 { return sum*31 + uint64(x) }

func variantsFor(n int) []variant {
	return []variant{
		{"container/heap, IntHeap", func(in []int, reps int) (pt, qt time.Duration, sum uint64) {
			h := make(IntHeap, 0, n)
			for range reps {
				t0 := time.Now()
				for _, x := range in {
					heap.Push(&h, x)
				}
				t1 := time.Now()
				sum = 0
				for h.Len() > 0 {
					sum = fold(sum, heap.Pop(&h).(int))
				}
				pt, qt = pt+t1.Sub(t0), qt+time.Since(t1)
			}
			return
		}},
		{"generic Heap[int], less func", func(in []int, reps int) (pt, qt time.Duration, sum uint64) {
			h := &Heap[int]{a: make([]int, 0, n), less: func(a, b int) bool { return a < b }}
			for range reps {
				t0 := time.Now()
				for _, x := range in {
					h.Push(x)
				}
				t1 := time.Now()
				sum = 0
				for len(h.a) > 0 {
					sum = fold(sum, h.Pop())
				}
				pt, qt = pt+t1.Sub(t0), qt+time.Since(t1)
			}
			return
		}},
		{"specialised []int, < inline", func(in []int, reps int) (pt, qt time.Duration, sum uint64) {
			h := make([]int, 0, n)
			for range reps {
				t0 := time.Now()
				for _, x := range in {
					pushInt(&h, x)
				}
				t1 := time.Now()
				sum = 0
				for len(h) > 0 {
					sum = fold(sum, popInt(&h))
				}
				pt, qt = pt+t1.Sub(t0), qt+time.Since(t1)
			}
			return
		}},
		{"slices.Sort (baseline)", func(in []int, reps int) (pt, qt time.Duration, sum uint64) {
			buf := make([]int, n)
			for range reps {
				copy(buf, in) // not timed
				t0 := time.Now()
				slices.Sort(buf)
				qt += time.Since(t0)
			}
			for _, x := range buf {
				sum = fold(sum, x)
			}
			return
		}},
	}
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	// (a) Allocations per Push and per Pop: reserved capacity, a VARYING value on
	// every call (the runtime boxes constants and values below 256 for free).
	fmt.Println("(a) allocations per operation (testing.AllocsPerRun, capacity reserved, varying values):")
	fmt.Printf("  %-34s %10s %10s\n", "container/heap, element type", "heap.Push", "heap.Pop")
	newInts := func(f func(i int) int) *IntHeap {
		h := make(IntHeap, runs+2)
		for i := range h {
			h[i] = f(i)
		}
		heap.Init(&h)
		return &h
	}
	empty := func() *IntHeap { h := make(IntHeap, 0, runs+2); return &h }
	small, large := func(i int) int { return i % 200 }, func(i int) int { return 1000 + i }
	row("int below 256", pushAllocs(empty(), func(i int) any { return small(i) }), popAllocs(newInts(small)))
	row("int, 256 or more", pushAllocs(empty(), func(i int) any { return large(i) }), popAllocs(newInts(large)))
	items := make(ItemHeap, runs+2)
	for i := range items {
		items[i] = Item{i, i}
	}
	heap.Init(&items)
	ie := make(ItemHeap, 0, runs+2)
	row("Item{pri, id int}, 16 bytes", pushAllocs(&ie, func(i int) any { return Item{1000 + i, i} }), popAllocs(&items))
	ptrs := make([]*Item, runs+2)
	for i := range ptrs {
		ptrs[i] = &Item{i, i}
	}
	pe := make(PtrHeap, 0, runs+2)
	pp := PtrHeap(slices.Clone(ptrs))
	heap.Init(&pp)
	row("*Item, already allocated", pushAllocs(&pe, func(i int) any { return ptrs[i] }), popAllocs(&pp))
	pe2 := make(PtrHeap, 0, runs+2)
	pp2 := PtrHeap(slices.Clone(ptrs))
	heap.Init(&pp2)
	row("*Item, &Item{} at the call", pushAllocs(&pe2, func(i int) any { return &Item{i, i} }), popAllocs(&pp2))
	gi := &Heap[int]{a: make([]int, 0, runs+2), less: func(a, b int) bool { return a < b }}
	gp := &Heap[int]{a: make([]int, runs+2), less: func(a, b int) bool { return a < b }}
	pi := 0
	pushG := testing.AllocsPerRun(runs, func() { gi.Push(1000 + pi); pi++ })
	row("generic Heap[int], 256 or more", pushG, testing.AllocsPerRun(runs, func() { gp.Pop() }))
	gs := &Heap[Item]{a: make([]Item, 0, runs+2), less: func(a, b Item) bool { return a.pri < b.pri }}
	gq := &Heap[Item]{a: make([]Item, runs+2), less: func(a, b Item) bool { return a.pri < b.pri }}
	si := 0
	pushS := testing.AllocsPerRun(runs, func() { gs.Push(Item{1000 + si, si}); si++ })
	row("generic Heap[Item]", pushS, testing.AllocsPerRun(runs, func() { gq.Pop() }))
	grow := testing.AllocsPerRun(10, func() {
		var h IntHeap
		for i := range 1000 {
			heap.Push(&h, i%200)
		}
	})
	fmt.Printf("  no reservation: 1,000 pushes of ints below 256 into an empty IntHeap allocate %.0f times (slice growth only)\n", grow)

	// (b) Calls per operation, n = 2^16 random keys: container/heap against the
	// generic heap, which runs the same algorithm.
	const n = 1 << 16
	keys := make([]int, n)
	for i := range keys {
		keys[i] = r.IntN(1 << 30)
	}
	ch := &countedHeap{a: make([]int, 0, n)}
	for _, x := range keys {
		heap.Push(ch, x)
	}
	pushCalls := *ch
	*ch = countedHeap{a: ch.a}
	var order1 []int
	for len(ch.a) > 0 {
		order1 = append(order1, heap.Pop(ch).(int))
	}
	popCalls := *ch
	genCmps := 0
	gh := &Heap[int]{a: make([]int, 0, n), less: func(a, b int) bool { genCmps++; return a < b }}
	for _, x := range keys {
		gh.Push(x)
	}
	genPush := genCmps
	genCmps = 0
	var order2 []int
	for len(gh.a) > 0 {
		order2 = append(order2, gh.Pop())
	}
	genPop := genCmps
	fmt.Printf("\n(b) calls per operation for n = %d random keys (push n, then pop n):\n", n)
	fmt.Printf("  %-22s %6s %6s %6s %6s %6s   %s\n", "", "Len", "Less", "Swap", "Push", "Pop", "interface-method calls per op")
	fn := float64(n)
	total := func(c countedHeap) int { return c.lens + c.less + c.swaps + c.pushCalls + c.popCalls }
	fmt.Printf("  %-22s %6.2f %6.2f %6.2f %6.2f %6.2f   %.2f\n", "container/heap, push", float64(pushCalls.lens)/fn, float64(pushCalls.less)/fn, float64(pushCalls.swaps)/fn, float64(pushCalls.pushCalls)/fn, 0.0, float64(total(pushCalls))/fn)
	fmt.Printf("  %-22s %6.2f %6.2f %6.2f %6.2f %6.2f   %.2f\n", "container/heap, pop", float64(popCalls.lens)/fn, float64(popCalls.less)/fn, float64(popCalls.swaps)/fn, 0.0, float64(popCalls.popCalls)/fn, float64(total(popCalls))/fn)
	fmt.Printf("  %-22s %6s %6.2f %6s %6s %6s   %s\n", "generic Heap, push", "-", float64(genPush)/fn, "-", "-", "-", "0 (Less is a func value, the rest is inline)")
	fmt.Printf("  %-22s %6s %6.2f %6s %6s %6s   %s\n", "generic Heap, pop", "-", float64(genPop)/fn, "-", "-", "-", "0")
	fmt.Printf("  Less calls equal: push %v, pop %v;  popped orders identical and sorted: %v\n",
		pushCalls.less == genPush, popCalls.less == genPop, slices.Equal(order1, order2) && slices.IsSorted(order1))

	// (c) One timing table: push n random keys, pop them all.
	fmt.Println("\n(c) push n random keys, then pop them all (best of 5 interleaved rounds, 3 at n = 10^6):")
	fmt.Printf("  %8s  %-30s %8s %8s %9s %9s\n", "n", "variant", "push ns", "pop ns", "total ns", "vs inline")
	for _, n := range []int{1_000, 100_000, 1_000_000} {
		in := make([]int, n)
		for i := range in {
			in[i] = r.IntN(1_000_000_000)
		}
		want := slices.Clone(in)
		slices.Sort(want)
		var wantSum uint64
		for _, x := range want {
			wantSum = fold(wantSum, x)
		}
		vs := variantsFor(n)
		reps, rounds := max(1, 200_000/n), 5
		if n >= 1_000_000 {
			rounds = 3
		}
		type best struct{ push, pop, total float64 }
		bests := make([]best, len(vs))
		same := true
		for round := 0; round < rounds; round++ {
			for k, v := range vs {
				pt, qt, sum := v.run(in, reps)
				same = same && sum == wantSum
				per := float64(reps * n)
				b := best{float64(pt) / per, float64(qt) / per, float64(pt+qt) / per}
				if round == 0 || b.total < bests[k].total {
					bests[k] = b
				}
			}
		}
		for k, v := range vs {
			b := bests[k]
			if k == len(vs)-1 {
				fmt.Printf("  %8d  %-30s %8s %8s %9.1f %8.2fx\n", n, v.name, "-", "-", b.total, b.total/bests[2].total)
				continue
			}
			fmt.Printf("  %8d  %-30s %8.1f %8.1f %9.1f %8.2fx\n", n, v.name, b.push, b.pop, b.total, b.total/bests[2].total)
		}
		fmt.Printf("  %8s  every variant drained the keys in sorted order (checksum): %v\n", "", same)
	}
}
```

**Sample output:**

```
(a) allocations per operation (testing.AllocsPerRun, capacity reserved, varying values):
  container/heap, element type        heap.Push   heap.Pop
  int below 256                               0          0
  int, 256 or more                            1          1
  Item{pri, id int}, 16 bytes                 1          1
  *Item, already allocated                    0          0
  *Item, &Item{} at the call                  1          0
  generic Heap[int], 256 or more              0          0
  generic Heap[Item]                          0          0
  no reservation: 1,000 pushes of ints below 256 into an empty IntHeap allocate 13 times (slice growth only)

(b) calls per operation for n = 65536 random keys (push n, then pop n):
                            Len   Less   Swap   Push    Pop   interface-method calls per op
  container/heap, push     1.00   2.28   1.28   1.00   0.00   5.57
  container/heap, pop      1.00  27.03  14.36   0.00   1.00   43.40
  generic Heap, push          -   2.28      -      -      -   0 (Less is a func value, the rest is inline)
  generic Heap, pop           -  27.03      -      -      -   0
  Less calls equal: push true, pop true;  popped orders identical and sorted: true

(c) push n random keys, then pop them all (best of 5 interleaved rounds, 3 at n = 10^6):
         n  variant                         push ns   pop ns  total ns vs inline
      1000  container/heap, IntHeap            15.9     38.8      54.8     4.58x
      1000  generic Heap[int], less func        4.4     22.9      27.3     2.28x
      1000  specialised []int, < inline         2.3      9.7      11.9     1.00x
      1000  slices.Sort (baseline)                -        -       8.3     0.70x
            every variant drained the keys in sorted order (checksum): true
    100000  container/heap, IntHeap            21.2    117.6     138.7     2.10x
    100000  generic Heap[int], less func       12.8     90.6     103.4     1.56x
    100000  specialised []int, < inline        10.9     55.3      66.2     1.00x
    100000  slices.Sort (baseline)                -        -      47.5     0.72x
            every variant drained the keys in sorted order (checksum): true
   1000000  container/heap, IntHeap            20.5    154.7     175.2     2.08x
   1000000  generic Heap[int], less func       12.8    118.3     131.1     1.56x
   1000000  specialised []int, < inline        10.7     73.6      84.3     1.00x
   1000000  slices.Sort (baseline)                -        -      56.5     0.67x
            every variant drained the keys in sorted order (checksum): true
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- A counting global allocator: Rust's stand-in for testing.AllocsPerRun.
// Every allocation event (alloc, and the realloc a growing Vec performs) bumps
// the counter; frees are not counted. Deterministic, unlike Go's GC-noisy stats.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    // SAFETY (all three): forwards unchanged to the system allocator.
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

/// allocs_of: the allocation events while running `f`.
fn allocs_of(f: impl FnOnce()) -> usize {
    let a0 = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - a0
}

// Pcg: a port of Go's math/rand/v2 PCG-DXSM generator and its IntN, so the
// random keys here are the ones the Go program sees.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

// Item is the 16-byte element of the allocation table (Go's Item{pri, id int}).
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Item {
    pri: u64,
    id: u64,
}

// Heap<T, F> is the generic hand-written heap: the same up/down as Go's
// container/heap (swap at every level, two comparisons per level on the way
// down). `F` is a type parameter, so each instantiation gets its own copy of the
// code with `less` inlined; Go's `Heap[T]` holds a func VALUE instead.
struct Heap<T, F: Fn(&T, &T) -> bool> {
    a: Vec<T>,
    less: F,
}

impl<T, F: Fn(&T, &T) -> bool> Heap<T, F> {
    fn with_capacity(cap: usize, less: F) -> Self {
        Heap {
            a: Vec::with_capacity(cap),
            less,
        }
    }

    fn push(&mut self, x: T) {
        self.a.push(x);
        let mut j = self.a.len() - 1;
        while j > 0 {
            let i = (j - 1) / 2;
            if !(self.less)(&self.a[j], &self.a[i]) {
                break;
            }
            self.a.swap(i, j);
            j = i;
        }
    }

    fn pop(&mut self) -> Option<T> {
        let n = self.a.len().checked_sub(1)?;
        self.a.swap(0, n);
        self.down(0, n);
        self.a.pop()
    }

    fn down(&mut self, mut i: usize, n: usize) {
        loop {
            let j1 = 2 * i + 1;
            if j1 >= n {
                break;
            }
            let mut j = j1;
            if j1 + 1 < n && (self.less)(&self.a[j1 + 1], &self.a[j1]) {
                j = j1 + 1;
            }
            if !(self.less)(&self.a[j], &self.a[i]) {
                break;
            }
            self.a.swap(i, j);
            i = j;
        }
    }
}

// Cnt counts its own comparisons, so std's BinaryHeap can be asked how many it makes.
thread_local!(static CMPS: Cell<usize> = const { Cell::new(0) });

#[derive(PartialEq, Eq)]
struct Cnt(u64);
impl Ord for Cnt {
    fn cmp(&self, other: &Self) -> Ordering {
        CMPS.set(CMPS.get() + 1);
        self.0.cmp(&other.0)
    }
}
impl PartialOrd for Cnt {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

// ---- (c) timing: std's BinaryHeap, the own heap twice, and a sort
type Timed = (Duration, Duration, u64);
type Runner = fn(&[u64], usize) -> Timed;

// the two value streams of the allocation table: below 256, and 256 or more
fn small(i: u64) -> u64 {
    i % 200
}
fn large(i: u64) -> u64 {
    1000 + i
}

fn fold(sum: u64, x: u64) -> u64 {
    sum.wrapping_mul(31).wrapping_add(x)
}

fn run_std(input: &[u64], reps: usize) -> Timed {
    let mut h = BinaryHeap::with_capacity(input.len());
    let (mut pt, mut qt, mut sum) = (Duration::ZERO, Duration::ZERO, 0);
    for _ in 0..reps {
        let t0 = Instant::now();
        for &x in input {
            h.push(Reverse(x));
        }
        let t1 = Instant::now();
        sum = 0;
        while let Some(Reverse(x)) = h.pop() {
            sum = fold(sum, x);
        }
        pt += t1 - t0;
        qt += t1.elapsed();
    }
    (pt, qt, sum)
}

fn run_own(input: &[u64], reps: usize) -> Timed {
    let mut h = Heap::with_capacity(input.len(), |a: &u64, b: &u64| a < b);
    let (mut pt, mut qt, mut sum) = (Duration::ZERO, Duration::ZERO, 0);
    for _ in 0..reps {
        let t0 = Instant::now();
        for &x in input {
            h.push(x);
        }
        let t1 = Instant::now();
        sum = 0;
        while let Some(x) = h.pop() {
            sum = fold(sum, x);
        }
        pt += t1 - t0;
        qt += t1.elapsed();
    }
    (pt, qt, sum)
}

fn run_dyn(input: &[u64], reps: usize) -> Timed {
    // A trait object, hidden from the optimiser: every `less` is an indirect
    // call through a vtable, like Go's func value.
    let less: &dyn Fn(&u64, &u64) -> bool = black_box(&|a: &u64, b: &u64| a < b);
    let mut h = Heap::with_capacity(input.len(), less);
    let (mut pt, mut qt, mut sum) = (Duration::ZERO, Duration::ZERO, 0);
    for _ in 0..reps {
        let t0 = Instant::now();
        for &x in input {
            h.push(x);
        }
        let t1 = Instant::now();
        sum = 0;
        while let Some(x) = h.pop() {
            sum = fold(sum, x);
        }
        pt += t1 - t0;
        qt += t1.elapsed();
    }
    (pt, qt, sum)
}

fn run_sort(input: &[u64], reps: usize) -> Timed {
    let mut buf = vec![0u64; input.len()];
    let mut qt = Duration::ZERO;
    for _ in 0..reps {
        buf.copy_from_slice(input); // not timed
        let t0 = Instant::now();
        buf.sort_unstable();
        qt += t0.elapsed();
        black_box(&buf);
    }
    (Duration::ZERO, qt, buf.iter().fold(0, |s, &x| fold(s, x)))
}

fn main() {
    let mut r = Pcg::new(1, 2);

    // (a) Allocations per Push and per Pop: reserved capacity, a VARYING value on
    // every call. Rust has no `any` to box into, so the ints cost the same.
    const RUNS: usize = 1000;
    println!(
        "(a) allocations per operation (counting allocator, capacity reserved, varying values):"
    );
    println!(
        "  {:<34} {:>10} {:>10}",
        "BinaryHeap, element type", "push", "pop"
    );
    let per = |events: usize| events as f64 / RUNS as f64;
    let row = |name: &str, push: usize, pop: usize| {
        println!("  {:<34} {:>10.2} {:>10.2}", name, per(push), per(pop));
    };
    let ints = |f: fn(u64) -> u64| -> BinaryHeap<u64> { (0..RUNS as u64).map(f).collect() };
    let pop_ints = |mut h: BinaryHeap<u64>| allocs_of(|| while h.pop().is_some() {});
    let mut h = BinaryHeap::with_capacity(RUNS);
    let push = allocs_of(|| (0..RUNS as u64).for_each(|i| h.push(small(i))));
    row("u64 below 256", push, pop_ints(ints(small)));
    let mut h = BinaryHeap::with_capacity(RUNS);
    let push = allocs_of(|| (0..RUNS as u64).for_each(|i| h.push(large(i))));
    row("u64, 256 or more", push, pop_ints(ints(large)));
    let mut h = BinaryHeap::with_capacity(RUNS);
    let push = allocs_of(|| {
        for i in 0..RUNS as u64 {
            h.push(Item {
                pri: 1000 + i,
                id: i,
            });
        }
    });
    let mut full: BinaryHeap<Item> = (0..RUNS as u64).map(|i| Item { pri: i, id: i }).collect();
    let pop = allocs_of(|| while full.pop().is_some() {});
    row("Item{pri, id}, 16 bytes", push, pop);
    let boxes: Vec<Box<Item>> = (0..RUNS as u64)
        .map(|i| Box::new(Item { pri: i, id: i }))
        .collect();
    let mut h = BinaryHeap::with_capacity(RUNS);
    let push = allocs_of(|| boxes.into_iter().for_each(|b| h.push(b)));
    let pop = allocs_of(|| while h.pop().is_some() {});
    row("Box<Item>, already allocated", push, pop);
    let mut h = BinaryHeap::with_capacity(RUNS);
    let push = allocs_of(|| {
        for i in 0..RUNS as u64 {
            h.push(Box::new(Item { pri: i, id: i }));
        }
    });
    let pop = allocs_of(|| while h.pop().is_some() {});
    row("Box<Item>, Box::new at the call", push, pop);
    let mut h = Heap::with_capacity(RUNS, |a: &u64, b: &u64| a < b);
    let push = allocs_of(|| (0..RUNS as u64).for_each(|i| h.push(large(i))));
    let pop = allocs_of(|| while h.pop().is_some() {});
    row("own Heap<u64, F>, 256 or more", push, pop);
    let grow = allocs_of(|| {
        let mut h = BinaryHeap::new();
        for i in 0..RUNS as u64 {
            h.push(small(i));
        }
    });
    println!(
        "  no reservation: 1,000 pushes of u64 below 256 into BinaryHeap::new() make {grow} allocation events (Vec growth only)"
    );

    // (b) Comparisons per operation, n = 2^16 random keys.
    const N: usize = 1 << 16;
    let keys: Vec<u64> = (0..N).map(|_| r.int_n(1 << 30)).collect();
    let cmps = Cell::new(0usize);
    let mut own = Heap::with_capacity(N, |a: &u64, b: &u64| {
        cmps.set(cmps.get() + 1);
        a < b
    });
    for &x in &keys {
        own.push(x);
    }
    let own_push = cmps.replace(0);
    let mut order1 = Vec::with_capacity(N);
    while let Some(x) = own.pop() {
        order1.push(x);
    }
    let own_pop = cmps.get();
    CMPS.set(0);
    let mut std_heap = BinaryHeap::with_capacity(N);
    for &x in &keys {
        std_heap.push(Reverse(Cnt(x)));
    }
    let std_push = CMPS.replace(0);
    let mut order2 = Vec::with_capacity(N);
    while let Some(Reverse(Cnt(x))) = std_heap.pop() {
        order2.push(x);
    }
    let std_pop = CMPS.get();
    println!("\n(b) comparisons per operation for n = {N} random keys (push n, then pop n):");
    println!("  {:<24} {:>8} {:>8}", "", "push", "pop");
    let nf = N as f64;
    println!(
        "  {:<24} {:>8.2} {:>8.2}",
        "own Heap<u64, F>",
        own_push as f64 / nf,
        own_pop as f64 / nf
    );
    println!(
        "  {:<24} {:>8.2} {:>8.2}",
        "std BinaryHeap",
        std_push as f64 / nf,
        std_pop as f64 / nf
    );
    println!(
        "  popped orders identical and sorted: {};  trait objects in either heap: none, so there is no dispatch to count",
        order1 == order2 && order1.is_sorted()
    );

    // (c) One timing table: push n random keys, pop them all.
    println!(
        "\n(c) push n random keys, then pop them all (best of 5 interleaved rounds, 3 at n = 10^6):"
    );
    println!(
        "  {:>8}  {:<30} {:>8} {:>8} {:>9} {:>9}",
        "n", "variant", "push ns", "pop ns", "total ns", "vs own"
    );
    let variants: [(&str, Runner); 4] = [
        ("std BinaryHeap<Reverse<u64>>", run_std),
        ("own Heap, closure (generic)", run_own),
        ("own Heap, &dyn Fn (vtable)", run_dyn),
        ("sort_unstable (baseline)", run_sort),
    ];
    for n in [1_000usize, 100_000, 1_000_000] {
        let input: Vec<u64> = (0..n).map(|_| r.int_n(1_000_000_000)).collect();
        let mut want = input.clone();
        want.sort_unstable();
        let want_sum = want.iter().fold(0, |s, &x| fold(s, x));
        let reps = (200_000 / n).max(1);
        let rounds = if n >= 1_000_000 { 3 } else { 5 };
        let mut bests = [(0.0f64, 0.0f64, f64::INFINITY); 4];
        let mut same = true;
        for _ in 0..rounds {
            for (k, (_, run)) in variants.iter().enumerate() {
                let (pt, qt, sum) = run(&input, reps);
                same &= sum == want_sum;
                let per = (reps * n) as f64;
                let total = (pt + qt).as_nanos() as f64 / per;
                if total < bests[k].2 {
                    bests[k] = (
                        pt.as_nanos() as f64 / per,
                        qt.as_nanos() as f64 / per,
                        total,
                    );
                }
            }
        }
        for (k, (name, _)) in variants.iter().enumerate() {
            let (p, q, total) = bests[k];
            let ratio = total / bests[1].2;
            if k == variants.len() - 1 {
                println!(
                    "  {n:>8}  {name:<30} {:>8} {:>8} {total:>9.1} {ratio:>8.2}x",
                    "-", "-"
                );
            } else {
                println!("  {n:>8}  {name:<30} {p:>8.1} {q:>8.1} {total:>9.1} {ratio:>8.2}x");
            }
        }
        println!(
            "  {:>8}  every variant drained the keys in sorted order (checksum): {same}",
            ""
        );
    }
}
```

**Rust sample output:**

```
(a) allocations per operation (counting allocator, capacity reserved, varying values):
  BinaryHeap, element type                 push        pop
  u64 below 256                            0.00       0.00
  u64, 256 or more                         0.00       0.00
  Item{pri, id}, 16 bytes                  0.00       0.00
  Box<Item>, already allocated             0.00       0.00
  Box<Item>, Box::new at the call          1.00       0.00
  own Heap<u64, F>, 256 or more            0.00       0.00
  no reservation: 1,000 pushes of u64 below 256 into BinaryHeap::new() make 9 allocation events (Vec growth only)

(b) comparisons per operation for n = 65536 random keys (push n, then pop n):
                               push      pop
  own Heap<u64, F>             2.28    27.03
  std BinaryHeap               2.28    14.70
  popped orders identical and sorted: true;  trait objects in either heap: none, so there is no dispatch to count

(c) push n random keys, then pop them all (best of 5 interleaved rounds, 3 at n = 10^6):
         n  variant                         push ns   pop ns  total ns    vs own
      1000  std BinaryHeap<Reverse<u64>>        1.7     10.4      12.1     1.08x
      1000  own Heap, closure (generic)         2.0      9.3      11.3     1.00x
      1000  own Heap, &dyn Fn (vtable)          3.0     21.7      24.7     2.19x
      1000  sort_unstable (baseline)              -        -       4.3     0.39x
            every variant drained the keys in sorted order (checksum): true
    100000  std BinaryHeap<Reverse<u64>>        7.0     28.6      35.5     0.96x
    100000  own Heap, closure (generic)         7.5     29.6      37.1     1.00x
    100000  own Heap, &dyn Fn (vtable)          8.7     50.7      59.5     1.60x
    100000  sort_unstable (baseline)              -        -       8.2     0.22x
            every variant drained the keys in sorted order (checksum): true
   1000000  std BinaryHeap<Reverse<u64>>        7.8     45.1      52.8     0.92x
   1000000  own Heap, closure (generic)         7.9     49.7      57.5     1.00x
   1000000  own Heap, &dyn Fn (vtable)          9.3     93.1     102.4     1.78x
   1000000  sort_unstable (baseline)              -        -      10.0     0.17x
            every variant drained the keys in sorted order (checksum): true
```

> **Go vs Rust:** No `any` and no boxing: every `BinaryHeap<T>` and own-`Heap` row allocates 0 times per push and pop (Go: 1 for an int of 256 or more and for a 16-byte `Item`), the only allocation is the one you ask for with `Box::new` at the call, and the own heap makes exactly Go's 2.28 and 27.03 comparisons while std's `pop` makes 14.70 (`sift_down_to_bottom`, read in the source; example 15 counts it). The timings do not carry over from Go: across six runs std's heap stayed within 0.9-1.1x of the own heap, an indirect `less` through `&dyn Fn` cost about 1.6-1.8x at n of 10^5 and up (Go's func-value heap: about 1.5x the inline one), and `sort_unstable` took about 0.15-0.25x of the own heap's time at those sizes where Go's `slices.Sort` took about 0.66-0.73x of the specialised heap's.

**Complexity:** Θ(log n) per operation in every variant · allocations (deterministic): `heap.Push` and `heap.Pop` allocate once for an int of 256 or more and for a 16-byte `Item`, not at all for an int below 256 or a pointer (a `&Item{}` made at the call costs one, in the `Push`), and the generic heap allocates nothing · calls at n = 2^16: 5.57 interface-method calls per push and 43.40 per pop, with exactly the generic heap's `Less` counts (2.28 and 27.03) · timing (three runs here): `container/heap` took about 1.3x the generic heap with a `less` func at n = 10^5 and 10^6 (about 2.1x at n = 1,000) and 2.0-2.1x the specialised heap at those two sizes, and `slices.Sort` took 0.66-0.73x the specialised heap's time at every n

---

## 15. Pop with fewer comparisons: bottom-up and d-ary

`🔴 hard` · *Bottom-up sift / d-ary heap*

The element a pop moves to the root came from the bottom of the heap, so it usually sinks right back to near the bottom, and the textbook sift wastes a comparison per level checking that it has not arrived yet. **Bottom-up** pop (the sift Rust's `BinaryHeap::pop` uses, `sift_down_to_bottom`) walks the smaller-child path to a leaf with **one** comparison per level and then climbs back up a short way, about `⌊lg n⌋` comparisons per pop against the textbook's `2⌊lg n⌋`; a **d-ary** heap has `log_d n` levels but scans up to d children on each, which trades more comparisons for fewer levels and fewer moves. Heapsort stays Θ(n log n) either way, so the question is the constant, and the right tool is to count first (comparisons and element moves are deterministic) and only then time (lesson 13 example 6 counted the textbook form inside heapsort; this one re-counts it with the cheaper sift).

**Steps:**

1. Write one counted d-ary min-heap with a textbook sift (d = 2 is `container/heap`'s `down`), a bottom-up sift for d = 2, Floyd's heapify, and `Push`; carry the sinking element in a hole so a level costs one move.
2. Heapify n = 2^10, 2^14, 2^18 distinct keys and pop them all with each variant, count comparisons and moves per element, sweep d = 2, 3, 4, 8, 16, and check every output against `slices.Sort` and the arrays against each other.
3. Total the comparisons of a 100,000-element heapsort, then time the three pops (and `container/heap`) at n = 2^10, 2^16, 2^22 on counter-free copies of the same loops.

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"sort"
	"time"
)

// Heap is a hand-written d-ary min-heap of ints with two counters: cmps (one per
// element comparison) and moves (one per write into the array). Every sift here
// carries the sinking element x in a "hole" instead of swapping at each level, so
// a level costs ONE write; the comparison counts equal the swap version's (example 2).
type Heap struct {
	d           int
	a           []int
	cmps, moves int
}

func (h *Heap) less(x, y int) bool { h.cmps++; return x < y }
func (h *Heap) reset()             { h.cmps, h.moves = 0, 0 }

// minChild: the smallest of the (up to d) children of i among a[:n]. d-1
// comparisons at most; the leftmost wins ties. The caller knows i has a child.
func (h *Heap) minChild(i, n int) int {
	c := h.d*i + 1
	best := c
	for k := c + 1; k < min(c+h.d, n); k++ {
		if h.less(h.a[k], h.a[best]) {
			best = k
		}
	}
	return best
}

// siftDown is the TEXTBOOK sift: at every level find the smallest child (d-1
// comparisons), then compare it with x (1 more); stop when x is not larger.
// For d = 2 that is container/heap's down: two comparisons per level.
func (h *Heap) siftDown(i, n, x int) {
	for h.d*i+1 < n {
		c := h.minChild(i, n)
		if !h.less(h.a[c], x) {
			break
		}
		h.a[i] = h.a[c]
		h.moves++
		i = c
	}
	h.a[i] = x
	h.moves++
}

// siftDownBottomUp is Rust std's sift_down_to_bottom (binary only). The hole
// walks down the smaller-child path with ONE comparison per level (child vs
// child, never against x) all the way to a leaf; then x climbs from there while
// it is smaller than its parent. A leaf's only child costs no comparison at all.
func (h *Heap) siftDownBottomUp(n, x int) {
	i := 0
	for 2*i+2 < n { // both children exist
		c := 2*i + 1
		if h.less(h.a[c+1], h.a[c]) {
			c++
		}
		h.a[i] = h.a[c]
		h.moves++
		i = c
	}
	if 2*i+1 < n { // exactly one child, a leaf
		h.a[i] = h.a[2*i+1]
		h.moves++
		i = 2*i + 1
	}
	for i > 0 { // x was the LAST element: it usually belongs near here
		p := (i - 1) / 2
		if !h.less(x, h.a[p]) {
			break
		}
		h.a[i] = h.a[p]
		h.moves++
		i = p
	}
	h.a[i] = x
	h.moves++
}

// heapify is Floyd's: sift down every parent, last one first.
func (h *Heap) heapify() {
	n := len(h.a)
	if n < 2 {
		return
	}
	for i := (n - 2) / h.d; i >= 0; i-- {
		h.siftDown(i, n, h.a[i])
	}
}

// pop takes the root, moves the LAST element out of the array and lets it find
// its place from the empty root with the chosen sift (bottom-up is binary only).
func (h *Heap) pop(bottomUp bool) int {
	top, x, n := h.a[0], h.a[len(h.a)-1], len(h.a)-1
	h.a = h.a[:n]
	if n > 0 && bottomUp {
		h.siftDownBottomUp(n, x)
	} else if n > 0 {
		h.siftDown(0, n, x)
	}
	return top
}

// stdHeap is container/heap's Interface on ints, with a counter in Less.
type stdHeap struct {
	sort.IntSlice
	cmps int
}

func (h *stdHeap) Less(i, j int) bool { h.cmps++; return h.IntSlice[i] < h.IntSlice[j] }
func (h *stdHeap) Push(x any)         { h.IntSlice = append(h.IntSlice, x.(int)) }
func (h *stdHeap) Pop() any {
	n := len(h.IntSlice) - 1
	x := h.IntSlice[n]
	h.IntSlice = h.IntSlice[:n]
	return x
}

// run builds a heap from keys, then pops everything; it returns the heapify and
// pop-all comparison counts, the pop-all moves, and the popped order.
type run struct {
	heapifyCmps, cmps, moves int
	out                      []int
}

func popAll(d int, bottomUp bool, keys []int) run {
	h := &Heap{d: d, a: slices.Clone(keys)}
	h.heapify()
	r := run{heapifyCmps: h.cmps, out: make([]int, 0, len(keys))}
	h.reset()
	for len(h.a) > 0 {
		r.out = append(r.out, h.pop(bottomUp))
	}
	r.cmps, r.moves = h.cmps, h.moves
	return r
}

func popAllStd(keys []int) run {
	h := &stdHeap{IntSlice: slices.Clone(keys)}
	heap.Init(h)
	r := run{heapifyCmps: h.cmps, out: make([]int, 0, len(keys))}
	h.cmps = 0
	for h.Len() > 0 {
		r.out = append(r.out, heap.Pop(h).(int))
	}
	r.cmps = h.cmps
	return r
}

func permutation(r *rand.Rand, n int) []int {
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

// The same three pop-all loops with the counters removed, for timing. They are
// heapsort-style and in place: the minimum goes into the slot the heap just
// freed, so a ends up sorted DESCENDING.
func fastTextbook(a []int) {
	for n := len(a) - 1; n > 0; n-- {
		top, x, i := a[0], a[n], 0
		for 2*i+1 < n {
			c := 2*i + 1
			if c+1 < n && a[c+1] < a[c] {
				c++
			}
			if a[c] >= x {
				break
			}
			a[i] = a[c]
			i = c
		}
		a[i], a[n] = x, top
	}
}

func fastBottomUp(a []int) {
	for n := len(a) - 1; n > 0; n-- {
		top, x, i := a[0], a[n], 0
		for 2*i+2 < n {
			c := 2*i + 1
			if a[c+1] < a[c] {
				c++
			}
			a[i] = a[c]
			i = c
		}
		if 2*i+1 < n {
			a[i] = a[2*i+1]
			i = 2*i + 1
		}
		for i > 0 && x < a[(i-1)/2] {
			a[i] = a[(i-1)/2]
			i = (i - 1) / 2
		}
		a[i], a[n] = x, top
	}
}

func fastFourAry(a []int) {
	for n := len(a) - 1; n > 0; n-- {
		top, x, i := a[0], a[n], 0
		for 4*i+1 < n {
			best := 4*i + 1
			for k := best + 1; k < min(4*i+5, n); k++ {
				if a[k] < a[best] {
					best = k
				}
			}
			if a[best] >= x {
				break
			}
			a[i] = a[best]
			i = best
		}
		a[i], a[n] = x, top
	}
}

// the counter in Less stays: one add next to an interface call
func fastStd(a []int) {
	h := &stdHeap{IntSlice: a}
	for h.Len() > 0 {
		heap.Pop(h) // the popped value is boxed into an any: that is part of the cost
	}
}

// sortsAll: do the three own pops and container/heap all equal slices.Sort?
func sortsAll(keys []int) bool {
	want := slices.Sorted(slices.Values(keys))
	return slices.Equal(popAll(2, false, keys).out, want) && slices.Equal(popAll(2, true, keys).out, want) &&
		slices.Equal(popAll(4, false, keys).out, want) && slices.Equal(popAllStd(keys).out, want)
}

// sameLayout: after one pop (or after every pop, if lockstep), do the textbook
// and the bottom-up sift leave the very same array?
func sameLayout(keys []int, lockstep bool) bool {
	a := &Heap{d: 2, a: slices.Clone(keys)}
	a.heapify()
	b := &Heap{d: 2, a: slices.Clone(a.a)}
	for len(a.a) > 0 {
		a.pop(false)
		b.pop(true)
		if !slices.Equal(a.a, b.a) {
			return false
		}
		if !lockstep {
			break
		}
	}
	return true
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	sizes := []int{1 << 10, 1 << 14, 1 << 18}
	perms := make([][]int, len(sizes))

	// (a) Heapify, then pop all n, on one random permutation of distinct keys per n.
	fmt.Println("(a) heapify, then pop all n (counts per element; distinct keys):")
	for k, n := range sizes {
		perms[k] = permutation(r, n)
		lg := bits.Len(uint(n)) - 1
		fmt.Printf("  n = %d: lg n = %d, 2 lg n = %d\n", n, lg, 2*lg)
		fmt.Printf("    %-15s %9s %9s %9s\n", "variant", "heapify", "cmp/pop", "move/pop")
		text, bottom, four, std := popAll(2, false, perms[k]), popAll(2, true, perms[k]), popAll(4, false, perms[k]), popAllStd(perms[k])
		per := func(x int) float64 { return float64(x) / float64(n) }
		for _, row := range []struct {
			name string
			r    run
		}{{"textbook", text}, {"bottom-up", bottom}, {"4-ary textbook", four}} {
			fmt.Printf("    %-15s %9.2f %9.2f %9.2f\n", row.name, per(row.r.heapifyCmps), per(row.r.cmps), per(row.r.moves))
		}
		fmt.Printf("    %-15s %9.2f %9.2f %9s\n", "container/heap", per(std.heapifyCmps), per(std.cmps), "-")
		fmt.Printf("    container/heap == own textbook (heapify and pop-all comparisons): %v\n",
			std.heapifyCmps == text.heapifyCmps && std.cmps == text.cmps)
		fmt.Printf("    bottom-up makes %.2f of the textbook's pop comparisons, 4-ary makes %.2f of them and %.2f of its moves\n",
			per(bottom.cmps)/per(text.cmps), per(four.cmps)/per(text.cmps), per(four.moves)/per(text.moves))
	}

	// (b) d-ary: fewer levels, more children to scan on each. Same permutation, n = 2^18.
	fmt.Printf("\n(b) d-ary heaps, textbook sift, n = %d (counts per element):\n", sizes[2])
	fmt.Printf("  %3s %6s %7s %9s %9s %9s\n", "d", "levels", "d/lg d", "heapify", "cmp/pop", "move/pop")
	for _, d := range []int{2, 3, 4, 8, 16} {
		levels := 0
		for i := sizes[2] - 1; i > 0; i = (i - 1) / d {
			levels++
		}
		ru, n := popAll(d, false, perms[2]), float64(sizes[2])
		fmt.Printf("  %3d %6d %7.2f %9.2f %9.2f %9.2f\n", d, levels, float64(d)/math.Log2(float64(d)),
			float64(ru.heapifyCmps)/n, float64(ru.cmps)/n, float64(ru.moves)/n)
	}

	// (c) Every variant sorts: against slices.Sort, on permutations and on heavy ties.
	fmt.Println("\n(c) pop order equals slices.Sort (textbook, bottom-up, 4-ary and container/heap):")
	for k, n := range sizes {
		fmt.Printf("  n = %6d, distinct keys: all four equal: %v\n", n, sortsAll(perms[k]))
	}
	bad := 0
	for range 2000 { // sizes 0..59, keys IntN(10)
		keys := make([]int, r.IntN(60))
		for i := range keys {
			keys[i] = r.IntN(10)
		}
		if !sortsAll(keys) {
			bad++
		}
	}
	fmt.Printf("  2000 random arrays with keys IntN(10), sizes 0..59: mismatches: %d\n", bad)

	// (d) Heapsort = heapify + n pops, n = 100,000.
	hn := 100000
	hkeys := permutation(r, hn)
	nlgn := float64(hn) * math.Log2(float64(hn))
	fmt.Printf("\n(d) heapsort of n = %d distinct keys = heapify + n pops (n lg n = %.0f):\n", hn, nlgn)
	fmt.Printf("  %-15s %10s %10s %10s %9s\n", "variant", "heapify", "pops", "total", "/ n lg n")
	for _, v := range []struct {
		name     string
		d        int
		bottomUp bool
	}{{"textbook", 2, false}, {"bottom-up", 2, true}, {"4-ary textbook", 4, false}} {
		ru := popAll(v.d, v.bottomUp, hkeys)
		total := ru.heapifyCmps + ru.cmps
		fmt.Printf("  %-15s %10d %10d %10d %9.2f\n", v.name, ru.heapifyCmps, ru.cmps, total, float64(total)/nlgn)
	}

	// (e) Textbook and bottom-up both put x on the smaller-child path, so with
	// distinct keys they leave identical arrays, not just equal output.
	fmt.Println("\n(e) layouts after the pop, textbook vs bottom-up:")
	for _, tc := range []struct {
		label    string
		trials   int
		size     int // 0: random in 2..201
		ties     bool
		lockstep bool
	}{
		{"one pop, distinct keys, sizes 2..201", 10000, 0, false, false},
		{"one pop, ties (keys IntN(size/4)), sizes 2..201", 10000, 0, true, false},
		{"every pop of a pop-all, distinct keys, size 500", 200, 500, false, true},
	} {
		identical := 0
		for range tc.trials {
			size := tc.size
			if size == 0 {
				size = 2 + r.IntN(200)
			}
			keys := permutation(r, size)
			for i := range keys {
				if tc.ties {
					keys[i] = r.IntN(max(2, size/4))
				}
			}
			if sameLayout(keys, tc.lockstep) {
				identical++
			}
		}
		fmt.Printf("  %-50s identical arrays %d/%d\n", tc.label, identical, tc.trials)
	}
	fmt.Println("  (swapping two equal ints changes nothing, so ties cannot show in an array of plain ints)")

	// (f) Timing. The counter-free loops above, each on a fresh copy of the same
	// heapified array; "copy only" is the control for that copy. Rounds are
	// interleaved and a cell is the lowest-highest ns per pop over 3 rounds.
	fmt.Println("\n(f) ns per pop (copy + pop all, over n; lowest-highest of 3 rounds):")
	fmt.Printf("  %8s %11s %11s %11s %11s %15s\n", "n", "copy only", "textbook", "bottom-up", "4-ary", "container/heap")
	sorted := true
	var ratios []string
	for _, n := range []int{1 << 10, 1 << 16, 1 << 22} {
		keys := permutation(r, n)
		bin, quad := &Heap{d: 2, a: slices.Clone(keys)}, &Heap{d: 4, a: slices.Clone(keys)}
		bin.heapify()
		quad.heapify()
		cs := []struct {
			pristine []int
			run      func([]int)
		}{{bin.a, func([]int) {}}, {bin.a, fastTextbook}, {bin.a, fastBottomUp}, {quad.a, fastFourAry}, {bin.a, fastStd}}
		work := make([]int, n)
		for _, c := range cs[1:4] { // the fast loops really sort (descending: minimum last)
			copy(work, c.pristine)
			c.run(work)
			sorted = sorted && slices.IsSortedFunc(work, func(a, b int) int { return b - a })
		}
		reps := max(1, (1<<20)/n)
		lo, hi := make([]float64, len(cs)), make([]float64, len(cs))
		for k := range lo {
			lo[k] = math.Inf(1)
		}
		for range 3 {
			for k, c := range cs {
				t0 := time.Now()
				for range reps {
					copy(work, c.pristine)
					c.run(work)
				}
				ns := float64(time.Since(t0).Nanoseconds()) / float64(reps*n)
				lo[k], hi[k] = min(lo[k], ns), max(hi[k], ns)
			}
		}
		cell := func(k int) string { return fmt.Sprintf("%.1f-%.1f", lo[k], hi[k]) }
		fmt.Printf("  %8d %11s %11s %11s %11s %15s\n", n, cell(0), cell(1), cell(2), cell(3), cell(4))
		ratios = append(ratios, fmt.Sprintf("  n = %7d: bottom-up %.2f, 4-ary %.2f, container/heap %.2f", n, lo[2]/lo[1], lo[3]/lo[1], lo[4]/lo[1]))
	}
	fmt.Println("lowest time over the rounds, as a multiple of the textbook's:")
	for _, line := range ratios {
		fmt.Println(line)
	}
	fmt.Println("  the counter-free loops sort correctly:", sorted)
}
```

**Sample output:**

```
(a) heapify, then pop all n (counts per element; distinct keys):
  n = 1024: lg n = 10, 2 lg n = 20
    variant           heapify   cmp/pop  move/pop
    textbook             1.86     15.04      8.36
    bottom-up            1.86      8.71      8.72
    4-ary textbook       1.32     16.36      5.00
    container/heap       1.86     15.04         -
    container/heap == own textbook (heapify and pop-all comparisons): true
    bottom-up makes 0.58 of the textbook's pop comparisons, 4-ary makes 1.09 of them and 0.60 of its moves
  n = 16384: lg n = 14, 2 lg n = 28
    variant           heapify   cmp/pop  move/pop
    textbook             1.88     23.04     12.36
    bottom-up            1.88     12.71     12.71
    4-ary textbook       1.32     24.35      7.00
    container/heap       1.88     23.04         -
    container/heap == own textbook (heapify and pop-all comparisons): true
    bottom-up makes 0.55 of the textbook's pop comparisons, 4-ary makes 1.06 of them and 0.57 of its moves
  n = 262144: lg n = 18, 2 lg n = 36
    variant           heapify   cmp/pop  move/pop
    textbook             1.88     31.03     16.36
    bottom-up            1.88     16.71     16.71
    4-ary textbook       1.32     32.36      9.00
    container/heap       1.88     31.03         -
    container/heap == own textbook (heapify and pop-all comparisons): true
    bottom-up makes 0.54 of the textbook's pop comparisons, 4-ary makes 1.04 of them and 0.55 of its moves

(b) d-ary heaps, textbook sift, n = 262144 (counts per element):
    d levels  d/lg d   heapify   cmp/pop  move/pop
    2     18    2.00      1.88     31.03     16.36
    3     11    1.89      1.47     30.11     10.93
    4      9    2.00      1.32     32.36      9.00
    8      6    2.67      1.14     44.24      6.48
   16      5    4.00      1.07     69.05      5.29

(c) pop order equals slices.Sort (textbook, bottom-up, 4-ary and container/heap):
  n =   1024, distinct keys: all four equal: true
  n =  16384, distinct keys: all four equal: true
  n = 262144, distinct keys: all four equal: true
  2000 random arrays with keys IntN(10), sizes 0..59: mismatches: 0

(d) heapsort of n = 100000 distinct keys = heapify + n pops (n lg n = 1660964):
  variant            heapify       pops      total  / n lg n
  textbook            187977    2831812    3019789      1.82
  bottom-up           187977    1534662    1722639      1.04
  4-ary textbook      131903    2945680    3077583      1.85

(e) layouts after the pop, textbook vs bottom-up:
  one pop, distinct keys, sizes 2..201               identical arrays 10000/10000
  one pop, ties (keys IntN(size/4)), sizes 2..201    identical arrays 10000/10000
  every pop of a pop-all, distinct keys, size 500    identical arrays 200/200
  (swapping two equal ints changes nothing, so ties cannot show in an array of plain ints)

(f) ns per pop (copy + pop all, over n; lowest-highest of 3 rounds):
         n   copy only    textbook   bottom-up       4-ary  container/heap
      1024     0.1-0.1     9.5-9.6     9.5-9.7   15.4-15.7       40.3-40.9
     65536     0.1-0.1   51.5-53.5   50.8-51.2   57.4-58.4     115.6-116.7
   4194304     0.2-0.2   94.5-95.4   93.1-93.9 105.6-110.2     194.4-199.9
lowest time over the rounds, as a multiple of the textbook's:
  n =    1024: bottom-up 1.00, 4-ary 1.62, container/heap 4.23
  n =   65536: bottom-up 0.99, 4-ary 1.12, container/heap 2.25
  n = 4194304: bottom-up 0.99, 4-ary 1.12, container/heap 2.06
  the counter-free loops sort correctly: true
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;
use std::hint::black_box;
use std::time::Instant;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same permutations as Go.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

/// Heap is a hand-written d-ary min-heap of u64 with two counters: cmps (one per
/// element comparison) and moves (one per write into the array). Every sift here
/// carries the sinking element x in a "hole" instead of swapping at each level, so
/// a level costs ONE write; the comparison counts equal the swap version's (example 2).
struct Heap {
    d: usize,
    a: Vec<u64>,
    cmps: u64,
    moves: u64,
}

impl Heap {
    fn new(d: usize, a: Vec<u64>) -> Self {
        Heap {
            d,
            a,
            cmps: 0,
            moves: 0,
        }
    }

    fn less(&mut self, x: u64, y: u64) -> bool {
        self.cmps += 1;
        x < y
    }

    fn reset(&mut self) {
        self.cmps = 0;
        self.moves = 0;
    }

    /// min_child: the smallest of the (up to d) children of i among a[..n]. d-1
    /// comparisons at most; the leftmost wins ties. The caller knows i has a child.
    fn min_child(&mut self, i: usize, n: usize) -> usize {
        let c = self.d * i + 1;
        let mut best = c;
        for k in c + 1..(c + self.d).min(n) {
            if self.less(self.a[k], self.a[best]) {
                best = k;
            }
        }
        best
    }

    /// sift_down is the TEXTBOOK sift: at every level find the smallest child (d-1
    /// comparisons), then compare it with x (1 more); stop when x is not larger.
    /// For d = 2 that is container/heap's down: two comparisons per level.
    fn sift_down(&mut self, mut i: usize, n: usize, x: u64) {
        while self.d * i + 1 < n {
            let c = self.min_child(i, n);
            if !self.less(self.a[c], x) {
                break;
            }
            self.a[i] = self.a[c];
            self.moves += 1;
            i = c;
        }
        self.a[i] = x;
        self.moves += 1;
    }

    /// sift_down_bottom_up is Rust std's sift_down_to_bottom (binary only). The
    /// hole walks down the smaller-child path with ONE comparison per level (child
    /// vs child, never against x) all the way to a leaf; then x climbs from there
    /// while it is smaller than its parent. A leaf's only child costs no comparison.
    fn sift_down_bottom_up(&mut self, n: usize, x: u64) {
        let mut i = 0;
        while 2 * i + 2 < n {
            // both children exist
            let mut c = 2 * i + 1;
            if self.less(self.a[c + 1], self.a[c]) {
                c += 1;
            }
            self.a[i] = self.a[c];
            self.moves += 1;
            i = c;
        }
        if 2 * i + 1 < n {
            // exactly one child, a leaf
            self.a[i] = self.a[2 * i + 1];
            self.moves += 1;
            i = 2 * i + 1;
        }
        while i > 0 {
            // x was the LAST element: it usually belongs near here
            let p = (i - 1) / 2;
            if !self.less(x, self.a[p]) {
                break;
            }
            self.a[i] = self.a[p];
            self.moves += 1;
            i = p;
        }
        self.a[i] = x;
        self.moves += 1;
    }

    /// heapify is Floyd's: sift down every parent, last one first.
    fn heapify(&mut self) {
        let n = self.a.len();
        if n < 2 {
            return;
        }
        for i in (0..=(n - 2) / self.d).rev() {
            let x = self.a[i];
            self.sift_down(i, n, x);
        }
    }

    /// pop takes the root, moves the LAST element out of the array and lets it
    /// find its place from the empty root with the chosen sift (bottom-up is
    /// binary only). None on an empty heap (Go's version panics there).
    fn pop(&mut self, bottom_up: bool) -> Option<u64> {
        let x = self.a.pop()?;
        if self.a.is_empty() {
            return Some(x);
        }
        let top = self.a[0];
        let n = self.a.len();
        if bottom_up {
            self.sift_down_bottom_up(n, x);
        } else {
            self.sift_down(0, n, x);
        }
        Some(top)
    }
}

// Std's BinaryHeap is a MAX-heap and counts nothing, so the key is a newtype
// whose `cmp` is REVERSED (std's maximum is the smallest number) and bumps a
// thread-local counter. Std's `<=`, `>=` and `<` each call `partial_cmp` once,
// and that calls `cmp` once, so the counter equals the number of comparisons.
thread_local!(static STD_CMPS: Cell<u64> = const { Cell::new(0) });

fn take_std_cmps() -> u64 {
    STD_CMPS.with(|c| c.replace(0))
}

#[derive(Clone, Copy, PartialEq, Eq)]
struct Key(u64);

impl Ord for Key {
    fn cmp(&self, other: &Self) -> Ordering {
        STD_CMPS.with(|c| c.set(c.get() + 1));
        other.0.cmp(&self.0)
    }
}

impl PartialOrd for Key {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// Run: heapify and pop-all comparison counts, pop-all moves, and the popped order.
struct Run {
    heapify_cmps: u64,
    cmps: u64,
    moves: u64,
    out: Vec<u64>,
}

fn pop_all(d: usize, bottom_up: bool, keys: &[u64]) -> Run {
    let mut h = Heap::new(d, keys.to_vec());
    h.heapify();
    let heapify_cmps = h.cmps;
    h.reset();
    let mut out = Vec::with_capacity(keys.len());
    while let Some(x) = h.pop(bottom_up) {
        out.push(x);
    }
    Run {
        heapify_cmps,
        cmps: h.cmps,
        moves: h.moves,
        out,
    }
}

/// std_run: BinaryHeap::from(vec) (heapify), then either n `pop()`s or one
/// `into_sorted_vec()`. Moves are not counted by std, so 0.
fn std_run(into_sorted: bool, keys: &[u64]) -> Run {
    take_std_cmps();
    let mut h = BinaryHeap::from(keys.iter().map(|&k| Key(k)).collect::<Vec<_>>());
    let heapify_cmps = take_std_cmps();
    let out: Vec<u64> = if into_sorted {
        // ascending in Key order is descending as numbers
        h.into_sorted_vec().into_iter().rev().map(|k| k.0).collect()
    } else {
        let mut out = Vec::with_capacity(keys.len());
        while let Some(k) = h.pop() {
            out.push(k.0);
        }
        out
    };
    Run {
        heapify_cmps,
        cmps: take_std_cmps(),
        moves: 0,
        out,
    }
}

fn permutation(r: &mut Pcg, n: usize) -> Vec<u64> {
    let mut p: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

// The same three pop-all loops with the counters removed, for timing. They are
// heapsort-style and in place: the minimum goes into the slot the heap just
// freed, so a ends up sorted DESCENDING.
fn fast_textbook(a: &mut [u64]) {
    for n in (1..a.len()).rev() {
        let (top, x) = (a[0], a[n]);
        let mut i = 0;
        while 2 * i + 1 < n {
            let mut c = 2 * i + 1;
            if c + 1 < n && a[c + 1] < a[c] {
                c += 1;
            }
            if a[c] >= x {
                break;
            }
            a[i] = a[c];
            i = c;
        }
        a[i] = x;
        a[n] = top;
    }
}

fn fast_bottom_up(a: &mut [u64]) {
    for n in (1..a.len()).rev() {
        let (top, x) = (a[0], a[n]);
        let mut i = 0;
        while 2 * i + 2 < n {
            let mut c = 2 * i + 1;
            if a[c + 1] < a[c] {
                c += 1;
            }
            a[i] = a[c];
            i = c;
        }
        if 2 * i + 1 < n {
            a[i] = a[2 * i + 1];
            i = 2 * i + 1;
        }
        while i > 0 && x < a[(i - 1) / 2] {
            a[i] = a[(i - 1) / 2];
            i = (i - 1) / 2;
        }
        a[i] = x;
        a[n] = top;
    }
}

fn fast_four_ary(a: &mut [u64]) {
    for n in (1..a.len()).rev() {
        let (top, x) = (a[0], a[n]);
        let mut i = 0;
        while 4 * i + 1 < n {
            let mut best = 4 * i + 1;
            for k in best + 1..(4 * i + 5).min(n) {
                if a[k] < a[best] {
                    best = k;
                }
            }
            if a[best] >= x {
                break;
            }
            a[i] = a[best];
            i = best;
        }
        a[i] = x;
        a[n] = top;
    }
}

/// A counter-free pop-all loop, for timing.
type FastPop = fn(&mut [u64]);

/// sorts_all: do the three own pops and std's pop and into_sorted_vec all equal
/// a sorted copy?
fn sorts_all(keys: &[u64]) -> bool {
    let mut want = keys.to_vec();
    want.sort_unstable();
    pop_all(2, false, keys).out == want
        && pop_all(2, true, keys).out == want
        && pop_all(4, false, keys).out == want
        && std_run(false, keys).out == want
        && std_run(true, keys).out == want
}

/// same_layout: after one pop (or after every pop, if lockstep), do the textbook
/// and the bottom-up sift leave the very same array?
fn same_layout(keys: &[u64], lockstep: bool) -> bool {
    let mut a = Heap::new(2, keys.to_vec());
    a.heapify();
    let mut b = Heap::new(2, a.a.clone());
    while !a.a.is_empty() {
        a.pop(false);
        b.pop(true);
        if a.a != b.a {
            return false;
        }
        if !lockstep {
            break;
        }
    }
    true
}

fn main() {
    let mut r = Pcg::new(1, 2);
    let sizes = [1usize << 10, 1 << 14, 1 << 18];
    let mut perms: Vec<Vec<u64>> = Vec::new();

    // (a) Heapify, then pop all n, on one random permutation of distinct keys per n.
    println!("(a) heapify, then pop all n (counts per element; distinct keys):");
    for &n in &sizes {
        perms.push(permutation(&mut r, n));
        let keys = &perms[perms.len() - 1];
        let lg = n.ilog2();
        println!("  n = {n}: lg n = {lg}, 2 lg n = {}", 2 * lg);
        println!(
            "    {:<19} {:>9} {:>9} {:>9}",
            "variant", "heapify", "cmp/pop", "move/pop"
        );
        let text = pop_all(2, false, keys);
        let bottom = pop_all(2, true, keys);
        let four = pop_all(4, false, keys);
        let std_pop = std_run(false, keys);
        let std_sorted = std_run(true, keys);
        let per = |x: u64| x as f64 / n as f64;
        for (name, run) in [
            ("textbook", &text),
            ("bottom-up", &bottom),
            ("4-ary textbook", &four),
        ] {
            println!(
                "    {:<19} {:>9.2} {:>9.2} {:>9.2}",
                name,
                per(run.heapify_cmps),
                per(run.cmps),
                per(run.moves)
            );
        }
        for (name, run) in [
            ("std pop()", &std_pop),
            ("std into_sorted_vec", &std_sorted),
        ] {
            println!(
                "    {:<19} {:>9.2} {:>9.2} {:>9}",
                name,
                per(run.heapify_cmps),
                per(run.cmps),
                "-"
            );
        }
        println!(
            "    std from(vec) == own heapify: {}",
            std_pop.heapify_cmps == text.heapify_cmps
        );
        println!(
            "    std pop-all == own bottom-up (comparisons): {}",
            std_pop.cmps == bottom.cmps
        );
        println!(
            "    std into_sorted_vec == own textbook (comparisons): {}",
            std_sorted.cmps == text.cmps
        );
        println!(
            "    bottom-up makes {:.2} of the textbook's pop comparisons, 4-ary makes {:.2} of them and {:.2} of its moves",
            per(bottom.cmps) / per(text.cmps),
            per(four.cmps) / per(text.cmps),
            per(four.moves) / per(text.moves)
        );
    }

    // (b) d-ary: fewer levels, more children to scan on each. Same permutation, n = 2^18.
    println!(
        "\n(b) d-ary heaps, textbook sift, n = {} (counts per element):",
        sizes[2]
    );
    println!(
        "  {:>3} {:>6} {:>7} {:>9} {:>9} {:>9}",
        "d", "levels", "d/lg d", "heapify", "cmp/pop", "move/pop"
    );
    for d in [2usize, 3, 4, 8, 16] {
        let mut levels = 0;
        let mut i = sizes[2] - 1;
        while i > 0 {
            i = (i - 1) / d;
            levels += 1;
        }
        let run = pop_all(d, false, &perms[2]);
        let n = sizes[2] as f64;
        println!(
            "  {:>3} {:>6} {:>7.2} {:>9.2} {:>9.2} {:>9.2}",
            d,
            levels,
            d as f64 / (d as f64).log2(),
            run.heapify_cmps as f64 / n,
            run.cmps as f64 / n,
            run.moves as f64 / n
        );
    }

    // (c) Every variant sorts: against a sorted copy, on permutations and on heavy ties.
    println!(
        "\n(c) pop order equals a sorted copy (textbook, bottom-up, 4-ary, std pop and into_sorted_vec):"
    );
    for (k, &n) in sizes.iter().enumerate() {
        println!(
            "  n = {n:>6}, distinct keys: all five equal: {}",
            sorts_all(&perms[k])
        );
    }
    let mut bad = 0;
    for _ in 0..2000 {
        // sizes 0..59, keys IntN(10)
        let len = r.int_n(60) as usize;
        let keys: Vec<u64> = (0..len).map(|_| r.int_n(10)).collect();
        if !sorts_all(&keys) {
            bad += 1;
        }
    }
    println!("  2000 random arrays with keys IntN(10), sizes 0..59: mismatches: {bad}");

    // (d) Heapsort = heapify + n pops, n = 100,000.
    let hn = 100_000usize;
    let hkeys = permutation(&mut r, hn);
    let nlgn = hn as f64 * (hn as f64).log2();
    println!("\n(d) heapsort of n = {hn} distinct keys = heapify + n pops (n lg n = {nlgn:.0}):");
    println!(
        "  {:<15} {:>10} {:>10} {:>10} {:>9}",
        "variant", "heapify", "pops", "total", "/ n lg n"
    );
    for (name, d, bottom_up) in [
        ("textbook", 2, false),
        ("bottom-up", 2, true),
        ("4-ary textbook", 4, false),
    ] {
        let run = pop_all(d, bottom_up, &hkeys);
        let total = run.heapify_cmps + run.cmps;
        println!(
            "  {:<15} {:>10} {:>10} {:>10} {:>9.2}",
            name,
            run.heapify_cmps,
            run.cmps,
            total,
            total as f64 / nlgn
        );
    }

    // (e) Textbook and bottom-up both put x on the smaller-child path, so with
    // distinct keys they leave identical arrays, not just equal output.
    println!("\n(e) layouts after the pop, textbook vs bottom-up:");
    // (label, trials, size (0: random in 2..201), ties, lockstep)
    let cases = [
        (
            "one pop, distinct keys, sizes 2..201",
            10000,
            0,
            false,
            false,
        ),
        (
            "one pop, ties (keys IntN(size/4)), sizes 2..201",
            10000,
            0,
            true,
            false,
        ),
        (
            "every pop of a pop-all, distinct keys, size 500",
            200,
            500,
            false,
            true,
        ),
    ];
    for (label, trials, size, ties, lockstep) in cases {
        let mut identical = 0;
        for _ in 0..trials {
            let size = if size == 0 {
                2 + r.int_n(200) as usize
            } else {
                size
            };
            let mut keys = permutation(&mut r, size);
            if ties {
                for key in keys.iter_mut() {
                    *key = r.int_n((size as u64 / 4).max(2));
                }
            }
            if same_layout(&keys, lockstep) {
                identical += 1;
            }
        }
        println!("  {label:<50} identical arrays {identical}/{trials}");
    }
    println!(
        "  (swapping two equal ints changes nothing, so ties cannot show in an array of plain ints)"
    );

    // (f) Timing. The counter-free loops above, each on a fresh copy of the same
    // heapified array; "copy only" is the control for that copy. Std's heap is
    // consumed by its pop-all, so its rows start from a clone (allocation plus
    // memcpy), and "clone only" is the control for that. Rounds are interleaved
    // and a cell is the lowest-highest ns per pop over 3 rounds.
    println!("\n(f) ns per pop (copy + pop all, over n; lowest-highest of 3 rounds):");
    println!(
        "  {:>8} {:>10} {:>10} {:>10} {:>10} {:>10} {:>10} {:>10}",
        "n", "copy only", "textbook", "bottom-up", "4-ary", "clone only", "std pop", "std sorted"
    );
    let mut sorted_ok = true;
    let mut ratios = Vec::new();
    for n in [1usize << 10, 1 << 16, 1 << 22] {
        let keys = permutation(&mut r, n);
        let mut bin = Heap::new(2, keys.clone());
        bin.heapify();
        let mut quad = Heap::new(4, keys);
        quad.heapify();
        let base: BinaryHeap<Reverse<u64>> =
            BinaryHeap::from(bin.a.iter().map(|&x| Reverse(x)).collect::<Vec<_>>());
        let mut work = vec![0u64; n];
        let fast: [(&[u64], FastPop); 3] = [
            (&bin.a, fast_textbook),
            (&bin.a, fast_bottom_up),
            (&quad.a, fast_four_ary),
        ];
        for (pristine, f) in fast {
            // the fast loops really sort (descending: minimum last)
            work.copy_from_slice(pristine);
            f(&mut work);
            sorted_ok &= work.windows(2).all(|w| w[0] >= w[1]);
        }
        let mut popped = base.clone();
        let mut prev = 0;
        while let Some(Reverse(x)) = popped.pop() {
            sorted_ok &= x >= prev;
            prev = x;
        }
        sorted_ok &= base
            .clone()
            .into_sorted_vec()
            .windows(2)
            .all(|w| w[0].0 >= w[1].0);

        let reps = ((1usize << 20) / n).max(1);
        let ns_per_pop = |t0: Instant| t0.elapsed().as_nanos() as f64 / (reps * n) as f64;
        let mut lo = [f64::INFINITY; 7];
        let mut hi = [0f64; 7];
        for _ in 0..3 {
            for k in 0..7 {
                let t0 = Instant::now();
                for _ in 0..reps {
                    match k {
                        0 => work.copy_from_slice(&bin.a),
                        1..=3 => {
                            let (pristine, f) = fast[k - 1];
                            work.copy_from_slice(pristine);
                            f(&mut work);
                        }
                        4 => {
                            black_box(black_box(&base).clone());
                        }
                        5 => {
                            let mut h = black_box(&base).clone();
                            let mut acc = 0;
                            while let Some(Reverse(x)) = h.pop() {
                                acc ^= x;
                            }
                            black_box(acc);
                        }
                        _ => {
                            black_box(black_box(&base).clone().into_sorted_vec());
                        }
                    }
                    black_box(&work);
                }
                let ns = ns_per_pop(t0);
                lo[k] = lo[k].min(ns);
                hi[k] = hi[k].max(ns);
            }
        }
        let cell = |k: usize| format!("{:.1}-{:.1}", lo[k], hi[k]);
        println!(
            "  {:>8} {:>10} {:>10} {:>10} {:>10} {:>10} {:>10} {:>10}",
            n,
            cell(0),
            cell(1),
            cell(2),
            cell(3),
            cell(4),
            cell(5),
            cell(6)
        );
        ratios.push(format!(
            "  n = {n:>7}: bottom-up {:.2}, 4-ary {:.2}, std pop {:.2}, std into_sorted_vec {:.2}",
            lo[2] / lo[1],
            lo[3] / lo[1],
            lo[5] / lo[1],
            lo[6] / lo[1]
        ));
    }
    println!("lowest time over the rounds, as a multiple of the textbook's:");
    for line in ratios {
        println!("{line}");
    }
    println!("  the counter-free loops and std's pops sort correctly: {sorted_ok}");
}
```

**Rust sample output:**

```
(a) heapify, then pop all n (counts per element; distinct keys):
  n = 1024: lg n = 10, 2 lg n = 20
    variant               heapify   cmp/pop  move/pop
    textbook                 1.86     15.04      8.36
    bottom-up                1.86      8.71      8.72
    4-ary textbook           1.32     16.36      5.00
    std pop()                1.86      8.71         -
    std into_sorted_vec      1.86     15.04         -
    std from(vec) == own heapify: true
    std pop-all == own bottom-up (comparisons): true
    std into_sorted_vec == own textbook (comparisons): true
    bottom-up makes 0.58 of the textbook's pop comparisons, 4-ary makes 1.09 of them and 0.60 of its moves
  n = 16384: lg n = 14, 2 lg n = 28
    variant               heapify   cmp/pop  move/pop
    textbook                 1.88     23.04     12.36
    bottom-up                1.88     12.71     12.71
    4-ary textbook           1.32     24.35      7.00
    std pop()                1.88     12.71         -
    std into_sorted_vec      1.88     23.04         -
    std from(vec) == own heapify: true
    std pop-all == own bottom-up (comparisons): true
    std into_sorted_vec == own textbook (comparisons): true
    bottom-up makes 0.55 of the textbook's pop comparisons, 4-ary makes 1.06 of them and 0.57 of its moves
  n = 262144: lg n = 18, 2 lg n = 36
    variant               heapify   cmp/pop  move/pop
    textbook                 1.88     31.03     16.36
    bottom-up                1.88     16.71     16.71
    4-ary textbook           1.32     32.36      9.00
    std pop()                1.88     16.71         -
    std into_sorted_vec      1.88     31.03         -
    std from(vec) == own heapify: true
    std pop-all == own bottom-up (comparisons): true
    std into_sorted_vec == own textbook (comparisons): true
    bottom-up makes 0.54 of the textbook's pop comparisons, 4-ary makes 1.04 of them and 0.55 of its moves

(b) d-ary heaps, textbook sift, n = 262144 (counts per element):
    d levels  d/lg d   heapify   cmp/pop  move/pop
    2     18    2.00      1.88     31.03     16.36
    3     11    1.89      1.47     30.11     10.93
    4      9    2.00      1.32     32.36      9.00
    8      6    2.67      1.14     44.24      6.48
   16      5    4.00      1.07     69.05      5.29

(c) pop order equals a sorted copy (textbook, bottom-up, 4-ary, std pop and into_sorted_vec):
  n =   1024, distinct keys: all five equal: true
  n =  16384, distinct keys: all five equal: true
  n = 262144, distinct keys: all five equal: true
  2000 random arrays with keys IntN(10), sizes 0..59: mismatches: 0

(d) heapsort of n = 100000 distinct keys = heapify + n pops (n lg n = 1660964):
  variant            heapify       pops      total  / n lg n
  textbook            187977    2831812    3019789      1.82
  bottom-up           187977    1534662    1722639      1.04
  4-ary textbook      131903    2945680    3077583      1.85

(e) layouts after the pop, textbook vs bottom-up:
  one pop, distinct keys, sizes 2..201               identical arrays 10000/10000
  one pop, ties (keys IntN(size/4)), sizes 2..201    identical arrays 10000/10000
  every pop of a pop-all, distinct keys, size 500    identical arrays 200/200
  (swapping two equal ints changes nothing, so ties cannot show in an array of plain ints)

(f) ns per pop (copy + pop all, over n; lowest-highest of 3 rounds):
         n  copy only   textbook  bottom-up      4-ary clone only    std pop std sorted
      1024    0.1-0.1    8.4-8.4    8.0-8.0  15.5-15.6    0.1-0.1    9.7-9.8    9.2-9.3
     65536    0.1-0.1  25.8-26.1  24.9-25.1  34.0-34.1    0.1-0.2  26.1-26.4  25.1-25.4
   4194304    0.2-0.2  75.7-79.7  71.5-80.9  73.0-78.6    0.6-0.8  71.0-78.3  75.5-80.1
lowest time over the rounds, as a multiple of the textbook's:
  n =    1024: bottom-up 0.96, 4-ary 1.85, std pop 1.16, std into_sorted_vec 1.11
  n =   65536: bottom-up 0.96, 4-ary 1.32, std pop 1.01, std into_sorted_vec 0.97
  n = 4194304: bottom-up 0.94, 4-ary 0.96, std pop 0.94, std into_sorted_vec 1.00
  the counter-free loops and std's pops sort correctly: true
```

> **Go vs Rust:** Every count of the own heaps is identical to the Go output, and the std lines are the finding: Go's `container/heap` made exactly the textbook counts (15.04 / 23.04 / 31.03 per pop), but `BinaryHeap::pop` made exactly the bottom-up counts (8.71 / 12.71 / 16.71), `into_sorted_vec` the textbook ones, and `BinaryHeap::from(vec)` the own heapify count, all equal to the own heap's on the same permutation. That matches the source (read in `alloc/src/collections/binary_heap/mod.rs`, not run): `pop` calls `sift_down_to_bottom`, `into_sorted_vec` and `from` call the textbook `sift_down_range`. The counts come from a thread-local counter in a reversed `Ord` newtype, because std's heap is a max-heap and counts nothing. Timings differ from Go (six runs on a shared machine; the 2^22 row flips order between runs, so read only 2^10 and 2^16): bottom-up was 4-8% faster than the textbook (Go: 0-3%), 4-ary was 1.3-1.9x slower (Go: 1.1-1.7x slower at all three sizes), and halving the comparisons was no clear win inside std either: its bottom-up `pop` was 3-7% slower than its textbook `into_sorted_vec`. At 2^22 every Rust row sat within about 15% of the textbook, 4-ary included, and which one was ahead changed from run to run.

**Complexity:** textbook pop ≤ `2⌊lg n⌋` comparisons, measured 15.04 / 23.04 / 31.03 per pop at n = 2^10 / 2^14 / 2^18 and identical to `container/heap`'s · bottom-up 8.71 / 12.71 / 16.71, which is 0.54–0.58 of that (`lg n − 1.29` at all three sizes), leaves the same array after every pop on distinct keys, and sorts 100,000 elements in 1.04 `n lg n` comparisons against the textbook's 1.82 · 4-ary: 9 levels instead of 18 and 0.55–0.60 of the moves, but 1.04–1.09 times the comparisons (a textbook d-ary pop costs about `(d / lg d) lg n`, so d = 16 makes 69.05 per pop) · timing, ranges over three runs: bottom-up 0.97–1.00 of the textbook's time at all three sizes despite about 0.55 of its comparisons, 4-ary 1.1–1.7 times slower, `container/heap` 2.0–4.3 times slower, so "fewer comparisons wins" did not hold in Go (plausible mechanism, not measured: the child-versus-child comparison, the hard one to predict, is still made once per level in both)

---

## 16. Capstone: `PQ[T]` with handles, built, proved, measured

`🔴 hard` · *Indexed priority queue*

The structure this lesson built, packaged as one generic type: `Push` returns a **handle**, and `Update` and `Remove` take it, so changing or cancelling any element is Θ(log n) with no search and no array indices in the caller's hands (`container/heap` makes you store the index inside your element and keep it right in `Swap`, and Rust's `BinaryHeap` cannot reach below the top at all). The heap array of `{value, slot}` cells is paired with `pos[slot]`, the element's current index, which every swap repairs; freed slots go on a free list, and a per-slot generation makes a handle from a popped or removed element **stale** even after its slot holds someone else. That is a lot of bookkeeping to trust, so it is proved three ways: against an O(n) model on 100,000 random operations over a narrow priority range (ties everywhere), by a checker that re-verifies heap order and the `pos` and free-list consistency after every single operation, and by counting its comparisons against `container/heap` on the same input before timing it. `Push`, `Pop`, `Update`, `Remove` are Θ(log n), `Peek` is Θ(1), `FromSlice` is Θ(n).

**Steps:**

1. Write `PQ[T]` (heap of `{val, slot}`, `pos`, `gen`, free list, sifts copied from `container/heap` with `swap` repairing `pos`) and a `check()` that verifies every invariant without counting comparisons.
2. Drive the queue and an O(n) slice model with the same random Push/Pop/Update/Remove/Peek stream, trying a stale handle after every operation and comparing everything after every operation; then walk the stale-handle cases and `FromSlice` by hand.
3. Compare comparison counts with `container/heap` and `Update` with `Remove + Push`, then time push-n-pop-n and a steady-state workload with 20% `Update` at n = 10^3, 10^5, 10^6.

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
	"math/rand/v2"
	"runtime"
	"slices"
	"sort"
	"time"
)

// Handle names ONE element for its whole stay in the queue. It is plain data
// (the caller keeps it anywhere); slot says where the queue keeps the element's
// bookkeeping and gen says which occupant of that slot this handle was made for.
// gen starts at 1, so the zero Handle is never valid.
type Handle struct {
	slot int
	gen  uint32
}

type entry[T any] struct {
	val  T
	slot int // which slot's pos[] entry points back at this heap cell
}

// PQ is a binary min-heap (by less) with handles, so Update and Remove need no
// search: pos[slot] is the element's current index in the heap array, kept right
// by swap. Slots are recycled through a free list; gen[slot] counts how many
// elements have left the slot, so a handle from a popped or removed element
// is STALE even after the slot holds somebody else.
type PQ[T any] struct {
	less func(a, b T) bool
	heap []entry[T] // the heap, in array order
	pos  []int      // pos[slot]: index into heap, or -1 while the slot is free
	gen  []uint32   // gen[slot]: generation of the slot's current (or next) occupant
	free []int      // free slots, used as a stack
	Cmps int        // element comparisons made by sifts so far
}

func New[T any](less func(a, b T) bool) *PQ[T] { return &PQ[T]{less: less} }

// FromSlice builds a queue from xs in O(n) (Floyd) and returns xs[i]'s handle at [i].
func FromSlice[T any](xs []T, less func(a, b T) bool) (*PQ[T], []Handle) {
	q, n := New(less), len(xs)
	q.heap, q.pos, q.gen = make([]entry[T], n), make([]int, n), make([]uint32, n)
	hs := make([]Handle, n)
	for i, x := range xs {
		q.heap[i], q.pos[i], q.gen[i], hs[i] = entry[T]{x, i}, i, 1, Handle{i, 1}
	}
	for i := n/2 - 1; i >= 0; i-- {
		q.down(i, n)
	}
	return q, hs
}

func (q *PQ[T]) Len() int { return len(q.heap) }

func (q *PQ[T]) Push(x T) Handle {
	var s int
	if k := len(q.free); k > 0 {
		s, q.free = q.free[k-1], q.free[:k-1]
	} else {
		s = len(q.pos)
		q.pos, q.gen = append(q.pos, 0), append(q.gen, 1)
	}
	q.heap = append(q.heap, entry[T]{x, s})
	q.pos[s] = len(q.heap) - 1
	q.up(len(q.heap) - 1)
	return Handle{s, q.gen[s]}
}

func (q *PQ[T]) Peek() (T, bool) {
	if len(q.heap) == 0 {
		var zero T
		return zero, false
	}
	return q.heap[0].val, true
}

func (q *PQ[T]) Pop() (T, bool) {
	if len(q.heap) == 0 {
		var zero T
		return zero, false
	}
	return q.removeAt(0), true
}

// index turns a handle into a heap index, or reports that it is stale or forged.
func (q *PQ[T]) index(h Handle) (int, bool) {
	if h.slot < 0 || h.slot >= len(q.pos) || q.gen[h.slot] != h.gen || q.pos[h.slot] < 0 {
		return 0, false
	}
	return q.pos[h.slot], true
}

func (q *PQ[T]) Get(h Handle) (T, bool) {
	i, ok := q.index(h)
	if !ok {
		var zero T
		return zero, false
	}
	return q.heap[i].val, true
}

// Update replaces the element's value and moves it up OR down, Theta(log n).
// It is container/heap's Fix, found through the handle instead of a stored index.
func (q *PQ[T]) Update(h Handle, x T) bool {
	i, ok := q.index(h)
	if !ok {
		return false
	}
	q.heap[i].val = x
	if !q.down(i, len(q.heap)) {
		q.up(i)
	}
	return true
}

func (q *PQ[T]) Remove(h Handle) (T, bool) {
	i, ok := q.index(h)
	if !ok {
		var zero T
		return zero, false
	}
	return q.removeAt(i), true
}

// removeAt is container/heap's Remove: swap the cell with the last, shrink, repair
// the moved element in whichever direction it needs, then retire the slot.
func (q *PQ[T]) removeAt(i int) T {
	n := len(q.heap) - 1
	if n != i {
		q.swap(i, n)
		if !q.down(i, n) {
			q.up(i)
		}
	}
	e := q.heap[n]
	q.heap[n] = entry[T]{} // let the GC have a pointer-bearing T
	q.heap = q.heap[:n]
	q.pos[e.slot] = -1
	q.gen[e.slot]++ // every handle made for this occupant is stale from here on
	q.free = append(q.free, e.slot)
	return e.val
}

func (q *PQ[T]) lessAt(i, j int) bool {
	q.Cmps++
	return q.less(q.heap[i].val, q.heap[j].val)
}

// swap exchanges two heap cells AND repairs both pos entries: the whole price of handles.
func (q *PQ[T]) swap(i, j int) {
	q.heap[i], q.heap[j] = q.heap[j], q.heap[i]
	q.pos[q.heap[i].slot], q.pos[q.heap[j].slot] = i, j
}

// up and down are container/heap's, loop for loop (strict <, left child on ties).
func (q *PQ[T]) up(j int) {
	for {
		i := (j - 1) / 2 // parent; at the root (0-1)/2 == 0 == j in Go
		if i == j || !q.lessAt(j, i) {
			break
		}
		q.swap(i, j)
		j = i
	}
}

func (q *PQ[T]) down(i0, n int) bool {
	i := i0
	for {
		j := 2*i + 1
		if j >= n {
			break
		}
		if j2 := j + 1; j2 < n && q.lessAt(j2, j) {
			j = j2
		}
		if !q.lessAt(j, i) {
			break
		}
		q.swap(i, j)
		i = j
	}
	return i > i0
}

// check verifies every structural invariant and returns "" or the first
// violation. It calls q.less directly, so it never touches Cmps.
func (q *PQ[T]) check() string {
	for i := 1; i < len(q.heap); i++ {
		if q.less(q.heap[i].val, q.heap[(i-1)/2].val) {
			return fmt.Sprintf("heap order broken at index %d", i)
		}
	}
	for i, e := range q.heap {
		if e.slot < 0 || e.slot >= len(q.pos) || q.pos[e.slot] != i {
			return fmt.Sprintf("pos does not point back at heap index %d", i)
		}
	}
	live := 0
	for _, p := range q.pos {
		if p >= 0 {
			live++
		}
	}
	if live != len(q.heap) {
		return fmt.Sprintf("%d live slots for %d heap cells", live, len(q.heap))
	}
	seen := make([]bool, len(q.pos))
	for _, s := range q.free {
		if s < 0 || s >= len(q.pos) || q.pos[s] >= 0 || seen[s] {
			return fmt.Sprintf("free list entry %d is out of range, live or repeated", s)
		}
		seen[s] = true
	}
	if len(q.free)+len(q.heap) != len(q.pos) {
		return "a slot is neither live nor free"
	}
	return ""
}

// ---- the oracle: a slice of live (id, priority) pairs, scanned in O(n) ----

type Item struct{ id, prio int }

func byPrio(a, b Item) bool { return a.prio < b.prio }

type liveItem struct {
	Item
	h Handle
}

type model struct {
	live  []liveItem
	dead  [64]Handle // ring of the most recently dead handles
	deadN int
}

func (m *model) minPrio() int {
	lo := m.live[0].prio
	for _, e := range m.live[1:] {
		lo = min(lo, e.prio)
	}
	return lo
}

func (m *model) find(id int) int {
	return slices.IndexFunc(m.live, func(e liveItem) bool { return e.id == id })
}

// kill removes live[i] from the model and remembers its handle as dead.
func (m *model) kill(i int) {
	m.dead[m.deadN%len(m.dead)] = m.live[i].h
	m.deadN++
	m.live[i] = m.live[len(m.live)-1]
	m.live = m.live[:len(m.live)-1]
}

type stats struct {
	push, pop, update, remove, peek, skipped int
	maxLive, slots, reused                   int
	staleTries, staleOnReused                int
	mismatches, violations, staleErrors      int
	first                                    string
}

// runModel drives a PQ[Item] and the model with the same random operations and
// compares them after EVERY operation. start > 0 begins from FromSlice(start items).
func runModel(seed uint64, ops, start int) stats {
	r := rand.New(rand.NewPCG(seed, 2))
	var st stats
	var m model
	q := New(byPrio)
	nextID := 0
	if start > 0 {
		xs := make([]Item, start)
		for i := range xs {
			xs[i] = Item{i, r.IntN(20)}
		}
		var hs []Handle
		q, hs = FromSlice(xs, byPrio)
		for i, x := range xs {
			m.live = append(m.live, liveItem{x, hs[i]})
		}
		nextID = start
	}
	bad := func(msg string) {
		st.mismatches++
		if st.first == "" {
			st.first = msg
		}
	}
	for range ops {
		switch x := r.IntN(100); {
		case x < 35: // Push
			it := Item{nextID, r.IntN(20)}
			nextID++
			before := len(q.pos)
			h := q.Push(it)
			if h.slot < before {
				st.reused++
			}
			m.live = append(m.live, liveItem{it, h})
			st.push++
		case x < 55: // Pop: must be a live element holding the minimum priority
			got, ok := q.Pop()
			st.pop++
			if len(m.live) == 0 {
				if ok {
					bad("Pop on an empty queue returned an element")
				}
			} else if i := m.find(got.id); !ok || i < 0 || got != m.live[i].Item || got.prio != m.minPrio() {
				bad(fmt.Sprintf("Pop returned %v, ok=%v, model minimum %d", got, ok, m.minPrio()))
			} else {
				m.kill(i)
			}
		case x < 75 && len(m.live) > 0: // Update a random live handle, priority up or down
			i, np := r.IntN(len(m.live)), r.IntN(20)
			if !q.Update(m.live[i].h, Item{m.live[i].id, np}) {
				bad("Update through a live handle returned false")
			}
			m.live[i].prio = np
			st.update++
		case x < 90 && len(m.live) > 0: // Remove a random live handle
			i := r.IntN(len(m.live))
			if got, ok := q.Remove(m.live[i].h); !ok || got != m.live[i].Item {
				bad(fmt.Sprintf("Remove returned %v, ok=%v, wanted %v", got, ok, m.live[i].Item))
			}
			m.kill(i)
			st.remove++
		case x >= 90: // Peek: no state change
			got, ok := q.Peek()
			if i := m.find(got.id); len(m.live) == 0 && ok || len(m.live) > 0 && (!ok || i < 0 || got != m.live[i].Item) {
				bad("Peek returned an element the model does not hold")
			}
			st.peek++
		default: // an Update or Remove with nothing live to aim at
			st.skipped++
		}

		// A stale handle, tried through all three entry points: nothing may happen.
		if n := min(m.deadN, len(m.dead)); n > 0 {
			d := m.dead[r.IntN(n)]
			st.staleTries++
			if d.slot < len(q.pos) && q.pos[d.slot] >= 0 {
				st.staleOnReused++ // the slot holds ANOTHER element now: only gen protects it
			}
			_, g := q.Get(d)
			u := q.Update(d, Item{-1, -1})
			_, rm := q.Remove(d)
			if g || u || rm {
				st.staleErrors++
			}
		}

		// Invariants and the oracle's verdict on the whole queue, after every operation.
		if msg := q.check(); msg != "" {
			st.violations++
			if st.first == "" {
				st.first = msg
			}
		}
		if q.Len() != len(m.live) {
			bad("Len disagrees with the model")
		}
		if top, ok := q.Peek(); len(m.live) > 0 && (!ok || top.prio != m.minPrio()) {
			bad("Peek is not the model minimum")
		}
		st.maxLive = max(st.maxLive, len(m.live))
	}
	st.slots = len(q.pos)
	return st
}

// ---- helpers for the count and timing sections ----

func permutation(r *rand.Rand, n int) []int {
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

func intLess(a, b int) bool { return a < b }

// stdHeap is container/heap's Interface on ints; cmps counts Less calls (a plain
// run for timing just ignores the field).
type stdHeap struct {
	sort.IntSlice
	cmps int
}

func (h *stdHeap) Less(i, j int) bool { h.cmps++; return h.IntSlice[i] < h.IntSlice[j] }
func (h *stdHeap) Push(x any)         { h.IntSlice = append(h.IntSlice, x.(int)) }
func (h *stdHeap) Pop() any {
	n := len(h.IntSlice) - 1
	x := h.IntSlice[n]
	h.IntSlice = h.IntSlice[:n]
	return x
}

// hItem and hq are the container/heap idiom for updates, as in the package's own
// example: the item stores its index, Swap keeps it right, heap.Fix repairs it.
type hItem struct{ id, key, index int }
type hq []*hItem

func (h hq) Len() int           { return len(h) }
func (h hq) Less(i, j int) bool { return h[i].key < h[j].key }
func (h hq) Swap(i, j int)      { h[i], h[j] = h[j], h[i]; h[i].index, h[j].index = i, j }
func (h *hq) Push(x any)        { it := x.(*hItem); it.index = len(*h); *h = append(*h, it) }
func (h *hq) Pop() any {
	old := *h
	it := old[len(old)-1]
	old[len(old)-1] = nil
	*h = old[:len(old)-1]
	return it
}

type op struct {
	upd     bool // else: pop the minimum and push it back with a new key
	id, key int
}

// timeAll runs every contender (an untimed setup that returns the timed work)
// for 3 interleaved rounds and returns the lowest and highest ns per operation.
func timeAll(cs []func() func(), ops int) (lo, hi []float64) {
	lo, hi = make([]float64, len(cs)), make([]float64, len(cs))
	for i := range lo {
		lo[i] = math.Inf(1)
	}
	for range 3 {
		for k, c := range cs {
			run := c()
			runtime.GC()
			t0 := time.Now()
			run()
			ns := float64(time.Since(t0).Nanoseconds()) / float64(ops)
			lo[k], hi[k] = min(lo[k], ns), max(hi[k], ns)
		}
	}
	return lo, hi
}

func main() {
	// (a) + (b): the model oracle with invariants checked after every operation.
	fmt.Println("(a) PQ[Item] against an O(n) model, (b) every invariant checked after EVERY operation;")
	fmt.Println("    mix: Push 35%, Pop 20%, Update 20%, Remove 15%, Peek 10%; priorities IntN(20), so ties are everywhere")
	fmt.Printf("  %4s %6s %7s | %6s %6s %6s %6s %6s %6s | %8s %6s %7s\n",
		"seed", "start", "ops", "push", "pop", "update", "remove", "peek", "skip", "max live", "slots", "reused")
	var runs []stats
	for _, c := range []struct {
		seed       uint64
		ops, start int
	}{{1, 100000, 0}, {2, 100000, 0}, {3, 100000, 0}, {4, 100000, 0}, {5, 100000, 0}, {6, 20000, 4000}} {
		st := runModel(c.seed, c.ops, c.start)
		runs = append(runs, st)
		fmt.Printf("  %4d %6d %7d | %6d %6d %6d %6d %6d %6d | %8d %6d %7d\n", c.seed, c.start, c.ops,
			st.push, st.pop, st.update, st.remove, st.peek, st.skipped, st.maxLive, st.slots, st.reused)
	}
	fmt.Printf("\n  %4s | %11s %14s | %10s %10s %12s\n", "seed", "stale tries", "on reused slot", "mismatches", "violations", "stale errors")
	var tot stats
	for i, st := range runs {
		fmt.Printf("  %4d | %11d %14d | %10d %10d %12d\n", i+1, st.staleTries, st.staleOnReused, st.mismatches, st.violations, st.staleErrors)
		tot.mismatches += st.mismatches
		tot.violations += st.violations
		tot.staleErrors += st.staleErrors
		if st.first != "" {
			fmt.Println("  first failure:", st.first)
		}
	}
	fmt.Printf("  mismatches: %d  invariant violations: %d  stale-handle errors: %d\n", tot.mismatches, tot.violations, tot.staleErrors)

	// (c) Stale handles, step by step.
	fmt.Println("\n(c) stale handles, directed (Get / Update / Remove must all say false):")
	q := New(intLess)
	h5, h3, h9 := q.Push(5), q.Push(3), q.Push(9)
	staleBad := 0
	try := func(label string, h Handle) {
		_, g := q.Get(h)
		u := q.Update(h, 4)
		_, rm := q.Remove(h)
		fmt.Printf("  %-44s %v: Get %-5v Update %-5v Remove %-5v\n", label, h, g, u, rm)
		if g || u || rm {
			staleBad++
		}
	}
	v, _ := q.Pop()
	try(fmt.Sprintf("after Pop (returned %d)", v), h3)
	v, _ = q.Remove(h9)
	try(fmt.Sprintf("after Remove (returned %d)", v), h9)
	h7 := q.Push(7)
	fmt.Printf("  Push(7) reuses the freed slot: %v then %v, same slot: %v, new generation: %v\n", h9, h7, h7.slot == h9.slot, h7.gen != h9.gen)
	try("old handle, slot now holds the 7", h9)
	v, ok := q.Get(h7)
	fmt.Printf("  the 7 is untouched: Get -> %d, %v; Len %d\n", v, ok, q.Len())
	try("the zero Handle", Handle{})
	try("a slot that never existed", Handle{99, 1})
	try("a negative slot", Handle{-1, 1})
	q.Update(h5, 1)
	top, _ := q.Peek()
	fmt.Printf("  a live handle still works: Update(h5, 1) puts %d on top; stale-handle errors: %d\n", top, staleBad)

	// (d) FromSlice: O(n) heapify, with a handle per input element.
	r := rand.New(rand.NewPCG(1, 2))
	const n = 10000
	xs := make([]int, n)
	for i := range xs {
		xs[i] = r.IntN(1_000_000)
	}
	fq, hs := FromSlice(xs, intLess)
	pushed := New(intLess)
	for _, x := range xs {
		pushed.Push(x)
	}
	fmt.Printf("\n(d) FromSlice of %d values: %d comparisons = %.2f n (<= 2n: %v); %d Push calls on the same values: %d\n",
		n, fq.Cmps, float64(fq.Cmps)/n, fq.Cmps <= 2*n, n, pushed.Cmps)
	fresh := fq.check() == ""
	same, left := true, []int(nil)
	for i, h := range hs {
		if v, ok := fq.Get(h); !ok || v != xs[i] {
			same = false
		}
	}
	for i := range xs {
		if i%3 == 0 { // remove every third element through its handle
			if v, ok := fq.Remove(hs[i]); !ok || v != xs[i] {
				same = false
			}
		} else {
			left = append(left, xs[i])
		}
	}
	slices.Sort(left)
	afterRemoves := fq.check() == ""
	var popped []int
	for fq.Len() > 0 {
		v, _ := fq.Pop()
		popped = append(popped, v)
	}
	fmt.Printf("    check() after FromSlice: %v, after removing every 3rd element by handle: %v\n", fresh, afterRemoves)
	fmt.Printf("    handle i returns xs[i] and removal by handle returns it: %v; the pops then equal the sorted rest: %v\n",
		same, slices.Equal(popped, left))

	// (e) Counts: the sifts are container/heap's, so the counts must match it exactly.
	cn := 1 << 14
	keys := permutation(r, cn)
	pq, ch := New(intLess), &stdHeap{}
	for _, k := range keys {
		pq.Push(k)
		heap.Push(ch, k)
	}
	pushPQ, pushCH := pq.Cmps, ch.cmps
	pq.Cmps, ch.cmps = 0, 0
	same = true
	for pq.Len() > 0 {
		a, _ := pq.Pop()
		same = same && a == heap.Pop(ch).(int)
	}
	fmt.Printf("\n(e) n = %d random keys, comparisons per operation:\n", cn)
	fmt.Printf("  %-16s %9s %9s\n", "", "per Push", "per Pop")
	fmt.Printf("  %-16s %9.2f %9.2f\n", "PQ[int]", float64(pushPQ)/float64(cn), float64(pq.Cmps)/float64(cn))
	fmt.Printf("  %-16s %9.2f %9.2f\n", "container/heap", float64(pushCH)/float64(cn), float64(ch.cmps)/float64(cn))
	fmt.Printf("  identical counts and pop order: %v\n", pushPQ == pushCH && pq.Cmps == ch.cmps && same)

	type kv struct{ id, key int }
	byKey := func(a, b kv) bool { return a.key < b.key }
	build := func() (*PQ[kv], []Handle) {
		q, hs := New(byKey), make([]Handle, cn)
		for id, k := range keys {
			hs[id] = q.Push(kv{id, k})
		}
		q.Cmps = 0
		return q, hs
	}
	rekey := func(label string, newKey func(old int) int) { // the same new key for the same element, two ways
		qa, ha := build()
		qb, hb := build()
		for range cn {
			id := r.IntN(cn)
			old, _ := qa.Get(ha[id])
			nk := newKey(old.key)
			qa.Update(ha[id], kv{id, nk})
			qb.Remove(hb[id])
			hb[id] = qb.Push(kv{id, nk})
		}
		cmpsA, cmpsB := qa.Cmps, qb.Cmps // read BEFORE draining the queues below
		sameKeys := true
		for qa.Len() > 0 {
			a, _ := qa.Pop()
			b, _ := qb.Pop()
			sameKeys = sameKeys && a.key == b.key
		}
		fmt.Printf("  %d %-34s Update %5.2f comparisons each, Remove + Push %5.2f; same keys come out: %v\n",
			cn, label, float64(cmpsA)/float64(cn), float64(cmpsB)/float64(cn), sameKeys)
	}
	rekey("re-keyings, new key IntN(2n):", func(int) int { return r.IntN(2 * cn) })
	rekey("decrease-keys, new key IntN(old+1):", func(old int) int { return r.IntN(old + 1) })

	// (f) Timing: lowest-highest ns per operation over 3 interleaved rounds.
	fmt.Println("\n(f) ns per operation (lowest-highest of 3 rounds; PQ also bumps its Cmps counter):")
	fmt.Printf("  %9s | %12s %14s | %12s %17s\n", "n", "PQ[int]", "container/heap", "PQ[Item]", "container/heap+Fix")
	for _, n := range []int{1000, 100000, 1000000} {
		keys := make([]int, n)
		for i := range keys {
			keys[i] = r.IntN(1 << 30)
		}
		ops := make([]op, 400000)
		for i := range ops {
			ops[i] = op{r.IntN(10) < 2, r.IntN(n), r.IntN(1 << 30)}
		}
		reps := max(1, 500000/(2*n))
		ppLo, ppHi := timeAll([]func() func(){
			func() func() {
				return func() {
					for range reps {
						q := New(intLess)
						for _, k := range keys {
							q.Push(k)
						}
						for q.Len() > 0 {
							q.Pop()
						}
					}
				}
			},
			func() func() {
				return func() {
					for range reps {
						h := &stdHeap{}
						for _, k := range keys {
							heap.Push(h, k)
						}
						for h.Len() > 0 {
							heap.Pop(h)
						}
					}
				}
			},
		}, 2*n*reps)
		mxLo, mxHi := timeAll([]func() func(){
			func() func() {
				q, hs := New(byKey), make([]Handle, n)
				for id, k := range keys {
					hs[id] = q.Push(kv{id, k})
				}
				return func() {
					for _, o := range ops {
						if o.upd {
							q.Update(hs[o.id], kv{o.id, o.key})
						} else {
							it, _ := q.Pop()
							hs[it.id] = q.Push(kv{it.id, o.key})
						}
					}
				}
			},
			func() func() {
				h, items := &hq{}, make([]*hItem, n)
				for id, k := range keys {
					items[id] = &hItem{id: id, key: k}
					heap.Push(h, items[id])
				}
				return func() {
					for _, o := range ops {
						if o.upd {
							items[o.id].key = o.key
							heap.Fix(h, items[o.id].index)
						} else {
							it := heap.Pop(h).(*hItem)
							it.key = o.key
							heap.Push(h, it)
						}
					}
				}
			},
		}, len(ops))
		cell := func(lo, hi []float64, k int) string { return fmt.Sprintf("%.0f-%.0f", lo[k], hi[k]) }
		fmt.Printf("  %9d | %12s %14s | %12s %17s\n", n, cell(ppLo, ppHi, 0), cell(ppLo, ppHi, 1), cell(mxLo, mxHi, 0), cell(mxLo, mxHi, 1))
	}
}
```

**Sample output:**

```
(a) PQ[Item] against an O(n) model, (b) every invariant checked after EVERY operation;
    mix: Push 35%, Pop 20%, Update 20%, Remove 15%, Peek 10%; priorities IntN(20), so ties are everywhere
  seed  start     ops |   push    pop update remove   peek   skip | max live  slots  reused
     1      0  100000 |  35131  19862  20104  14965   9900     38 |      329    329   34802
     2      0  100000 |  34993  19950  19929  15087   9874    167 |      336    336   34657
     3      0  100000 |  35003  20141  19928  14673  10053    202 |      348    348   34655
     4      0  100000 |  34820  19942  19915  15012   9959    352 |      183    183   34637
     5      0  100000 |  35059  19921  20001  14968  10037     14 |      485    485   34574
     6   4000   20000 |   7078   3885   3994   3044   1999      0 |     4167   4167    6911

  seed | stale tries on reused slot | mismatches violations stale errors
     1 |       99995          79852 |          0          0            0
     2 |       99998          76437 |          0          0            0
     3 |       99995          76511 |          0          0            0
     4 |       99996          73933 |          0          0            0
     5 |       99993          81351 |          0          0            0
     6 |       19997          17146 |          0          0            0
  mismatches: 0  invariant violations: 0  stale-handle errors: 0

(c) stale handles, directed (Get / Update / Remove must all say false):
  after Pop (returned 3)                       {1 1}: Get false Update false Remove false
  after Remove (returned 9)                    {2 1}: Get false Update false Remove false
  Push(7) reuses the freed slot: {2 1} then {2 2}, same slot: true, new generation: true
  old handle, slot now holds the 7             {2 1}: Get false Update false Remove false
  the 7 is untouched: Get -> 7, true; Len 2
  the zero Handle                              {0 0}: Get false Update false Remove false
  a slot that never existed                    {99 1}: Get false Update false Remove false
  a negative slot                              {-1 1}: Get false Update false Remove false
  a live handle still works: Update(h5, 1) puts 1 on top; stale-handle errors: 0

(d) FromSlice of 10000 values: 18793 comparisons = 1.88 n (<= 2n: true); 10000 Push calls on the same values: 22849
    check() after FromSlice: true, after removing every 3rd element by handle: true
    handle i returns xs[i] and removal by handle returns it: true; the pops then equal the sorted rest: true

(e) n = 16384 random keys, comparisons per operation:
                    per Push   per Pop
  PQ[int]               2.28     23.04
  container/heap        2.28     23.04
  identical counts and pop order: true
  16384 re-keyings, new key IntN(2n):      Update  2.88 comparisons each, Remove + Push  3.90; same keys come out: true
  16384 decrease-keys, new key IntN(old+1): Update  2.90 comparisons each, Remove + Push  4.72; same keys come out: true

(f) ns per operation (lowest-highest of 3 rounds; PQ also bumps its Cmps counter):
          n |      PQ[int] container/heap |     PQ[Item] container/heap+Fix
       1000 |        19-21          32-35 |        61-62             63-64
     100000 |        64-65          70-73 |      112-117           132-135
    1000000 |        89-95          88-88 |      227-235           293-324
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;
use std::hint::black_box;
use std::time::Instant;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seeds feed this program the same operation streams as Go.
struct Pcg {
    hi: u64,
    lo: u64,
}

impl Pcg {
    fn new(seed1: u64, seed2: u64) -> Self {
        Pcg {
            hi: seed1,
            lo: seed2,
        }
    }

    fn next_u64(&mut self) -> u64 {
        const MUL: u128 = (2549297995355413924 << 64) | 4865540595714422341;
        const INC: u128 = (6364136223846793005 << 64) | 1442695040888963407;
        // state = state * MUL + INC, with explicit 128-bit wraparound
        let state = ((u128::from(self.hi) << 64) | u128::from(self.lo))
            .wrapping_mul(MUL)
            .wrapping_add(INC);
        self.hi = (state >> 64) as u64;
        self.lo = state as u64;

        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
        assert!(n > 0, "invalid argument to int_n");
        if n & (n - 1) == 0 {
            return self.next_u64() & (n - 1);
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as u64
    }
}

/// Handle names ONE element for its whole stay in the queue. It is plain `Copy`
/// data, so holding one borrows nothing from the queue. `slot` says where the
/// queue keeps the element's bookkeeping and `generation` says which occupant of that
/// slot this handle was made for. `generation` starts at 1, so the zero Handle is never
/// valid. (Go's slot is a signed int and can be -1; here it is an unsigned index.
/// And `gen`, Go's field name, is a reserved keyword in Rust 2024: hence `generation`.)
#[derive(Clone, Copy, PartialEq, Eq)]
struct Handle {
    slot: u32,
    generation: u32,
}

impl Handle {
    /// Go prints a Handle as {1 1}; Rust's derived Debug would print a long form.
    fn show(self) -> String {
        format!("{{{} {}}}", self.slot, self.generation)
    }
}

struct Entry<T> {
    val: T,
    slot: u32, // which slot's pos[] entry points back at this heap cell
}

/// Pq is a binary min-heap (by `less`) with handles, so `update` and `remove`
/// need no search: pos[slot] is the element's current index in the heap array,
/// kept right by `swap`. Slots are recycled through a free list; generation[slot] counts
/// how many elements have left the slot, so a handle from a popped or removed
/// element is STALE even after the slot holds somebody else.
struct Pq<T, F: Fn(&T, &T) -> bool> {
    less: F,
    heap: Vec<Entry<T>>,   // the heap, in array order
    pos: Vec<Option<u32>>, // pos[slot]: index into heap, or None while the slot is free
    generation: Vec<u32>,  // generation[slot]: the slot's current (or next) occupant
    free: Vec<u32>,        // free slots, used as a stack
    cmps: u64,             // element comparisons made by sifts so far
}

impl<T, F: Fn(&T, &T) -> bool> Pq<T, F> {
    fn new(less: F) -> Self {
        Pq {
            less,
            heap: Vec::new(),
            pos: Vec::new(),
            generation: Vec::new(),
            free: Vec::new(),
            cmps: 0,
        }
    }

    /// from_slice builds a queue from `xs` in O(n) (Floyd) and returns xs[i]'s handle at [i].
    fn from_slice(xs: Vec<T>, less: F) -> (Self, Vec<Handle>) {
        let mut q = Pq::new(less);
        let n = xs.len();
        let mut hs = Vec::with_capacity(n);
        for (i, x) in xs.into_iter().enumerate() {
            let slot = i as u32;
            q.heap.push(Entry { val: x, slot });
            q.pos.push(Some(slot));
            q.generation.push(1);
            hs.push(Handle {
                slot,
                generation: 1,
            });
        }
        for i in (0..n / 2).rev() {
            q.down(i, n);
        }
        (q, hs)
    }

    fn len(&self) -> usize {
        self.heap.len()
    }

    fn push(&mut self, x: T) -> Handle {
        let s = match self.free.pop() {
            Some(s) => s,
            None => {
                self.pos.push(None);
                self.generation.push(1);
                (self.pos.len() - 1) as u32
            }
        };
        self.heap.push(Entry { val: x, slot: s });
        let at = self.heap.len() - 1;
        self.pos[s as usize] = Some(at as u32);
        self.up(at);
        Handle {
            slot: s,
            generation: self.generation[s as usize],
        }
    }

    fn peek(&self) -> Option<&T> {
        self.heap.first().map(|e| &e.val)
    }

    fn pop(&mut self) -> Option<T> {
        self.remove_at(0)
    }

    /// index turns a handle into a heap index, or None if it is stale or forged.
    fn index(&self, h: Handle) -> Option<usize> {
        let s = h.slot as usize;
        if *self.generation.get(s)? != h.generation {
            return None;
        }
        self.pos[s].map(|p| p as usize)
    }

    fn get(&self, h: Handle) -> Option<&T> {
        self.index(h).map(|i| &self.heap[i].val)
    }

    /// update replaces the element's value and moves it up OR down, Theta(log n).
    /// It is container/heap's Fix, found through the handle instead of a stored index.
    fn update(&mut self, h: Handle, x: T) -> bool {
        let Some(i) = self.index(h) else {
            return false;
        };
        self.heap[i].val = x;
        if !self.down(i, self.heap.len()) {
            self.up(i);
        }
        true
    }

    fn remove(&mut self, h: Handle) -> Option<T> {
        let i = self.index(h)?;
        self.remove_at(i)
    }

    /// remove_at is container/heap's Remove: swap the cell with the last, shrink,
    /// repair the moved element in whichever direction it needs, then retire the
    /// slot. None if `i` is out of range (an empty queue for `pop`).
    fn remove_at(&mut self, i: usize) -> Option<T> {
        let n = self.heap.len().checked_sub(1)?;
        if i > n {
            return None;
        }
        if n != i {
            self.swap(i, n);
            if !self.down(i, n) {
                self.up(i);
            }
        }
        let e = self.heap.pop()?;
        self.pos[e.slot as usize] = None;
        self.generation[e.slot as usize] += 1; // every handle made for this occupant is stale from here on
        self.free.push(e.slot);
        Some(e.val)
    }

    fn less_at(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        (self.less)(&self.heap[i].val, &self.heap[j].val)
    }

    /// swap exchanges two heap cells AND repairs both pos entries: the whole price of handles.
    fn swap(&mut self, i: usize, j: usize) {
        self.heap.swap(i, j);
        self.pos[self.heap[i].slot as usize] = Some(i as u32);
        self.pos[self.heap[j].slot as usize] = Some(j as u32);
    }

    /// up and down are container/heap's, loop for loop (strict <, left child on
    /// ties). usize has no -1, so the loop condition is `j > 0` where Go tests `i == j`.
    fn up(&mut self, mut j: usize) {
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less_at(j, i) {
                break;
            }
            self.swap(i, j);
            j = i;
        }
    }

    fn down(&mut self, i0: usize, n: usize) -> bool {
        let mut i = i0;
        loop {
            let mut j = 2 * i + 1;
            if j >= n {
                break;
            }
            if j + 1 < n && self.less_at(j + 1, j) {
                j += 1;
            }
            if !self.less_at(j, i) {
                break;
            }
            self.swap(i, j);
            i = j;
        }
        i > i0
    }

    /// check verifies every structural invariant and returns None or the first
    /// violation. It calls `self.less` directly, so it never touches `cmps`.
    fn check(&self) -> Option<String> {
        for i in 1..self.heap.len() {
            if (self.less)(&self.heap[i].val, &self.heap[(i - 1) / 2].val) {
                return Some(format!("heap order broken at index {i}"));
            }
        }
        for (i, e) in self.heap.iter().enumerate() {
            if self.pos.get(e.slot as usize).copied().flatten() != Some(i as u32) {
                return Some(format!("pos does not point back at heap index {i}"));
            }
        }
        let live = self.pos.iter().flatten().count();
        if live != self.heap.len() {
            return Some(format!(
                "{live} live slots for {} heap cells",
                self.heap.len()
            ));
        }
        let mut seen = vec![false; self.pos.len()];
        for &s in &self.free {
            let s = s as usize;
            if s >= self.pos.len() || self.pos[s].is_some() || seen[s] {
                return Some(format!(
                    "free list entry {s} is out of range, live or repeated"
                ));
            }
            seen[s] = true;
        }
        if self.free.len() + self.heap.len() != self.pos.len() {
            return Some("a slot is neither live nor free".to_string());
        }
        None
    }
}

// ---- the oracle: a vector of live (id, priority) pairs, scanned in O(n) ----

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
struct Item {
    id: i64,
    prio: i64,
}

fn by_prio(a: &Item, b: &Item) -> bool {
    a.prio < b.prio
}

struct LiveItem {
    item: Item,
    h: Handle,
}

struct Model {
    live: Vec<LiveItem>,
    dead: [Handle; 64], // ring of the most recently dead handles
    dead_n: usize,
}

impl Model {
    fn min_prio(&self) -> i64 {
        self.live.iter().map(|e| e.item.prio).min().unwrap_or(0)
    }

    fn find(&self, id: i64) -> Option<usize> {
        self.live.iter().position(|e| e.item.id == id)
    }

    /// kill removes live[i] from the model and remembers its handle as dead.
    fn kill(&mut self, i: usize) {
        self.dead[self.dead_n % 64] = self.live[i].h;
        self.dead_n += 1;
        self.live.swap_remove(i);
    }
}

#[derive(Default)]
struct Stats {
    push: usize,
    pop: usize,
    update: usize,
    remove: usize,
    peek: usize,
    skipped: usize,
    max_live: usize,
    slots: usize,
    reused: usize,
    stale_tries: usize,
    stale_on_reused: usize,
    mismatches: usize,
    violations: usize,
    stale_errors: usize,
    first: String,
}

impl Stats {
    fn bad(&mut self, msg: String) {
        self.mismatches += 1;
        if self.first.is_empty() {
            self.first = msg;
        }
    }
}

/// run_model drives a Pq<Item> and the model with the same random operations and
/// compares them after EVERY operation. start > 0 begins from from_slice(start items).
fn run_model(seed: u64, ops: usize, start: usize) -> Stats {
    let mut r = Pcg::new(seed, 2);
    let mut st = Stats::default();
    let mut m = Model {
        live: Vec::new(),
        dead: [Handle {
            slot: 0,
            generation: 0,
        }; 64],
        dead_n: 0,
    };
    let mut q = Pq::new(by_prio);
    let mut next_id = 0i64;
    if start > 0 {
        let xs: Vec<Item> = (0..start)
            .map(|i| Item {
                id: i as i64,
                prio: r.int_n(20) as i64,
            })
            .collect();
        let (fq, hs) = Pq::from_slice(xs.clone(), by_prio);
        q = fq;
        for (x, h) in xs.into_iter().zip(hs) {
            m.live.push(LiveItem { item: x, h });
        }
        next_id = start as i64;
    }
    for _ in 0..ops {
        let x = r.int_n(100);
        match x {
            0..=34 => {
                // Push
                let it = Item {
                    id: next_id,
                    prio: r.int_n(20) as i64,
                };
                next_id += 1;
                let before = q.pos.len();
                let h = q.push(it);
                if (h.slot as usize) < before {
                    st.reused += 1;
                }
                m.live.push(LiveItem { item: it, h });
                st.push += 1;
            }
            35..=54 => {
                // Pop: must be a live element holding the minimum priority
                let got = q.pop();
                st.pop += 1;
                if m.live.is_empty() {
                    if got.is_some() {
                        st.bad("Pop on an empty queue returned an element".to_string());
                    }
                } else {
                    let found = got.and_then(|g| m.find(g.id).map(|i| (g, i)));
                    match found {
                        Some((g, i)) if g == m.live[i].item && g.prio == m.min_prio() => m.kill(i),
                        _ => {
                            let msg =
                                format!("Pop returned {got:?}, model minimum {}", m.min_prio());
                            st.bad(msg);
                        }
                    }
                }
            }
            55..=74 if !m.live.is_empty() => {
                // Update a random live handle, priority up or down
                let i = r.int_n(m.live.len() as u64) as usize;
                let np = r.int_n(20) as i64;
                let id = m.live[i].item.id;
                if !q.update(m.live[i].h, Item { id, prio: np }) {
                    st.bad("Update through a live handle returned false".to_string());
                }
                m.live[i].item.prio = np;
                st.update += 1;
            }
            75..=89 if !m.live.is_empty() => {
                // Remove a random live handle
                let i = r.int_n(m.live.len() as u64) as usize;
                match q.remove(m.live[i].h) {
                    Some(got) if got == m.live[i].item => {}
                    got => {
                        let msg = format!("Remove returned {got:?}, wanted {:?}", m.live[i].item);
                        st.bad(msg);
                    }
                }
                m.kill(i);
                st.remove += 1;
            }
            90..=99 => {
                // Peek: no state change
                let ok = match (q.peek(), m.live.is_empty()) {
                    (None, empty) => empty,
                    (Some(got), false) => m.find(got.id).is_some_and(|i| *got == m.live[i].item),
                    (Some(_), true) => false,
                };
                if !ok {
                    st.bad("Peek returned an element the model does not hold".to_string());
                }
                st.peek += 1;
            }
            _ => st.skipped += 1, // an Update or Remove with nothing live to aim at
        }

        // A stale handle, tried through all three entry points: nothing may happen.
        let n = m.dead_n.min(m.dead.len());
        if n > 0 {
            let d = m.dead[r.int_n(n as u64) as usize];
            st.stale_tries += 1;
            if (d.slot as usize) < q.pos.len() && q.pos[d.slot as usize].is_some() {
                st.stale_on_reused += 1; // the slot holds ANOTHER element now: only generation protects it
            }
            let g = q.get(d).is_some();
            let u = q.update(d, Item { id: -1, prio: -1 });
            let rm = q.remove(d).is_some();
            if g || u || rm {
                st.stale_errors += 1;
            }
        }

        // Invariants and the oracle's verdict on the whole queue, after every operation.
        if let Some(msg) = q.check() {
            st.violations += 1;
            if st.first.is_empty() {
                st.first = msg;
            }
        }
        if q.len() != m.live.len() {
            st.bad("Len disagrees with the model".to_string());
        }
        if !m.live.is_empty() && q.peek().is_none_or(|top| top.prio != m.min_prio()) {
            st.bad("Peek is not the model minimum".to_string());
        }
        st.max_live = st.max_live.max(m.live.len());
    }
    st.slots = q.pos.len();
    st
}

// ---- helpers for the count and timing sections ----

fn permutation(r: &mut Pcg, n: usize) -> Vec<i64> {
    let mut p: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

fn int_less(a: &i64, b: &i64) -> bool {
    a < b
}

// Std's BinaryHeap is a MAX-heap and counts nothing, so for the one labelled
// std line the key is a newtype whose `cmp` is REVERSED and bumps a counter.
thread_local!(static STD_CMPS: Cell<u64> = const { Cell::new(0) });

#[derive(Clone, Copy, PartialEq, Eq)]
struct Key(i64);

impl Ord for Key {
    fn cmp(&self, other: &Self) -> Ordering {
        STD_CMPS.with(|c| c.set(c.get() + 1));
        other.0.cmp(&self.0)
    }
}

impl PartialOrd for Key {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

/// Kv is the (id, key) pair of the re-keying and timing sections.
#[derive(Clone, Copy)]
struct Kv {
    id: usize,
    key: i64,
}

fn by_key(a: &Kv, b: &Kv) -> bool {
    a.key < b.key
}

/// build pushes every key as (id, key) and returns the queue and each id's handle.
fn build<F: Fn(&Kv, &Kv) -> bool>(keys: &[i64], less: F) -> (Pq<Kv, F>, Vec<Handle>) {
    let mut q = Pq::new(less);
    let hs: Vec<Handle> = keys
        .iter()
        .enumerate()
        .map(|(id, &key)| q.push(Kv { id, key }))
        .collect();
    q.cmps = 0;
    (q, hs)
}

/// rekey gives the same new key to the same element two ways: `update`, and
/// `remove` + `push`. It prints the comparisons each way spent.
fn rekey(label: &str, r: &mut Pcg, keys: &[i64], new_key: impl Fn(i64, &mut Pcg) -> i64) {
    let cn = keys.len();
    let (mut qa, ha) = build(keys, by_key);
    let (mut qb, mut hb) = build(keys, by_key);
    for _ in 0..cn {
        let id = r.int_n(cn as u64) as usize;
        let old = qa.get(ha[id]).map_or(0, |kv| kv.key);
        let nk = new_key(old, r);
        qa.update(ha[id], Kv { id, key: nk });
        qb.remove(hb[id]);
        hb[id] = qb.push(Kv { id, key: nk });
    }
    let (cmps_a, cmps_b) = (qa.cmps, qb.cmps); // read BEFORE draining the queues below
    let mut same_keys = true;
    while let (Some(a), Some(b)) = (qa.pop(), qb.pop()) {
        same_keys &= a.key == b.key;
    }
    println!(
        "  {cn} {label:<34} Update {:5.2} comparisons each, Remove + Push {:5.2}; same keys come out: {same_keys}",
        cmps_a as f64 / cn as f64,
        cmps_b as f64 / cn as f64
    );
}

struct Op {
    upd: bool, // else: pop the minimum and push it back with a new key
    id: usize,
    key: i64,
}

/// time_all runs every contender (each does its untimed setup, then returns the
/// ns per operation of its timed work) for 3 interleaved rounds and returns the
/// lowest and highest of each.
fn time_all(cs: &[Box<dyn Fn() -> f64 + '_>]) -> (Vec<f64>, Vec<f64>) {
    let mut lo = vec![f64::INFINITY; cs.len()];
    let mut hi = vec![0f64; cs.len()];
    for _ in 0..3 {
        for (k, c) in cs.iter().enumerate() {
            let ns = c();
            lo[k] = lo[k].min(ns);
            hi[k] = hi[k].max(ns);
        }
    }
    (lo, hi)
}

fn main() {
    // (a) + (b): the model oracle with invariants checked after every operation.
    println!(
        "(a) Pq<Item> against an O(n) model, (b) every invariant checked after EVERY operation;"
    );
    println!(
        "    mix: Push 35%, Pop 20%, Update 20%, Remove 15%, Peek 10%; priorities IntN(20), so ties are everywhere"
    );
    println!(
        "  {:>4} {:>6} {:>7} | {:>6} {:>6} {:>6} {:>6} {:>6} {:>6} | {:>8} {:>6} {:>7}",
        "seed",
        "start",
        "ops",
        "push",
        "pop",
        "update",
        "remove",
        "peek",
        "skip",
        "max live",
        "slots",
        "reused"
    );
    let mut runs = Vec::new();
    for (seed, ops, start) in [
        (1u64, 100_000usize, 0usize),
        (2, 100_000, 0),
        (3, 100_000, 0),
        (4, 100_000, 0),
        (5, 100_000, 0),
        (6, 20_000, 4000),
    ] {
        let st = run_model(seed, ops, start);
        println!(
            "  {:>4} {:>6} {:>7} | {:>6} {:>6} {:>6} {:>6} {:>6} {:>6} | {:>8} {:>6} {:>7}",
            seed,
            start,
            ops,
            st.push,
            st.pop,
            st.update,
            st.remove,
            st.peek,
            st.skipped,
            st.max_live,
            st.slots,
            st.reused
        );
        runs.push(st);
    }
    println!(
        "\n  {:>4} | {:>11} {:>14} | {:>10} {:>10} {:>12}",
        "seed", "stale tries", "on reused slot", "mismatches", "violations", "stale errors"
    );
    let (mut mismatches, mut violations, mut stale_errors) = (0, 0, 0);
    for (i, st) in runs.iter().enumerate() {
        println!(
            "  {:>4} | {:>11} {:>14} | {:>10} {:>10} {:>12}",
            i + 1,
            st.stale_tries,
            st.stale_on_reused,
            st.mismatches,
            st.violations,
            st.stale_errors
        );
        mismatches += st.mismatches;
        violations += st.violations;
        stale_errors += st.stale_errors;
        if !st.first.is_empty() {
            println!("  first failure: {}", st.first);
        }
    }
    println!(
        "  mismatches: {mismatches}  invariant violations: {violations}  stale-handle errors: {stale_errors}"
    );

    // (c) Stale handles, step by step.
    println!("\n(c) stale handles, directed (Get / Update / Remove must all say false):");
    let mut q = Pq::new(int_less);
    let h5 = q.push(5);
    let h3 = q.push(3);
    let h9 = q.push(9);
    let mut stale_bad = 0;
    // a fn rather than a closure: a closure would keep `q` borrowed while the steps mutate it
    fn try_handle<F: Fn(&i64, &i64) -> bool>(
        q: &mut Pq<i64, F>,
        label: &str,
        h: Handle,
        stale_bad: &mut i32,
    ) {
        let g = q.get(h).is_some();
        let u = q.update(h, 4);
        let rm = q.remove(h).is_some();
        println!(
            "  {label:<44} {}: Get {g:<5} Update {u:<5} Remove {rm:<5}",
            h.show()
        );
        if g || u || rm {
            *stale_bad += 1;
        }
    }
    let v = q.pop();
    try_handle(
        &mut q,
        &format!("after Pop (returned {})", v.unwrap_or(-1)),
        h3,
        &mut stale_bad,
    );
    let v = q.remove(h9);
    try_handle(
        &mut q,
        &format!("after Remove (returned {})", v.unwrap_or(-1)),
        h9,
        &mut stale_bad,
    );
    let h7 = q.push(7);
    println!(
        "  Push(7) reuses the freed slot: {} then {}, same slot: {}, new generation: {}",
        h9.show(),
        h7.show(),
        h7.slot == h9.slot,
        h7.generation != h9.generation
    );
    try_handle(
        &mut q,
        "old handle, slot now holds the 7",
        h9,
        &mut stale_bad,
    );
    println!(
        "  the 7 is untouched: Get -> {}, {}; Len {}",
        q.get(h7).copied().unwrap_or(-1),
        q.get(h7).is_some(),
        q.len()
    );
    try_handle(
        &mut q,
        "the zero Handle",
        Handle {
            slot: 0,
            generation: 0,
        },
        &mut stale_bad,
    );
    try_handle(
        &mut q,
        "a slot that never existed",
        Handle {
            slot: 99,
            generation: 1,
        },
        &mut stale_bad,
    );
    try_handle(
        &mut q,
        "the largest possible slot",
        Handle {
            slot: u32::MAX,
            generation: 1,
        },
        &mut stale_bad,
    );
    q.update(h5, 1);
    println!(
        "  a live handle still works: Update(h5, 1) puts {} on top; stale-handle errors: {stale_bad}",
        q.peek().copied().unwrap_or(-1)
    );

    // (d) from_slice: O(n) heapify, with a handle per input element.
    let mut r = Pcg::new(1, 2);
    let n = 10_000usize;
    let xs: Vec<i64> = (0..n).map(|_| r.int_n(1_000_000) as i64).collect();
    let (mut fq, hs) = Pq::from_slice(xs.clone(), int_less);
    let mut pushed = Pq::new(int_less);
    for &x in &xs {
        pushed.push(x);
    }
    println!(
        "\n(d) from_slice of {n} values: {} comparisons = {:.2} n (<= 2n: {}); {n} push calls on the same values: {}",
        fq.cmps,
        fq.cmps as f64 / n as f64,
        fq.cmps as usize <= 2 * n,
        pushed.cmps
    );
    let fresh = fq.check().is_none();
    let mut same = hs
        .iter()
        .zip(&xs)
        .all(|(&h, &x)| fq.get(h).copied() == Some(x));
    let mut left = Vec::new();
    for (i, &x) in xs.iter().enumerate() {
        if i % 3 == 0 {
            // remove every third element through its handle
            same &= fq.remove(hs[i]) == Some(x);
        } else {
            left.push(x);
        }
    }
    left.sort_unstable();
    let after_removes = fq.check().is_none();
    let mut popped = Vec::new();
    while let Some(v) = fq.pop() {
        popped.push(v);
    }
    println!(
        "    check() after from_slice: {fresh}, after removing every 3rd element by handle: {after_removes}"
    );
    println!(
        "    handle i returns xs[i] and removal by handle returns it: {same}; the pops then equal the sorted rest: {}",
        popped == left
    );

    // (e) Counts: the sifts are container/heap's, so the counts match the Go numbers.
    let cn = 1usize << 14;
    let keys = permutation(&mut r, cn);
    let mut pq = Pq::new(int_less);
    for &k in &keys {
        pq.push(k);
    }
    let push_pq = pq.cmps;
    pq.cmps = 0;
    let mut popped = Vec::new();
    while let Some(v) = pq.pop() {
        popped.push(v);
    }
    let mut sorted = keys.clone();
    sorted.sort_unstable();
    STD_CMPS.with(|c| c.set(0));
    let mut sh = BinaryHeap::new();
    for &k in &keys {
        sh.push(Key(k));
    }
    let push_std = STD_CMPS.with(|c| c.replace(0));
    let mut std_popped = Vec::new();
    while let Some(Key(k)) = sh.pop() {
        std_popped.push(k);
    }
    let pop_std = STD_CMPS.with(|c| c.replace(0));
    println!("\n(e) n = {cn} random keys, comparisons per operation:");
    println!("  {:<16} {:>9} {:>9}", "", "per Push", "per Pop");
    println!(
        "  {:<16} {:>9.2} {:>9.2}",
        "Pq<i64>",
        push_pq as f64 / cn as f64,
        pq.cmps as f64 / cn as f64
    );
    println!(
        "  {:<16} {:>9.2} {:>9.2}",
        "std BinaryHeap",
        push_std as f64 / cn as f64,
        pop_std as f64 / cn as f64
    );
    println!(
        "  both pop in sorted order: {}",
        popped == sorted && std_popped == sorted
    );

    rekey("re-keyings, new key IntN(2n):", &mut r, &keys, |_, r| {
        r.int_n(2 * cn as u64) as i64
    });
    rekey(
        "decrease-keys, new key IntN(old+1):",
        &mut r,
        &keys,
        |old, r| r.int_n(old as u64 + 1) as i64,
    );

    // (f) Timing: lowest-highest ns per operation over 3 interleaved rounds.
    println!(
        "\n(f) ns per operation (lowest-highest of 3 rounds; Pq also bumps its cmps counter):"
    );
    println!(
        "  {:>9} | {:>12} {:>14} | {:>12}",
        "n", "Pq<i64>", "std BinaryHeap", "Pq<Kv>"
    );
    for n in [1000usize, 100_000, 1_000_000] {
        let keys: Vec<i64> = (0..n).map(|_| r.int_n(1 << 30) as i64).collect();
        let ops: Vec<Op> = (0..400_000)
            .map(|_| Op {
                upd: r.int_n(10) < 2,
                id: r.int_n(n as u64) as usize,
                key: r.int_n(1 << 30) as i64,
            })
            .collect();
        let reps = (500_000 / (2 * n)).max(1);
        let (pp_lo, pp_hi) = time_all(&[
            Box::new(|| {
                let t0 = Instant::now();
                for _ in 0..reps {
                    let mut q = Pq::new(int_less);
                    for &k in &keys {
                        q.push(k);
                    }
                    while let Some(v) = q.pop() {
                        black_box(v);
                    }
                }
                t0.elapsed().as_nanos() as f64 / (2 * n * reps) as f64
            }),
            Box::new(|| {
                let t0 = Instant::now();
                for _ in 0..reps {
                    let mut h = BinaryHeap::new();
                    for &k in &keys {
                        h.push(Reverse(k));
                    }
                    while let Some(v) = h.pop() {
                        black_box(v);
                    }
                }
                t0.elapsed().as_nanos() as f64 / (2 * n * reps) as f64
            }),
        ]);
        let (mx_lo, mx_hi) = time_all(&[Box::new(|| {
            let (mut q, mut hs) = build(&keys, by_key);
            let t0 = Instant::now();
            for o in &ops {
                if o.upd {
                    q.update(
                        hs[o.id],
                        Kv {
                            id: o.id,
                            key: o.key,
                        },
                    );
                } else if let Some(it) = q.pop() {
                    hs[it.id] = q.push(Kv {
                        id: it.id,
                        key: o.key,
                    });
                }
            }
            black_box(&q);
            t0.elapsed().as_nanos() as f64 / ops.len() as f64
        })]);
        let cell = |lo: &[f64], hi: &[f64], k: usize| format!("{:.0}-{:.0}", lo[k], hi[k]);
        println!(
            "  {:>9} | {:>12} {:>14} | {:>12}",
            n,
            cell(&pp_lo, &pp_hi, 0),
            cell(&pp_lo, &pp_hi, 1),
            cell(&mx_lo, &mx_hi, 0)
        );
    }
}
```

**Rust sample output:**

```
(a) Pq<Item> against an O(n) model, (b) every invariant checked after EVERY operation;
    mix: Push 35%, Pop 20%, Update 20%, Remove 15%, Peek 10%; priorities IntN(20), so ties are everywhere
  seed  start     ops |   push    pop update remove   peek   skip | max live  slots  reused
     1      0  100000 |  35131  19862  20104  14965   9900     38 |      329    329   34802
     2      0  100000 |  34993  19950  19929  15087   9874    167 |      336    336   34657
     3      0  100000 |  35003  20141  19928  14673  10053    202 |      348    348   34655
     4      0  100000 |  34820  19942  19915  15012   9959    352 |      183    183   34637
     5      0  100000 |  35059  19921  20001  14968  10037     14 |      485    485   34574
     6   4000   20000 |   7078   3885   3994   3044   1999      0 |     4167   4167    6911

  seed | stale tries on reused slot | mismatches violations stale errors
     1 |       99995          79852 |          0          0            0
     2 |       99998          76437 |          0          0            0
     3 |       99995          76511 |          0          0            0
     4 |       99996          73933 |          0          0            0
     5 |       99993          81351 |          0          0            0
     6 |       19997          17146 |          0          0            0
  mismatches: 0  invariant violations: 0  stale-handle errors: 0

(c) stale handles, directed (Get / Update / Remove must all say false):
  after Pop (returned 3)                       {1 1}: Get false Update false Remove false
  after Remove (returned 9)                    {2 1}: Get false Update false Remove false
  Push(7) reuses the freed slot: {2 1} then {2 2}, same slot: true, new generation: true
  old handle, slot now holds the 7             {2 1}: Get false Update false Remove false
  the 7 is untouched: Get -> 7, true; Len 2
  the zero Handle                              {0 0}: Get false Update false Remove false
  a slot that never existed                    {99 1}: Get false Update false Remove false
  the largest possible slot                    {4294967295 1}: Get false Update false Remove false
  a live handle still works: Update(h5, 1) puts 1 on top; stale-handle errors: 0

(d) from_slice of 10000 values: 18793 comparisons = 1.88 n (<= 2n: true); 10000 push calls on the same values: 22849
    check() after from_slice: true, after removing every 3rd element by handle: true
    handle i returns xs[i] and removal by handle returns it: true; the pops then equal the sorted rest: true

(e) n = 16384 random keys, comparisons per operation:
                    per Push   per Pop
  Pq<i64>               2.28     23.04
  std BinaryHeap        2.28     12.70
  both pop in sorted order: true
  16384 re-keyings, new key IntN(2n):      Update  2.88 comparisons each, Remove + Push  3.90; same keys come out: true
  16384 decrease-keys, new key IntN(old+1): Update  2.90 comparisons each, Remove + Push  4.72; same keys come out: true

(f) ns per operation (lowest-highest of 3 rounds; Pq also bumps its cmps counter):
          n |      Pq<i64> std BinaryHeap |       Pq<Kv>
       1000 |        12-12            6-6 |        45-46
     100000 |        34-36          18-18 |        80-82
    1000000 |        65-71          27-28 |      208-223
```

> **Go vs Rust:** The oracle streams (same PCG seeds, same draw order), the table of operations, the stale-handle counts and every comparison count are identical to the Go output, so the checks behave the same way in both languages; what differs is the shape. `Handle` is `Copy` plain data, so holding one borrows nothing from the queue; Go's field name `gen` is a reserved keyword in Rust 2024, so it is `generation`; `slot` is unsigned, so the "negative slot" case becomes `u32::MAX`; `pop`/`remove`/`get` return `Option` and `peek` a borrow instead of `(T, bool)`; and `less` is a type parameter that the compiler inlines where Go calls a func value. The labelled `std` line is Rust only: `BinaryHeap` makes the same 2.28 comparisons per push but 12.70 per pop instead of 23.04 (its bottom-up pop, example 15), and by the lowest times of nine runs it was about 2-2.5x faster than `Pq<i64>`, a comparison that is not like for like because std cannot `update` or `remove` (plausible mechanism, not measured: the hole technique writes no `pos` entries). Rust's `Pq<i64>` ran 11-12 / 32-41 / 62-86 ns per operation against Go's 19-24 / 64-86 / 87-123.

**Complexity:** `Push`, `Pop`, `Update`, `Remove` Θ(log n), `Peek` and `Get` Θ(1), `FromSlice` Θ(n) · space Θ(n) plus one `int` and one `uint32` per slot, and the slot count never exceeded the largest live count (329 / 336 / 348 / 183 / 485 slots for those maxima) · proved on 520,000 operations in six streams with 0 mismatches, 0 invariant violations and 0 stale-handle errors, where 73,933–81,351 of the roughly 100,000 stale tries per stream (one handle after every operation) landed on a slot already holding another element, which only the generation protects · `FromSlice` 1.88 n comparisons (≤ 2n) against 2.28 n for n `Push` calls · identical to `container/heap` where the sifts are the same: 2.28 per `Push`, 23.04 per `Pop` at n = 16,384 · `Update` about 2.9 comparisons against 3.9–4.7 for `Remove` + `Push` · timing over nine runs on a shared machine: push-n-pop-n `PQ[int]` was 1.3–1.9x faster than `container/heap` at n = 10^3 (about 20 against about 33 ns per operation) and level with it at 10^5 and 10^6 (ranges overlap: 64–86 against 70–103, 87–123 against 87–111), so the lead is gone at 10^6 (plausible mechanism, not measured: each swap also writes `pos` at a random slot); in the steady-state 20% `Update` workload `PQ[Item]` was level with `container/heap` + `Fix` at 10^3 (61–70 against 62–73 ns) and used 10–30% less time at 10^5 (110–140 against 128–165) and 20–35% less at 10^6 (225–281 against 290–426)

---

> Next lesson: [21 — Tries & Prefix Structures](../../21-tries.md) — the prefix tree, and a heap on top
> of it for autocomplete.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
