# Step 11 — Strings, Runes & Bytes · 🟡 Medium

Examples **7–11**: the string problem set — anagram, palindrome, reverse words, run-length compression
and the longest substring without repeats — each solved correctly for UTF-8, not just ASCII.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic; **Sample output:** ones print timings or memory, which vary between runs.

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

## 7. Anagrams: sort, count, and the byte trap

`🟡 medium` · *Anagram*

Two strings are anagrams when they hold the same characters the same number of times, and there are three
classic checks: sort both and compare (Θ(n log n) comparisons and two copies), a `[26]int` counter (Θ(n) time,
Θ(1) space, lowercase ASCII only), and a `map[rune]int` (Θ(n), any valid UTF-8). The counter is the right tool
whenever the alphabet is known and small; the map is the right tool for text. The trap sits in the first one:
sorting **bytes** says `"é£"` and `"ã©"` are anagrams, because they share the byte multiset `a3 a9 c2 c3` and not
a single rune. (Grouping anagrams was lesson 10 example 12.)

**Steps:**

1. Write the three checks plus a sorted-runes oracle, and run them on a table of pairs that includes `"é£"/"ã©"`.
2. Count the work: sort comparisons on random a–z strings and on near-distinct CJK runes, against the counter's exact 2n + 26.
3. Count allocations, then run 20,000 random pairs over `{é, £, ã, ©}` against the oracle and count the wrong answers.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
)

// Two strings are anagrams when they hold the same characters the same number
// of times. Three classic ways to check, and one question they all hide:
// "character" means BYTE, RUNE, or something else?
//
//   sortBytes  copy both, sort, compare      Theta(n log n), 2 copies
//   count26    [26]int: +1 for a, -1 for b   Theta(n), Theta(1) space, a-z only
//   runeMap    map[rune]int, same idea       Theta(n), any valid UTF-8

// sortBytes sorts the BYTES of each string. cmps counts the comparisons the
// sort makes, so we can watch the n log n.
func sortBytes(a, b string, cmps *int) bool {
	if len(a) != len(b) {
		return false
	}
	byCmp := func(x, y byte) int { *cmps++; return int(x) - int(y) }
	x, y := []byte(a), []byte(b) // the two copies: a string is immutable
	slices.SortFunc(x, byCmp)
	slices.SortFunc(y, byCmp)
	return string(x) == string(y) // compiler-recognised: no allocation
}

// count26 is the fast one, and it only knows 26 letters. ok=false means the
// input had something outside a-z, so the method does not apply.
func count26(a, b string, ops *int) (same, ok bool) {
	if len(a) != len(b) {
		return false, true
	}
	var c [26]int
	for i := 0; i < len(a); i++ {
		if a[i] < 'a' || a[i] > 'z' || b[i] < 'a' || b[i] > 'z' {
			return false, false
		}
		c[a[i]-'a']++
		c[b[i]-'a']--
		*ops += 2
	}
	for _, v := range c { // always 26 checks, whatever n is
		*ops++
		if v != 0 {
			return false, true
		}
	}
	return true, true
}

// runeMap counts code points, so a 2-byte letter is one thing, not two.
func runeMap(a, b string) bool {
	c := make(map[rune]int)
	for _, r := range a {
		c[r]++
	}
	for _, r := range b {
		c[r]--
	}
	for _, v := range c {
		if v != 0 {
			return false
		}
	}
	return true
}

// sortRunes sorts the RUNES: right for any valid UTF-8, and our oracle.
func sortRunes(a, b string, cmps *int) bool {
	byCmp := func(x, y rune) int { *cmps++; return int(x) - int(y) }
	x, y := []rune(a), []rune(b)
	slices.SortFunc(x, byCmp)
	slices.SortFunc(y, byCmp)
	return slices.Equal(x, y)
}

func oracle(a, b string) bool {
	var n int
	return sortRunes(a, b, &n)
}

func yn(v bool) string {
	if v {
		return "yes"
	}
	return "no"
}

var sink bool

func main() {
	fmt.Println("(a) three answers per pair; '!' marks a wrong one (truth = sorted runes)")
	fmt.Println()
	fmt.Printf("  %-22s %-10s %-10s %s\n", "pair", "sortBytes", "count26", "runeMap")
	pairs := [][2]string{{"listen", "silent"}, {"rat", "car"}, {"anagram", "nagaram"},
		{"aab", "abb"}, {"", ""}, {"é£", "ã©"}}
	var junk int
	for _, p := range pairs {
		truth := oracle(p[0], p[1])
		mark := func(v bool) string {
			if v != truth {
				return yn(v) + " !"
			}
			return yn(v)
		}
		c26 := "n/a"
		if same, ok := count26(p[0], p[1], &junk); ok {
			c26 = mark(same)
		}
		label := fmt.Sprintf("\"%s\" / \"%s\"", p[0], p[1])
		fmt.Printf("  %-22s %-10s %-10s %s\n", label,
			mark(sortBytes(p[0], p[1], &junk)), c26, mark(runeMap(p[0], p[1])))
	}
	fmt.Println()
	fmt.Println("  why: the bytes of \"é£\" and \"ã©\" are the same multiset, the runes are not")
	for _, s := range []string{"é£", "ã©"} {
		b := []byte(s)
		fmt.Printf("    %-4s bytes % x   sorted % x   runes %U\n", s, b, sorted(b), []rune(s))
	}

	fmt.Println()
	fmt.Println("(b) comparisons on random anagram pairs (PCG seed 1,2): b = a shuffled")
	fmt.Println()
	fmt.Printf("  %5s %6s | %10s %8s | %10s %8s | %11s\n",
		"n", "log2 n", "a-z sort", "per byte", "CJK sort", "per rune", "count26 ops")
	rng := rand.New(rand.NewPCG(1, 2))
	var perB, perR []float64
	for _, n := range []int{1000, 2000, 4000} {
		a := make([]byte, n) // 26 distinct keys
		for i := range a {
			a[i] = byte('a' + rng.IntN(26))
		}
		b := slices.Clone(a)
		rng.Shuffle(n, func(i, j int) { b[i], b[j] = b[j], b[i] })
		c := make([]rune, n) // 20,992 distinct keys: nearly all different
		for i := range c {
			c[i] = rune(0x4E00 + rng.IntN(20992))
		}
		d := slices.Clone(c)
		rng.Shuffle(n, func(i, j int) { d[i], d[j] = d[j], d[i] })
		cmpB, cmpR, ops := 0, 0, 0
		okB := sortBytes(string(a), string(b), &cmpB)
		okR := sortRunes(string(c), string(d), &cmpR)
		okC, _ := count26(string(a), string(b), &ops)
		if !okB || !okR || !okC || !runeMap(string(c), string(d)) {
			panic("a shuffled copy must be an anagram")
		}
		perB = append(perB, float64(cmpB)/float64(2*n)) // two sorts of n keys
		perR = append(perR, float64(cmpR)/float64(2*n))
		fmt.Printf("  %5d %6.1f | %10d %8.2f | %10d %8.2f | %11d\n",
			n, math.Log2(float64(n)), cmpB, perB[len(perB)-1], cmpR, perR[len(perR)-1], ops)
	}
	fmt.Println()
	fmt.Printf("  count26: exactly 2n + 26 ops. CJK sort: %.2f -> %.2f comparisons per key,\n", perR[0], perR[2])
	fmt.Println("  growing with log2 n -- that is n log n. a-z sort: per-key cost went")
	fmt.Printf("  %.2f -> %.2f, not growing: with 26 distinct keys, pdqsort's partitionEqual sets aside\n", perB[0], perB[2])
	fmt.Println("  every key equal to the pivot in one pass (slices/zsortanyfunc.go), so the work")
	fmt.Println("  is bounded by the 26 keys, not by log n -- a small alphabet is cheaper to sort")

	s := strings.Repeat("abcdefghij", 100)
	t := strings.Repeat("jihgfedcba", 100)
	fmt.Println()
	fmt.Println("(c) allocations for one check of two 1,000-byte strings (testing.AllocsPerRun)")
	fmt.Printf("  sortBytes %4.0f   (the two []byte copies)\n",
		testing.AllocsPerRun(100, func() { sink = sortBytes(s, t, &junk) }))
	fmt.Printf("  count26   %4.0f\n",
		testing.AllocsPerRun(100, func() { sink, _ = count26(s, t, &junk) }))
	fmt.Printf("  runeMap   %4.0f   (the map, for 10 distinct runes)\n",
		testing.AllocsPerRun(100, func() { sink = runeMap(s, t) }))

	fmt.Println()
	fmt.Println("(d) oracle check: 20,000 random pairs over {é, £, ã, ©}, length 0-6 runes")
	alpha := []string{"é", "£", "ã", "©"}
	word := func() string {
		var sb strings.Builder
		for range rng.IntN(7) {
			sb.WriteString(alpha[rng.IntN(len(alpha))])
		}
		return sb.String()
	}
	wrongS, falseYes, wrongM, yes := 0, 0, 0, 0
	for range 20000 {
		a, b := word(), word()
		truth := oracle(a, b)
		if truth {
			yes++
		}
		if got := sortBytes(a, b, &junk); got != truth {
			wrongS++
			if got {
				falseYes++
			}
		}
		if runeMap(a, b) != truth {
			wrongM++
		}
	}
	fmt.Printf("  true anagrams %d   sortBytes wrong %d   runeMap wrong %d\n", yes, wrongS, wrongM)
	fmt.Printf("  of the %d sortBytes errors, %d are false 'yes' -- bytes can only merge letters,\n", wrongS, falseYes)
	fmt.Println("  never lose one, so equal runes always mean equal bytes (é+£ = ã+© = c2 c3 a3 a9)")
}

func sorted(b []byte) []byte {
	c := slices.Clone(b)
	slices.Sort(c)
	return c
}
```

**Output:**

```
(a) three answers per pair; '!' marks a wrong one (truth = sorted runes)

  pair                   sortBytes  count26    runeMap
  "listen" / "silent"    yes        yes        yes
  "rat" / "car"          no         no         no
  "anagram" / "nagaram"  yes        yes        yes
  "aab" / "abb"          no         no         no
  "" / ""                yes        yes        yes
  "é£" / "ã©"            yes !      n/a        no

  why: the bytes of "é£" and "ã©" are the same multiset, the runes are not
    é£   bytes c3 a9 c2 a3   sorted a3 a9 c2 c3   runes [U+00E9 U+00A3]
    ã©   bytes c3 a3 c2 a9   sorted a3 a9 c2 c3   runes [U+00E3 U+00A9]

(b) comparisons on random anagram pairs (PCG seed 1,2): b = a shuffled

      n log2 n |   a-z sort per byte |   CJK sort per rune | count26 ops
   1000   10.0 |      13446     6.72 |      20471    10.24 |        2026
   2000   11.0 |      25669     6.42 |      45726    11.43 |        4026
   4000   12.0 |      50571     6.32 |      98515    12.31 |        8026

  count26: exactly 2n + 26 ops. CJK sort: 10.24 -> 12.31 comparisons per key,
  growing with log2 n -- that is n log n. a-z sort: per-key cost went
  6.72 -> 6.32, not growing: with 26 distinct keys, pdqsort's partitionEqual sets aside
  every key equal to the pivot in one pass (slices/zsortanyfunc.go), so the work
  is bounded by the 26 keys, not by log n -- a small alphabet is cheaper to sort

(c) allocations for one check of two 1,000-byte strings (testing.AllocsPerRun)
  sortBytes    2   (the two []byte copies)
  count26      0
  runeMap      3   (the map, for 10 distinct runes)

(d) oracle check: 20,000 random pairs over {é, £, ã, ©}, length 0-6 runes
  true anagrams 609   sortBytes wrong 70   runeMap wrong 0
  of the 70 sortBytes errors, 70 are false 'yes' -- bytes can only merge letters,
  never lose one, so equal runes always mean equal bytes (é+£ = ã+© = c2 c3 a3 a9)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Two strings are anagrams when they hold the same characters the same number
// of times. Three classic ways to check, and one question they all hide:
// "character" means BYTE, CHAR (code point), or something else?
//
//   sort_bytes  copy both, sort, compare        Theta(n log n), 2 copies
//   count26     [i32; 26]: +1 for a, -1 for b   Theta(n), Theta(1) space, a-z only
//   char_map    HashMap<char, i32>, same idea   Theta(n), any &str (always valid UTF-8)

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // alloc + realloc events

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

/// average allocation events per call, after one warm-up call (like AllocsPerRun)
fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> f64 {
    f();
    let a0 = ALLOCS.load(Relaxed);
    for _ in 0..runs {
        f();
    }
    (ALLOCS.load(Relaxed) - a0) as f64 / runs as f64
}

// Pcg: a port of Go's math/rand/v2 PCG-DXSM and of its IntN and Shuffle, so
// this program sees exactly the random strings the Go program sees.
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
    /// like rand.IntN(n): exactly uniform in [0, n)
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
    /// like rand.Shuffle: Fisher-Yates from the back
    fn shuffle<T>(&mut self, v: &mut [T]) {
        for i in (1..v.len()).rev() {
            let j = self.int_n(i + 1);
            v.swap(i, j);
        }
    }
}

// sort_bytes sorts the BYTES of each string; cmps counts the comparisons.
fn sort_bytes(a: &str, b: &str, cmps: &mut u64) -> bool {
    if a.len() != b.len() {
        return false;
    }
    // the two copies: a &str is a shared borrow, so we may not sort it in place
    let (mut x, mut y) = (a.as_bytes().to_vec(), b.as_bytes().to_vec());
    let mut by_cmp = |p: &u8, q: &u8| {
        *cmps += 1;
        p.cmp(q)
    };
    x.sort_unstable_by(&mut by_cmp);
    y.sort_unstable_by(&mut by_cmp);
    x == y
}

// count26 is the fast one, and it only knows 26 letters. None means the input
// had something outside a-z, so the method does not apply.
fn count26(a: &str, b: &str, ops: &mut u64) -> Option<bool> {
    if a.len() != b.len() {
        return Some(false);
    }
    let mut c = [0i32; 26];
    for (&x, &y) in a.as_bytes().iter().zip(b.as_bytes()) {
        if !x.is_ascii_lowercase() || !y.is_ascii_lowercase() {
            return None;
        }
        c[usize::from(x - b'a')] += 1;
        c[usize::from(y - b'a')] -= 1;
        *ops += 2;
    }
    for v in c {
        // always 26 checks, whatever n is
        *ops += 1;
        if v != 0 {
            return Some(false);
        }
    }
    Some(true)
}

// char_map counts code points, so a 2-byte letter is one thing, not two.
fn char_map(a: &str, b: &str) -> bool {
    let mut c: HashMap<char, i32> = HashMap::new();
    for r in a.chars() {
        *c.entry(r).or_default() += 1;
    }
    for r in b.chars() {
        *c.entry(r).or_default() -= 1;
    }
    c.values().all(|&v| v == 0)
}

// sort_chars sorts the CHARS: right for any &str, and our oracle.
fn sort_chars(a: &str, b: &str, cmps: &mut u64) -> bool {
    let (mut x, mut y): (Vec<char>, Vec<char>) = (a.chars().collect(), b.chars().collect());
    let mut by_cmp = |p: &char, q: &char| {
        *cmps += 1;
        p.cmp(q)
    };
    x.sort_unstable_by(&mut by_cmp);
    y.sort_unstable_by(&mut by_cmp);
    x == y
}

