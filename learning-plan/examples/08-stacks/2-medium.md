# Step 08 — Stacks · 🟡 Medium

Examples **7–11**: parsing, augmentation, and the amortized argument.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 7, 9 and 10 are deterministic. Examples 8 and 11 measure — sample output from an Apple M4, Go 1.26.3.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Shunting yard

`🟡 medium` · *Precedence as data*

Where RPN comes from. One operator stack converts infix to postfix, encoding precedence and
associativity as two comparisons rather than a grammar — the explicit-stack counterpart to
lesson 05's recursive-descent parser.

**Steps:**

1. Pop operators that bind at least as tightly, then push.
2. Make the test strict for right-associative operators.
3. Check `10-2-3` and `2^3^2`, which is where associativity shows.

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

// Example 4 evaluated RPN with one stack. This is where RPN comes from.
//
// The SHUNTING-YARD algorithm (Dijkstra, 1961) converts infix to postfix using
// one stack of operators, and it encodes precedence and associativity as two
// comparisons rather than a grammar. Lesson 05's recursive-descent parser solved
// the same problem with the CALL stack; this solves it with an explicit one.
//
// Two ways to parse an expression, and the difference between them is exactly
// the difference between example 6's recursive and iterative DFS.

type op struct {
	prec       int
	rightAssoc bool
}

var ops = map[string]op{
	"+": {1, false},
	"-": {1, false},
	"*": {2, false},
	"/": {2, false},
	"^": {3, true}, // right-associative: 2^3^2 is 2^(3^2)
}

func tokenize(s string) []string {
	for _, sym := range []string{"+", "-", "*", "/", "^", "(", ")"} {
		s = strings.ReplaceAll(s, sym, " "+sym+" ")
	}
	return strings.Fields(s)
}

// toPostfix converts infix to RPN.
//
// The rule for an incoming operator: pop everything already on the stack that
// binds at least as tightly, THEN push. "At least as tightly" is where
// associativity lives -- for a right-associative operator the test is strict.
func toPostfix(tokens []string) ([]string, error) {
	var out []string
	var stack []string

	for _, tok := range tokens {
		switch {
		case tok == "(":
			stack = append(stack, tok)

		case tok == ")":
			for len(stack) > 0 && stack[len(stack)-1] != "(" {
				out = append(out, stack[len(stack)-1])
				stack = stack[:len(stack)-1]
			}
			if len(stack) == 0 {
				return nil, fmt.Errorf("unmatched )")
			}
			stack = stack[:len(stack)-1] // discard the "("

		case isOp(tok):
			o := ops[tok]
			for len(stack) > 0 {
				top := stack[len(stack)-1]
				if !isOp(top) {
					break
				}
				t := ops[top]
				// pop while the top binds tighter, or equally tight and we are
				// left-associative
				if t.prec > o.prec || (t.prec == o.prec && !o.rightAssoc) {
					out = append(out, top)
					stack = stack[:len(stack)-1]
					continue
				}
				break
			}
			stack = append(stack, tok)

		default:
			if _, err := strconv.ParseFloat(tok, 64); err != nil {
				return nil, fmt.Errorf("not a number: %q", tok)
			}
			out = append(out, tok)
		}
	}

	for len(stack) > 0 {
		top := stack[len(stack)-1]
		if top == "(" {
			return nil, fmt.Errorf("unmatched (")
		}
		out = append(out, top)
		stack = stack[:len(stack)-1]
	}
	return out, nil
}

func isOp(s string) bool { _, ok := ops[s]; return ok }

// evalRPN, unchanged from example 4 apart from ^.
func evalRPN(tokens []string) (float64, error) {
	var st []float64
	for _, tok := range tokens {
		if !isOp(tok) {
			f, err := strconv.ParseFloat(tok, 64)
			if err != nil {
				return 0, err
			}
			st = append(st, f)
			continue
		}
		if len(st) < 2 {
			return 0, fmt.Errorf("operator %q needs two operands", tok)
		}
		b, a := st[len(st)-1], st[len(st)-2]
		st = st[:len(st)-2]
		var v float64
		switch tok {
		case "+":
			v = a + b
		case "-":
			v = a - b
		case "*":
			v = a * b
		case "/":
			if b == 0 {
				return 0, fmt.Errorf("division by zero")
			}
			v = a / b
		case "^":
			v = 1
			for i := 0; i < int(b); i++ {
				v *= a
			}
		}
		st = append(st, v)
	}
	if len(st) != 1 {
		return 0, fmt.Errorf("malformed expression")
	}
	return st[0], nil
}

