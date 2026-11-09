# Step 11 — Strings, Runes & Bytes · Examples

A library of **16 runnable examples**, split into three files by difficulty.

All sixteen are single-file `package main` programs.

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Every example was `gofmt`-checked, `go vet`-ed and **run** before being added.

- Examples whose result block is marked **Output:** are deterministic: your output should match character for character.
- Examples marked **Sample output:** print timings or memory, which vary from run to run and machine to machine.

**Every example also has a Rust twin**, in the same tier file: Go → Go output → Rust → Rust output →
a short *Go vs Rust* note. The twins are std-only single files, built with `rustc --edition 2024 -O`, and
each one passed `rustfmt`, `clippy -D warnings` and a real run before being added; the Rust output
blocks are the binary's real stdout, captured — never typed. Where the two languages disagree, the note
says so and the Rust program prints what *its* numbers show, not Go's.

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste a Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

| Tier | File | Examples |
|------|------|----------|
| 🟢 Easy | [1-easy.md](1-easy.md) | 1–6 |
| 🟡 Medium | [2-medium.md](2-medium.md) | 7–11 |
| 🔴 Hard | [3-hard.md](3-hard.md) | 12–16 |

> Progress tracker: [PROGRESS.md](PROGRESS.md). Want more examples? Just ask and I'll append them to the right tier file.

## Index

### 🟢 [Easy](1-easy.md) — what a string is: bytes, runes, immutability, and what each operation costs

- [1. One string, three lengths](1-easy.md#1-one-string-three-lengths)
- [2. Immutable, and why a substring is free](1-easy.md#2-immutable-and-why-a-substring-is-free)
- [3. The `string` / `[]byte` / `[]rune` triangle](1-easy.md#3-the-string--byte--rune-triangle)
- [4. UTF-8 indexing traps](1-easy.md#4-utf-8-indexing-traps)
- [5. `+=` vs `strings.Builder`, counted](1-easy.md#5--vs-stringsbuilder-counted)
- [6. Comparison is bytes: equality, order and case](1-easy.md#6-comparison-is-bytes-equality-order-and-case)

### 🟡 [Medium](2-medium.md) — the string problem set, correct for UTF-8

- [7. Anagrams: sort, count, and the byte trap](2-medium.md#7-anagrams-sort-count-and-the-byte-trap)
- [8. Palindromes: two pointers over runes](2-medium.md#8-palindromes-two-pointers-over-runes)
- [9. Reversing bytes, runes and words](2-medium.md#9-reversing-bytes-runes-and-words)
- [10. Run-length compression, and when it makes things longer](2-medium.md#10-run-length-compression-and-when-it-makes-things-longer)
- [11. Longest substring without repeating characters](2-medium.md#11-longest-substring-without-repeating-characters)

### 🔴 [Hard](3-hard.md) — the cost model measured, the leak, zero-copy parsing, graphemes, the capstone

- [12. The cost model, measured](3-hard.md#12-the-cost-model-measured)
- [13. The substring that kept its parent alive](3-hard.md#13-the-substring-that-kept-its-parent-alive)
- [14. A tokenizer that never allocates](3-hard.md#14-a-tokenizer-that-never-allocates)
- [15. Graphemes: when a rune is not a character](3-hard.md#15-graphemes-when-a-rune-is-not-a-character)
- [16. Capstone: the string problem set, proved and measured](3-hard.md#16-capstone-the-string-problem-set-proved-and-measured)

> Lesson: [11-strings-bytes.md](../../11-strings-bytes.md) · Previous: [10 — Hash Tables & Sets](../10-hash-tables/README.md)