fn oracle(a: &str, b: &str) -> bool {
    sort_chars(a, b, &mut 0)
}

fn yn(v: bool) -> &'static str {
    if v { "yes" } else { "no" }
}

// Go's "% x" and "%U" of a []rune, by hand (Rust's {:?} would print [195, 169])
fn hex(b: &[u8]) -> String {
    b.iter()
        .map(|x| format!("{x:02x}"))
        .collect::<Vec<_>>()
        .join(" ")
}
fn code_points(s: &str) -> String {
    let v: Vec<String> = s.chars().map(|c| format!("U+{:04X}", c as u32)).collect();
    format!("[{}]", v.join(" "))
}

fn main() {
    println!("(a) three answers per pair; '!' marks a wrong one (truth = sorted chars)");
    println!();
    println!(
        "  {:<22} {:<10} {:<10} char_map",
        "pair", "sort_bytes", "count26"
    );
    let pairs = [
        ("listen", "silent"),
        ("rat", "car"),
        ("anagram", "nagaram"),
        ("aab", "abb"),
        ("", ""),
        ("é£", "ã©"),
    ];
    let mut junk = 0u64;
    for (a, b) in pairs {
        let truth = oracle(a, b);
        let mark = |v: bool| {
            if v != truth {
                format!("{} !", yn(v))
            } else {
                yn(v).to_string()
            }
        };
        let c26 = count26(a, b, &mut junk).map_or("n/a".to_string(), mark);
        let label = format!("\"{a}\" / \"{b}\"");
        println!(
            "  {:<22} {:<10} {:<10} {}",
            label,
            mark(sort_bytes(a, b, &mut junk)),
            c26,
            mark(char_map(a, b))
        );
    }
    println!();
    println!("  why: the bytes of \"é£\" and \"ã©\" are the same multiset, the chars are not");
    for s in ["é£", "ã©"] {
        let mut sorted = s.as_bytes().to_vec();
        sorted.sort_unstable();
        println!(
            "    {:<4} bytes {}   sorted {}   chars {}",
            s,
            hex(s.as_bytes()),
            hex(&sorted),
            code_points(s)
        );
    }

    println!();
    println!("(b) comparisons on random anagram pairs (PCG seed 1,2): b = a shuffled");
    println!();
    println!(
        "  {:>5} {:>6} | {:>10} {:>8} | {:>10} {:>8} | {:>11}",
        "n", "log2 n", "a-z sort", "per byte", "CJK sort", "per char", "count26 ops"
    );
    let mut rng = Pcg::new(1, 2);
    let (mut per_b, mut per_r) = (Vec::new(), Vec::new());
    for n in [1000usize, 2000, 4000] {
        // 26 distinct keys
        let a: Vec<u8> = (0..n).map(|_| b'a' + rng.int_n(26) as u8).collect();
        let mut b = a.clone();
        rng.shuffle(&mut b);
        // 20,992 distinct keys: nearly all different
        let c: Vec<char> = (0..n)
            .map(|_| char::from_u32(0x4E00 + rng.int_n(20992) as u32).unwrap())
            .collect();
        let mut d = c.clone();
        rng.shuffle(&mut d);
        // the a-z bytes are ASCII, so they are valid UTF-8: from_utf8 cannot fail
        let (sa, sb) = (String::from_utf8(a).unwrap(), String::from_utf8(b).unwrap());
        let (sc, sd): (String, String) = (c.iter().collect(), d.iter().collect());
        let (mut cmp_b, mut cmp_r, mut ops) = (0u64, 0u64, 0u64);
        let ok_b = sort_bytes(&sa, &sb, &mut cmp_b);
        let ok_r = sort_chars(&sc, &sd, &mut cmp_r);
        let ok_c = count26(&sa, &sb, &mut ops) == Some(true);
        assert!(
            ok_b && ok_r && ok_c && char_map(&sc, &sd),
            "a shuffled copy must be an anagram"
        );
        per_b.push(cmp_b as f64 / (2 * n) as f64); // two sorts of n keys
        per_r.push(cmp_r as f64 / (2 * n) as f64);
        println!(
            "  {:>5} {:>6.1} | {:>10} {:>8.2} | {:>10} {:>8.2} | {:>11}",
            n,
            (n as f64).log2(),
            cmp_b,
            per_b[per_b.len() - 1],
            cmp_r,
            per_r[per_r.len() - 1],
            ops
        );
    }
    println!();
    println!(
        "  count26: exactly 2n + 26 ops. CJK sort: {:.2} -> {:.2} comparisons per key,",
        per_r[0], per_r[2]
    );
    println!("  growing with log2 n -- that is n log n. a-z sort: per-key cost went");
    println!(
        "  {:.2} -> {:.2}, not growing: with 26 distinct keys, ipnsort's equal-pivot partition",
        per_b[0], per_b[2]
    );
    println!("  sets aside every key equal to the pivot in one pass (core/src/slice/sort/");
    println!("  unstable/quicksort.rs), so the work is bounded by the 26 keys, not by log n");

    let s = "abcdefghij".repeat(100);
    let t = "jihgfedcba".repeat(100);
    println!();
    println!("(c) allocations for one check of two 1,000-byte strings (counting allocator)");
    let n = allocs_per_run(100, || {
        black_box(sort_bytes(black_box(&s), black_box(&t), &mut junk));
    });
    println!("  sort_bytes {n:4.0}   (the two Vec<u8> copies)");
    let n = allocs_per_run(100, || {
        black_box(count26(black_box(&s), black_box(&t), &mut junk));
    });
    println!("  count26    {n:4.0}");
    let n = allocs_per_run(100, || {
        black_box(char_map(black_box(&s), black_box(&t)));
    });
    println!("  char_map   {n:4.0}   (the map, for 10 distinct chars)");

    println!();
    println!("(d) oracle check: 20,000 random pairs over {{é, £, ã, ©}}, length 0-6 chars");
    let alpha = ["é", "£", "ã", "©"];
    let word = |rng: &mut Pcg| -> String {
        let k = rng.int_n(7);
        (0..k).map(|_| alpha[rng.int_n(alpha.len())]).collect()
    };
    let (mut wrong_s, mut false_yes, mut wrong_m, mut yes) = (0, 0, 0, 0);
    for _ in 0..20000 {
        let a = word(&mut rng);
        let b = word(&mut rng);
        let truth = oracle(&a, &b);
        if truth {
            yes += 1;
        }
        let got = sort_bytes(&a, &b, &mut junk);
        if got != truth {
            wrong_s += 1;
            if got {
                false_yes += 1;
            }
        }
        if char_map(&a, &b) != truth {
            wrong_m += 1;
        }
    }
    println!("  true anagrams {yes}   sort_bytes wrong {wrong_s}   char_map wrong {wrong_m}");
    println!(
        "  of the {wrong_s} sort_bytes errors, {false_yes} are false 'yes' -- bytes can only merge letters,"
    );
    println!("  never lose one, so equal chars always mean equal bytes (é+£ = ã+© = c2 c3 a3 a9)");
}
```

**Rust output:**

```
(a) three answers per pair; '!' marks a wrong one (truth = sorted chars)

  pair                   sort_bytes count26    char_map
  "listen" / "silent"    yes        yes        yes
  "rat" / "car"          no         no         no
  "anagram" / "nagaram"  yes        yes        yes
  "aab" / "abb"          no         no         no
  "" / ""                yes        yes        yes
  "é£" / "ã©"            yes !      n/a        no

  why: the bytes of "é£" and "ã©" are the same multiset, the chars are not
    é£   bytes c3 a9 c2 a3   sorted a3 a9 c2 c3   chars [U+00E9 U+00A3]
    ã©   bytes c3 a3 c2 a9   sorted a3 a9 c2 c3   chars [U+00E3 U+00A9]

(b) comparisons on random anagram pairs (PCG seed 1,2): b = a shuffled

      n log2 n |   a-z sort per byte |   CJK sort per char | count26 ops
   1000   10.0 |      13618     6.81 |      20535    10.27 |        2026
   2000   11.0 |      25367     6.34 |      45615    11.40 |        4026
   4000   12.0 |      49708     6.21 |      99963    12.50 |        8026

  count26: exactly 2n + 26 ops. CJK sort: 10.27 -> 12.50 comparisons per key,
  growing with log2 n -- that is n log n. a-z sort: per-key cost went
  6.81 -> 6.21, not growing: with 26 distinct keys, ipnsort's equal-pivot partition
  sets aside every key equal to the pivot in one pass (core/src/slice/sort/
  unstable/quicksort.rs), so the work is bounded by the 26 keys, not by log n

(c) allocations for one check of two 1,000-byte strings (counting allocator)
  sort_bytes    2   (the two Vec<u8> copies)
  count26       0
  char_map      3   (the map, for 10 distinct chars)

(d) oracle check: 20,000 random pairs over {é, £, ã, ©}, length 0-6 chars
  true anagrams 609   sort_bytes wrong 70   char_map wrong 0
  of the 70 sort_bytes errors, 70 are false 'yes' -- bytes can only merge letters,
  never lose one, so equal chars always mean equal bytes (é+£ = ã+© = c2 c3 a3 a9)
```

> **Go vs Rust:** Every answer, the allocation counts (2 / 0 / 3) and the oracle run (609 anagrams, 70 byte false-positives, 0 map errors) are identical — the PCG-DXSM port feeds both programs the same strings. Only the sort comparison counts differ (e.g. 13,446 vs 13,618 at n = 1,000), because `slices.SortFunc` is pdqsort and `sort_unstable_by` is ipnsort; both have the equal-pivot partition, so both show the same shape: ~6–7 comparisons per key on 26 letters, growing with log₂ n on CJK. A `&str` cannot hold invalid UTF-8, so `chars()` never meets a stray byte the way Go's `range` can.

**Complexity:** sort Θ(n log n) comparisons in general (CJK runes: 10.2 → 12.3 per key as n went 1,000 → 4,000), but a flat 6.3–6.7 per key on a 26-letter alphabet · `[26]int` Θ(n) time, Θ(1) space, exactly 2n + 26 operations, 0 allocations · `map[rune]int` Θ(n) time, Θ(distinct runes) space, 3 allocations here · byte methods give false "yes" answers on non-ASCII input (70 of 20,000 random pairs).

---

## 8. Palindromes: two pointers over runes

`🟡 medium` · *Palindrome*

Two pointers walk in from both ends, skip anything that is not a letter or digit, fold case and compare —
Θ(n) time whatever they step over. What they step over is the whole question. On **bytes** the check is wrong
for every multi-byte letter (`"été"` compares `c3` with `a9`); on a `[]rune` it is right but costs 4 bytes per
rune; decoding one rune from each end with `utf8.DecodeRuneInString` / `utf8.DecodeLastRuneInString` is right
**and** allocation-free, which makes it the right tool. The interview variant — delete at most one character —
stays Θ(n): the first mismatch tries both deletions, each a single extra scan.

**Steps:**

1. Write the byte, `[]rune` and decode walks, each counting comparisons, and run them on ASCII, accented, NFD and long inputs.
2. Count allocations with `testing.AllocsPerRun`, then run valid-palindrome-II on `abca`, `abc`, `deeee`, `ébcé`.
3. Check all three walks against a filter-fold-reverse oracle on 20,000 random strings and count the wrong answers.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
	"unicode"
	"unicode/utf8"
)

// A palindrome check is two pointers walking in from both ends, skipping what
// is not a letter or digit and folding case. The algorithm is trivial; the
// question is what the pointers step over: bytes, runes, or decoded runes.

// keep is the classic "isalnum": letters and digits survive, the rest is noise.
func keep(r rune) bool { return unicode.IsLetter(r) || unicode.IsDigit(r) }

// palBytes steps over BYTES. It cannot tell what a byte >= 0x80 means, so it
// keeps it as "a letter" -- and compares halves of letters with each other.
func palBytes(s string) (bool, int) {
	isKept := func(c byte) bool { return c >= 0x80 || keep(rune(c)) }
	lower := func(c byte) byte {
		if 'A' <= c && c <= 'Z' {
			return c + 'a' - 'A'
		}
		return c
	}
	cmps := 0
	for i, j := 0, len(s)-1; ; i, j = i+1, j-1 {
		for i < j && !isKept(s[i]) {
			i++
		}
		for i < j && !isKept(s[j]) {
			j--
		}
		if i >= j {
			return true, cmps
		}
		cmps++
		if lower(s[i]) != lower(s[j]) {
			return false, cmps
		}
	}
}

// palRunes converts to []rune first: right, but 4 bytes per rune.
func palRunes(s string) (bool, int) {
	r := []rune(s)
	cmps := 0
	for i, j := 0, len(r)-1; ; i, j = i+1, j-1 {
		for i < j && !keep(r[i]) {
			i++
		}
		for i < j && !keep(r[j]) {
			j--
		}
		if i >= j {
			return true, cmps
		}
		cmps++
		if unicode.ToLower(r[i]) != unicode.ToLower(r[j]) {
			return false, cmps
		}
	}
}

// palDecode decodes one rune from the front and one from the back of s[i:j].
// Right AND allocation-free. i and j are byte offsets that only ever land on
// rune boundaries, because each step moves by a decoded width.
func palDecode(s string) (bool, int) {
	cmps := 0
	i, j := 0, len(s)
	for {
		var a, b rune
		var wa, wb int
		for ; i < j; i += wa { // next kept rune from the front (break skips i += wa)
			if a, wa = utf8.DecodeRuneInString(s[i:j]); keep(a) {
				break
			}
		}
		for ; i < j; j -= wb { // next kept rune from the back
			if b, wb = utf8.DecodeLastRuneInString(s[i:j]); keep(b) {
				break
			}
		}
		if j-i <= wa { // the pointers met, or only the middle rune is left
			return true, cmps
		}
		cmps++
		if unicode.ToLower(a) != unicode.ToLower(b) {
			return false, cmps
		}
		i, j = i+wa, j-wb
	}
}

// pal2 is "valid palindrome II" (interview version: no skipping, no folding).
// On the first mismatch it tries both deletions, each one more Theta(n) scan
// with skips = 0 -- so the whole thing is still Theta(n).
func pal2(t string, skips int, deleted *string, cmps *int) bool {
	for len(t) > 0 {
		a, wa := utf8.DecodeRuneInString(t)
		b, wb := utf8.DecodeLastRuneInString(t)
		if len(t) == wa { // a single rune left
			return true
		}
		*cmps++
		if a != b {
			if skips == 0 {
				return false
			}
			if pal2(t[wa:], 0, deleted, cmps) {
				*deleted = string(a)
				return true
			}
			*deleted = string(b)
			return pal2(t[:len(t)-wb], 0, deleted, cmps)
		}
		t = t[wa : len(t)-wb]
	}
	return true
}

func yn(v bool) string { return map[bool]string{true: "yes", false: "no"}[v] }

var sink bool

func main() {
	type input struct{ label, s, reads string }
	inputs := []input{
		{`"racecar"`, "racecar", "yes"},
		{`"A man, a plan, a canal: Panama"`, "A man, a plan, a canal: Panama", "yes"},
		{`"No 'x' in Nixon"`, "No 'x' in Nixon", "yes"},
		{`"été"`, "été", "yes"},
		{`"ab"`, "ab", "no"},
		{`""`, "", "yes"},
		{`"été " x 250`, strings.Repeat("été ", 250), "yes"},
		{`NFD "e\u0301te\u0301" (été)`, "e\u0301te\u0301", "yes"},
		{`NFD "e\u0301te" (éte)`, "e\u0301te", "no"},
		{`NFC "\u00e9te" (éte)`, "\u00e9te", "no"},
	}
	fmt.Println("(a) three walks; '!' = disagrees with what a reader sees")
	fmt.Println()
	fmt.Printf("  %-34s %-5s %-5s %-5s %-6s %13s %11s\n",
		"input", "reads", "bytes", "runes", "decode", "cmps b/r/d", "allocs b/r/d")
	walks := []func(string) (bool, int){palBytes, palRunes, palDecode}
	for _, in := range inputs {
		var res [3]string
		var cmps, allocs [3]any
		for k, walk := range walks {
			ok, n := walk(in.s)
			if res[k], cmps[k] = yn(ok), n; res[k] != in.reads {
				res[k] += "!"
			}
			allocs[k] = testing.AllocsPerRun(20, func() { sink, _ = walk(in.s) })
		}
		fmt.Printf("  %-34s %-5s %-5s %-5s %-6s %13s %11s\n", in.label, in.reads,
			res[0], res[1], res[2], fmt.Sprintf("%d/%d/%d", cmps[:]...),
			fmt.Sprintf("%.0f/%.0f/%.0f", allocs[:]...))
	}
	fmt.Println()
	fmt.Println("  bytes: wrong on every non-ASCII palindrome -- on \"été\" it compares c3 with a9.")
	fmt.Println("  runes: right; allocates only for the 1,000-rune input, because a []rune that does")
	fmt.Println("  not escape and has <= 32 runes lives in a stack buffer (runtime stringtoslicerune).")
	fmt.Println("  decode: right, 0 allocations always, the same comparisons as runes.")
	fmt.Println(`  NFD: IsLetter(U+0301) is false, so the accent is skipped like a comma and "e\u0301te"`)
	fmt.Println("  passes as \"ete\"; the same text in NFC fails. Normalize first (example 15).")

	fmt.Println()
	fmt.Println("(b) valid palindrome II: delete at most one rune (no skipping, no folding)")
	fmt.Println()
	for _, s := range []string{"racecar", "abca", "abc", "deeee", "ébcé"} {
		del, cmps := "", 0
		ok := pal2(s, 1, &del, &cmps)
		how := "nothing to delete"
		if !ok {
			how = "one deletion is not enough"
		} else if del != "" {
			how = "delete '" + del + "'"
		}
		fmt.Printf("  %-9s %-4s %-28s comparisons %d\n", `"`+s+`"`, yn(ok), how, cmps)
	}
	fmt.Println()
	fmt.Println("(c) case folding is rune -> rune in Go")
	r := unicode.ToLower('İ')
	ok, _ := palDecode("İi")
	fmt.Printf("  unicode.ToLower('İ') = %U (one rune), so palDecode(\"İi\") = %v\n", r, ok)

	fmt.Println()
	fmt.Println("(d) oracle check: 20,000 random strings over {a, b, é, É, space, comma}, half mirrored")
	alpha := []rune{'a', 'b', 'é', 'É', ' ', ','}
	rng := rand.New(rand.NewPCG(1, 2))
	oracle := func(s string) bool { // filter, fold, compare with the reverse
		var r []rune
		for _, c := range s {
			if keep(c) {
				r = append(r, unicode.ToLower(c))
			}
		}
		rev := slices.Clone(r)
		slices.Reverse(rev)
		return slices.Equal(r, rev)
	}
	var wrong [3]int
	pals := 0
	for range 20000 {
		w := make([]rune, rng.IntN(8))
		for i := range w {
			w[i] = alpha[rng.IntN(len(alpha))]
		}
		if rng.IntN(2) == 0 { // mirror: w + reverse(w)
			rev := slices.Clone(w)
			slices.Reverse(rev)
			w = append(w, rev...)
		}
		s := string(w)
		truth := oracle(s)
		if truth {
			pals++
		}
		for k, walk := range walks {
			if got, _ := walk(s); got != truth {
				wrong[k]++
			}
		}
	}
	fmt.Printf("  palindromes %d   wrong: bytes %d   runes %d   decode %d\n",
		pals, wrong[0], wrong[1], wrong[2])
}
```