func main() {
	fmt.Println("infix -> postfix -> value, with one operator stack:")
	fmt.Println()
	fmt.Printf("  %-24s %-24s %s\n", "infix", "postfix (RPN)", "value")

	for _, expr := range []string{
		"3 + 4",
		"3 + 4 * 2",
		"(3 + 4) * 2",
		"10 - 2 - 3",
		"2 ^ 3 ^ 2",
		"(1 + 2) * (3 + 4)",
		"100 / 5 / 2",
	} {
		post, err := toPostfix(tokenize(expr))
		if err != nil {
			fmt.Printf("  %-24s ERR %v\n", expr, err)
			continue
		}
		v, err := evalRPN(post)
		if err != nil {
			fmt.Printf("  %-24s %-24s ERR %v\n", expr, strings.Join(post, " "), err)
			continue
		}
		fmt.Printf("  %-24s %-24s %g\n", expr, strings.Join(post, " "), v)
	}

	fmt.Println()
	fmt.Println("the two rows that prove the rule is right:")
	fmt.Println()
	fmt.Println("  \"10 - 2 - 3\"  -> \"10 2 - 3 -\"  = 5, not 11")
	fmt.Println("     LEFT-associative: the second - pops the first before pushing.")
	fmt.Println()
	fmt.Println("  \"2 ^ 3 ^ 2\"   -> \"2 3 2 ^ ^\"   = 512, not 64")
	fmt.Println("     RIGHT-associative: the second ^ does NOT pop the first, so the")
	fmt.Println("     right side is evaluated first. That is the single `!o.rightAssoc`")
	fmt.Println("     in the comparison, and it is the whole of associativity.")

	fmt.Println()
	fmt.Println("errors:")
	for _, expr := range []string{"(1 + 2", "1 + 2)", "1 + x"} {
		_, err := toPostfix(tokenize(expr))
		fmt.Printf("  %-12q -> %v\n", expr, err)
	}

	fmt.Println()
	fmt.Println("two ways to parse the same grammar:")
	fmt.Println()
	fmt.Printf("  %-22s %-22s %s\n", "", "recursive descent", "shunting yard")
	for _, r := range [][3]string{
		{"the stack is", "the CALL stack", "an explicit slice"},
		{"precedence from", "the call graph", "a number you compare"},
		{"associativity from", "loop vs recursion", "strict vs non-strict test"},
		{"depth limit", "the goroutine stack", "available memory"},
		{"adding an operator", "a new function", "a row in a table"},
	} {
		fmt.Printf("  %-22s %-22s %s\n", r[0], r[1], r[2])
	}
	fmt.Println()
	fmt.Println("  the last row is the practical difference. Shunting yard puts the")
	fmt.Println("  grammar in DATA, so a new operator is one map entry. Recursive")
	fmt.Println("  descent puts it in code, which is easier to read and harder to")
	fmt.Println("  extend at runtime. Both are correct; they trade the same way")
	fmt.Println("  example 6's two DFS versions do.")
}
```

**Output:**

```
infix -> postfix -> value, with one operator stack:

  infix                    postfix (RPN)            value
  3 + 4                    3 4 +                    7
  3 + 4 * 2                3 4 2 * +                11
  (3 + 4) * 2              3 4 + 2 *                14
  10 - 2 - 3               10 2 - 3 -               5
  2 ^ 3 ^ 2                2 3 2 ^ ^                512
  (1 + 2) * (3 + 4)        1 2 + 3 4 + *            21
  100 / 5 / 2              100 5 / 2 /              10

the two rows that prove the rule is right:

  "10 - 2 - 3"  -> "10 2 - 3 -"  = 5, not 11
     LEFT-associative: the second - pops the first before pushing.

  "2 ^ 3 ^ 2"   -> "2 3 2 ^ ^"   = 512, not 64
     RIGHT-associative: the second ^ does NOT pop the first, so the
     right side is evaluated first. That is the single `!o.rightAssoc`
     in the comparison, and it is the whole of associativity.

errors:
  "(1 + 2"     -> unmatched (
  "1 + 2)"     -> unmatched )
  "1 + x"      -> not a number: "x"

two ways to parse the same grammar:

                         recursive descent      shunting yard
  the stack is           the CALL stack         an explicit slice
  precedence from        the call graph         a number you compare
  associativity from     loop vs recursion      strict vs non-strict test
  depth limit            the goroutine stack    available memory
  adding an operator     a new function         a row in a table

  the last row is the practical difference. Shunting yard puts the
  grammar in DATA, so a new operator is one map entry. Recursive
  descent puts it in code, which is easier to read and harder to
  extend at runtime. Both are correct; they trade the same way
  example 6's two DFS versions do.
