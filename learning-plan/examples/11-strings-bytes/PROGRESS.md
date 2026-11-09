# Step 11 — Strings, Runes & Bytes · Progress

Type & run each example; tick once your output matches. Examples are split by tier:
[🟢 easy](1-easy.md) · [🟡 medium](2-medium.md) · [🔴 hard](3-hard.md).

> ▶ **Resume here:** 🟢 **easy** tier — start with example **1. One string, three lengths**. None ticked yet.

### 🟢 easy — [1-easy.md](1-easy.md)
- [ ] 1. One string, three lengths
- [ ] 2. Immutable, and why a substring is free
- [ ] 3. The `string` / `[]byte` / `[]rune` triangle
- [ ] 4. UTF-8 indexing traps
- [ ] 5. `+=` vs `strings.Builder`, counted
- [ ] 6. Comparison is bytes: equality, order and case

### 🟡 medium — [2-medium.md](2-medium.md)
- [ ] 7. Anagrams: sort, count, and the byte trap
- [ ] 8. Palindromes: two pointers over runes
- [ ] 9. Reversing bytes, runes and words
- [ ] 10. Run-length compression, and when it makes things longer
- [ ] 11. Longest substring without repeating characters

### 🔴 hard — [3-hard.md](3-hard.md)
- [ ] 12. The cost model, measured
- [ ] 13. The substring that kept its parent alive
- [ ] 14. A tokenizer that never allocates
- [ ] 15. Graphemes: when a rune is not a character
- [ ] 16. Capstone: the string problem set, proved and measured

## The three-pass build

In `practice/11-strings-bytes/`:

- [ ] **Build it** — `IsAnagram` with a counter, rune-correct, plus an ASCII `[26]int` fast path
- [ ] **Build it** — `IsPalindrome` with two pointers that decode from both ends and allocate nothing
- [ ] **Build it** — `ReverseWords` in place on a `[]byte`: reverse all, reverse each word, single spaces
- [ ] **Build it** — `Compress` / `Decompress` (run-length) with one `Grow` and multi-digit counts
- [ ] **Build it** — `LengthOfLongestSubstring` over runes, with `left = max(left, last+1)`
- [ ] **Prove it** — each one against a brute-force oracle on `[]rune`, over ≥100,000 random strings
- [ ] **Prove it** — a **mixed alphabet**: ASCII, 2-, 3- and 4-byte runes, and spaces — never ASCII only
- [ ] **Prove it** — every output passes `utf8.ValidString`
- [ ] **Prove it** — `Decompress(Compress(s)) == s` on a digit-free alphabet, and a test that shows why digits break it
- [ ] **Prove it** — the `"abba"`, `"é£"`/`"ã©"` and `"été"` cases as named regression tests
- [ ] **Measure it** — allocations per call for every function (`testing.AllocsPerRun`)
- [ ] **Measure it** — ASCII fast path vs rune path vs oracle, on 1 MB of ASCII and 1 MB of mixed text
- [ ] `./check.sh` prints **all gates passed**

## Recall drill

| Question | Answer |
|---|---|
| What does `len(s)` count? | **bytes** — read from the header, Θ(1) |
| What is a rune? | a code point (`int32`) — not a character |
| The three lengths of a string? | bytes · runes · grapheme clusters (what a reader sees) |
| How many bytes is a UTF-8 code point? | 1–4; the lead byte says which |
| Can an ASCII byte appear inside a multi-byte rune? | **no** — every byte of one is ≥ `0x80` |
| How big is a string header? | 16 bytes: pointer + length |
| Why is `s[i:j]` free? | strings are immutable, so the substring can share the bytes |
| …and what does that cost? | a tiny substring pins its whole parent; `strings.Clone` fixes it |
| Which triangle conversions allocate? | all of them, except the ones the compiler recognises |
| Three free `[]byte`→`string` cases? | `m[string(b)]`, `string(b) == "lit"`, `"<"+string(b)+">"` |
| What is `string("héllo"[1])`? | `"Ã"` — a byte converted as a code point |
| What does `range` over a string yield? | byte offset + decoded rune; `U+FFFD` width 1 on bad bytes |
| Cost of the k-th rune? | Θ(k) |
| Why is `s += x` in a loop Θ(n²) in Go? | every `+=` allocates and copies all of `s` |
| What does `Builder.String()` copy? | **nothing** — so a non-empty Builder must not be copied |
| Is `s += &x` quadratic in Rust? | **no** — it is `push_str`; `format!("{s}{x}")` in a loop is |
| Why does `"Éclair"` sort after `"zebra"`? | comparison is bytes, and `É` starts with `0xC3` |
| Case-insensitive equality without allocating? | `strings.EqualFold` |
| Why is byte-counting wrong for anagrams? | `"é£"` and `"ã©"` have the same bytes and no common rune |
| Why is reverse-all-then-each-word UTF-8-safe? | each word's bytes are reversed twice |
| When does run-length encoding fail to round-trip? | when the input contains digits |
| The sliding-window bug? | letting `left` move backwards — test `"abba"` |
| What does Rust do on `&s[0..2]` mid-letter? | **panics**; `s.get(0..2)` returns `None` |
| Does either standard library segment graphemes or normalize? | **no** |

## Numbers to find on your own machine

- [ ] Allocations for each edge of the triangle, and which conversions come out at **0**
- [ ] Allocations and bytes copied for `+=` vs `Builder` vs `Grow` vs `Join` as n doubles
- [ ] The capacity sequence of a `strings.Builder` — and of a Rust `String`
- [ ] How many bytes 50 kept 16-byte substrings of 1 MB lines pin, with and without `strings.Clone`
- [ ] Allocations per line for four log-line parsers — get one to **0**
- [ ] Which string operations the ratio test classifies as Θ(1), and which as Θ(n)

> Lesson: [11-strings-bytes.md](../../11-strings-bytes.md) · Index: [README.md](README.md)