**Output:**

```
(a) three walks; '!' = disagrees with what a reader sees

  input                              reads bytes runes decode    cmps b/r/d allocs b/r/d
  "racecar"                          yes   yes   yes   yes            3/3/3       0/0/0
  "A man, a plan, a canal: Panama"   yes   yes   yes   yes         10/10/10       0/0/0
  "No 'x' in Nixon"                  yes   yes   yes   yes            5/5/5       0/0/0
  "été"                              yes   no!   yes   yes            1/1/1       0/0/0
  "ab"                               no    no    no    no             1/1/1       0/0/0
  ""                                 yes   yes   yes   yes            0/0/0       0/0/0
  "été " x 250                       yes   no!   yes   yes        1/375/375       0/1/0
  NFD "e\u0301te\u0301" (été)        yes   no!   yes   yes            1/1/1       0/0/0
  NFD "e\u0301te" (éte)              no    no    yes!  yes!           2/1/1       0/0/0
  NFC "\u00e9te" (éte)               no    no    no    no             1/1/1       0/0/0

  bytes: wrong on every non-ASCII palindrome -- on "été" it compares c3 with a9.
  runes: right; allocates only for the 1,000-rune input, because a []rune that does
  not escape and has <= 32 runes lives in a stack buffer (runtime stringtoslicerune).
  decode: right, 0 allocations always, the same comparisons as runes.
  NFD: IsLetter(U+0301) is false, so the accent is skipped like a comma and "e\u0301te"
  passes as "ete"; the same text in NFC fails. Normalize first (example 15).

(b) valid palindrome II: delete at most one rune (no skipping, no folding)

  "racecar" yes  nothing to delete            comparisons 3
  "abca"    yes  delete 'b'                   comparisons 2
  "abc"     no   one deletion is not enough   comparisons 3
  "deeee"   yes  delete 'd'                   comparisons 3
  "ébcé"    yes  delete 'b'                   comparisons 2

(c) case folding is rune -> rune in Go
  unicode.ToLower('İ') = U+0069 (one rune), so palDecode("İi") = true

(d) oracle check: 20,000 random strings over {a, b, é, É, space, comma}, half mirrored
  palindromes 15311   wrong: bytes 8496   runes 0   decode 0
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// A palindrome check is two pointers walking in from both ends, skipping what
// is not a letter or digit and folding case. The algorithm is trivial; the
// question is what the pointers step over: bytes, chars, or decoded chars.

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // alloc + realloc events

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

/// average allocation events per call, after one warm-up call (like AllocsPerRun)
fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> f64 {
    f();
    let a0 = ALLOCS.load(Relaxed);
    for _ in 0..runs {
        f();
    }
    (ALLOCS.load(Relaxed) - a0) as f64 / runs as f64
}

// Pcg: a port of Go's math/rand/v2 PCG-DXSM and IntN, so this program sees
// exactly the random strings the Go program sees.
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
    /// like rand.IntN(n): exactly uniform in [0, n)
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
}

// keep is "isalnum". Rust's is_alphanumeric = Alphabetic || Numeric, which is
// close to Go's IsLetter || IsDigit but not equal (Numeric also takes '½').
fn keep(c: char) -> bool {
    c.is_alphanumeric()
}

// Rust's case mapping is the FULL one: to_lowercase() yields an iterator,
// because one char may lower to several. Compare the iterators.
fn same_fold(a: char, b: char) -> bool {
    a.to_lowercase().eq(b.to_lowercase())
}

// pal_bytes steps over BYTES. It cannot tell what a byte >= 0x80 means, so it
// keeps it as "a letter" -- and compares halves of letters with each other.
fn pal_bytes(s: &str) -> (bool, u32) {
    let b = s.as_bytes();
    let kept = |c: u8| c >= 0x80 || c.is_ascii_alphanumeric();
    let mut cmps = 0;
    let (mut i, mut j) = (0usize, b.len()); // j is one past the right pointer
    loop {
        while i < j && !kept(b[i]) {
            i += 1;
        }
        while i < j && !kept(b[j - 1]) {
            j -= 1;
        }
        if j - i <= 1 {
            return (true, cmps);
        }
        cmps += 1;
        if !b[i].eq_ignore_ascii_case(&b[j - 1]) {
            return (false, cmps);
        }
        i += 1;
        j -= 1;
    }
}

// pal_chars collects a Vec<char> first: right, but 4 bytes per char.
fn pal_chars(s: &str) -> (bool, u32) {
    let r: Vec<char> = s.chars().collect();
    let mut cmps = 0;
    let (mut i, mut j) = (0usize, r.len());
    loop {
        while i < j && !keep(r[i]) {
            i += 1;
        }
        while i < j && !keep(r[j - 1]) {
            j -= 1;
        }
        if j - i <= 1 {
            return (true, cmps);
        }
        cmps += 1;
        if !same_fold(r[i], r[j - 1]) {
            return (false, cmps);
        }
        i += 1;
        j -= 1;
    }
}

// pal_decode: Chars is a DoubleEndedIterator, so next() decodes from the
// front and next_back() from the back -- Go's DecodeRuneInString and
// DecodeLastRuneInString in one type. Filtering keeps it double-ended, and
// the two ends can never cross: right AND allocation-free.
fn pal_decode(s: &str) -> (bool, u32) {
    let mut it = s.chars().filter(|&c| keep(c));
    let mut cmps = 0;
    loop {
        let Some(a) = it.next() else {
            return (true, cmps);
        };
        let Some(b) = it.next_back() else {
            return (true, cmps); // a was the middle char
        };
        cmps += 1;
        if !same_fold(a, b) {
            return (false, cmps);
        }
    }
}

// pal2 is "valid palindrome II" (interview version: no skipping, no folding).
// On the first mismatch it tries both deletions, each one more Theta(n) scan
// with skips = 0 -- so the whole thing is still Theta(n). char_indices gives
// byte offsets of char starts, so every slice below is on a char boundary.
fn pal2(t: &str, skips: u32, deleted: &mut Option<char>, cmps: &mut u32) -> bool {
    let mut it = t.char_indices();
    loop {
        let (Some((i, a)), Some((j, b))) = (it.next(), it.next_back()) else {
            return true; // met, or a single char left
        };
        *cmps += 1;
        if a != b {
            if skips == 0 {
                return false;
            }
            if pal2(&t[i + a.len_utf8()..j + b.len_utf8()], 0, deleted, cmps) {
                *deleted = Some(a);
                return true;
            }
            *deleted = Some(b);
            return pal2(&t[i..j], 0, deleted, cmps);
        }
    }
}

/// a palindrome walk: (is it one?, comparisons made)
type Walk = fn(&str) -> (bool, u32);

fn yn(v: bool) -> &'static str {
    if v { "yes" } else { "no" }
}

fn main() {
    let long = "été ".repeat(250);
    let inputs: [(&str, &str, &str); 10] = [
        ("\"racecar\"", "racecar", "yes"),
        (
            "\"A man, a plan, a canal: Panama\"",
            "A man, a plan, a canal: Panama",
            "yes",
        ),
        ("\"No 'x' in Nixon\"", "No 'x' in Nixon", "yes"),
        ("\"été\"", "été", "yes"),
        ("\"ab\"", "ab", "no"),
        ("\"\"", "", "yes"),
        ("\"été \" x 250", &long, "yes"),
        (
            r#"NFD "e\u{301}te\u{301}" (été)"#,
            "e\u{301}te\u{301}",
            "yes",
        ),
        (r#"NFD "e\u{301}te" (éte)"#, "e\u{301}te", "no"),
        (r#"NFC "\u{e9}te" (éte)"#, "\u{e9}te", "no"),
    ];
    let walks: [Walk; 3] = [pal_bytes, pal_chars, pal_decode];
    println!("(a) three walks; '!' = disagrees with what a reader sees");
    println!();
    println!(
        "  {:<34} {:<5} {:<5} {:<5} {:<6} {:>13} {:>11}",
        "input", "reads", "bytes", "chars", "decode", "cmps b/c/d", "allocs b/c/d"
    );
    let (mut lo, mut hi) = (f64::MAX, 0f64); // pal_chars allocations, non-empty inputs
    for (label, s, reads) in inputs {
        let mut res = [String::new(), String::new(), String::new()];
        let (mut cmps, mut allocs) = ([0u32; 3], [0f64; 3]);
        for (k, walk) in walks.iter().enumerate() {
            let (ok, n) = walk(s);
            res[k] = yn(ok).to_string();
            if res[k] != reads {
                res[k].push('!');
            }
            cmps[k] = n;
            allocs[k] = allocs_per_run(20, || {
                black_box(walk(black_box(s)));
            });
        }
        if !s.is_empty() {
            (lo, hi) = (lo.min(allocs[1]), hi.max(allocs[1]));
        }
        println!(
            "  {:<34} {:<5} {:<5} {:<5} {:<6} {:>13} {:>11}",
            label,
            reads,
            res[0],
            res[1],
            res[2],
            format!("{}/{}/{}", cmps[0], cmps[1], cmps[2]),
            format!("{:.0}/{:.0}/{:.0}", allocs[0], allocs[1], allocs[2])
        );
    }
    println!();
    println!("  bytes: wrong on every non-ASCII palindrome -- on \"été\" it compares c3 with a9.");
    println!(
        "  chars: right, but collect() sizes the Vec from Chars' size_hint lower bound (bytes/4,"
    );
    println!(
        "  rounded up) and regrows: {lo:.0}-{hi:.0} allocation events on every non-empty input."
    );
    println!("  Rust has no stack buffer for a short Vec the way Go's []rune(s) has one.");
    println!("  decode: right, 0 allocations always, the same comparisons as chars.");
    println!(
        r#"  NFD: '\u{{301}}'.is_alphanumeric() is false, so the accent is skipped like a comma and"#
    );
    println!(
        r#"  "e\u{{301}}te" passes as "ete"; the same text in NFC fails. Normalize first (example 15)."#
    );

    println!();
    println!("(b) valid palindrome II: delete at most one char (no skipping, no folding)");
    println!();
    for s in ["racecar", "abca", "abc", "deeee", "ébcé"] {
        let (mut del, mut cmps) = (None, 0);
        let ok = pal2(s, 1, &mut del, &mut cmps);
        let how = match (ok, del) {
            (false, _) => "one deletion is not enough".to_string(),
            (true, None) => "nothing to delete".to_string(),
            (true, Some(c)) => format!("delete '{c}'"),
        };
        println!(
            "  {:<9} {:<4} {:<28} comparisons {}",
            format!("\"{s}\""),
            yn(ok),
            how,
            cmps
        );
    }

    println!();
    println!("(c) case folding is the full mapping in Rust");
    let lower: Vec<String> = 'İ'
        .to_lowercase()
        .map(|c| format!("U+{:04X}", c as u32))
        .collect();
    let (ok, _) = pal_decode("İi");
    println!(
        "  'İ'.to_lowercase() = {} ({} chars), so pal_decode(\"İi\") = {}",
        lower.join(" "),
        lower.len(),
        ok
    );

    println!();
    println!(
        "(d) oracle check: 20,000 random strings over {{a, b, é, É, space, comma}}, half mirrored"
    );
    let alpha = ['a', 'b', 'é', 'É', ' ', ','];
    let mut rng = Pcg::new(1, 2);
    // the oracle: filter, fold, compare with the reverse
    let oracle = |s: &str| {
        let r: Vec<char> = s
            .chars()
            .filter(|&c| keep(c))
            .flat_map(char::to_lowercase)
            .collect();
        r.iter().eq(r.iter().rev())
    };
    let (mut wrong, mut pals) = ([0u32; 3], 0);
    for _ in 0..20000 {
        let k = rng.int_n(8);
        let mut w: Vec<char> = (0..k).map(|_| alpha[rng.int_n(alpha.len())]).collect();
        if rng.int_n(2) == 0 {
            // mirror: w + reverse(w)
            let rev: Vec<char> = w.iter().rev().copied().collect();
            w.extend(rev);
        }
        let s: String = w.into_iter().collect();
        let truth = oracle(&s);
        if truth {
            pals += 1;
        }
        for (k, walk) in walks.iter().enumerate() {
            if walk(&s).0 != truth {
                wrong[k] += 1;
            }
        }
    }
    println!(
        "  palindromes {pals}   wrong: bytes {}   chars {}   decode {}",
        wrong[0], wrong[1], wrong[2]
    );
}
```