```

**Complexity:** Θ(n) time, Θ(depth) space · the grammar lives in a table, so a new operator is one map entry

---

## 8. Min-stack in Θ(1)

`🟡 medium` · *Augmenting a stack*

"Give me the minimum at any point" sounds like it needs a heap. On a stack it does not — because
a stack's contents are strictly nested, so an answer computed at push time stays valid exactly as
long as the element does.

**Steps:**

1. Scan on demand, then pair each element with the min-so-far.
2. Then keep a second stack, pushed only when the minimum changes.
3. Find the duplicate-value input that breaks the strict `<` version.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"runtime"
	"testing"
)

// "Give me the minimum, in Theta(1), at any point" sounds like it needs a heap
// (lesson 20). On a STACK it does not -- because a stack's history is a strict
// nesting, and that lets you carry the answer along with each element.
//
// Three designs, in increasing order of cleverness.

// --- 1. scan on demand: Theta(n) per Min ---

type scanStack struct{ xs []int }

func (s *scanStack) Push(v int) { s.xs = append(s.xs, v) }
func (s *scanStack) Pop() (int, bool) {
	if len(s.xs) == 0 {
		return 0, false
	}
	v := s.xs[len(s.xs)-1]
	s.xs = s.xs[:len(s.xs)-1]
	return v, true
}
func (s *scanStack) Min() (int, bool) {
	if len(s.xs) == 0 {
		return 0, false
	}
	m := math.MaxInt
	for _, v := range s.xs { // Theta(n), every call
		if v < m {
			m = v
		}
	}
	return m, true
}

// --- 2. pair each element with the min AT THAT MOMENT: Theta(1), 2x memory ---
//
// The key insight: when v is on the stack, the minimum of everything below it
// can never change until v is popped. So it is safe to freeze it alongside v.

type pairStack struct {
	xs []struct{ val, min int }
}

func (s *pairStack) Push(v int) {
	m := v
	if len(s.xs) > 0 && s.xs[len(s.xs)-1].min < m {
		m = s.xs[len(s.xs)-1].min
	}
	s.xs = append(s.xs, struct{ val, min int }{v, m})
}
func (s *pairStack) Pop() (int, bool) {
	if len(s.xs) == 0 {
		return 0, false
	}
	v := s.xs[len(s.xs)-1].val
	s.xs = s.xs[:len(s.xs)-1]
	return v, true
}
func (s *pairStack) Min() (int, bool) {
	if len(s.xs) == 0 {
		return 0, false
	}
	return s.xs[len(s.xs)-1].min, true
}

// --- 3. a second stack, pushed only when the minimum CHANGES ---
//
// Same Theta(1), and it only stores a min when one is actually new. On data
// with few new minima -- which is most data -- the auxiliary stack stays tiny.

type minStack struct {
	xs   []int
	mins []int
}

func (s *minStack) Push(v int) {
	s.xs = append(s.xs, v)
	if len(s.mins) == 0 || v <= s.mins[len(s.mins)-1] { // <= matters: see below
		s.mins = append(s.mins, v)
	}
}
func (s *minStack) Pop() (int, bool) {
	if len(s.xs) == 0 {
		return 0, false
	}
	v := s.xs[len(s.xs)-1]
	s.xs = s.xs[:len(s.xs)-1]
	if len(s.mins) > 0 && v == s.mins[len(s.mins)-1] {
		s.mins = s.mins[:len(s.mins)-1]
	}
	return v, true
}
func (s *minStack) Min() (int, bool) {
	if len(s.mins) == 0 {
		return 0, false
	}
	return s.mins[len(s.mins)-1], true
}
func (s *minStack) auxLen() int { return len(s.mins) }

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("all three agree, step by step:")
	fmt.Println()
	fmt.Printf("  %-12s %-10s %-10s %-10s\n", "op", "scan", "pair", "two-stack")

	var a scanStack
	var b pairStack
	var c minStack
	show := func(label string) {
		m1, _ := a.Min()
		m2, _ := b.Min()
		m3, _ := c.Min()
		fmt.Printf("  %-12s %-10d %-10d %-10d\n", label, m1, m2, m3)
	}
	for _, v := range []int{5, 3, 7, 3, 8, 1} {
		a.Push(v)
		b.Push(v)
		c.Push(v)
		show(fmt.Sprintf("push %d", v))
	}
	for i := 0; i < 4; i++ {
		a.Pop()
		b.Pop()
		c.Pop()
		show("pop")
	}

	fmt.Println()
	fmt.Println("the subtle bug in design 3, and why the test is <= and not <:")
	fmt.Println()
	fmt.Println("  push 3, then push 3 again. With a strict <, the second 3 does not")
	fmt.Println("  go on the min stack -- and then popping ONE of them removes the")
	fmt.Println("  only recorded 3, leaving Min() wrong while a 3 is still present.")
	fmt.Println()
	var wrong minStack
	for _, v := range []int{3, 3} {
		wrong.xs = append(wrong.xs, v)
		if len(wrong.mins) == 0 || v < wrong.mins[len(wrong.mins)-1] { // the BUG: <
			wrong.mins = append(wrong.mins, v)
		}
	}
	wrong.Pop()
	m, _ := wrong.Min()
	fmt.Printf("  with `<`:  push 3, push 3, pop -> Min()=%d, and the stack is %v\n", m, wrong.xs)
	var right minStack
	right.Push(3)
	right.Push(3)
	right.Pop()
	m2, _ := right.Min()
	fmt.Printf("  with `<=`: push 3, push 3, pop -> Min()=%d   correct\n", m2)
	fmt.Println()
	fmt.Println("  duplicates are exactly the case a hand-written test forgets, and")
	fmt.Println("  exactly the case a model oracle finds immediately (example 16).")

	fmt.Println()
	fmt.Println("cost, over 100,000 pushes with a Min() after each:")
	fmt.Println()
	const n = 100_000
	rng := rand.New(rand.NewPCG(1, 2))
	vals := make([]int, n)
	for i := range vals {
		vals[i] = rng.IntN(1 << 20)
	}

	tScan := nsPerOp(func() {
		var s scanStack
		for _, v := range vals[:2000] { // only 2000: it is quadratic
			s.Push(v)
			m, _ := s.Min()
			sinkI = m
		}
	})
	tPair := nsPerOp(func() {
		var s pairStack
		for _, v := range vals[:2000] {
			s.Push(v)
			m, _ := s.Min()
			sinkI = m
		}
	})
	tTwo := nsPerOp(func() {
		var s minStack
		for _, v := range vals[:2000] {
			s.Push(v)
			m, _ := s.Min()
			sinkI = m
		}
	})
	fmt.Printf("  %-26s %14s %12s\n", "design (2000 elements)", "ns/op", "vs pair")
	fmt.Printf("  %-26s %14.0f %11.1fx\n", "1. scan on demand", tScan, tScan/tPair)
	fmt.Printf("  %-26s %14.0f %11.1fx\n", "2. pair (val, min)", tPair, 1.0)
	fmt.Printf("  %-26s %14.0f %11.1fx\n", "3. two stacks", tTwo, tTwo/tPair)

	// How big does the auxiliary stack actually get?
	var full minStack
	for _, v := range vals {
		full.Push(v)
	}
	fmt.Printf("\n  after %d random pushes the auxiliary min stack holds %d entries\n",
		n, full.auxLen())
	fmt.Printf("  -- %.2f%% of the data, because a new minimum gets rarer as you go.\n",
		100*float64(full.auxLen())/float64(n))
	fmt.Println()
	fmt.Println("  worst case is still Theta(n): a strictly decreasing input pushes a")
	fmt.Println("  new minimum every time, and design 3 degenerates to design 2.")

	var dec minStack
	for i := n; i > 0; i-- {
		dec.Push(i)
	}
	fmt.Printf("  with a strictly DECREASING input the aux stack holds %d entries.\n", dec.auxLen())

	fmt.Println()
	fmt.Println("what makes this possible at all:")
	fmt.Println()
	fmt.Println("  a stack's contents are strictly nested -- what is below an element")
	fmt.Println("  cannot change while that element is present. So an answer computed")
	fmt.Println("  at push time stays valid for exactly as long as the element does.")
	fmt.Println()
	fmt.Println("  that argument works for ANY associative query, not just min: max,")
	fmt.Println("  sum, gcd, running product. It does NOT work for a queue, where the")
	fmt.Println("  far end changes underneath you -- which is why a min-QUEUE needs")
	fmt.Println("  the monotonic deque of lesson 09.")

	runtime.KeepAlive(full)
}
```

