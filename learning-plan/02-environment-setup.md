# 02 — Environment & Toolkit

> Part of **Part 1 — Foundations**. Lesson [01](01-introduction.md) argued that you should measure
> instead of guess and prove instead of hope; this lesson builds the equipment that makes both cheap
> enough to actually do. The output is a **harness you copy into all 43 remaining lessons**. The
> one-line thesis: **an algorithm you cannot test and cannot measure is an algorithm you do not know
> is correct or fast.**

## Goals
- Lay out the `practice/` module and know where lesson code, tests and benchmarks live.
- Write **table-driven tests with subtests** by reflex, and start every table with the **edge-case checklist**.
- Know why `slices.Equal` — not `reflect.DeepEqual` or `==` — is the right comparison for algorithm output.
- Read `go test -cover`, and know exactly what 100% coverage does and does not promise.
- Write benchmarks that survive scrutiny: `b.Loop`, sub-benchmark **sweeps**, an **empty control**, and `benchstat` for before/after.
- Catch what tests can't: `-race` for concurrency, `go test -fuzz` for inputs you never thought of, `pprof` for where the time went.
- Assemble the **reusable harness**: a generic differential oracle, invariant checks, generators, a benchmark skeleton, and a one-command gate script.

## Concepts

- **The module layout.** All practice code is one module, one package per lesson. A lesson folder holds
  the implementation, its tests, its benchmarks, and (once you get there) a fuzz corpus:
  ```
  practice/
  ├── go.mod                     # github.com/Matrosovdream/dsa-mastery/practice
  └── 07-linked-lists/
      ├── list.go                # what the lesson asked for
      ├── list_test.go           # table + invariants + oracle   <- you write this
      ├── harness_test.go        # the reusable helpers          <- copied, unchanged
      ├── bench_test.go          # control + size sweep          <- copied, edited lightly
      ├── check.sh               # the gates                     <- copied, unchanged
      └── testdata/fuzz/...      # written by the fuzzer, committed
  ```
  Tests live *beside* the code in the same package, so they can reach unexported helpers — which for
  data structures (where the invariant you want to assert is usually a private field) matters a lot.

- **Table-driven, always.** One slice of cases, one loop, one `t.Run` per case. Adding a case is adding
  a line; a failure names itself; and `-run 'TestMax/negatives'` re-runs exactly one case while you debug.
  ```go
  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) { /* one case */ })
  }
  ```
  Use `t.Errorf` to record a failure and keep going, `t.Fatalf` when everything after would be nonsense.

- **The edge-case checklist comes first.** Before the interesting case, every sequence algorithm gets:
  **nil · empty · single · two · all-equal · duplicates · already-sorted · reverse-sorted · negatives ·
  extremes**. Most algorithm bugs live in those first four rows, not in the logic you were concentrating on.

- **`slices.Equal`, not `reflect.DeepEqual`.** You cannot compare slices with `==`. The two real options
  disagree about exactly one thing: `slices.Equal(nil, []int{})` is **true** (same contents),
  `reflect.DeepEqual(nil, []int{})` is **false** (different representation). Since `var out []T` stays
  nil until the first `append`, half the functions you write in this plan return nil for "no results" —
  compare contents unless nil-ness is genuinely part of the contract.

- **Coverage is a floor, not a verdict.** Coverage tells you which lines *ran*, never whether the
  assertions were right. This lesson ships a binary search with the classic `lo < hi` bug and a table
  that gives it **100.0% statement coverage while every case passes** — because none of the cases
  happens to narrow the range to a single element. An 8-line exhaustive oracle finds it immediately.
  Treat coverage as "which code did I forget to exercise", then let the oracle judge correctness.

- **Benchmarks need a control.** `BenchmarkEmptyControl` — an empty `for b.Loop() {}` — belongs in every
  package. On this machine it costs **~1.65 ns/op**, and a benchmark of a one-multiply function reports
  **1.61 ns/op**: *below* the control, i.e. pure noise. Without the control you would have published
  that number ([01](01-introduction.md) example 12).

