# 11 — Strings, Runes & Bytes

> Part of **Part 2 — Linear Structures**, and the last of it. A string looks like the simplest structure
> in this repo — an array of characters — and it is not: it is an array of **bytes** that *encode*
> characters, and the encoding has a variable width. [06](06-arrays-slices.md) gave you the slice, and a
> Go string is a read-only slice of bytes; [10](10-hash-tables.md) gave you the map, which half of
> the string problems use as a counter. The one-line thesis: **a string has three lengths — bytes, code
> points, and what a reader sees — and every string bug and every string cost comes from using one where
> you meant another.**

## Goals
- Say what `len(s)` counts, what a rune is, and why neither is "a character".
- Explain why a Go string is immutable, and the two things that buys: free substrings and safe sharing.
- Know the price of every edge of the `string` / `[]byte` / `[]rune` triangle — and the conversions the compiler makes free.
- Recognise the five UTF-8 indexing traps and fix each one.
- Explain why `s += x` in a loop is Θ(n²) in Go, why `strings.Builder` is Θ(n), and why the same operator is **not** quadratic in Rust.
- Know that `==`, `<`, sorting and map keys compare bytes, and what that means for accents and case.
- Solve the string problem set — anagram, palindrome, reverse words, compress, longest substring without repeats — correctly for UTF-8, not just ASCII.
- State the cost of any string operation before you call it: Θ(1) on the header, Θ(n) over the bytes, or Θ(n) plus an allocation.

## Concepts

- **A string is a header and some bytes.** Two words: a pointer and a length — `unsafe.Sizeof("")` is
  **16**, against 24 for a `[]byte` (which also carries a capacity). `len(s)` reads the header, so it is
  Θ(1) and it counts **bytes**. Assigning a string copies the header, never the bytes.

- **The bytes are conventionally UTF-8, not necessarily.** A string literal in Go source is UTF-8; a
  string built from arbitrary bytes is whatever those bytes are. Go never validates — `"a\xffb"` is a
  perfectly legal three-byte string. `utf8.ValidString` is how you ask.

- **UTF-8 in one table.** A code point takes 1 to 4 bytes, and the **lead byte says how many**:

  | Code points | Bytes | Lead byte | Example |
  |---|---|---|---|
  | U+0000–U+007F | 1 | `0xxxxxxx` | `A` |
  | U+0080–U+07FF | 2 | `110xxxxx` | `é` |
  | U+0800–U+FFFF | 3 | `1110xxxx` | `€` |
  | U+10000–U+10FFFF | 4 | `11110xxx` | `😀` |

  Every continuation byte is `10xxxxxx`, so from any position you can find the start of the current code
  point in at most three steps back (`utf8.RuneStart`). And no byte of a multi-byte sequence is below
  `0x80` — so an ASCII delimiter can never match in the middle of a letter. Example 14 builds a whole
  parser on that one fact.