**Sample output:**

```
all three agree, step by step:

  op           scan       pair       two-stack 
  push 5       5          5          5         
  push 3       3          3          3         
  push 7       3          3          3         
  push 3       3          3          3         
  push 8       3          3          3         
  push 1       1          1          1         
  pop          3          3          3         
  pop          3          3          3         
  pop          3          3          3         
  pop          3          3          3         

the subtle bug in design 3, and why the test is <= and not <:

  push 3, then push 3 again. With a strict <, the second 3 does not
  go on the min stack -- and then popping ONE of them removes the
  only recorded 3, leaving Min() wrong while a 3 is still present.

  with `<`:  push 3, push 3, pop -> Min()=0, and the stack is [3]
  with `<=`: push 3, push 3, pop -> Min()=3   correct

  duplicates are exactly the case a hand-written test forgets, and
  exactly the case a model oracle finds immediately (example 16).

cost, over 100,000 pushes with a Min() after each:

  design (2000 elements)              ns/op      vs pair
  1. scan on demand                  996668        94.9x
  2. pair (val, min)                  10508         1.0x
  3. two stacks                       12287         1.2x

  after 100000 random pushes the auxiliary min stack holds 15 entries
  -- 0.01% of the data, because a new minimum gets rarer as you go.

  worst case is still Theta(n): a strictly decreasing input pushes a
  new minimum every time, and design 3 degenerates to design 2.
  with a strictly DECREASING input the aux stack holds 100000 entries.

what makes this possible at all:

  a stack's contents are strictly nested -- what is below an element
  cannot change while that element is present. So an answer computed
  at push time stays valid for exactly as long as the element does.

  that argument works for ANY associative query, not just min: max,
  sum, gcd, running product. It does NOT work for a queue, where the
  far end changes underneath you -- which is why a min-QUEUE needs
  the monotonic deque of lesson 09.
```

**Complexity:** Min Θ(1) for both augmented designs · the auxiliary stack is Θ(1) on typical data and Θ(n) on descending input

---

## 9. A queue from two stacks

`🟡 medium` · *The aggregate argument*

A classic interview question whose interesting part is not the construction but the **amortized
proof** — lesson 03's aggregate method applied to something less obvious than `append`.

**Steps:**

1. Pour `in` into `out` only when `out` is empty; the pour reverses.
2. Count every transfer and compare against the number of operations.
3. Then try the mirror — a stack from two queues — and see why it fails.