- **One number tells you nothing; a sweep tells you the complexity.** Use `b.Run(fmt.Sprintf("n=%d", n))`
  to benchmark the same operation across growing n, then read the *column*:
  ```
  linear/n=64        20.73 ns      binary/n=64       3.750 ns
  linear/n=512      125.6  ns      binary/n=512      6.236 ns
  linear/n=4096     946.3  ns      binary/n=4096     9.061 ns
  linear/n=262144 60921    ns      binary/n=262144  14.66  ns
  ```
  8× the input costs the scan ~8× the time (O(n)); it costs binary search a flat **+3 ns per 8×** — the
  signature of O(log n). You just measured a complexity class.

- **`benchstat` for before/after.** Eyeballing two ns/op numbers is not a comparison — run each side
  `-count=10` and let `benchstat` do the statistics. Selecting the two versions with a **build tag**
  (`//go:build prealloc`) keeps one benchmark name, which is what benchstat matches on:
  ```
  Grow-10   25.471µ ± 2%   5.647µ ± 1%  -77.83% (p=0.000 n=10)   # sec/op
  Grow-10   19.000  ± 0%   1.000  ± 0%  -94.74% (p=0.000 n=10)   # allocs/op
  ```

- **`-race` finds what a passing test cannot.** A parallel max-reduction into a shared variable passed
  its test **5 runs out of 5** — and `go test -race` reported the data race with file and line numbers
  on the first try. Any lesson with a goroutine in it runs under `-race`, always. The fix is usually
  structural, not a mutex: give each worker its own accumulator and combine once.

- **`go test -fuzz` writes your regression tests for you.** Give it seeds, shape the random bytes into
  a valid input for your precondition, and compare against an oracle. Pointed at the same buggy binary
  search, the fuzzer found a counterexample in **0.02 seconds**, minimized it to a 3-element slice, and
  wrote `testdata/fuzz/FuzzSearch/d8dadd70ed1a164a` — which is now a permanent, deterministic
  regression test that runs on every plain `go test`. **Commit that file.**

- **`pprof` answers "where did the time go".** `go test -bench=. -cpuprofile=cpu.out` then
  `go tool pprof -top`. On a function that does one O(n) pass and one O(n²) pass, the profile is
  unambiguous: `CountPairs 80.85%` flat, `normalize 1.06%`. Profile before optimizing — the hot spot is
  frequently not where you'd bet.

- **An allocation budget is a unit test.** `testing.AllocsPerRun` in a `Test` function, compared against
  a number, fails the build when someone regresses it — no human required to read a benchmark and judge.
  Set the budget to the value you **measured**, not to a round number you hoped for. (Writing this
  lesson, I guessed 5 for a dedupe; it was 6.)

## The gates

A lesson is done when all five pass. `check.sh` in example 16 runs them in one command:

| Gate | Command | Catches |
|------|---------|---------|
| Format | `gofmt -l .` | style drift (must print nothing) |
| Vet | `go vet ./...` | printf mismatches, lost returns, shadowed errors |
| Test | `go test -race ./...` | wrong answers **and** data races |
| Coverage | `go test -cover ./...` | code you forgot to exercise |
| Benchmarks | `go test -bench=. -benchmem -run='^$' ./...` | complexity regressions, surprise allocations |

`-run='^$'` matches no test, so the benchmark run skips tests. `-benchmem` adds B/op and allocs/op.

## Exercises
1. Create `practice/02-environment-setup/` in the practice module. Write `Max([]int) (int, bool)` and a plain test for it; run `go test -v` and read every line of the output.
2. Rewrite that test as a table with subtests. Add the full edge-case checklist. Then run just one case with `-run 'TestMax/negatives'`.
3. Write a function returning a filtered slice. Prove to yourself with a test that its "no results" return is nil, that `slices.Equal` calls it equal to `[]int{}`, and that `reflect.DeepEqual` does not.
4. Copy this lesson's buggy `Search`. Write a table that passes and reports 100% coverage, then write the 8-line exhaustive oracle that catches the bug. Fix it, and confirm both stay green.
5. Write `BenchmarkEmptyControl` and a benchmark of a one-line function. Compare them. Then batch 1000 calls per op and report the per-call cost.
6. Benchmark linear vs binary search as a sub-benchmark sweep from n=64 to n=262144. From the ns/op column alone, argue which is O(n) and which is O(log n).
7. Use a build tag to keep two versions of one function, run each `-count=10`, and compare with `benchstat`. Report the delta and the p-value.
8. Write a parallel reduction with a shared accumulator. Confirm the test passes several times, then run `go test -race` and read the report. Fix it without a mutex.
9. Write a fuzz target comparing an implementation against an oracle. Run `-fuzztime=30s`, let it fail, inspect the file it wrote in `testdata/`, and confirm plain `go test` now reproduces the failure.
10. Stretch — assemble your own `harness_test.go`, `bench_test.go` and `check.sh`, drop them into a practice folder, and get `./check.sh` to print "all gates passed". You'll use these files for the next 43 lessons.