**Rust output:**

```
(a) three walks; '!' = disagrees with what a reader sees

  input                              reads bytes chars decode    cmps b/c/d allocs b/c/d
  "racecar"                          yes   yes   yes   yes            3/3/3       0/2/0
  "A man, a plan, a canal: Panama"   yes   yes   yes   yes         10/10/10       0/3/0
  "No 'x' in Nixon"                  yes   yes   yes   yes            5/5/5       0/3/0
  "été"                              yes   no!   yes   yes            1/1/1       0/1/0
  "ab"                               no    no    no    no             1/1/1       0/1/0
  ""                                 yes   yes   yes   yes            0/0/0       0/0/0
  "été " x 250                       yes   no!   yes   yes        1/375/375       0/3/0
  NFD "e\u{301}te\u{301}" (été)      yes   no!   yes   yes            1/1/1       0/2/0
  NFD "e\u{301}te" (éte)             no    no    yes!  yes!           2/1/1       0/1/0
  NFC "\u{e9}te" (éte)               no    no    no    no             1/1/1       0/1/0

  bytes: wrong on every non-ASCII palindrome -- on "été" it compares c3 with a9.
  chars: right, but collect() sizes the Vec from Chars' size_hint lower bound (bytes/4,
  rounded up) and regrows: 1-3 allocation events on every non-empty input.
  Rust has no stack buffer for a short Vec the way Go's []rune(s) has one.
  decode: right, 0 allocations always, the same comparisons as chars.
  NFD: '\u{301}'.is_alphanumeric() is false, so the accent is skipped like a comma and
  "e\u{301}te" passes as "ete"; the same text in NFC fails. Normalize first (example 15).

(b) valid palindrome II: delete at most one char (no skipping, no folding)

  "racecar" yes  nothing to delete            comparisons 3
  "abca"    yes  delete 'b'                   comparisons 2
  "abc"     no   one deletion is not enough   comparisons 3
  "deeee"   yes  delete 'd'                   comparisons 3
  "ébcé"    yes  delete 'b'                   comparisons 2

(c) case folding is the full mapping in Rust
  'İ'.to_lowercase() = U+0069 U+0307 (2 chars), so pal_decode("İi") = false

(d) oracle check: 20,000 random strings over {a, b, é, É, space, comma}, half mirrored
  palindromes 15311   wrong: bytes 8496   chars 0   decode 0
```

> **Go vs Rust:** Every answer, every comparison count and the oracle run (15,311 palindromes; bytes wrong 8,496, the other two 0) are identical — same algorithm, same PCG strings. Two real differences: collecting a `Vec<char>` costs 1–3 allocation events per input (`collect` starts from `Chars`' lower size hint and regrows), where Go's `[]rune` costs 0 up to 32 runes thanks to a stack buffer and 1 at 1,000 runes; and Rust's `to_lowercase` is the full mapping, so `'İ'` lowers to two chars and `"İi"` is not a palindrome in Rust while Go's rune→rune `unicode.ToLower` says it is. `is_alphanumeric` (Alphabetic ∪ Numeric) is close to Go's `IsLetter || IsDigit` but wider (it accepts `'½'`); on these inputs they agree.

**Complexity:** Θ(n) time for all three walks, ⌊k/2⌋ comparisons for k kept runes (375 for the 1,000-rune input) · bytes and decode Θ(1) space and 0 allocations; `[]rune` Θ(n) space — 1 allocation at 1,000 runes, 0 up to 32 runes thanks to the runtime's stack buffer · valid palindrome II Θ(n): at most one branch, each Θ(n) · the byte walk was wrong on 8,496 of 20,000 random strings, the rune walks on none.

---

## 9. Reversing bytes, runes and words

`🟡 medium` · *Reverse words*

Reverse the **bytes** of a string and every multi-byte letter breaks, because its lead byte ends up last. Reverse
the **runes** and letters survive, but a combining mark moves onto the wrong letter. Reverse the **words** with the
classic in-place trick (reverse the whole buffer, then reverse each word back) and the result is valid UTF-8,
because every byte of a word is reversed twice and an ASCII space can never match inside a multi-byte letter. All
of them are Θ(n). The in-place version is the right tool when you own a `[]byte`: Θ(1) extra space and zero
allocations, where `Fields` + `Join` costs one slice and one string.

**Steps:**

1. Reverse `"héllo"` by bytes and by runes, and NFC vs NFD `"café"` by runes, printing bytes and code points.
2. Reverse words two ways: `Fields` + `slices.Reverse` + `Join`, and in place with a write index, counting allocations, swaps and moved bytes.
3. Show why `Split(s, " ")` is the wrong splitter, then check the in-place version against the library one on 10,000 random mixed-script strings.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
	"unicode/utf8"
)

// "Reverse a string" has three answers, and only the unit you reverse decides
// whether the result is still text:
//
//   bytes  breaks every multi-byte letter (its lead byte ends up last)
//   runes  keeps letters whole, but a combining mark lands on the wrong letter
//   words  reverse everything, then reverse each word back: every byte of a
//          word is reversed TWICE, so its letters come out intact
//
// The last one is safe on UTF-8 as long as the separator is ASCII: no byte of
// a multi-byte rune is below 0x80, so a ' ' can never be found inside a letter.

func reverseBytes(s string) []byte {
	b := []byte(s)
	slices.Reverse(b)
	return b
}

func reverseRunes(s string) string {
	r := []rune(s)
	slices.Reverse(r)
	return string(r)
}

// Version A: the library way -- one []string (the words are substrings of s,
// they share its bytes), reversed in place, joined into one new string.
func reverseWordsFields(s string) string {
	w := strings.Fields(s)
	slices.Reverse(w)
	return strings.Join(w, " ")
}

// Version B: in place on a []byte. Reverse the whole buffer, then walk it with
// a write index w (lesson 06 example 3): copy each word down to w, collapsing
// runs of spaces to one, and reverse that word back. No allocation at all.
func reverseWordsInPlace(b []byte) (out []byte, swaps, moved int) {
	rev := func(x []byte) { slices.Reverse(x); swaps += len(x) / 2 }
	rev(b)
	w := 0
	for i := 0; i < len(b); {
		if b[i] == ' ' {
			i++
			continue
		}
		if w > 0 { // one space between words, none at the ends
			b[w] = ' '
			w++
		}
		start := w
		for ; i < len(b) && b[i] != ' '; i, w = i+1, w+1 {
			if w != i {
				b[w] = b[i] // w <= i always, so this never overwrites unread bytes
				moved++
			}
		}
		rev(b[start:w])
	}
	return b[:w], swaps, moved
}

// codePoints prints runes as U+XXXX so an invisible combining mark shows up.
func codePoints(s string) string {
	var parts []string
	for _, r := range s {
		parts = append(parts, fmt.Sprintf("U+%04X", r))
	}
	return strings.Join(parts, " ")
}

var sinkS string
var sinkB []byte

func main() {
	fmt.Println("(a) reversing the units of one word")
	fmt.Println()
	b := reverseBytes("héllo")
	fmt.Printf("  bytes of \"héllo\"  % x   ->  % x\n", []byte("héllo"), b)
	fmt.Printf("    as a string: %q   utf8.Valid: %v  (Go keeps the broken bytes anyway)\n", b, utf8.Valid(b))
	fmt.Printf("  runes of \"héllo\"  -> %q   valid: %v\n", reverseRunes("héllo"), utf8.ValidString(reverseRunes("héllo")))
	for _, c := range []struct{ label, s string }{
		{`"caf\u00e9" (NFC)`, "caf\u00e9"}, {`"cafe\u0301" (NFD)`, "cafe\u0301"}} {
		fmt.Printf("  runes of %-18s %-34s ->  %s\n", c.label, codePoints(c.s), codePoints(reverseRunes(c.s)))
	}
	fmt.Println("    NFC: the é is one rune and survives. NFD: U+0301 now comes FIRST, with no")
	fmt.Println("    letter before it, and the e lost its accent -- valid UTF-8, wrong text.")

	fmt.Println()
	fmt.Println("(b) reverse the words: A = Fields + Reverse + Join, B = in place on a []byte")
	fmt.Println()
	buf := make([]byte, 64)
	for _, s := range []string{"  the sky   is blue  ", "naïve café über", "a", ""} {
		a := reverseWordsFields(s)
		in := []byte(s) // B works on its own copy; this copy is the only allocation
		out, swaps, moved := reverseWordsInPlace(in)
		allocA := testing.AllocsPerRun(50, func() { sinkS = reverseWordsFields(s) })
		allocB := testing.AllocsPerRun(50, func() {
			n := copy(buf, s)
			sinkB, _, _ = reverseWordsInPlace(buf[:n])
		})
		fmt.Printf("  in  [%s]  len %d\n", s, len(s))
		fmt.Printf("    A [%s]  allocs %.0f\n", a, allocA)
		fmt.Printf("    B [%s]  allocs %.0f  swaps %d  bytes moved %d  valid %v  same as A: %v\n",
			out, allocB, swaps, moved, utf8.Valid(out), string(out) == a)
	}
	fmt.Println("  A: one []string from Fields + one string from Join (a lone word is returned as is).")
	fmt.Println("  B: no allocation; accented words come back valid (each of their bytes is reversed twice).")

	fmt.Println()
	fmt.Println("(c) why A uses Fields, not Split(s, \" \")")
	parts := strings.Split("  the sky   is blue  ", " ")
	fmt.Printf("  Split gives %d pieces: %q\n", len(parts), parts)
	fmt.Printf("  Fields gives %d: %q\n", len(strings.Fields("  the sky   is blue  ")),
		strings.Fields("  the sky   is blue  "))

	fmt.Println()
	fmt.Println("(d) 10,000 random strings over {a, é, €, 😀, space}, 0-12 runes (PCG seed 1,2)")
	alpha := []string{"a", "é", "€", "😀", " "}
	rng := rand.New(rand.NewPCG(1, 2))
	diff, invalidB, invalidBytes, totalSwaps, totalLen := 0, 0, 0, 0, 0
	for range 10000 {
		var sb strings.Builder
		for range rng.IntN(13) {
			sb.WriteString(alpha[rng.IntN(len(alpha))])
		}
		s := sb.String()
		out, swaps, _ := reverseWordsInPlace([]byte(s))
		if string(out) != reverseWordsFields(s) {
			diff++
		}
		if !utf8.Valid(out) {
			invalidB++
		}
		if !utf8.Valid(reverseBytes(s)) {
			invalidBytes++
		}
		totalSwaps += swaps
		totalLen += len(s)
	}
	fmt.Printf("  B differs from A: %d   B output invalid: %d\n", diff, invalidB)
	fmt.Printf("  plain byte reversal invalid: %d of 10,000\n", invalidBytes)
	fmt.Printf("  swaps: %d for %d input bytes = %.2f per byte (at most 1: n/2 + n/2)\n",
		totalSwaps, totalLen, float64(totalSwaps)/float64(totalLen))
}
```

**Output:**

```
(a) reversing the units of one word

  bytes of "héllo"  68 c3 a9 6c 6c 6f   ->  6f 6c 6c a9 c3 68
    as a string: "oll\xa9\xc3h"   utf8.Valid: false  (Go keeps the broken bytes anyway)
  runes of "héllo"  -> "olléh"   valid: true
  runes of "caf\u00e9" (NFC)  U+0063 U+0061 U+0066 U+00E9        ->  U+00E9 U+0066 U+0061 U+0063
  runes of "cafe\u0301" (NFD) U+0063 U+0061 U+0066 U+0065 U+0301 ->  U+0301 U+0065 U+0066 U+0061 U+0063
    NFC: the é is one rune and survives. NFD: U+0301 now comes FIRST, with no
    letter before it, and the e lost its accent -- valid UTF-8, wrong text.

(b) reverse the words: A = Fields + Reverse + Join, B = in place on a []byte

  in  [  the sky   is blue  ]  len 21
    A [blue is sky the]  allocs 2
    B [blue is sky the]  allocs 0  swaps 15  bytes moved 12  valid true  same as A: true
  in  [naïve café über]  len 18
    A [über café naïve]  allocs 2
    B [über café naïve]  allocs 0  swaps 16  bytes moved 0  valid true  same as A: true
  in  [a]  len 1
    A [a]  allocs 1
    B [a]  allocs 0  swaps 0  bytes moved 0  valid true  same as A: true
  in  []  len 0
    A []  allocs 0
    B []  allocs 0  swaps 0  bytes moved 0  valid true  same as A: true
  A: one []string from Fields + one string from Join (a lone word is returned as is).
  B: no allocation; accented words come back valid (each of their bytes is reversed twice).

(c) why A uses Fields, not Split(s, " ")
  Split gives 10 pieces: ["" "" "the" "sky" "" "" "is" "blue" "" ""]
  Fields gives 4: ["the" "sky" "is" "blue"]

(d) 10,000 random strings over {a, é, €, 😀, space}, 0-12 runes (PCG seed 1,2)
  B differs from A: 0   B output invalid: 0
  plain byte reversal invalid: 8727 of 10,000
  swaps: 119795 for 131924 input bytes = 0.91 per byte (at most 1: n/2 + n/2)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// "Reverse a string" has three answers, and only the unit you reverse decides
// whether the result is still text:
//
//   bytes  breaks every multi-byte letter (its lead byte ends up last)
//   chars  keeps letters whole, but a combining mark lands on the wrong letter
//   words  reverse everything, then reverse each word back: every byte of a
//          word is reversed TWICE, so its letters come out intact
//
// The last one is safe on UTF-8 as long as the separator is ASCII: no byte of
// a multi-byte char is below 0x80, so a b' ' can never be found inside a letter.
// Rust adds one thing Go does not: a broken result cannot BECOME a String.

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // alloc + realloc events

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

/// average allocation events per call, after one warm-up call (like AllocsPerRun)
fn allocs_per_run(runs: usize, mut f: impl FnMut()) -> f64 {
    f();
    let a0 = ALLOCS.load(Relaxed);
    for _ in 0..runs {
        f();
    }
    (ALLOCS.load(Relaxed) - a0) as f64 / runs as f64
}

// Pcg: a port of Go's math/rand/v2 PCG-DXSM and IntN, so this program sees
// exactly the random strings the Go program sees.
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
    /// like rand.IntN(n): exactly uniform in [0, n)
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
        if n & (n - 1) == 0 {
            return (self.next_u64() & (n - 1)) as usize;
        }
        let mut m = u128::from(self.next_u64()) * u128::from(n);
        if (m as u64) < n {
            let thresh = n.wrapping_neg() % n;
            while (m as u64) < thresh {
                m = u128::from(self.next_u64()) * u128::from(n);
            }
        }
        (m >> 64) as usize
    }
}