```go
package main

import "fmt"

// A queue from two stacks. It is a classic interview question, and the
// interesting part is not the construction -- it is the AMORTIZED ARGUMENT,
// which is lesson 03's aggregate method applied to something less obvious than
// append.
//
//	in    everything pushed since the last transfer, newest on top
//	out   everything ready to leave, OLDEST on top
//
// Dequeue takes from `out`. When `out` is empty, pour ALL of `in` into it --
// which reverses the order, turning LIFO into FIFO.

type Queue struct {
	in, out []int
	moves   int // count every element transfer, so we can check the bound
}

func (q *Queue) Len() int { return len(q.in) + len(q.out) }

// Enqueue is always Theta(1).
func (q *Queue) Enqueue(v int) { q.in = append(q.in, v) }

// Dequeue is Theta(n) on the transfer and Theta(1) otherwise -- amortized
// Theta(1), because each element is transferred exactly ONCE in its lifetime.
func (q *Queue) Dequeue() (int, bool) {
	if len(q.out) == 0 {
		for len(q.in) > 0 { // pour in -> out, reversing
			v := q.in[len(q.in)-1]
			q.in = q.in[:len(q.in)-1]
			q.out = append(q.out, v)
			q.moves++
		}
	}
	if len(q.out) == 0 {
		return 0, false
	}
	v := q.out[len(q.out)-1]
	q.out = q.out[:len(q.out)-1]
	return v, true
}

func (q *Queue) Peek() (int, bool) {
	if len(q.out) == 0 {
		for len(q.in) > 0 {
			v := q.in[len(q.in)-1]
			q.in = q.in[:len(q.in)-1]
			q.out = append(q.out, v)
			q.moves++
		}
	}
	if len(q.out) == 0 {
		return 0, false
	}
	return q.out[len(q.out)-1], true
}

func (q *Queue) show(label string) {
	fmt.Printf("  %-18s in=%v out=%v\n", label, q.in, q.out)
}

// --- and the mirror: a stack from two queues, which is genuinely worse ---

type StackFromQueues struct {
	a, b  []int
	moves int
}

// Push is Theta(n): enqueue onto the empty queue, then move everything else
// behind it so the newest element is at the FRONT.
func (s *StackFromQueues) Push(v int) {
	s.b = append(s.b, v)
	for len(s.a) > 0 {
		s.b = append(s.b, s.a[0])
		s.a = s.a[1:]
		s.moves++
	}
	s.a, s.b = s.b, s.a
}

func (s *StackFromQueues) Pop() (int, bool) {
	if len(s.a) == 0 {
		return 0, false
	}
	v := s.a[0]
	s.a = s.a[1:]
	return v, true
}

func main() {
	var q Queue
	fmt.Println("a FIFO queue built from two LIFO stacks:")
	fmt.Println()
	for _, v := range []int{1, 2, 3} {
		q.Enqueue(v)
		q.show(fmt.Sprintf("enqueue %d", v))
	}

	fmt.Println()
	v, _ := q.Dequeue()
	fmt.Printf("  dequeue -> %d      <- POURED in into out, reversing it\n", v)
	q.show("")

	v, _ = q.Dequeue()
	fmt.Printf("  dequeue -> %d      <- no pour needed, out is not empty\n", v)
	q.show("")

	fmt.Println()
	q.Enqueue(4)
	q.show("enqueue 4")
	v, _ = q.Dequeue()
	fmt.Printf("  dequeue -> %d      <- still 3, because out is drained FIRST\n", v)
	v, _ = q.Dequeue()
	fmt.Printf("  dequeue -> %d      <- now the pour happens\n", v)
	_, ok := q.Dequeue()
	fmt.Printf("  dequeue -> ok=%v\n", ok)

	fmt.Println()
	fmt.Println("the amortized argument, counted rather than asserted:")
	fmt.Println()
	fmt.Printf("  %10s %14s %16s %s\n", "n", "transfers", "per operation", "worst single dequeue")
	for _, n := range []int{100, 1_000, 10_000, 100_000} {
		var r Queue
		worst := 0
		for i := 0; i < n; i++ {
			r.Enqueue(i)
		}
		for i := 0; i < n; i++ {
			before := r.moves
			r.Dequeue()
			if d := r.moves - before; d > worst {
				worst = d
			}
		}
		fmt.Printf("  %10d %14d %16.2f %d\n", n, r.moves, float64(r.moves)/float64(2*n), worst)
	}
	fmt.Println()
	fmt.Println("  transfers = n exactly, for 2n operations. Each element moves from")
	fmt.Println("  `in` to `out` ONCE and never goes back, so the total is bounded by")
	fmt.Println("  the number of elements -- amortized Theta(1) per operation, by")
	fmt.Println("  exactly the aggregate argument lesson 03 used for append.")
	fmt.Println()
	fmt.Println("  and note the last column: one unlucky dequeue moves n elements.")
	fmt.Println("  Amortized Theta(1) and worst-case Theta(n), simultaneously true --")
	fmt.Println("  lesson 03, example 8. If you have a latency SLO, this structure")
	fmt.Println("  has a tail.")

	fmt.Println()
	fmt.Println("interleaving is the case that breaks a naive analysis:")
	fmt.Println()
	var alt Queue
	for i := 0; i < 5; i++ {
		alt.Enqueue(i)
		alt.Dequeue()
	}
	fmt.Printf("  5 x (enqueue then dequeue): %d transfers -- one per element, still\n", alt.moves)
	fmt.Println("  (each pour finds exactly one waiting element, so it is Theta(1) each)")

	fmt.Println()
	fmt.Println("the mirror image -- a stack from two queues -- is genuinely worse:")
	fmt.Println()
	var s StackFromQueues
	for _, n := range []int{100, 1000} {
		var t StackFromQueues
		for i := 0; i < n; i++ {
			t.Push(i)
		}
		fmt.Printf("  %5d pushes -> %d element moves  (that is n^2/2)\n", n, t.moves)
	}
	_ = s
	fmt.Println()
	fmt.Println("  Push must move the ENTIRE queue every time, and no amortization")
	fmt.Println("  saves it: elements move again and again. Theta(n) per push, always.")
	fmt.Println()
	fmt.Println("  the asymmetry is worth noticing. Two stacks make a fine queue")
	fmt.Println("  because pouring REVERSES, and reversing twice is the identity.")
	fmt.Println("  Two queues cannot reverse anything, so the trick has no mirror.")

	fmt.Println()
	fmt.Println("would you ship this? Almost never -- lesson 09's ring buffer is a")
	fmt.Println("better queue in every way. Learn it for the amortized argument, which")
	fmt.Println("generalises far beyond queues.")
}
```

**Output:**