- **A rune is a code point, not a character.** `rune` is `int32`. `"héllo"` has `len` 6 and 5 runes.
  But `"e\u0301"` — `e` followed by a combining acute — is 2 runes and one visible `é`, and the
  family emoji `👨‍👩‍👧` is **5 runes and 18 bytes** for one picture. What a reader sees is a
  *grapheme cluster* (Unicode UAX #29). Neither Go's nor Rust's standard library segments them;
  example 15 builds a small, explicitly partial segmenter to show what the real one does.

- **The three lengths cost differently.** Bytes: Θ(1), from the header. Runes: Θ(n) —
  `utf8.RuneCountInString` must decode every byte. Graphemes: Θ(n) and a table of Unicode properties.
  Indexing the k-th rune is Θ(k); there is no shortcut, because widths vary.

- **Immutable, and that is the point.** `s[0] = 'x'` does not compile: *cannot assign to s[0] (neither
  addressable nor a map index expression)*. Because no one can change the bytes, a substring `s[i:j]`
  can **share** them: it is a new header pointing `i` bytes into the same array — Θ(1), no allocation.
  Strings can be passed to any goroutine without a lock for the same reason. Any "edit" builds a new
  string: Θ(n) and one allocation.

- **The price of sharing.** A 16-byte substring of a one-megabyte string holds a pointer into the
  megabyte, so the garbage collector keeps **all of it**. This is lesson 06's backing-array leak in a new
  costume. `strings.Clone` exists for exactly this — it always copies into a fresh allocation. Example 13
  measures how much a handful of tiny keys can pin.

- **The triangle.** `string` (immutable bytes), `[]byte` (mutable bytes), `[]rune` (one `int32` per code
  point). Every conversion between them copies:

  | Conversion | Time | Allocation | Note |
  |---|---|---|---|
  | `[]byte(s)` | Θ(n) | 1 | must copy: the slice is mutable |
  | `string(b)` | Θ(n) | 1 | must copy: the string must not change later |
  | `[]rune(s)` | Θ(n) | 1, **4 bytes per rune** | two passes: count, then fill |
  | `string(r)` | Θ(n) | 1 | re-encodes to UTF-8 |
  | `s[i:j]` | Θ(1) | 0 | shares the bytes |

  The runtime and compiler skip the copy where they can prove it is safe: `m[string(b)]`,
  `string(b) == "lit"` and `"<" + string(b) + ">"` do not allocate (the comment on
  `slicebytetostringtmp` in `runtime/string.go` lists them), a one-byte `string(b)` points into a static
  table, and a short result that does not escape can live in a 32-byte stack buffer. Example 3 counts
  every edge.

- **The five indexing traps.** (1) `s[i]` is a **byte**, and `string(s[i])` converts that byte as a
  *code point* — `string("héllo"[1])` is `"Ã"`. (2) `s[:k]` cuts at a byte and can split a letter,
  leaving invalid UTF-8 — Go will not stop you. (3) `for i, r := range s` decodes runes but `i` is a
  **byte offset** (`0 1 3 4 5` for `"héllo"`). (4) "The k-th character" is Θ(k), or Θ(n) and an
  allocation via `[]rune(s)[k]`. (5) On invalid input `range` yields `U+FFFD` with width 1, while the
  string still holds the raw byte. Example 4 walks all five.

- **`+=` in a loop is Θ(n²) in Go.** `s += x` allocates `len(s)+len(x)` bytes and copies both — every
  time. n appends of a fixed piece copy Θ(n²) bytes in n allocations ([03](03-complexity-analysis.md)
  example 13 measured the per-element growth). `strings.Builder` appends into a growing `[]byte`
  (amortized Θ(1) per byte, O(log n) allocations), and its `String()` hands the buffer over **without
  copying** — it is `unsafe.String(unsafe.SliceData(b.buf), len(b.buf))`. That aliasing is why a
  non-empty Builder must not be copied: its `copyCheck` panics with *illegal use of non-zero Builder
  copied by value*. `Grow(n)` pre-sizes; `strings.Join` computes the total length and `Grow`s once, so it
  is a single allocation. Example 5 counts allocations for six ways to build a string.

- **Comparison is bytes.** `==` checks the lengths first, then the bytes (`len(a)==len(b) &&
  memequal`), so unequal lengths cost Θ(1). `<`, `strings.Compare`, `slices.Sort` and map keys are all
  byte-wise: `"Éclair"` sorts after `"zebra"` because `É` starts with `0xC3`. `"\u00e9"` (one code
  point) and `"e\u0301"` (two) look identical and are **unequal** — that is *normalization*, and Go's
  standard library does not do it.

- **Case is not a byte operation.** `strings.EqualFold` folds case without allocating and knows that the
  Kelvin sign `"\u212a"` folds to `k`. `strings.ToLower(a) == strings.ToLower(b)` allocates up to twice — except
  that `ToLower` returns `s` itself when it is ASCII with no uppercase. Go maps case rune-for-rune:
  `strings.ToUpper("ß")` is `"ß"`, and `EqualFold("ß", "SS")` is false.

- **The problem set, and where each one breaks on UTF-8.**
  - *Anagram* — sort (Θ(n log n)), `[26]int` (Θ(n), Θ(1) space, ASCII letters only), or `map[rune]int`
    (Θ(n), any text). Counting **bytes** is wrong: `"é£"` and `"ã©"` have the same bytes and no common
    rune.
  - *Palindrome* — two pointers from both ends. On bytes, `"été"` fails; on `[]rune` it is right and
    costs 4n bytes; decoding from both ends (`utf8.DecodeRuneInString`, `DecodeLastRuneInString`) is
    right and allocation-free.
  - *Reverse words* — `strings.Fields` + `slices.Reverse` + `strings.Join`, or in place: reverse the
    whole buffer, then each word. The in-place version is UTF-8-safe when the separators are ASCII,
    because each word's bytes are reversed twice. Reversing **bytes** breaks letters; reversing
    **runes** breaks combining marks and emoji.
  - *Compress* (run-length) — one `Grow`, one allocation; runs are of runes; the worst case doubles the
    length (so return the original when it is not shorter); and if the input may contain digits the
    output cannot be decoded.
  - *Longest substring without repeats* — the variable sliding window, Θ(n): when the new character was
    seen inside the window, jump `left` past it, and **never let `left` move backwards**
    (`left = max(left, last[c]+1)`; `"abba"` is the test). Run it over runes, not bytes. Lesson 16
    generalises the window.

- **The cost model.** Every string operation is one of three shapes, and you can tell which from what it
  must touch:

  | Shape | Operations |
  |---|---|
  | Θ(1), header only | `len`, `s[i]`, `s[i:j]`, assignment, `==` on unequal lengths |
  | Θ(n), reads bytes | `==`, `<`, `Compare`, `HasPrefix` (Θ(prefix)), `Index`, `Contains`, `Count`, `RuneCountInString`, hashing for `m[s]`, `EqualFold` |
  | Θ(n) + allocation | every conversion, `+`, `ToLower`/`ToUpper` (0 if unchanged ASCII), `Replace` (0 if nothing matches), `Repeat`, `Split`/`Fields` (one slice; the pieces share) |

  `strings.Index` is a hybrid — a single byte uses `IndexByte`, a short needle in a short haystack uses
  brute force, and otherwise it scans with `IndexByte` and falls back to brute force or Rabin–Karp when
  false positives pile up (`internal/stringslite`). The algorithms behind that — KMP, Rabin–Karp, Z —
  are lesson 35. Example 12 measures each row and classifies it by the ratio test.

### Go and Rust side by side

- `String` is `struct String { vec: Vec<u8> }`, owned and **mutable**; `&str` is the borrowed view — immutability belongs to the borrow. `size_of`: `String` 24, `&str` 16, `char` 4.
- A `str` is **always valid UTF-8**: `String::from_utf8` returns `Err` on bad bytes, `from_utf8_lossy` substitutes `U+FFFD`. Go lets any bytes be a string.
- `s[0]` does not compile (*the type `str` cannot be indexed by `{integer}`*); `&s[0..2]` across a letter **panics** (*byte index 2 is not a char boundary*); `s.get(0..2)` is `None`; `floor_char_boundary` (stable since 1.91) finds a safe cut.
- `.bytes()` / `.chars()` / `.char_indices()` are `s[i]`, rune iteration and `range s`; `chars()` is double-ended, so a palindrome check allocates nothing. `as_bytes()` and `into_bytes()` are free: the triangle's edges are borrows and moves.
- `s + &t` takes `s` **by value** and `push_str`es into its buffer, and `s += &t` is `push_str` — neither is quadratic (lesson 03 example 13). Rust's Θ(n²) spelling is `s = format!("{s}{t}")`. Capacity doubles (or jumps to what is required); Go's Builder follows `append`'s policy.
- Comparison is byte-wise in both. Rust case mapping is the **full** mapping (`"ß".to_uppercase()` is `"SS"`); `eq_ignore_ascii_case` is ASCII-only and there is no std `EqualFold`.
- Neither std normalizes or segments graphemes, and Rust's has no general-category lookup like Go's `unicode.Mn`.
- A `&str` cannot outlive its `String`, so a substring cannot silently pin its parent (example 13).

## Complexity Table

| Operation | Time | Allocations | Note |
|---|---|---|---|
| `len(s)` | Θ(1) | 0 | bytes, from the header |
| `utf8.RuneCountInString(s)` | Θ(n) | 0 | decodes every byte |
| k-th rune | Θ(k) | 0 | Θ(n) via `[]rune(s)[k]`, plus 1 allocation once the slice escapes or exceeds 32 runes (the runtime's stack buffer) |
| `s[i:j]` | Θ(1) | 0 | shares bytes; may pin a large parent |
| `[]byte(s)`, `string(b)`, `string(r)` | Θ(n) | 1 | free in the compiler-recognised temporaries |
| `[]rune(s)` | Θ(n) | 1 | 4 bytes per rune |
| `a == b` | Θ(1) if lengths differ, else Θ(n) | 0 | |
| `a < b`, `strings.Compare` | Θ(min(\|a\|, \|b\|)) | 0 | byte order, not dictionary order |
| `strings.Index(s, sub)` | Θ(n·m) worst, short needles; Θ(n+m) expected otherwise | 0 | hybrid; see lesson 35 |
| `strings.EqualFold` | Θ(n) | 0 | |
| `strings.ToLower` | Θ(n) | 0 or 1 | 0 if already lowercase ASCII |
| `s += x` in a loop, n times | **Θ(n²)** bytes copied | **n** | |
| `strings.Builder`, n appends | Θ(n) amortized | O(log n) | `String()` copies nothing |
| `Builder.Grow(total)` / `strings.Join` | Θ(n) | **1** | |
| `strings.Split` / `Fields` | Θ(n) | 1 | the pieces are substrings |
| Anagram: sort / counter | Θ(n log n) / Θ(n) | copies both / 0 | counter is ASCII-only with `[26]int` |
| Palindrome, two pointers | Θ(n) | 0 | decode from both ends |
| Reverse words in place | Θ(n) | 0 | reverse all, then each word |
| Run-length compress | Θ(n) | 1 with `Grow` | output ≤ 2× input before the "not shorter" check |
| Longest substring without repeats | Θ(n) | 0–1 | Θ(min(n, alphabet)) space |

## Exercises
1. For `"hello"`, `"héllo"`, `"日本語"`, `"😀"` and `"e\u0301"`, predict `len`, the rune count, and what you see. Then print the UTF-8 bytes of each in binary and point at the lead bytes.
2. Prove a substring shares its parent's bytes with `unsafe.StringData`. Then show the "edit" `"X" + s[1:]` does not.
3. Count allocations for every edge of the triangle with `testing.AllocsPerRun`. Find the three conversions the compiler makes free, and one that looks free and is not.
4. Write `TruncateRunes(s, k)` that never returns invalid UTF-8, and `RuneAt(s, k)` without converting to `[]rune`. Test them on `"a\xffb"`.
5. Build a 10,000-piece string with `+=`, `Builder`, `Builder.Grow` and `strings.Join`. Report allocations and bytes copied, and how each changes when n doubles.
6. Copy a non-empty `strings.Builder` by value and write to the copy. Explain the panic from `String()`'s source.
7. Sort `["banana", "Apple", "Éclair", "zebra"]` and explain the order byte by byte. Then compare `EqualFold` with `ToLower(a) == ToLower(b)` on allocations.
8. Write `IsAnagram` three ways. Find a pair on which the byte-counting version is wrong.
9. Write `IsPalindrome` that ignores punctuation and case, is correct for `"été"`, and allocates nothing. Then the one-deletion variant.
10. Reverse the words of a string in place in a `[]byte`, with single spaces in the output. Prove the result is valid UTF-8 for accented input.
11. Write `Compress`/`Decompress` for run-length encoding. Find the shortest input that does not round-trip, and fix the format.
12. Write `LengthOfLongestSubstring`. Write the version without `max` and find the input that breaks it; then check both against a brute-force oracle on 20,000 random strings over a four-letter alphabet.
13. Keep the first 16 bytes of 50 one-megabyte strings and measure the live heap. Repeat with `strings.Clone`.
14. Parse 100,000 `key=value` log lines four ways and count allocations per line. Get one parser to zero.
15. Find three strings where rune reversal produces something visibly wrong, and explain each with its code points.
16. Stretch — package the five problems with an ASCII fast path and a rune path, prove both against brute-force oracles on random mixed-script input, and measure fast path vs rune path vs oracle.

Write them in `practice/11-strings-bytes/`.

## Best Practices & Pitfalls
- **Decide which length you mean** — bytes, runes or graphemes — and name the variable after it.
- **Pitfall — `string(s[i])`.** It converts a byte as a code point. Use `s[i:i+1]` for the byte, or decode the rune.
- **Pitfall — slicing at an arbitrary byte.** Back off to a rune start, or slice at offsets that came from `range`.
- **Iterate with `range s`** when you want runes; index `s[i]` only when the problem is genuinely about bytes (ASCII delimiters, hex, protocols).
- **Convert to `[]rune` only when you need random access to runes** — it is 4 bytes per rune, and an allocation as soon as the result escapes or holds more than 32 runes (short, local conversions use a 32-rune stack buffer — example 8 counts both cases).
- **Build with `strings.Builder`, and `Grow` it** when you know the size. Use `strings.Join` when you have the pieces.
- **Pitfall — `s += x` in a loop.** Quadratic in Go. (Fine in Rust; `format!` in a loop is the Rust version of the mistake.)
- **Pitfall — copying a `strings.Builder` by value.** Pass `*strings.Builder`.
- **Pitfall — keeping small substrings of big strings.** `strings.Clone` the ones you keep.
- **Use `strings.EqualFold`** for case-insensitive equality, not two `ToLower` calls.
- **Pitfall — assuming sort order is dictionary order.** It is byte order. Human collation is a library concern.
- **Pitfall — comparing user text without normalizing.** `"é"` has two spellings.
- **Validate at the boundary** (`utf8.ValidString`) when bytes come from outside; Go will not do it for you.
- **Scan bytes for ASCII delimiters** — it is UTF-8-safe and allocation-free.
- **Test string code on non-ASCII input.** `"été"`, `"é£"`/`"ã©"`, `"e\u0301"` and `"👨‍👩‍👧"` catch most of the bugs in this lesson.

## Checklist
- [ ] I can say what `len(s)` counts, what a rune is, and why neither is a character.
- [ ] I can read a UTF-8 lead byte and say how long the sequence is.
- [ ] I know why strings are immutable, and that substrings are Θ(1) and may pin their parent.
- [ ] I can price every edge of the `string`/`[]byte`/`[]rune` triangle, and name the free ones.
- [ ] I can name and fix the five indexing traps.
- [ ] I can explain why `+=` is Θ(n²) in Go and not in Rust, and what `Builder.String()` does not copy.
- [ ] I know comparison, sorting and map keys are byte-wise, and what that does to accents.
- [ ] I can solve anagram, palindrome, reverse words, compress and longest-unique-substring correctly for UTF-8.
- [ ] I can find the `"abba"` bug in a sliding window.
- [ ] I can classify any string operation as Θ(1), Θ(n), or Θ(n) + an allocation, before calling it.
- [ ] I can say what a grapheme cluster is and that neither standard library segments them.

## Resources
- "Strings, bytes, runes and characters in Go" (Rob Pike): https://go.dev/blog/strings
- Go spec on string types, conversions and `for range`: https://go.dev/ref/spec#String_types
- `strings.Builder`, worth reading in full — it is short: https://github.com/golang/go/blob/master/src/strings/builder.go
- Go's string runtime (concatenation, conversions, the compiler's free cases): https://github.com/golang/go/blob/master/src/runtime/string.go
- `unicode/utf8`: https://pkg.go.dev/unicode/utf8
- The Rust Book, "Storing UTF-8 Encoded Text with Strings": https://doc.rust-lang.org/book/ch08-02-strings.html
- Rust `String` source: https://github.com/rust-lang/rust/blob/master/library/alloc/src/string.rs
- Unicode UAX #29, Text Segmentation (grapheme clusters): https://www.unicode.org/reports/tr29/
- Unicode UAX #15, Normalization Forms: https://www.unicode.org/reports/tr15/
- Examples: [examples/11-strings-bytes](examples/11-strings-bytes/) (16).
- Next: [12 — Elementary Sorts](12-elementary-sorts.md) — Part 3 begins, and comparison counting becomes the whole method.