## Best Practices & Pitfalls
- **Test in the same package** (`package algo`, not `algo_test`). Data-structure invariants live in unexported fields, and you want to assert them.
- **`t.Helper()` in every assertion helper**, or failures report the helper's line number instead of the caller's — and you lose the only useful information in the message.
- **Failure messages carry the input.** `got X, want Y` is half a message; `f(input) = X, want Y` is one you can act on without re-reading the test.
- **Pitfall — `reflect.DeepEqual` on slices.** It says nil ≠ empty, so a perfectly correct function fails its test on the "no results" case. Use `slices.Equal`.
- **Pitfall — trusting coverage.** 100% statement coverage with all-green tests is compatible with a completely broken function. Coverage finds unexercised code; only an oracle or a property finds wrong code.
- **Pitfall — a benchmark without a control.** Anything reporting under ~2 ns/op is measuring the harness. Add the empty benchmark and compare.
- **Pitfall — mutating shared benchmark input.** If the benchmarked function sorts or reverses in place, copy inside the loop — and make sure the competing implementation pays the same copy.
- **Keep generated inputs small.** `len ≤ 10`, values in `[0,6)`. Small inputs make collisions and duplicates frequent (better coverage) and make counterexamples readable (`[4 48 49]`, not 200 numbers).
- **Fix the seed.** `rand.NewPCG(42, 7)` means a failure reproduces exactly on the next run. An unseeded generator turns a real bug into a rumour.
- **Commit `testdata/fuzz/`.** Those files are the regression suite the fuzzer earned; deleting them throws away the discovery.
- **`-race` costs ~10× time and memory** — it's a CI/pre-commit gate, not a benchmark mode. Never benchmark under `-race`.
- **Profile before optimizing.** `-cpuprofile` + `pprof -top` takes thirty seconds and routinely contradicts your guess about the hot spot.

## Checklist
- [ ] I can scaffold a lesson folder and explain what each file in it is for.
- [ ] I write table-driven tests with subtests by default, starting from the edge-case checklist.
- [ ] I know why `slices.Equal` beats `reflect.DeepEqual` for algorithm output, and what nil-vs-empty means in Go.
- [ ] I can state exactly what 100% coverage promises, and demonstrate a 100%-covered function that is wrong.
- [ ] Every benchmark I write has an empty control, and I sweep n instead of quoting one number.
- [ ] I can run a before/after comparison through `benchstat` and read the delta and p-value.
- [ ] I run `-race` on anything with a goroutine, and I can explain why a passing test proves nothing there.
- [ ] I can write a fuzz target with seeds, an oracle, and input shaping — and I commit the corpus it produces.
- [ ] I can get a CPU profile of a benchmark and name the hot function from `pprof -top`.
- [ ] I have `harness_test.go`, `bench_test.go` and `check.sh` saved, and `./check.sh` prints "all gates passed".

## Resources
- `go test` flags — the full reference: https://pkg.go.dev/cmd/go#hdr-Testing_flags
- The `testing` package (`T`, `B`, `F`, `AllocsPerRun`, `b.Loop`): https://pkg.go.dev/testing
- Go Fuzzing — the official tutorial and reference: https://go.dev/doc/security/fuzz/
- Data Race Detector: https://go.dev/doc/articles/race_detector
- Profiling Go Programs: https://go.dev/blog/pprof
- Diagnostics — profiling, tracing, debugging: https://go.dev/doc/diagnostics
- `benchstat` (install: `go install golang.org/x/perf/cmd/benchstat@latest`): https://pkg.go.dev/golang.org/x/perf/cmd/benchstat
- `slices` — `Equal`, `Clone`, `Sort`, `BinarySearch`: https://pkg.go.dev/slices
- Examples: [examples/02-environment-setup](examples/02-environment-setup/) (16).
- Next: [03 — Complexity Analysis](03-complexity-analysis.md) makes the notation precise; you'll verify its claims with the harness built here.