```
a FIFO queue built from two LIFO stacks:

  enqueue 1          in=[1] out=[]
  enqueue 2          in=[1 2] out=[]
  enqueue 3          in=[1 2 3] out=[]

  dequeue -> 1      <- POURED in into out, reversing it
                     in=[] out=[3 2]
  dequeue -> 2      <- no pour needed, out is not empty
                     in=[] out=[3]

  enqueue 4          in=[4] out=[3]
  dequeue -> 3      <- still 3, because out is drained FIRST
  dequeue -> 4      <- now the pour happens
  dequeue -> ok=false

the amortized argument, counted rather than asserted:

           n      transfers    per operation worst single dequeue
         100            100             0.50 100
        1000           1000             0.50 1000
       10000          10000             0.50 10000
      100000         100000             0.50 100000

  transfers = n exactly, for 2n operations. Each element moves from
  `in` to `out` ONCE and never goes back, so the total is bounded by
  the number of elements -- amortized Theta(1) per operation, by
  exactly the aggregate argument lesson 03 used for append.

  and note the last column: one unlucky dequeue moves n elements.
  Amortized Theta(1) and worst-case Theta(n), simultaneously true --
  lesson 03, example 8. If you have a latency SLO, this structure
  has a tail.

interleaving is the case that breaks a naive analysis:

  5 x (enqueue then dequeue): 5 transfers -- one per element, still
  (each pour finds exactly one waiting element, so it is Theta(1) each)

the mirror image -- a stack from two queues -- is genuinely worse:

    100 pushes -> 4950 element moves  (that is n^2/2)
   1000 pushes -> 499500 element moves  (that is n^2/2)

  Push must move the ENTIRE queue every time, and no amortization
  saves it: elements move again and again. Theta(n) per push, always.

  the asymmetry is worth noticing. Two stacks make a fine queue
  because pouring REVERSES, and reversing twice is the identity.
  Two queues cannot reverse anything, so the trick has no mirror.

would you ship this? Almost never -- lesson 09's ring buffer is a
better queue in every way. Learn it for the amortized argument, which
generalises far beyond queues.
```

**Complexity:** amortized Θ(1) per operation, Θ(n) worst case for one dequeue · a stack from two queues is Θ(n) per push with no amortization available

---

## 10. Writing down the call stack

`🟡 medium` · *What the `stage` field really is*

Lesson 05 said an explicit stack is what the call stack was doing for you. Here that is made
literal: converting a recursion by hand means writing down the frame the compiler would have built.

**Steps:**

1. Convert a commutative fold — a plain stack of nodes is enough.
2. Convert post-order, where the work happens *after* both calls.
3. Then convert `fib`, which has two resume points.

```go
package main

import "fmt"

// Lesson 05 said "an explicit stack is what the call stack was doing for you."
// This is that sentence made literal: a recursive function converted into a
// loop by writing down the FRAME the compiler would have built.
//
// The technique matters when the recursion has work AFTER the recursive call,
// because then a simple stack of arguments is not enough -- you also have to
// remember WHERE you were.

// --- the recursion we are going to dismantle ---

type tree struct {
	val         int
	left, right *tree
}

func sumRec(t *tree) int {
	if t == nil {
		return 0
	}
	return t.val + sumRec(t.left) + sumRec(t.right)
}

// --- attempt 1: a stack of nodes. Works, because + is associative. ---

func sumStack(t *tree) int {
	if t == nil {
		return 0
	}
	total := 0
	stack := []*tree{t}
	for len(stack) > 0 {
		n := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		total += n.val
		if n.right != nil {
			stack = append(stack, n.right)
		}
		if n.left != nil {
			stack = append(stack, n.left)
		}
	}
	return total
}

// --- attempt 2: when the order of the work matters, you need the STATE ---
//
// Post-order visits a node AFTER both children, so a popped node is not
// finished -- it must be pushed back with a marker saying "children done".
// That marker IS the program counter the CPU would have saved.

type stage int

const (
	stageEnter stage = iota // first arrival: schedule the children
	stageExit               // children finished: do the work
)

type frame struct {
	node  *tree
	stage stage
}

func postorderStack(t *tree) []int {
	var out []int
	if t == nil {
		return out
	}
	stack := []frame{{t, stageEnter}}

	for len(stack) > 0 {
		f := stack[len(stack)-1]
		stack = stack[:len(stack)-1]

		switch f.stage {
		case stageEnter:
			// Re-push OURSELVES for later, then the children on top.
			stack = append(stack, frame{f.node, stageExit})
			if f.node.right != nil {
				stack = append(stack, frame{f.node.right, stageEnter})
			}
			if f.node.left != nil {
				stack = append(stack, frame{f.node.left, stageEnter})
			}
		case stageExit:
			out = append(out, f.node.val) // the work AFTER both calls
		}
	}
	return out
}

func postorderRec(t *tree, out *[]int) {
	if t == nil {
		return
	}
	postorderRec(t.left, out)
	postorderRec(t.right, out)
	*out = append(*out, t.val)
}

// --- the same technique on a non-tree recursion: fib with an explicit stack ---

type fibFrame struct {
	n     int
	stage int // 0 = compute n-1, 1 = compute n-2, 2 = combine
	a, b  int
}

func fibStack(n int) int {
	var result int
	stack := []fibFrame{{n: n}}

	for len(stack) > 0 {
		f := &stack[len(stack)-1]

		if f.n < 2 {
			result = f.n
			stack = stack[:len(stack)-1]
			continue
		}

		switch f.stage {
		case 0:
			f.stage = 1
			stack = append(stack, fibFrame{n: f.n - 1})
		case 1:
			f.a = result
			f.stage = 2
			stack = append(stack, fibFrame{n: f.n - 2})
		case 2:
			f.b = result
			result = f.a + f.b
			stack = stack[:len(stack)-1]
		}
	}
	return result
}

func fibRec(n int) int {
	if n < 2 {
		return n
	}
	return fibRec(n-1) + fibRec(n-2)
}

func build() *tree {
	//        1
	//      /   \
	//     2     3
	//    / \
	//   4   5
	return &tree{1,
		&tree{2, &tree{val: 4}, &tree{val: 5}},
		&tree{val: 3},
	}
}

func main() {
	t := build()

	fmt.Println("attempt 1 -- a plain stack of nodes, for a commutative fold:")
	fmt.Printf("  sumRec   = %d\n", sumRec(t))
	fmt.Printf("  sumStack = %d\n", sumStack(t))
	fmt.Println()
	fmt.Println("  this works because addition does not care about order. Nothing")
	fmt.Println("  happens 'after' the recursive calls that a plain stack cannot do.")

	fmt.Println()
	fmt.Println("attempt 2 -- post-order, where the work happens AFTER both calls:")
	var rec []int
	postorderRec(t, &rec)
	fmt.Printf("  recursive %v\n", rec)
	fmt.Printf("  explicit  %v\n", postorderStack(t))
	fmt.Println()
	fmt.Println("  a popped node is only HALF done: its children have not run yet.")
	fmt.Println("  So it is pushed back with stage=exit, underneath its children.")
	fmt.Println("  When it surfaces again, the children really are finished.")
	fmt.Println()
	fmt.Println("  that `stage` field is the RETURN ADDRESS. The CPU saves one on")
	fmt.Println("  every call so it knows where to resume; converting a recursion by")
	fmt.Println("  hand means writing it down yourself.")

	fmt.Println()
	fmt.Println("the same on a non-tree recursion -- fib has TWO resume points:")
	fmt.Println()
	fmt.Printf("  %5s %12s %12s\n", "n", "recursive", "explicit")
	for _, n := range []int{0, 1, 5, 10, 20} {
		fmt.Printf("  %5d %12d %12d\n", n, fibRec(n), fibStack(n))
	}
	fmt.Println()
	fmt.Println("  three stages: compute fib(n-1), compute fib(n-2), then add. Each")
	fmt.Println("  one is a place the recursion would have resumed at, so each one")
	fmt.Println("  needs its own case -- and `a` and `b` are the saved locals.")

	fmt.Println()
	fmt.Println("what this tells you about when to bother:")
	fmt.Println()
	fmt.Println("  the conversion is MECHANICAL but not free -- you are hand-writing")
	fmt.Println("  what the compiler generates. Do it when the depth can grow with")
	fmt.Println("  the input (lesson 05, example 15), and leave the recursion alone")
	fmt.Println("  when it cannot.")
	fmt.Println()
	fmt.Println("  the tell for how hard the conversion will be: count the RESUME")
	fmt.Println("  POINTS. One recursive call at the very end (a tail call) needs no")
	fmt.Println("  stack at all -- it is a loop. Work after one call needs a stage")
	fmt.Println("  field. Work between two calls needs saved locals as well.")
}
```