// Reversing bytes can only give a Vec<u8>: whether it is text is a separate
// question, asked with String::from_utf8 (which checks, and refuses).
fn reverse_bytes(s: &str) -> Vec<u8> {
    let mut b = s.as_bytes().to_vec();
    b.reverse();
    b
}

// Chars is double-ended, so rev() decodes from the back; no Vec<char> needed.
fn reverse_chars(s: &str) -> String {
    s.chars().rev().collect()
}

// Version A: the library way -- the words are &str views into s, collected
// into one Vec, reversed by the iterator and joined into one new String.
fn reverse_words_split(s: &str) -> String {
    s.split_whitespace().rev().collect::<Vec<&str>>().join(" ")
}

// Version B: in place on a Vec<u8>. Reverse the whole buffer, then walk it
// with a write index w (lesson 06 example 3): copy each word down to w,
// collapsing runs of spaces to one, and reverse that word back. truncate()
// keeps the capacity, so there is no allocation at all.
fn reverse_words_in_place(b: &mut Vec<u8>) -> (usize, usize) {
    let (mut swaps, mut moved) = (b.len() / 2, 0);
    b.reverse();
    let (mut w, mut i) = (0, 0);
    while i < b.len() {
        if b[i] == b' ' {
            i += 1;
            continue;
        }
        if w > 0 {
            // one space between words, none at the ends
            b[w] = b' ';
            w += 1;
        }
        let start = w;
        while i < b.len() && b[i] != b' ' {
            if w != i {
                b[w] = b[i]; // w <= i always, so this never overwrites unread bytes
                moved += 1;
            }
            i += 1;
            w += 1;
        }
        b[start..w].reverse();
        swaps += (w - start) / 2;
    }
    b.truncate(w);
    (swaps, moved)
}

// code_points prints chars as U+XXXX so an invisible combining mark shows up.
fn code_points(s: &str) -> String {
    let v: Vec<String> = s.chars().map(|c| format!("U+{:04X}", c as u32)).collect();
    v.join(" ")
}

fn hex(b: &[u8]) -> String {
    b.iter()
        .map(|x| format!("{x:02x}"))
        .collect::<Vec<_>>()
        .join(" ")
}

// Go's %q of a []string, by hand: ["" "the" ...]
fn quoted(v: &[&str]) -> String {
    let q: Vec<String> = v.iter().map(|p| format!("\"{p}\"")).collect();
    format!("[{}]", q.join(" "))
}

