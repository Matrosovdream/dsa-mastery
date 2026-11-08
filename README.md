# dsa-mastery

My notes and practice for data structures & algorithms — implemented, benchmarked and tested in **Go**,
with a **Rust twin** beside every runnable example so the two languages can be read side by side.

Companion repo to [golang-learning](https://github.com/Matrosovdream/golang-learning): that one teaches
the language and backend services, this one teaches the structures and algorithms underneath them.
Same shape (lessons → graded examples → practice → projects), different subject.

## The lessons

45 lessons in [learning-plan/](learning-plan/), numbered 01 to 45. They run from complexity analysis
and Go's memory model, through the classic structures (lists, heaps, trees, graphs) and the paradigms
(greedy, divide & conquer, backtracking, DP), and end with a capstone. Each lesson is one markdown
file with notes, exercises, pitfalls and a checklist. Each lesson also gets a set of small runnable
examples under [learning-plan/examples/](learning-plan/examples/), graded easy → medium → hard. Every
example is a Go program with its real output, followed by the same example in **Rust** with *its* real
output and a short *Go vs Rust* note — where the two languages disagree, the Rust program prints what
its own numbers show rather than repeating Go's claim. See
[learning-plan/examples/README.md](learning-plan/examples/README.md) for what "the same example" means.

Where I'm at is tracked in [learning-plan/PROGRESS.md](learning-plan/PROGRESS.md).

## The rule of this repo

**Implement it, then prove it.** Every structure gets three things:

1. A **from-scratch implementation** — so the mechanics are understood, not memorized.
2. A **test** — table-driven, plus a brute-force oracle or a `go test -fuzz` property where one fits.
3. A **benchmark** — `testing.B` with `ReportAllocs`, because "O(n log n)" and "fast" are different claims.

Then, when the stdlib already ships it (`slices.Sort`, `container/heap`, `map`, `sync.Map`), the lesson
says so and shows the idiomatic version. Hand-rolling is for learning; production code uses the stdlib.

## How to use it

1. Open the next lesson, e.g. [learning-plan/03-complexity-analysis.md](learning-plan/03-complexity-analysis.md).
2. Read it, then retype the examples into a scratch folder and run them:
   ```bash
   mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex
   go mod init scratch        # first time only
   # paste an example into main.go, then:
   go run .
   ```
   To run the Rust twin instead — std only, nothing to install but `rustc`:
   ```bash
   mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
   # paste the Rust block into main.rs, then (always -O: the examples measure time):
   rustc --edition 2024 -O main.rs && ./main
   ```
3. Write your own answers to the exercises in [practice/](practice/) — one folder per lesson.
4. Run the gates before you call a lesson done:
   ```bash
   gofmt -l . && go vet ./... && go test -race ./... && go test -bench=. -benchmem ./...
   ```
   and for any Rust twin you add or change:
   ```bash
   rustfmt --edition 2024 --check main.rs \
     && clippy-driver --edition 2024 -D warnings main.rs \
     && rustc --edition 2024 -O main.rs && ./main
   ```
5. Update PROGRESS.md when a lesson is done.

## Layout

```
dsa-mastery/
├── learning-plan/
│   ├── README.md              # the plan — 45 lessons in 9 parts
│   ├── PROGRESS.md            # where I am
│   ├── NN-topic.md            # one lesson per file
│   ├── examples/NN-topic/     # graded runnable examples (1-easy / 2-medium / 3-hard)
│   └── example-projects/      # bigger builds that apply several lessons at once
└── practice/
    ├── go.mod                 # one module for all practice code
    └── NN-topic/              # my own solutions to the exercises
```

## Adding more practice examples

The examples live in `learning-plan/examples/<lesson>/`, split across `1-easy.md`, `2-medium.md`,
and `3-hard.md`. To add more, keep the numbering going in the matching tier file and add the entry
to that lesson's `README.md` index and `PROGRESS.md`. Each example is a full `package main` program
with its real output underneath — run it before adding it.