**Output:**

```
attempt 1 -- a plain stack of nodes, for a commutative fold:
  sumRec   = 15
  sumStack = 15

  this works because addition does not care about order. Nothing
  happens 'after' the recursive calls that a plain stack cannot do.

attempt 2 -- post-order, where the work happens AFTER both calls:
  recursive [4 5 2 3 1]
  explicit  [4 5 2 3 1]

  a popped node is only HALF done: its children have not run yet.
  So it is pushed back with stage=exit, underneath its children.
  When it surfaces again, the children really are finished.

  that `stage` field is the RETURN ADDRESS. The CPU saves one on
  every call so it knows where to resume; converting a recursion by
  hand means writing it down yourself.

the same on a non-tree recursion -- fib has TWO resume points:

      n    recursive     explicit
      0            0            0
      1            1            1
      5            5            5
     10           55           55
     20         6765         6765

  three stages: compute fib(n-1), compute fib(n-2), then add. Each
  one is a place the recursion would have resumed at, so each one
  needs its own case -- and `a` and `b` are the saved locals.

what this tells you about when to bother:

  the conversion is MECHANICAL but not free -- you are hand-writing
  what the compiler generates. Do it when the depth can grow with
  the input (lesson 05, example 15), and leave the recursion alone
  when it cannot.

  the tell for how hard the conversion will be: count the RESUME
  POINTS. One recursive call at the very end (a tail call) needs no
  stack at all -- it is a loop. Work after one call needs a stage
  field. Work between two calls needs saved locals as well.
```

**Complexity:** unchanged · the `stage` field is the RETURN ADDRESS, and counting resume points tells you how hard a conversion will be

---

## 11. Which implementation?

`🟡 medium` · *The comparison most favourable to a list*

Lesson 07 measured a linked list losing almost everything. A stack is the cleanest possible
rematch: both structures do only Θ(1) operations at their best end, with no indexing and no
middle insertion.

**Steps:**

1. Compare a slice, a linked stack, a pooled linked stack and `container/list`.
2. Pool the nodes to remove the per-push allocation.
3. Then decide whether anything beats the slice.