fn main() {
    println!("(a) reversing the units of one word");
    println!();
    let b = reverse_bytes("héllo");
    println!(
        "  bytes of \"héllo\"  {}   ->  {}",
        hex("héllo".as_bytes()),
        hex(&b)
    );
    match String::from_utf8(b) {
        Ok(s) => println!("    as a String: {s}"),
        Err(e) => println!("    String::from_utf8: Err({e})  -- refused"),
    }
    let r = reverse_chars("héllo");
    println!("  chars of \"héllo\"  -> \"{r}\"   (a String, so valid by construction)");
    for (label, s) in [
        (r#""caf\u{e9}" (NFC)"#, "caf\u{e9}"),
        (r#""cafe\u{301}" (NFD)"#, "cafe\u{301}"),
    ] {
        println!(
            "  chars of {:<18} {:<34} ->  {}",
            label,
            code_points(s),
            code_points(&reverse_chars(s))
        );
    }
    println!("    NFC: the é is one char and survives. NFD: U+0301 now comes FIRST, with no");
    println!("    letter before it, and the e lost its accent -- valid UTF-8, wrong text.");

    println!();
    println!("(b) reverse the words: A = split_whitespace + rev + join, B = in place on a Vec<u8>");
    println!();
    let mut buf: Vec<u8> = Vec::with_capacity(64);
    for s in ["  the sky   is blue  ", "naïve café über", "a", ""] {
        let a = reverse_words_split(s);
        let mut v = s.as_bytes().to_vec(); // B works on its own copy: the only allocation
        let (swaps, moved) = reverse_words_in_place(&mut v);
        let alloc_a = allocs_per_run(50, || {
            black_box(reverse_words_split(black_box(s)));
        });
        let alloc_b = allocs_per_run(50, || {
            buf.clear();
            buf.extend_from_slice(black_box(s).as_bytes()); // fits in the capacity: no allocation
            black_box(reverse_words_in_place(&mut buf));
        });
        // String::from_utf8 takes the Vec by value: Ok means valid, and it reuses the buffer
        let out = String::from_utf8(v).expect("word reversal keeps UTF-8 valid");
        println!("  in  [{s}]  len {}", s.len());
        println!("    A [{a}]  allocs {alloc_a:.0}");
        println!(
            "    B [{out}]  allocs {alloc_b:.0}  swaps {swaps}  bytes moved {moved}  from_utf8 Ok  same as A: {}",
            out == a
        );
    }
    println!("  A: one Vec<&str> from collect + one String from join (join copies even one word).");
    println!(
        "  B: no allocation; accented words come back valid (each of their bytes is reversed twice)."
    );

    println!();
    println!("(c) why A uses split_whitespace, not split(' ')");
    let s = "  the sky   is blue  ";
    let parts: Vec<&str> = s.split(' ').collect();
    println!("  split gives {} pieces: {}", parts.len(), quoted(&parts));
    let words: Vec<&str> = s.split_whitespace().collect();
    println!(
        "  split_whitespace gives {}: {}",
        words.len(),
        quoted(&words)
    );

    println!();
    println!("(d) 10,000 random strings over {{a, é, €, 😀, space}}, 0-12 chars (PCG seed 1,2)");
    let alpha = ["a", "é", "€", "😀", " "];
    let mut rng = Pcg::new(1, 2);
    let (mut diff, mut invalid_b, mut invalid_bytes, mut total_swaps, mut total_len) =
        (0, 0, 0, 0, 0);
    for _ in 0..10000 {
        let k = rng.int_n(13);
        let s: String = (0..k).map(|_| alpha[rng.int_n(alpha.len())]).collect();
        let mut v = s.as_bytes().to_vec();
        let (swaps, _) = reverse_words_in_place(&mut v);
        match String::from_utf8(v) {
            Ok(out) if out == reverse_words_split(&s) => {}
            Ok(_) => diff += 1,
            Err(_) => invalid_b += 1,
        }
        if String::from_utf8(reverse_bytes(&s)).is_err() {
            invalid_bytes += 1;
        }
        total_swaps += swaps;
        total_len += s.len();
    }
    println!("  B differs from A: {diff}   B output invalid: {invalid_b}");
    println!("  plain byte reversal invalid: {invalid_bytes} of 10,000");
    println!(
        "  swaps: {total_swaps} for {total_len} input bytes = {:.2} per byte (at most 1: n/2 + n/2)",
        total_swaps as f64 / total_len as f64
    );
}
```

**Rust output:**

```
(a) reversing the units of one word

  bytes of "héllo"  68 c3 a9 6c 6c 6f   ->  6f 6c 6c a9 c3 68
    String::from_utf8: Err(invalid utf-8 sequence of 1 bytes from index 3)  -- refused
  chars of "héllo"  -> "olléh"   (a String, so valid by construction)
  chars of "caf\u{e9}" (NFC)  U+0063 U+0061 U+0066 U+00E9        ->  U+00E9 U+0066 U+0061 U+0063
  chars of "cafe\u{301}" (NFD) U+0063 U+0061 U+0066 U+0065 U+0301 ->  U+0301 U+0065 U+0066 U+0061 U+0063
    NFC: the é is one char and survives. NFD: U+0301 now comes FIRST, with no
    letter before it, and the e lost its accent -- valid UTF-8, wrong text.

(b) reverse the words: A = split_whitespace + rev + join, B = in place on a Vec<u8>

  in  [  the sky   is blue  ]  len 21
    A [blue is sky the]  allocs 2
    B [blue is sky the]  allocs 0  swaps 15  bytes moved 12  from_utf8 Ok  same as A: true
  in  [naïve café über]  len 18
    A [über café naïve]  allocs 2
    B [über café naïve]  allocs 0  swaps 16  bytes moved 0  from_utf8 Ok  same as A: true
  in  [a]  len 1
    A [a]  allocs 2
    B [a]  allocs 0  swaps 0  bytes moved 0  from_utf8 Ok  same as A: true
  in  []  len 0
    A []  allocs 0
    B []  allocs 0  swaps 0  bytes moved 0  from_utf8 Ok  same as A: true
  A: one Vec<&str> from collect + one String from join (join copies even one word).
  B: no allocation; accented words come back valid (each of their bytes is reversed twice).

(c) why A uses split_whitespace, not split(' ')
  split gives 10 pieces: ["" "" "the" "sky" "" "" "is" "blue" "" ""]
  split_whitespace gives 4: ["the" "sky" "is" "blue"]

(d) 10,000 random strings over {a, é, €, 😀, space}, 0-12 chars (PCG seed 1,2)
  B differs from A: 0   B output invalid: 0
  plain byte reversal invalid: 8727 of 10,000
  swaps: 119795 for 131924 input bytes = 0.91 per byte (at most 1: n/2 + n/2)
```

> **Go vs Rust:** Swaps, moved bytes, results and the random-set counts (0 differences, 0 invalid, 8,727 invalid byte reversals, 0.91 swaps per byte) are identical. The difference is where invalid UTF-8 can live: Go happily keeps `"oll\xa9\xc3h"` as a string and you must ask `utf8.Valid`, while Rust's reversed bytes stay a `Vec<u8>` and `String::from_utf8` returns `Err`. Allocations agree except for one word: Go's `strings.Join` returns a lone element as is (1 allocation, the `[]string`), Rust's `join` always builds a new `String` (2).

**Complexity:** all Θ(n) · in-place word reversal Θ(1) extra space and 0 allocations, ≤ 1 swap per byte (0.91 on the random set) · `Fields` + `Join`: one slice + one string (2 allocations) · byte reversal is UTF-8-unsafe (8,727 of 10,000 random strings came out invalid), rune reversal is grapheme-unsafe (NFD `"café"` loses its accent), word reversal with ASCII separators is safe (0 invalid, 0 differences from the library version).

---

## 10. Run-length compression, and when it makes things longer

`🟡 medium` · *Compress*

`"aabcccccaaa"` becomes `"a2b1c5a3"`: each run is written as the character and its count, and the original comes
back unchanged when the encoding is not strictly shorter. It is a one-pass Θ(n) problem, and a `strings.Builder`
problem: since anything as long as the input is thrown away, one `Grow(len(s))` is exactly enough and the whole
function makes one allocation. The rest is edge cases: counts of 10 or more need several digits, runs are runs of
**runes** (a byte-wise encoder writes digits into the middle of `é`), the worst case doubles the length, and if the
input may contain digits the output cannot be decoded at all.

**Steps:**

1. Write `rle(s, limit)` with one `Grow(limit)`, digits from `strconv.AppendInt` into a stack buffer, and an early exit once the output would reach `limit`.
2. Compress five inputs and count allocations with `testing.AllocsPerRun`; encode the all-distinct worst case without the "not shorter" rule.
3. Round-trip 10,000 random strings over `ab` and over `a1`, and read the failure count and the first counter-example.

```go
package main

import (
	"errors"
	"fmt"
	"math/rand/v2"
	"strconv"
	"strings"
	"testing"
	"unicode/utf8"
)

// Run-length encoding: "aabcccccaaa" -> "a2b1c5a3". Each run is written as
// the character followed by its count in decimal. Three details decide
// whether an implementation is right:
//
//   - a run is a run of RUNES: "ééé" is one run of 3, not six 1-byte runs;
//   - counts can have several digits: twelve a's are "a12";
//   - the output can be LONGER than the input ("abc" -> "a1b1c1"), so
//     Compress returns the original unless the encoding is strictly shorter.

// sink is package-level so results escape to the heap and allocation counts
// are honest (a non-escaping result could live on the stack).
var sink string

// rle encodes s, but gives up (ok=false) as soon as the output would reach
// limit bytes. That is what makes one Grow enough: the Builder never needs
// more than limit bytes, so it never grows a second time.
func rle(s string, limit int) (out string, ok bool) {
	var b strings.Builder
	b.Grow(limit)    // the one allocation
	var num [20]byte // digits of the count, built on the stack
	for i := 0; i < len(s); {
		r, w := utf8.DecodeRuneInString(s[i:])
		j := i + w
		for j < len(s) && strings.HasPrefix(s[j:], s[i:i+w]) {
			j += w // same rune again: extend the run
		}
		count := (j - i) / w
		digits := strconv.AppendInt(num[:0], int64(count), 10)
		if b.Len()+w+len(digits) >= limit {
			return "", false // would not be shorter than limit: stop early
		}
		b.WriteRune(r)
		b.Write(digits)
		i = j
	}
	return b.String(), true // String() hands over the buffer: no copy
}

// Compress returns the run-length encoding of s, or s itself when the
// encoding is not strictly shorter (in bytes).
func Compress(s string) string {
	if s == "" {
		return s
	}
	if out, ok := rle(s, len(s)); ok {
		return out
	}
	return s
}

// encode is the raw encoder without the "not shorter" rule. The output is at
// most 2 bytes per input byte (an ASCII rune with count 1), so this limit is
// never reached.
func encode(s string) string {
	out, _ := rle(s, 2*len(s)+1)
	return out
}

// decode reads (rune, digits) pairs. It cannot tell a digit that was DATA
// from a digit that was a COUNT -- that is the format's weakness.
func decode(enc string) (string, error) {
	var b strings.Builder
	for i := 0; i < len(enc); {
		r, w := utf8.DecodeRuneInString(enc[i:])
		i += w
		j := i
		for j < len(enc) && enc[j] >= '0' && enc[j] <= '9' {
			j++
		}
		if j == i {
			return "", errors.New("missing count")
		}
		n, err := strconv.Atoi(enc[i:j])
		if err != nil || n > 1<<20 {
			return "", errors.New("bad count " + enc[i:j])
		}
		for range n {
			b.WriteRune(r)
		}
		i = j
	}
	return b.String(), nil
}

// show prints bytes like %q, but by hand, so the Rust twin can match it:
// valid runes as themselves, every invalid byte as \xNN.
func show(s string) string {
	var b strings.Builder
	b.WriteByte('"')
	for i := 0; i < len(s); {
		r, w := utf8.DecodeRuneInString(s[i:])
		if r == utf8.RuneError && w == 1 {
			fmt.Fprintf(&b, `\x%02x`, s[i])
		} else {
			b.WriteRune(r)
		}
		i += w
	}
	b.WriteByte('"')
	return b.String()
}

// byteRLE is the bug: runs of BYTES. Multi-byte letters are split apart and
// digits land between a lead byte and its continuation byte.
func byteRLE(s string) string {
	var b strings.Builder
	for i := 0; i < len(s); {
		j := i
		for j < len(s) && s[j] == s[i] {
			j++
		}
		b.WriteByte(s[i])
		b.WriteString(strconv.Itoa(j - i))
		i = j
	}
	return b.String()
}

func main() {
	fmt.Println("(a) Compress: shorter, or the original")
	fmt.Printf("  %-14s %-12s %6s %6s %7s\n", "input", "output", "bytes", "runes", "allocs")
	allocSet := map[float64]bool{}
	for _, s := range []string{"aabcccccaaa", "abc", strings.Repeat("a", 12), "éééab", ""} {
		out := Compress(s)
		allocs := testing.AllocsPerRun(100, func() { sink = Compress(s) })
		if s != "" {
			allocSet[allocs] = true
		}
		note := ""
		if out == s && s != "" {
			note = "  <- not shorter: original returned"
		}
		fmt.Printf("  %-14s %-12s %2d->%-2d %2d->%-2d %7.0f%s\n", show(s), show(out),
			len(s), len(out), utf8.RuneCountInString(s), utf8.RuneCountInString(out), allocs, note)
	}
	e := Compress("éééab")
	fmt.Printf("  \"éééab\" is shorter in bytes (%d -> %d) but longer in runes (%d -> %d): the rule counts bytes.\n",
		len("éééab"), len(e), utf8.RuneCountInString("éééab"), utf8.RuneCountInString(e))
	if len(allocSet) == 1 && allocSet[1] {
		fmt.Println("  every non-empty input costs exactly 1 allocation -- the Grow: the count digits")
		fmt.Println("  use a stack buffer, String() copies nothing, and \"abc\" pays it and returns the original.")
	} else {
		fmt.Println("  allocations differ between inputs: see the table")
	}

	fmt.Println()
	fmt.Println("runs must be runes, not bytes -- a byte-wise encoder on \"ééé\":")
	bad := byteRLE("ééé")
	fmt.Printf("  byte runs: %s  valid UTF-8: %v\n", show(bad), utf8.ValidString(bad))
	fmt.Printf("  rune runs: %s  valid UTF-8: %v\n", show(encode("ééé")), utf8.ValidString(encode("ééé")))

	fmt.Println()
	fmt.Println("(b) the worst case: every run has length 1")
	for _, s := range []string{"abcdef", "é€😀"} {
		e := encode(s)
		fmt.Printf("  %-10s -> %-16s bytes %2d -> %2d (%.2fx)   runes %d -> %d (%.2fx)\n",
			show(s), show(e), len(s), len(e), float64(len(e))/float64(len(s)),
			utf8.RuneCountInString(s), utf8.RuneCountInString(e),
			float64(utf8.RuneCountInString(e))/float64(utf8.RuneCountInString(s)))
	}
	fmt.Println("  each run adds at least one digit: at most 2x the runes, and 2x the bytes only")
	fmt.Println("  when every run is a single 1-byte rune")

	fmt.Println()
	fmt.Println("(c) round trip decode(encode(s)) == s on 10,000 random strings, length 0..12")
	for _, alphabet := range []string{"ab", "a1"} {
		letters := []rune(alphabet)
		rng := rand.New(rand.NewPCG(1, 2))
		failures, shorter := 0, 0
		first := ""
		for range 10_000 {
			n := rng.IntN(13)
			rs := make([]rune, n)
			for k := range rs {
				rs[k] = letters[rng.IntN(len(letters))]
			}
			s := string(rs)
			if Compress(s) != s {
				shorter++
			}
			got, err := decode(encode(s))
			if err != nil || got != s {
				if failures == 0 {
					first = fmt.Sprintf("%s -> %s -> decodes to %d runes", show(s), show(encode(s)),
						utf8.RuneCountInString(got))
				}
				failures++
			}
		}
		fmt.Printf("  alphabet %-5s failures %5d   Compress was shorter on %d of the 10000\n",
			show(alphabet), failures, shorter)
		if failures > 0 {
			fmt.Printf("    first counter-example: %s\n", first)
		}
	}
	fmt.Println("  same seed: the \"a1\" strings are the \"ab\" strings with b -> 1, so the runs (and")
	fmt.Println("  the shorter count) are identical -- only decoding tells them apart.")
	fmt.Println("  a digit in the input is read back as part of a count: the format needs a")
	fmt.Println("  digit-free alphabet, or an escape character, to be decodable.")
}
```

**Output:**

```
(a) Compress: shorter, or the original
  input          output        bytes  runes  allocs
  "aabcccccaaa"  "a2b1c5a3"   11->8  11->8        1
  "abc"          "abc"         3->3   3->3        1  <- not shorter: original returned
  "aaaaaaaaaaaa" "a12"        12->3  12->3        1
  "éééab"        "é3a1b1"      8->7   5->6        1
  ""             ""            0->0   0->0        0
  "éééab" is shorter in bytes (8 -> 7) but longer in runes (5 -> 6): the rule counts bytes.
  every non-empty input costs exactly 1 allocation -- the Grow: the count digits
  use a stack buffer, String() copies nothing, and "abc" pays it and returns the original.

runs must be runes, not bytes -- a byte-wise encoder on "ééé":
  byte runs: "\xc31\xa91\xc31\xa91\xc31\xa91"  valid UTF-8: false
  rune runs: "é3"  valid UTF-8: true

(b) the worst case: every run has length 1
  "abcdef"   -> "a1b1c1d1e1f1"   bytes  6 -> 12 (2.00x)   runes 6 -> 12 (2.00x)
  "é€😀"      -> "é1€1😀1"         bytes  9 -> 12 (1.33x)   runes 3 -> 6 (2.00x)
  each run adds at least one digit: at most 2x the runes, and 2x the bytes only
  when every run is a single 1-byte rune

(c) round trip decode(encode(s)) == s on 10,000 random strings, length 0..12
  alphabet "ab"  failures     0   Compress was shorter on 2082 of the 10000
  alphabet "a1"  failures  6958   Compress was shorter on 2082 of the 10000
    first counter-example: "aaaaa1a1a1" -> "a511a111a111" -> decodes to 733 runes
  same seed: the "a1" strings are the "ab" strings with b -> 1, so the runs (and
  the shorter count) are identical -- only decoding tells them apart.
  a digit in the input is read back as part of a count: the format needs a
  digit-free alphabet, or an escape character, to be decodable.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::borrow::Cow;
use std::collections::HashSet;
use std::fmt::Write;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Run-length encoding: "aabcccccaaa" -> "a2b1c5a3". Each run is written as
// the character followed by its count in decimal. Three details decide
// whether an implementation is right:
//
//   - a run is a run of CHARS: "ééé" is one run of 3, not six 1-byte runs;
//   - counts can have several digits: twelve a's are "a12";
//   - the output can be LONGER than the input ("abc" -> "a1b1c1"), so
//     compress returns the original unless the encoding is strictly shorter.

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
// Fresh allocations and reallocs are counted separately: a realloc may grow
// in place, so it is not "one more buffer", but it is still a call.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static REALLOCS: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        REALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocs, reallocs) made while running `f`.
fn allocs_of(f: impl FnOnce()) -> (usize, usize) {
    let (a0, r0) = (ALLOCS.load(Relaxed), REALLOCS.load(Relaxed));
    f();
    (ALLOCS.load(Relaxed) - a0, REALLOCS.load(Relaxed) - r0)
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) produces exactly the strings Go tests.
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

/// rle encodes s, but gives up (None) as soon as the output would reach
/// `limit` bytes. That is what makes one with_capacity enough: the String
/// never needs more than `limit` bytes, so it never reallocates.
fn rle(s: &str, limit: usize) -> Option<String> {
    let mut out = String::with_capacity(limit); // the one allocation
    let mut chars = s.chars().peekable();
    while let Some(c) = chars.next() {
        let mut count: u32 = 1;
        while chars.next_if_eq(&c).is_some() {
            count += 1; // same char again: extend the run
        }
        // digits of the count, computed without formatting (and so without
        // allocating): ilog10 of 1..9 is 0, of 10..99 is 1, ...
        let digits = count.ilog10() as usize + 1;
        if out.len() + c.len_utf8() + digits >= limit {
            return None; // would not be shorter than limit: stop early
        }
        // write! goes through fmt::Write for String, i.e. push_str into the
        // existing buffer; no temporary String is created.
        write!(out, "{c}{count}").expect("writing to a String cannot fail");
    }
    Some(out)
}

/// compress returns the run-length encoding of s, or s itself when the
/// encoding is not strictly shorter (in bytes). Cow says "maybe borrowed":
/// Go just returns the same string header; Rust spells the choice in the type.
fn compress(s: &str) -> Cow<'_, str> {
    if s.is_empty() {
        return Cow::Borrowed(s);
    }
    match rle(s, s.len()) {
        Some(out) => Cow::Owned(out),
        None => Cow::Borrowed(s),
    }
}

/// encode is the raw encoder without the "not shorter" rule. The output is at
/// most 2 bytes per input byte (an ASCII char with count 1), so this limit is
/// never reached.
fn encode(s: &str) -> String {
    rle(s, 2 * s.len() + 1).expect("limit is above the worst case")
}

/// decode reads (char, digits) pairs. It cannot tell a digit that was DATA
/// from a digit that was a COUNT -- that is the format's weakness.
fn decode(enc: &str) -> Result<String, String> {
    let mut out = String::new();
    let mut chars = enc.chars().peekable();
    while let Some(c) = chars.next() {
        let mut n: u32 = 0;
        let mut ndigits = 0;
        while let Some(d) = chars.peek().and_then(|d| d.to_digit(10)) {
            chars.next();
            ndigits += 1;
            n = n.saturating_mul(10).saturating_add(d);
        }
        if ndigits == 0 {
            return Err("missing count".to_string());
        }
        if n > 1 << 20 {
            return Err(format!("bad count {n}"));
        }
        out.extend(std::iter::repeat_n(c, n as usize));
    }
    Ok(out)
}

/// show prints bytes the way the Go version does: valid UTF-8 as itself,
/// every invalid byte as \xNN. utf8_chunks splits any byte slice into
/// (valid str, invalid bytes) pieces -- no slicing off a char boundary.
fn show(bytes: &[u8]) -> String {
    let mut out = String::from("\"");
    for chunk in bytes.utf8_chunks() {
        out.push_str(chunk.valid());
        for b in chunk.invalid() {
            write!(out, "\\x{b:02x}").expect("writing to a String cannot fail");
        }
    }
    out.push('"');
    out
}

/// byte_rle is the bug: runs of BYTES. Multi-byte letters are split apart and
/// digits land between a lead byte and its continuation byte. The result can't
/// even be a String -- it has to stay a Vec<u8>.
fn byte_rle(s: &str) -> Vec<u8> {
    let b = s.as_bytes();
    let mut out = Vec::new();
    let mut i = 0;
    while i < b.len() {
        let j = i + b[i..].iter().take_while(|&&x| x == b[i]).count();
        out.push(b[i]);
        out.extend_from_slice((j - i).to_string().as_bytes());
        i = j;
    }
    out
}

fn main() {
    println!("(a) compress: shorter, or the original");
    println!(
        "  {:<14} {:<12} {:>6} {:>6} {:>7}",
        "input", "output", "bytes", "runes", "allocs"
    );
    let twelve = "a".repeat(12);
    let mut alloc_set = HashSet::new();
    let mut reallocs_total = 0;
    for s in ["aabcccccaaa", "abc", twelve.as_str(), "éééab", ""] {
        let out = compress(s);
        let (allocs, reallocs) = allocs_of(|| {
            black_box(compress(black_box(s)));
        });
        reallocs_total += reallocs;
        if !s.is_empty() {
            alloc_set.insert(allocs);
        }
        let note = if matches!(out, Cow::Borrowed(_)) && !s.is_empty() {
            "  <- not shorter: original returned"
        } else {
            ""
        };
        println!(
            "  {:<14} {:<12} {:>2}->{:<2} {:>2}->{:<2} {:>7}{}",
            show(s.as_bytes()),
            show(out.as_bytes()),
            s.len(),
            out.len(),
            s.chars().count(),
            out.chars().count(),
            allocs,
            note
        );
    }
    let e = compress("éééab");
    println!(
        "  \"éééab\" is shorter in bytes ({} -> {}) but longer in runes ({} -> {}): the rule counts bytes.",
        "éééab".len(),
        e.len(),
        "éééab".chars().count(),
        e.chars().count()
    );
    if alloc_set.len() == 1 && alloc_set.contains(&1) && reallocs_total == 0 {
        println!(
            "  every non-empty input costs exactly 1 allocation and 0 reallocs -- the with_capacity:"
        );
        println!(
            "  write! pushes into that buffer, and \"abc\" pays it and returns the original (borrowed)."
        );
    } else {
        println!("  allocations differ between inputs ({reallocs_total} reallocs): see the table");
    }

    println!();
    println!("runs must be runes, not bytes -- a byte-wise encoder on \"ééé\":");
    let bad = byte_rle("ééé");
    // String::from_utf8 is the validity check: Err for the byte-wise output.
    let bad_ok = String::from_utf8(bad.clone()).is_ok();
    println!("  byte runs: {}  valid UTF-8: {}", show(&bad), bad_ok);
    let good = encode("ééé"); // a String: valid by construction
    println!(
        "  rune runs: {}  valid UTF-8: {}",
        show(good.as_bytes()),
        std::str::from_utf8(good.as_bytes()).is_ok()
    );

    println!();
    println!("(b) the worst case: every run has length 1");
    for s in ["abcdef", "é€😀"] {
        let e = encode(s);
        let (rs, re) = (s.chars().count(), e.chars().count());
        println!(
            "  {:<10} -> {:<16} bytes {:>2} -> {:>2} ({:.2}x)   runes {} -> {} ({:.2}x)",
            show(s.as_bytes()),
            show(e.as_bytes()),
            s.len(),
            e.len(),
            e.len() as f64 / s.len() as f64,
            rs,
            re,
            re as f64 / rs as f64
        );
    }
    println!("  each run adds at least one digit: at most 2x the runes, and 2x the bytes only");
    println!("  when every run is a single 1-byte rune");

    println!();
    println!("(c) round trip decode(encode(s)) == s on 10,000 random strings, length 0..12");
    for alphabet in ["ab", "a1"] {
        let letters: Vec<char> = alphabet.chars().collect();
        let mut rng = Pcg::new(1, 2);
        let (mut failures, mut shorter) = (0, 0);
        let mut first = String::new();
        for _ in 0..10_000 {
            let n = rng.int_n(13);
            let s: String = (0..n)
                .map(|_| letters[rng.int_n(letters.len() as u64) as usize])
                .collect();
            if matches!(compress(&s), Cow::Owned(_)) {
                shorter += 1;
            }
            let enc = encode(&s);
            match decode(&enc) {
                Ok(got) if got == s => {}
                other => {
                    if failures == 0 {
                        let what = match other {
                            Ok(got) => format!("decodes to {} runes", got.chars().count()),
                            Err(e) => format!("error: {e}"),
                        };
                        first = format!(
                            "{} -> {} -> {what}",
                            show(s.as_bytes()),
                            show(enc.as_bytes())
                        );
                    }
                    failures += 1;
                }
            }
        }
        println!(
            "  alphabet {:<5} failures {:>5}   compress was shorter on {} of the 10000",
            show(alphabet.as_bytes()),
            failures,
            shorter
        );
        if failures > 0 {
            println!("    first counter-example: {first}");
        }
    }
    println!(
        "  same seed: the \"a1\" strings are the \"ab\" strings with b -> 1, so the runs (and"
    );
    println!("  the shorter count) are identical -- only decoding tells them apart.");
    println!("  a digit in the input is read back as part of a count: the format needs a");
    println!("  digit-free alphabet, or an escape character, to be decodable.");
}
```

**Rust output:**

```
(a) compress: shorter, or the original
  input          output        bytes  runes  allocs
  "aabcccccaaa"  "a2b1c5a3"   11->8  11->8        1
  "abc"          "abc"         3->3   3->3        1  <- not shorter: original returned
  "aaaaaaaaaaaa" "a12"        12->3  12->3        1
  "éééab"        "é3a1b1"      8->7   5->6        1
  ""             ""            0->0   0->0        0
  "éééab" is shorter in bytes (8 -> 7) but longer in runes (5 -> 6): the rule counts bytes.
  every non-empty input costs exactly 1 allocation and 0 reallocs -- the with_capacity:
  write! pushes into that buffer, and "abc" pays it and returns the original (borrowed).

runs must be runes, not bytes -- a byte-wise encoder on "ééé":
  byte runs: "\xc31\xa91\xc31\xa91\xc31\xa91"  valid UTF-8: false
  rune runs: "é3"  valid UTF-8: true

(b) the worst case: every run has length 1
  "abcdef"   -> "a1b1c1d1e1f1"   bytes  6 -> 12 (2.00x)   runes 6 -> 12 (2.00x)
  "é€😀"      -> "é1€1😀1"         bytes  9 -> 12 (1.33x)   runes 3 -> 6 (2.00x)
  each run adds at least one digit: at most 2x the runes, and 2x the bytes only
  when every run is a single 1-byte rune

(c) round trip decode(encode(s)) == s on 10,000 random strings, length 0..12
  alphabet "ab"  failures     0   compress was shorter on 2082 of the 10000
  alphabet "a1"  failures  6958   compress was shorter on 2082 of the 10000
    first counter-example: "aaaaa1a1a1" -> "a511a111a111" -> decodes to 733 runes
  same seed: the "a1" strings are the "ab" strings with b -> 1, so the runs (and
  the shorter count) are identical -- only decoding tells them apart.
  a digit in the input is read back as part of a count: the format needs a
  digit-free alphabet, or an escape character, to be decodable.
```

> **Go vs Rust:** Every algorithmic line is identical to Go's (diffed): the same outputs, lengths, 0 and 6,958 round-trip failures and the same first counter-example, because `Pcg` ports Go's PCG-DXSM and `IntN`. Both versions make exactly 1 allocation per non-empty input; the counting allocator also shows 0 reallocs, so `write!(out, "{c}{count}")` writes into the `with_capacity` buffer without a temporary. `compress` returns `Cow<str>` where Go returns the same string header, and the byte-wise encoder's output has to stay a `Vec<u8>`: `String::from_utf8` rejects it.

**Complexity:** Θ(n) time, one pass over the runes · output at most 2× the input runes (2× the bytes only for 1-byte runes) before the "not shorter" check · exactly 1 allocation per non-empty input with `Grow(len(s))` and an early exit · the round trip held for all 10,000 strings over `ab` and failed for 6,958 of 10,000 over `a1`: it needs a digit-free alphabet.

---

## 11. Longest substring without repeating characters

`🟡 medium` · *Sliding window*

The variable sliding window: move `right` one character at a time, and when the new character was already seen
**inside** the window, jump `left` to just past its last position. `left` only moves forward, so every position
enters the window once and leaves it at most once: Θ(n) steps, against Θ(n²) for trying every start. Two bugs
are classic. The first lets `left` move **backwards** (`left = last[c]+1` without the `max`), and `"abba"` catches
it. The second runs the window over bytes, which gives a length in bytes and a substring that starts in the middle
of a rune. Lesson 16 generalises the window; this example is about the string-specific parts.

**Steps:**

1. Write a brute-force oracle over `[]rune`, a map window over `range` (rune index for the length, byte offset for the cut), an ASCII array window, and the two buggy versions.
2. Run all of them on seven classic inputs, counting brute-force steps and the window's right moves and left jumps.
3. Check every version against the oracle on 20,000 PCG-random strings over `abcd` and over `abé€`, and watch which bug each alphabet exposes.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"strings"
	"unicode/utf8"
)

// Longest substring without repeating characters, the variable sliding
// window: move `right` one character at a time; when the new character was
// already seen INSIDE the window, jump `left` to just past that earlier
// occurrence. `left` only ever moves forward, so each position enters the
// window once and leaves it once: Theta(n).
//
// Every function returns the length in characters (runes), the substring,
// and a step count, so the cost is counted rather than claimed.

type result struct {
	n     int    // length of the answer (runes; bytes for the byte window)
	sub   string // the first longest window
	steps int    // brute: inner-loop steps; windows: right moves + left jumps
	jumps int    // windows only: how often `left` moved
}

// brute is the oracle: from every start, extend until a repeat. Theta(n^2).
func brute(s string) result {
	rs := []rune(s)
	seen := map[rune]bool{}
	var best, start, steps int
	for i := range rs {
		clear(seen)
		j := i
		for ; j < len(rs); j++ {
			steps++
			if seen[rs[j]] {
				break
			}
			seen[rs[j]] = true
		}
		if j-i > best {
			best, start = j-i, i
		}
	}
	return result{best, string(rs[start : start+best]), steps, 0}
}

// windowBytes is the window over BYTES with a last-seen array instead of a
// map. With asciiOnly it is the ASCII fast path (only entries 0..127 are ever
// used, and it refuses -- ok=false -- any byte >= 0x80). Without it, it is the
// Unicode bug: it answers in bytes and its substring can start mid-rune.
func windowBytes(s string, asciiOnly bool) (result, bool) {
	var last [256]int
	for i := range last {
		last[i] = -1 // -1: not seen yet
	}
	var left, best, bestL, jumps int
	for right := 0; right < len(s); right++ {
		c := s[right]
		if asciiOnly && c >= utf8.RuneSelf {
			return result{}, false
		}
		if last[c] >= left { // seen inside the window
			left = last[c] + 1
			jumps++
		}
		last[c] = right
		if right-left+1 > best {
			best, bestL = right-left+1, left
		}
	}
	return result{best, s[bestL : bestL+best], len(s) + jumps, jumps}, true
}

// seen remembers, for one rune, its rune index and the byte offset just past
// it -- where the window restarts if this rune repeats.
type seen struct{ idx, end int }

// window runs over runes with range, which yields byte offsets. Lengths are
// counted in runes (k), substrings are cut at byte offsets -- always at a rune
// boundary, so the substring is valid UTF-8. `max` is optional: buggy=true
// drops the "inside the window" test and lets `left` move backwards.
func window(s string, buggy bool) result {
	last := map[rune]seen{}
	var left, leftIdx, k, best, bestL, bestR, jumps int
	for i, r := range s {
		end := i + utf8.RuneLen(r) // valid UTF-8 assumed: width of r
		if p, ok := last[r]; ok && (buggy || p.idx >= leftIdx) {
			// correct: left = max(left, p.end) -- the test above IS the max
			left, leftIdx = p.end, p.idx+1
			jumps++
		}
		last[r] = seen{k, end}
		if k-leftIdx+1 > best {
			best, bestL, bestR = k-leftIdx+1, left, end
		}
		k++
	}
	return result{best, s[bestL:bestR], k + jumps, jumps}
}

// show quotes like %q, by hand so the Rust twin can match it: valid runes as
// themselves, every invalid byte as \xNN.
func show(s string) string {
	var b strings.Builder
	b.WriteByte('"')
	for i := 0; i < len(s); {
		r, w := utf8.DecodeRuneInString(s[i:])
		if r == utf8.RuneError && w == 1 {
			fmt.Fprintf(&b, `\x%02x`, s[i])
		} else {
			b.WriteRune(r)
		}
		i += w
	}
	b.WriteByte('"')
	return b.String()
}

func mark(got, want result) string {
	if got.n != want.n || got.sub != want.sub {
		return " X"
	}
	return ""
}

func main() {
	cases := []struct {
		s    string
		want int
	}{{"abcabcbb", 3}, {"bbbbb", 1}, {"pwwkew", 3}, {"", 0}, {"dvdf", 3}, {"abba", 2}, {"éèé", 2}}

	fmt.Println("correct implementations (answer, first longest substring, steps):")
	fmt.Printf("  %-11s %4s | %-12s %5s | %-12s %5s %5s | %6s\n",
		"input", "want", "brute", "steps", "map window", "moves", "jumps", "ASCII")
	for _, c := range cases {
		b := brute(c.s)
		w := window(c.s, false)
		a := "n/a"
		if r, ok := windowBytes(c.s, true); ok {
			a = fmt.Sprint(r.n)
		}
		fmt.Printf("  %-11s %4d | %d %-10s %5d | %d %-10s %5d %5d | %6s\n",
			show(c.s), c.want, b.n, show(b.sub), b.steps, w.n, show(w.sub), w.steps-w.jumps, w.jumps, a)
	}

	fmt.Println()
	fmt.Println("the two bugs (X = differs from brute force):")
	fmt.Printf("  %-11s %4s | %-16s | %s\n", "input", "want", "no max (left<-)", "byte window")
	for _, c := range cases {
		b := brute(c.s)
		nm := window(c.s, true)
		bw, _ := windowBytes(c.s, false)
		bwCell := fmt.Sprintf("%d %s%s", bw.n, show(bw.sub), mark(bw, b))
		if !utf8.ValidString(bw.sub) {
			bwCell += " invalid UTF-8"
		}
		fmt.Printf("  %-11s %4d | %-16s | %s\n", show(c.s), c.want,
			fmt.Sprintf("%d %s%s", nm.n, show(nm.sub), mark(nm, b)), bwCell)
	}
	fmt.Println("  \"abba\": at the second a, its last index 0 is already left of the window (left = 2);")
	fmt.Println("  without max, left jumps BACK to 1 and the window \"bba\" holds two b's.")

	fmt.Println()
	fmt.Println("cost on n distinct runes (U+4E00 onward), where brute force never breaks early:")
	fmt.Printf("  %6s %12s %8s %12s %8s\n", "n", "brute steps", "ratio", "window steps", "ratio")
	prevB, prevW := 0, 0
	for _, n := range []int{1000, 2000, 4000} {
		rs := make([]rune, n)
		for i := range rs {
			rs[i] = rune(0x4E00 + i)
		}
		s := string(rs)
		b, w := brute(s), window(s, false) // both find all n runes
		rb, rw := "-", "-"
		if prevB > 0 {
			rb = fmt.Sprintf("%.2fx", float64(b.steps)/float64(prevB))
			rw = fmt.Sprintf("%.2fx", float64(w.steps)/float64(prevW))
		}
		fmt.Printf("  %6d %12d %8s %12d %8s\n", n, b.steps, rb, w.steps, rw)
		prevB, prevW = b.steps, w.steps
	}

	fmt.Println()
	fmt.Println("oracle check: 20,000 random strings, length 0..30, mismatches vs brute force")
	fmt.Printf("  %-8s %15s %6s %7s %6s %14s\n", "alphabet", "ASCII", "map", "no max", "bytes", "worst steps/n")
	firstBug, bugMsg := "", ""
	byteBad := map[string]int{}
	worstAll := 0.0
	for _, alphabet := range []string{"abcd", "abé€"} {
		letters := []rune(alphabet)
		rng := rand.New(rand.NewPCG(1, 2))
		var bad128, skipped128, badMap, badNoMax, badBytes int
		worst := 0.0 // max of (window steps)/n: Theta(n) means it stays <= 2
		for range 20_000 {
			rs := make([]rune, rng.IntN(31))
			for k := range rs {
				rs[k] = letters[rng.IntN(len(letters))]
			}
			s := string(rs)
			want := brute(s)
			if r, ok := windowBytes(s, true); !ok {
				skipped128++
			} else if mark(r, want) != "" {
				bad128++
			}
			w := window(s, false)
			if mark(w, want) != "" {
				badMap++
			}
			if len(rs) > 0 {
				worst = max(worst, float64(w.steps)/float64(len(rs)))
			}
			if nm := window(s, true); mark(nm, want) != "" {
				if firstBug == "" || len(s) < len(firstBug) {
					firstBug = s
					bugMsg = fmt.Sprintf("%s: no-max says %d %s, brute force %d %s",
						show(s), nm.n, show(nm.sub), want.n, show(want.sub))
				}
				badNoMax++
			}
			if bw, _ := windowBytes(s, false); mark(bw, want) != "" {
				badBytes++
			}
		}
		c128 := fmt.Sprint(bad128)
		if skipped128 > 0 {
			c128 = fmt.Sprintf("%d (%d n/a)", bad128, skipped128)
		}
		byteBad[alphabet] = badBytes
		worstAll = max(worstAll, worst)
		fmt.Printf("  %-8s %15s %6d %7d %6d %13.2fn\n", show(alphabet), c128, badMap, badNoMax, badBytes, worst)
	}
	fmt.Println("  shortest no-max counter-example:", bugMsg)
	fmt.Println("  map and no-max see the same rune pattern in both alphabets (c->é, d->€),")
	fmt.Println("  so their counts match.")
	if worstAll <= 2 {
		fmt.Printf("  the window never took more than %.2fn steps: n right moves + at most n jumps.\n", worstAll)
	}
	if byteBad["abcd"] == 0 && byteBad["abé€"] > 0 {
		fmt.Println("  on ASCII input bytes ARE runes, so the byte window passed every ASCII test;")
		fmt.Println("  only the non-ASCII alphabet exposed it.")
	}
}
```

**Output:**

```
correct implementations (answer, first longest substring, steps):
  input       want | brute        steps | map window   moves jumps |  ASCII
  "abcabcbb"     3 | 3 "abc"         25 | 3 "abc"          8     5 |      3
  "bbbbb"        1 | 1 "b"            9 | 1 "b"            5     4 |      1
  "pwwkew"       3 | 3 "wke"         15 | 3 "wke"          6     2 |      3
  ""             0 | 0 ""             0 | 0 ""             0     0 |      0
  "dvdf"         3 | 3 "vdf"          9 | 3 "vdf"          4     1 |      3
  "abba"         2 | 2 "ab"           8 | 2 "ab"           4     1 |      2
  "éèé"          2 | 2 "éè"           6 | 2 "éè"           3     1 |    n/a

the two bugs (X = differs from brute force):
  input       want | no max (left<-)  | byte window
  "abcabcbb"     3 | 3 "abc"          | 3 "abc"
  "bbbbb"        1 | 1 "b"            | 1 "b"
  "pwwkew"       3 | 3 "wke"          | 3 "wke"
  ""             0 | 0 ""             | 0 ""
  "dvdf"         3 | 3 "vdf"          | 3 "vdf"
  "abba"         2 | 3 "bba" X        | 2 "ab"
  "éèé"          2 | 2 "éè"           | 3 "\xa9è" X invalid UTF-8
  "abba": at the second a, its last index 0 is already left of the window (left = 2);
  without max, left jumps BACK to 1 and the window "bba" holds two b's.

cost on n distinct runes (U+4E00 onward), where brute force never breaks early:
       n  brute steps    ratio window steps    ratio
    1000       500500        -         1000        -
    2000      2001000    4.00x         2000    2.00x
    4000      8002000    4.00x         4000    2.00x

oracle check: 20,000 random strings, length 0..30, mismatches vs brute force
  alphabet           ASCII    map  no max  bytes  worst steps/n
  "abcd"                 0      0   14957      0          1.83n
  "abé€"     0 (18658 n/a)      0   14957  18658          1.83n
  shortest no-max counter-example: "caac": no-max says 3 "aac", brute force 2 "ca"
  map and no-max see the same rune pattern in both alphabets (c->é, d->€),
  so their counts match.
  the window never took more than 1.83n steps: n right moves + at most n jumps.
  on ASCII input bytes ARE runes, so the byte window passed every ASCII test;
  only the non-ASCII alphabet exposed it.
```

**Rust:**

```rust
use std::collections::{HashMap, HashSet};
use std::fmt::Write;

// Longest substring without repeating characters, the variable sliding
// window: move `right` one character at a time; when the new character was
// already seen INSIDE the window, jump `left` to just past that earlier
// occurrence. `left` only ever moves forward, so each position enters the
// window once and leaves it once: Theta(n).
//
// Every function returns the length in characters (chars), the substring,
// and a step count, so the cost is counted rather than claimed.

struct Res<'a> {
    n: usize,      // length of the answer (chars; bytes for the byte window)
    sub: &'a [u8], // the first longest window, borrowed from the input
    steps: usize,  // brute: inner-loop steps; windows: right moves + left jumps
    jumps: usize,  // windows only: how often `left` moved
}
// `sub` is a byte slice so that the buggy byte window can return what it
// really found; the correct versions produce it from a `&str` slice, and
// `&s[a..b]` would panic if a or b were not on a char boundary.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) produces exactly the strings Go tests.
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

/// brute is the oracle: from every start, extend until a repeat. Theta(n^2).
/// It walks a Vec of (byte offset, char) so the answer can be cut out of `s`.
fn brute(s: &str) -> Res<'_> {
    let cs: Vec<(usize, char)> = s.char_indices().collect();
    let offset = |k: usize| cs.get(k).map_or(s.len(), |&(i, _)| i);
    let mut seen = HashSet::new();
    let (mut best, mut start, mut steps) = (0, 0, 0);
    for i in 0..cs.len() {
        seen.clear();
        let mut j = i;
        while j < cs.len() {
            steps += 1;
            if !seen.insert(cs[j].1) {
                break; // insert returns false: already in the set
            }
            j += 1;
        }
        if j - i > best {
            (best, start) = (j - i, i);
        }
    }
    let sub = &s[offset(start)..offset(start + best)];
    Res {
        n: best,
        sub: sub.as_bytes(),
        steps,
        jumps: 0,
    }
}

/// window_bytes is the window over BYTES with a last-seen array instead of a
/// map; Option<usize> replaces Go's -1 sentinel. With ascii_only it is the
/// ASCII fast path (only entries 0..127 are ever used, and it refuses -- None
/// -- any byte >= 0x80). Without it, it is the Unicode bug: it answers in
/// bytes and its substring can start mid-char.
fn window_bytes(s: &str, ascii_only: bool) -> Option<Res<'_>> {
    let b = s.as_bytes();
    let mut last: [Option<usize>; 256] = [None; 256];
    let (mut left, mut best, mut best_l, mut jumps) = (0, 0, 0, 0);
    for (right, &c) in b.iter().enumerate() {
        if ascii_only && !c.is_ascii() {
            return None;
        }
        if last[c as usize].is_some_and(|p| p >= left) {
            // seen inside the window
            left = last[c as usize]? + 1;
            jumps += 1;
        }
        last[c as usize] = Some(right);
        if right - left + 1 > best {
            (best, best_l) = (right - left + 1, left);
        }
    }
    Some(Res {
        n: best,
        sub: &b[best_l..best_l + best], // a byte slice: no boundary check
        steps: b.len() + jumps,
        jumps,
    })
}

/// For one char: its char index and the byte offset just past it -- where
/// the window restarts if this char repeats.
struct Seen {
    idx: usize,
    end: usize,
}

/// window runs over char_indices(), which yields the byte offset and the char
/// together. Lengths are counted in chars (k); the substring is cut at byte
/// offsets that char_indices produced, so `&s[l..r]` is always on a char
/// boundary. buggy=true drops the "inside the window" test and lets `left`
/// move backwards.
fn window(s: &str, buggy: bool) -> Res<'_> {
    let mut last: HashMap<char, Seen> = HashMap::new();
    let (mut left, mut left_idx, mut k) = (0, 0, 0);
    let (mut best, mut best_l, mut best_r, mut jumps) = (0, 0, 0, 0);
    for (i, c) in s.char_indices() {
        let end = i + c.len_utf8();
        if let Some(p) = last.get(&c)
            && (buggy || p.idx >= left_idx)
        {
            // correct: left = max(left, p.end) -- the test above IS the max
            (left, left_idx) = (p.end, p.idx + 1);
            jumps += 1;
        }
        last.insert(c, Seen { idx: k, end });
        if k - left_idx + 1 > best {
            (best, best_l, best_r) = (k - left_idx + 1, left, end);
        }
        k += 1;
    }
    // a &str slice: checked to be on char boundaries (it would panic otherwise)
    let sub: &str = &s[best_l..best_r];
    Res {
        n: best,
        sub: sub.as_bytes(),
        steps: k + jumps,
        jumps,
    }
}

/// show quotes like Go's %q would here: valid UTF-8 as itself, every invalid
/// byte as \xNN. utf8_chunks splits any byte slice into (valid, invalid) parts.
fn show(bytes: &[u8]) -> String {
    let mut out = String::from("\"");
    for chunk in bytes.utf8_chunks() {
        out.push_str(chunk.valid());
        for b in chunk.invalid() {
            write!(out, "\\x{b:02x}").expect("writing to a String cannot fail");
        }
    }
    out.push('"');
    out
}

fn mark(got: &Res, want: &Res) -> &'static str {
    if got.n != want.n || got.sub != want.sub {
        " X"
    } else {
        ""
    }
}

fn main() {
    let cases = [
        ("abcabcbb", 3),
        ("bbbbb", 1),
        ("pwwkew", 3),
        ("", 0),
        ("dvdf", 3),
        ("abba", 2),
        ("éèé", 2),
    ];

    println!("correct implementations (answer, first longest substring, steps):");
    println!(
        "  {:<11} {:>4} | {:<12} {:>5} | {:<12} {:>5} {:>5} | {:>6}",
        "input", "want", "brute", "steps", "map window", "moves", "jumps", "ASCII"
    );
    for &(s, want) in &cases {
        let b = brute(s);
        let w = window(s, false);
        let a = window_bytes(s, true).map_or("n/a".to_string(), |r| r.n.to_string());
        println!(
            "  {:<11} {:>4} | {} {:<10} {:>5} | {} {:<10} {:>5} {:>5} | {:>6}",
            show(s.as_bytes()),
            want,
            b.n,
            show(b.sub),
            b.steps,
            w.n,
            show(w.sub),
            w.steps - w.jumps,
            w.jumps,
            a
        );
    }

    println!();
    println!("the two bugs (X = differs from brute force):");
    println!(
        "  {:<11} {:>4} | {:<16} | byte window",
        "input", "want", "no max (left<-)"
    );
    for &(s, want) in &cases {
        let b = brute(s);
        let nm = window(s, true);
        let bw = window_bytes(s, false).expect("never refuses without ascii_only");
        let mut bw_cell = format!("{} {}{}", bw.n, show(bw.sub), mark(&bw, &b));
        // std::str::from_utf8 is Rust's utf8.Valid
        if std::str::from_utf8(bw.sub).is_err() {
            bw_cell += " invalid UTF-8";
        }
        let nm_cell = format!("{} {}{}", nm.n, show(nm.sub), mark(&nm, &b));
        println!(
            "  {:<11} {:>4} | {:<16} | {}",
            show(s.as_bytes()),
            want,
            nm_cell,
            bw_cell
        );
    }
    println!(
        "  \"abba\": at the second a, its last index 0 is already left of the window (left = 2);"
    );
    println!("  without max, left jumps BACK to 1 and the window \"bba\" holds two b's.");

    println!();
    println!("cost on n distinct runes (U+4E00 onward), where brute force never breaks early:");
    println!(
        "  {:>6} {:>12} {:>8} {:>12} {:>8}",
        "n", "brute steps", "ratio", "window steps", "ratio"
    );
    let (mut prev_b, mut prev_w) = (0, 0);
    for n in [1000u32, 2000, 4000] {
        let s: String = (0..n)
            .map(|i| char::from_u32(0x4E00 + i).expect("CJK range is valid"))
            .collect();
        let (b, w) = (brute(&s), window(&s, false)); // both find all n chars
        let (mut rb, mut rw) = ("-".to_string(), "-".to_string());
        if prev_b > 0 {
            rb = format!("{:.2}x", b.steps as f64 / prev_b as f64);
            rw = format!("{:.2}x", w.steps as f64 / prev_w as f64);
        }
        println!(
            "  {:>6} {:>12} {:>8} {:>12} {:>8}",
            n, b.steps, rb, w.steps, rw
        );
        (prev_b, prev_w) = (b.steps, w.steps);
    }

    println!();
    println!("oracle check: 20,000 random strings, length 0..30, mismatches vs brute force");
    println!(
        "  {:<8} {:>15} {:>6} {:>7} {:>6} {:>14}",
        "alphabet", "ASCII", "map", "no max", "bytes", "worst steps/n"
    );
    let mut first_bug: Option<(String, String)> = None; // (input, message)
    let mut byte_bad = HashMap::new();
    let mut worst_all: f64 = 0.0;
    for alphabet in ["abcd", "abé€"] {
        let letters: Vec<char> = alphabet.chars().collect();
        let mut rng = Pcg::new(1, 2);
        let (mut bad128, mut skipped128, mut bad_map, mut bad_no_max, mut bad_bytes) =
            (0, 0, 0, 0, 0);
        let mut worst: f64 = 0.0; // max of (window steps)/n: Theta(n) means it stays <= 2
        for _ in 0..20_000 {
            let len = rng.int_n(31) as usize;
            let s: String = (0..len)
                .map(|_| letters[rng.int_n(letters.len() as u64) as usize])
                .collect();
            let want = brute(&s);
            match window_bytes(&s, true) {
                None => skipped128 += 1,
                Some(r) if !mark(&r, &want).is_empty() => bad128 += 1,
                Some(_) => {}
            }
            let w = window(&s, false);
            if !mark(&w, &want).is_empty() {
                bad_map += 1;
            }
            if len > 0 {
                worst = worst.max(w.steps as f64 / len as f64);
            }
            let nm = window(&s, true);
            if !mark(&nm, &want).is_empty() {
                if first_bug.as_ref().is_none_or(|(f, _)| s.len() < f.len()) {
                    let msg = format!(
                        "{}: no-max says {} {}, brute force {} {}",
                        show(s.as_bytes()),
                        nm.n,
                        show(nm.sub),
                        want.n,
                        show(want.sub)
                    );
                    first_bug = Some((s.clone(), msg));
                }
                bad_no_max += 1;
            }
            let bw = window_bytes(&s, false).expect("never refuses without ascii_only");
            if !mark(&bw, &want).is_empty() {
                bad_bytes += 1;
            }
        }
        let c128 = if skipped128 > 0 {
            format!("{bad128} ({skipped128} n/a)")
        } else {
            bad128.to_string()
        };
        byte_bad.insert(alphabet, bad_bytes);
        worst_all = worst_all.max(worst);
        println!(
            "  {:<8} {:>15} {:>6} {:>7} {:>6} {:>13.2}n",
            show(alphabet.as_bytes()),
            c128,
            bad_map,
            bad_no_max,
            bad_bytes,
            worst
        );
    }
    let msg = first_bug.map_or(String::new(), |(_, m)| m);
    println!("  shortest no-max counter-example: {msg}");
    println!("  map and no-max see the same rune pattern in both alphabets (c->é, d->€),");
    println!("  so their counts match.");
    if worst_all <= 2.0 {
        println!(
            "  the window never took more than {worst_all:.2}n steps: n right moves + at most n jumps."
        );
    }
    if byte_bad["abcd"] == 0 && byte_bad["abé€"] > 0 {
        println!("  on ASCII input bytes ARE runes, so the byte window passed every ASCII test;");
        println!("  only the non-ASCII alphabet exposed it.");
    }
}
```

**Rust output:**

```
correct implementations (answer, first longest substring, steps):
  input       want | brute        steps | map window   moves jumps |  ASCII
  "abcabcbb"     3 | 3 "abc"         25 | 3 "abc"          8     5 |      3
  "bbbbb"        1 | 1 "b"            9 | 1 "b"            5     4 |      1
  "pwwkew"       3 | 3 "wke"         15 | 3 "wke"          6     2 |      3
  ""             0 | 0 ""             0 | 0 ""             0     0 |      0
  "dvdf"         3 | 3 "vdf"          9 | 3 "vdf"          4     1 |      3
  "abba"         2 | 2 "ab"           8 | 2 "ab"           4     1 |      2
  "éèé"          2 | 2 "éè"           6 | 2 "éè"           3     1 |    n/a

the two bugs (X = differs from brute force):
  input       want | no max (left<-)  | byte window
  "abcabcbb"     3 | 3 "abc"          | 3 "abc"
  "bbbbb"        1 | 1 "b"            | 1 "b"
  "pwwkew"       3 | 3 "wke"          | 3 "wke"
  ""             0 | 0 ""             | 0 ""
  "dvdf"         3 | 3 "vdf"          | 3 "vdf"
  "abba"         2 | 3 "bba" X        | 2 "ab"
  "éèé"          2 | 2 "éè"           | 3 "\xa9è" X invalid UTF-8
  "abba": at the second a, its last index 0 is already left of the window (left = 2);
  without max, left jumps BACK to 1 and the window "bba" holds two b's.

cost on n distinct runes (U+4E00 onward), where brute force never breaks early:
       n  brute steps    ratio window steps    ratio
    1000       500500        -         1000        -
    2000      2001000    4.00x         2000    2.00x
    4000      8002000    4.00x         4000    2.00x

oracle check: 20,000 random strings, length 0..30, mismatches vs brute force
  alphabet           ASCII    map  no max  bytes  worst steps/n
  "abcd"                 0      0   14957      0          1.83n
  "abé€"     0 (18658 n/a)      0   14957  18658          1.83n
  shortest no-max counter-example: "caac": no-max says 3 "aac", brute force 2 "ca"
  map and no-max see the same rune pattern in both alphabets (c->é, d->€),
  so their counts match.
  the window never took more than 1.83n steps: n right moves + at most n jumps.
  on ASCII input bytes ARE runes, so the byte window passed every ASCII test;
  only the non-ASCII alphabet exposed it.
```

> **Go vs Rust:** Identical output, byte for byte (diffed): same answers, step counts, and 0 / 14,957 / 18,658 mismatches, because `Pcg` ports Go's PCG-DXSM and `IntN`. `char_indices()` gives the byte offset and the char together, the way Go's `range` does; `Option<usize>` replaces the `-1` sentinel; and the correct windows cut `&s[l..r]`, which is checked to fall on char boundaries. The byte window can't produce a `&str` at all: it returns `&[u8]`, because `&s[1..4]` on `"éèé"` would panic where Go silently hands back `"\xa9è"`.

**Complexity:** window Θ(n) time — n right moves plus at most n jumps of `left` (at most 1.83n steps on the random inputs, n on all-distinct ones) · Θ(min(n, alphabet)) space · brute force Θ(n²): 500,500 → 2,001,000 → 8,002,000 steps for n = 1,000 / 2,000 / 4,000 · `left = max(left, last+1)` is the whole correctness argument: without it, 14,957 of 20,000 random strings fail · the byte window passed all 20,000 ASCII strings and failed 18,658 mixed ones.

---

> Next tier: [🔴 hard](3-hard.md) — the cost model measured, the substring leak, a parser that never
> allocates, graphemes, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