```go
package main

import (
	"container/list"
	"fmt"
	"runtime"
	"testing"
)

// Lesson 07 measured a linked list losing almost every comparison to a slice.
// A stack is the cleanest possible test of that, because the two structures do
// EXACTLY the same operations here -- push and pop at one end -- with no
// indexing, no middle insertion, and no traversal to muddy the comparison.
//
// If a list is ever going to be competitive, it is here.

type sliceStack struct{ xs []int }

func (s *sliceStack) Push(v int) { s.xs = append(s.xs, v) }
func (s *sliceStack) Pop() (int, bool) {
	if len(s.xs) == 0 {
		return 0, false
	}
	v := s.xs[len(s.xs)-1]
	s.xs = s.xs[:len(s.xs)-1]
	return v, true
}

type node struct {
	val  int
	next *node
}

// listStack pushes at the HEAD, which is a linked list's Theta(1) end.
type listStack struct{ head *node }

func (s *listStack) Push(v int) { s.head = &node{val: v, next: s.head} }
func (s *listStack) Pop() (int, bool) {
	if s.head == nil {
		return 0, false
	}
	v := s.head.val
	s.head = s.head.next
	return v, true
}

// listStackPooled reuses removed nodes, removing the per-push allocation --
// the fairest possible version of a list-based stack.
type listStackPooled struct {
	head, free *node
}

func (s *listStackPooled) Push(v int) {
	n := s.free
	if n != nil {
		s.free = n.next
		n.val = v
	} else {
		n = &node{val: v}
	}
	n.next = s.head
	s.head = n
}
func (s *listStackPooled) Pop() (int, bool) {
	if s.head == nil {
		return 0, false
	}
	n := s.head
	s.head = n.next
	n.next = s.free
	s.free = n
	return n.val, true
}

var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// heapObjects returns a SIGNED delta: the GC can free more between the two
// readings than the build allocated, and an unsigned subtraction wraps to
// nonsense (this printed 18446744073709551613 the first time).
func heapObjects(build func() any) int64 {
	runtime.GC()
	var a runtime.MemStats
	runtime.ReadMemStats(&a)
	v := build()
	runtime.GC()
	var b runtime.MemStats
	runtime.ReadMemStats(&b)
	runtime.KeepAlive(v)
	d := int64(b.HeapObjects) - int64(a.HeapObjects)
	if d < 0 {
		return 0
	}
	return d
}

func main() {
	const n = 10_000

	fmt.Printf("push %d then pop %d, four implementations:\n\n", n, n)

	tSlice := nsPerOp(func() {
		var s sliceStack
		for i := 0; i < n; i++ {
			s.Push(i)
		}
		for {
			v, ok := s.Pop()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	tSliceRes := nsPerOp(func() {
		s := sliceStack{xs: make([]int, 0, n)}
		for i := 0; i < n; i++ {
			s.Push(i)
		}
		for {
			v, ok := s.Pop()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	tList := nsPerOp(func() {
		var s listStack
		for i := 0; i < n; i++ {
			s.Push(i)
		}
		for {
			v, ok := s.Pop()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	tPooled := nsPerOp(func() {
		var s listStackPooled
		for i := 0; i < n; i++ {
			s.Push(i)
		}
		for {
			v, ok := s.Pop()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	tCList := nsPerOp(func() {
		l := list.New()
		for i := 0; i < n; i++ {
			l.PushFront(i)
		}
		for l.Len() > 0 {
			e := l.Front()
			l.Remove(e)
			sinkI = e.Value.(int)
		}
	})

	fmt.Printf("  %-28s %14s %12s\n", "", "ns/op", "vs slice")
	rows := []struct {
		name string
		t    float64
	}{
		{"[]int, grown by append", tSlice},
		{"[]int, preallocated", tSliceRes},
		{"linked list", tList},
		{"linked list + free pool", tPooled},
		{"container/list", tCList},
	}
	for _, r := range rows {
		fmt.Printf("  %-28s %14.0f %11.1fx\n", r.name, r.t, r.t/tSliceRes)
	}

	objSlice := heapObjects(func() any {
		s := sliceStack{xs: make([]int, 0, n)}
		for i := 0; i < n; i++ {
			s.Push(i)
		}
		return s.xs
	})
	objList := heapObjects(func() any {
		var s listStack
		for i := 0; i < n; i++ {
			s.Push(i)
		}
		return s.head
	})

	fmt.Println()
	fmt.Printf("  heap objects for %d elements: slice %d, list %d\n", n, objSlice, objList)

	fmt.Println()
	fmt.Println("this is the comparison most favourable to a linked list in the whole")
	fmt.Println("plan -- both structures do only Theta(1) operations at their best end,")
	fmt.Println("with no indexing and no middle insertion -- and it still loses.")
	fmt.Println()
	fmt.Println("  the plain list pays one allocation per push (lesson 07, example 12)")
	fmt.Println("  the POOLED list removes that, and is still behind: every pop and")
	fmt.Println("    push is a dependent pointer load, while the slice walks an array")
	fmt.Println("  container/list boxes every int on top of everything else")

	fmt.Println()
	fmt.Println("so the recommendation for a stack in Go is unambiguous, which is")
	fmt.Println("rare enough to be worth stating plainly:")
	fmt.Println()
	fmt.Println("      use a slice. Preallocate it if you know the depth.")
	fmt.Println()
	fmt.Println("there is no workload in this lesson where a linked stack wins.")
	fmt.Println("The one thing a list is good at -- Theta(1) splice at a held handle")
	fmt.Println("(lesson 07) -- is an operation a stack does not have.")
}
```

**Sample output:**

```
push 10000 then pop 10000, four implementations:

                                        ns/op     vs slice
  []int, grown by append                51166         1.3x
  []int, preallocated                   38342         1.0x
  linked list                          126504         3.3x
  linked list + free pool              114744         3.0x
  container/list                       240008         6.3x

  heap objects for 10000 elements: slice 0, list 10000

this is the comparison most favourable to a linked list in the whole
plan -- both structures do only Theta(1) operations at their best end,
with no indexing and no middle insertion -- and it still loses.

  the plain list pays one allocation per push (lesson 07, example 12)
  the POOLED list removes that, and is still behind: every pop and
    push is a dependent pointer load, while the slice walks an array
  container/list boxes every int on top of everything else

so the recommendation for a stack in Go is unambiguous, which is
rare enough to be worth stating plainly:

      use a slice. Preallocate it if you know the depth.

there is no workload in this lesson where a linked stack wins.
The one thing a list is good at -- Theta(1) splice at a held handle
(lesson 07) -- is an operation a stack does not have.
```

**Complexity:** all Θ(1) per operation · the constants decide, and they are not close

---

> Next tier: [🔴 hard](3-hard.md).

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*