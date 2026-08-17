# Step 08 — Stacks · 🟢 Easy

Examples **1–6**: the structure that needs no structure, and the problems that *are* stacks.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Deterministic except example 2, which benchmarks the generic wrapper.

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. A slice is already a stack

`🟢 easy` · *LIFO for free*

After two lessons of trade-offs, this is a relief. A stack only ever touches the END of the
sequence — the cheap end of a slice — so there is no structure to build.

**Steps:**

1. Push with `append`, peek at `len-1`, pop by reslicing.
2. Watch that nothing shifts and nothing is chased.
3. Then handle the one case that can panic.

```go
package main

import "fmt"

// After two lessons of trade-offs, the stack is a relief: it only ever touches
// the END of the sequence, which is the cheap end of a slice (lesson 06).
//
// So in Go a slice ALREADY IS a stack. There is no structure to build:
//
//	push    xs = append(xs, v)      Theta(1) amortized
//	peek    xs[len(xs)-1]           Theta(1)
//	pop     xs = xs[:len(xs)-1]     Theta(1)
//
// Every operation is at index len-1. Nothing shifts, nothing is chased, and the
// data stays contiguous. A stack has none of lesson 07's problems because it
// never asks for anything a slice is bad at.

func main() {
	fmt.Println("a []int used directly as a stack:")
	fmt.Println()

	var stack []int

	push := func(v int) {
		stack = append(stack, v)
		fmt.Printf("  push %-3d -> %v  (len=%d cap=%d)\n", v, stack, len(stack), cap(stack))
	}
	for _, v := range []int{1, 2, 3} {
		push(v)
	}

	fmt.Printf("\n  peek     -> %d   (the top is the LAST element, not the first)\n", stack[len(stack)-1])
	fmt.Printf("  len      -> %d\n", len(stack))

	fmt.Println()
	for len(stack) > 0 {
		top := stack[len(stack)-1]
		stack = stack[:len(stack)-1]
		fmt.Printf("  pop      -> %-3d  remaining %v\n", top, stack)
	}

	fmt.Println()
	fmt.Println("the ONE thing you must get right: popping an empty stack.")
	fmt.Println()
	fmt.Println("  stack[len(stack)-1] on an empty slice is stack[-1] -- that is a")
	fmt.Println("  PANIC, not a nil you can test. So the pop signature has to report")
	fmt.Println("  emptiness, and the idiomatic way is a second return value:")
	fmt.Println()
	fmt.Println("      func Pop() (T, bool)")
	fmt.Println()
	fmt.Println("  the same shape as a map lookup or a channel receive. Callers then")
	fmt.Println("  write `if v, ok := s.Pop(); ok {` and cannot forget the check.")

	fmt.Println()
	fmt.Println("popping empty, safely:")
	fmt.Println()
	pop := func(s []int) ([]int, int, bool) {
		if len(s) == 0 {
			return s, 0, false
		}
		return s[:len(s)-1], s[len(s)-1], true
	}
	empty := []int{}
	_, v, ok := pop(empty)
	fmt.Printf("  pop on empty -> (%d, %v)   <- no panic, and the caller knows\n", v, ok)

	fmt.Println()
	fmt.Println("why LIFO falls out of a slice for free:")
	fmt.Println()
	fmt.Println("  append writes at the tail, and the tail is where a slice has spare")
	fmt.Println("  capacity (lesson 06, example 7). Reslicing to drop the last element")
	fmt.Println("  moves no data at all -- it only changes len.")
	fmt.Println()
	fmt.Println("  compare that with a QUEUE, which needs both ends: taking from the")
	fmt.Println("  front is where the slice stops being free, and that is lesson 09's")
	fmt.Println("  entire subject.")

	fmt.Println()
	fmt.Println("one memory note, from lesson 06's example 2:")
	fmt.Println()
	fmt.Println("  popping only shrinks len. The popped value is still sitting in the")
	fmt.Println("  array's spare capacity, so if T contains a pointer it stays alive.")
	fmt.Println("  Zero the slot on the way out -- example 2 does exactly that.")
}
```

**Output:**

```
a []int used directly as a stack:

  push 1   -> [1]  (len=1 cap=1)
  push 2   -> [1 2]  (len=2 cap=2)
  push 3   -> [1 2 3]  (len=3 cap=4)

  peek     -> 3   (the top is the LAST element, not the first)
  len      -> 3

  pop      -> 3    remaining [1 2]
  pop      -> 2    remaining [1]
  pop      -> 1    remaining []

the ONE thing you must get right: popping an empty stack.

  stack[len(stack)-1] on an empty slice is stack[-1] -- that is a
  PANIC, not a nil you can test. So the pop signature has to report
  emptiness, and the idiomatic way is a second return value:

      func Pop() (T, bool)

  the same shape as a map lookup or a channel receive. Callers then
  write `if v, ok := s.Pop(); ok {` and cannot forget the check.

popping empty, safely:

  pop on empty -> (0, false)   <- no panic, and the caller knows

why LIFO falls out of a slice for free:

  append writes at the tail, and the tail is where a slice has spare
  capacity (lesson 06, example 7). Reslicing to drop the last element
  moves no data at all -- it only changes len.

  compare that with a QUEUE, which needs both ends: taking from the
  front is where the slice stops being free, and that is lesson 09's
  entire subject.

one memory note, from lesson 06's example 2:

  popping only shrinks len. The popped value is still sitting in the
  array's spare capacity, so if T contains a pointer it stays alive.
  Zero the slot on the way out -- example 2 does exactly that.
```

**Complexity:** push Θ(1) amortized · peek and pop Θ(1) · a stack has none of lesson 07's problems because it never asks for anything a slice is bad at

---

## 2. Stack[T], and what the wrapper costs

`🟢 easy` · *The idiomatic Go container*

Generic, and whose **zero value is a usable empty stack** — no constructor to forget. The
`(T, bool)` pop is the same contract as a map lookup.

**Steps:**

1. Write the type so `var s Stack[int]` just works.
2. Zero the vacated slot on pop, and prove it.
3. Measure it against a bare slice — and do not assume the answer.

```go
package main

import (
	"fmt"
	"testing"
)

// The idiomatic Go container: generic, and whose ZERO VALUE is a usable empty
// stack. No constructor, no New(), nothing to forget.
//
//	var s Stack[int]   // ready to use
//
// That design comes straight from lesson 07's sentinel discussion: if the zero
// value works, callers cannot construct it wrong.

type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Len() int      { return len(s.items) }
func (s *Stack[T]) IsEmpty() bool { return len(s.items) == 0 }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

// Pop returns the top and whether there was one. The (T, bool) shape is the
// same contract as a map lookup, and it makes the empty case impossible to
// ignore.
func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	i := len(s.items) - 1
	v := s.items[i]
	s.items[i] = zero // release any pointer inside T (lesson 06, example 2)
	s.items = s.items[:i]
	return v, true
}

func (s *Stack[T]) Peek() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	return s.items[len(s.items)-1], true
}

// Reserve pre-sizes the stack when you know the depth (lesson 03).
func (s *Stack[T]) Reserve(n int) {
	if cap(s.items) < n {
		bigger := make([]T, len(s.items), n)
		copy(bigger, s.items)
		s.items = bigger
	}
}

type job struct{ name *string }

var sinkI int

func main() {
	fmt.Println("the zero value is a usable stack -- no constructor:")
	fmt.Println()
	var s Stack[string]
	fmt.Printf("  var s Stack[string]   IsEmpty=%v Len=%d\n", s.IsEmpty(), s.Len())

	for _, v := range []string{"a", "b", "c"} {
		s.Push(v)
	}
	top, _ := s.Peek()
	fmt.Printf("  after 3 pushes        Len=%d Peek=%q\n", s.Len(), top)

	for {
		v, ok := s.Pop()
		if !ok {
			break
		}
		fmt.Printf("  pop -> %q\n", v)
	}
	_, ok := s.Pop()
	fmt.Printf("  pop on empty -> ok=%v\n", ok)

	fmt.Println()
	fmt.Println("it works with any type, including ones a non-generic stack would box:")
	fmt.Println()
	type point struct{ x, y int }
	var ps Stack[point]
	ps.Push(point{1, 2})
	ps.Push(point{3, 4})
	p, _ := ps.Pop()
	fmt.Printf("  Stack[point]  popped %+v  -- stored as a VALUE, no interface\n", p)

	fmt.Println()
	fmt.Println("why Pop zeroes the slot it vacates:")
	fmt.Println()
	name := "still-referenced"
	var js Stack[job]
	js.Push(job{&name})
	js.Pop()
	// Look past len, into the capacity, at what the array still holds.
	held := js.items[:cap(js.items)][0].name
	fmt.Printf("  after Pop, items[:cap][0].name = %v   <- the zeroing worked\n", held)
	fmt.Println()
	fmt.Println("  without that line the popped job would stay reachable for as long")
	fmt.Println("  as the stack lives (lesson 04, example 3). For a Stack[int] nobody")
	fmt.Println("  cares; for a Stack[*Request] in a server it is a leak.")

	fmt.Println()
	fmt.Println("the cost of the wrapper, measured against a bare slice:")
	fmt.Println()
	const n = 1000
	tBare := nsPerOp(func() {
		xs := make([]int, 0, n)
		for i := 0; i < n; i++ {
			xs = append(xs, i)
		}
		for len(xs) > 0 {
			sinkI = xs[len(xs)-1]
			xs = xs[:len(xs)-1]
		}
	})
	tStack := nsPerOp(func() {
		var st Stack[int]
		st.Reserve(n)
		for i := 0; i < n; i++ {
			st.Push(i)
		}
		for {
			v, ok := st.Pop()
			if !ok {
				break
			}
			sinkI = v
		}
	})
	fmt.Printf("  %-28s %12.0f ns\n", "bare slice push/pop x1000", tBare)
	fmt.Printf("  %-28s %12.0f ns  %.2fx\n", "Stack[int] push/pop x1000", tStack, tStack/tBare)
	fmt.Println()
	fmt.Printf("  that is about %.1f ns of overhead per operation, over 2000 ops.\n",
		(tStack-tBare)/(2*n))
	fmt.Println()
	fmt.Println("  do NOT wave that away -- the generic methods are not inlining here,")
	fmt.Println("  so each Push and Pop is a real call, plus Pop does an extra write to")
	fmt.Println("  zero the vacated slot. The wrapper is cheap, not free.")
	fmt.Println()
	fmt.Println("  whether ~2 ns matters is a question about YOUR workload, and lesson")
	fmt.Println("  03 already gave you the way to answer it: if the stack operation is")
	fmt.Println("  the innermost thing your program does, use the bare slice. If there")
	fmt.Println("  is any real work per element -- and there almost always is -- 2 ns")
	fmt.Println("  is invisible and the contract is worth having.")

	fmt.Println()
	fmt.Println("so: bare slice inside one function, where the stack is local and")
	fmt.Println("obvious and `stack = append(stack, x)` reads fine. The type when the")
	fmt.Println("stack crosses a function or package boundary, where (T, bool) and the")
	fmt.Println("pointer hygiene are doing real work.")
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}
```

**Sample output:**

```
the zero value is a usable stack -- no constructor:

  var s Stack[string]   IsEmpty=true Len=0
  after 3 pushes        Len=3 Peek="c"
  pop -> "c"
  pop -> "b"
  pop -> "a"
  pop on empty -> ok=false

it works with any type, including ones a non-generic stack would box:

  Stack[point]  popped {x:3 y:4}  -- stored as a VALUE, no interface

why Pop zeroes the slot it vacates:

  after Pop, items[:cap][0].name = <nil>   <- the zeroing worked

  without that line the popped job would stay reachable for as long
  as the stack lives (lesson 04, example 3). For a Stack[int] nobody
  cares; for a Stack[*Request] in a server it is a leak.

the cost of the wrapper, measured against a bare slice:

  bare slice push/pop x1000             671 ns
  Stack[int] push/pop x1000            4044 ns  6.03x

  that is about 1.7 ns of overhead per operation, over 2000 ops.

  do NOT wave that away -- the generic methods are not inlining here,
  so each Push and Pop is a real call, plus Pop does an extra write to
  zero the vacated slot. The wrapper is cheap, not free.

  whether ~2 ns matters is a question about YOUR workload, and lesson
  03 already gave you the way to answer it: if the stack operation is
  the innermost thing your program does, use the bare slice. If there
  is any real work per element -- and there almost always is -- 2 ns
  is invisible and the contract is worth having.

so: bare slice inside one function, where the stack is local and
obvious and `stack = append(stack, x)` reads fine. The type when the
stack crosses a function or package boundary, where (T, bool) and the
pointer hygiene are doing real work.
```

**Complexity:** every operation Θ(1) · the wrapper costs ~1.6 ns/op, which is cheap but *not* free

---

## 3. Balanced brackets

`🟢 easy` · *The giveaway phrase*

The canonical "this problem IS a stack" example. The tell is the phrase **most recently
opened** — and a counter cannot do it.

**Steps:**

1. Push openers, and require a closer to match the top.
2. Report *where* and *why*, not just a bool.
3. Find the input a counter accepts and a stack rejects.

```go
package main

import "fmt"

// Some problems do not USE a stack -- they ARE a stack. Bracket matching is the
// clearest case, and the giveaway is the phrase "most recently opened".
//
// Whenever a problem says "the most recent unmatched X", the answer is a stack,
// because that is the only structure whose defining property is "most recent".

type result struct {
	ok     bool
	reason string
	pos    int
}

var pairs = map[rune]rune{')': '(', ']': '[', '}': '{'}

// balanced reports whether every bracket is matched and correctly nested, and
// WHERE it first goes wrong -- an error message with a position is worth far
// more than a bool.
func balanced(s string) result {
	type frame struct {
		ch  rune
		pos int
	}
	var stack []frame

	for i, ch := range s {
		switch ch {
		case '(', '[', '{':
			stack = append(stack, frame{ch, i})

		case ')', ']', '}':
			if len(stack) == 0 {
				return result{false, fmt.Sprintf("closing %q with nothing open", ch), i}
			}
			top := stack[len(stack)-1]
			if top.ch != pairs[ch] {
				return result{false,
					fmt.Sprintf("closing %q but %q was opened at %d", ch, top.ch, top.pos), i}
			}
			stack = stack[:len(stack)-1]
		}
	}

	if len(stack) > 0 {
		top := stack[len(stack)-1]
		return result{false, fmt.Sprintf("%q was never closed", top.ch), top.pos}
	}
	return result{ok: true}
}

// depth returns the maximum nesting depth -- the stack's high-water mark, which
// is the same number as the recursion depth a recursive parser would need
// (lesson 05).
func depth(s string) int {
	cur, max := 0, 0
	for _, ch := range s {
		switch ch {
		case '(', '[', '{':
			cur++
			if cur > max {
				max = cur
			}
		case ')', ']', '}':
			cur--
		}
	}
	return max
}

func main() {
	fmt.Println("the stack holds every bracket that is OPEN but not yet closed.")
	fmt.Println("A closer must match the TOP -- the most recently opened one.")
	fmt.Println()

	inputs := []string{
		"()",
		"()[]{}",
		"([{}])",
		"a(b[c]{d})e",
		"",
		"(",
		")",
		"(]",
		"([)]",
		"((())",
	}

	fmt.Printf("  %-16s %-8s %-10s %s\n", "input", "ok", "at", "reason")
	for _, in := range inputs {
		r := balanced(in)
		if r.ok {
			fmt.Printf("  %-16q %-8v %-10s\n", in, true, "-")
			continue
		}
		fmt.Printf("  %-16q %-8v %-10d %s\n", in, false, r.pos, r.reason)
	}

	fmt.Println()
	fmt.Println("the third failing case is the one a counter cannot catch:")
	fmt.Println()
	fmt.Println("  \"([)]\" has two opens and two closes, perfectly balanced by COUNT.")
	fmt.Println("  It is still wrong, because the nesting interleaves. Only a stack")
	fmt.Println("  knows WHICH bracket is currently innermost.")
	fmt.Println()
	fmt.Println("  a counter is enough for ONE bracket type. The moment there are")
	fmt.Println("  several, you need the order, and order means a stack.")

	fmt.Println()
	fmt.Println("the stack's high-water mark is the nesting depth:")
	fmt.Println()
	for _, in := range []string{"()", "((()))", "(()(()))", "a(b)c"} {
		fmt.Printf("  %-12q depth %d\n", in, depth(in))
	}
	fmt.Println()
	fmt.Println("  which is exactly the recursion depth a recursive-descent parser")
	fmt.Println("  would use for the same input (lesson 05, example 16) -- because an")
	fmt.Println("  explicit stack and the call stack are the same structure.")

	fmt.Println()
	fmt.Println("the pattern to recognise, stated once:")
	fmt.Println()
	fmt.Println("  \"the most recently opened / seen / pushed X\"   -> stack")
	fmt.Println("  \"the oldest outstanding X\"                     -> queue (lesson 09)")
	fmt.Println("  \"the smallest outstanding X\"                   -> heap  (lesson 20)")
	fmt.Println()
	fmt.Println("that one line resolves most 'which structure?' questions on sight.")

	fmt.Println()
	fmt.Println("real uses of this exact algorithm: matching braces in an editor,")
	fmt.Println("validating JSON/XML nesting, checking that every opened HTML tag is")
	fmt.Println("closed, and the tokenizer stage of every parser you will ever write.")
}
```

**Output:**

```
the stack holds every bracket that is OPEN but not yet closed.
A closer must match the TOP -- the most recently opened one.

  input            ok       at         reason
  "()"             true     -         
  "()[]{}"         true     -         
  "([{}])"         true     -         
  "a(b[c]{d})e"    true     -         
  ""               true     -         
  "("              false    0          '(' was never closed
  ")"              false    0          closing ')' with nothing open
  "(]"             false    1          closing ']' but '(' was opened at 0
  "([)]"           false    2          closing ')' but '[' was opened at 1
  "((())"          false    0          '(' was never closed

the third failing case is the one a counter cannot catch:

  "([)]" has two opens and two closes, perfectly balanced by COUNT.
  It is still wrong, because the nesting interleaves. Only a stack
  knows WHICH bracket is currently innermost.

  a counter is enough for ONE bracket type. The moment there are
  several, you need the order, and order means a stack.

the stack's high-water mark is the nesting depth:

  "()"         depth 1
  "((()))"     depth 3
  "(()(()))"   depth 3
  "a(b)c"      depth 1

  which is exactly the recursion depth a recursive-descent parser
  would use for the same input (lesson 05, example 16) -- because an
  explicit stack and the call stack are the same structure.

the pattern to recognise, stated once:

  "the most recently opened / seen / pushed X"   -> stack
  "the oldest outstanding X"                     -> queue (lesson 09)
  "the smallest outstanding X"                   -> heap  (lesson 20)

that one line resolves most 'which structure?' questions on sight.

real uses of this exact algorithm: matching braces in an editor,
validating JSON/XML nesting, checking that every opened HTML tag is
closed, and the tokenizer stage of every parser you will ever write.
```

**Complexity:** Θ(n) time, Θ(depth) space — and that depth is exactly the recursion depth a recursive parser would use

---

## 4. RPN evaluation

`🟢 easy` · *No parentheses, no precedence*

Reverse Polish notation puts the operator after its operands, so evaluating it is a stack and
nothing else — twenty lines with no grammar and no recursion.

**Steps:**

1. Push numbers; on an operator pop two, apply, push the result.
2. Trace the stack after every token.
3. Then check the operand order with `-` and `/`, not `+` and `*`.

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

// Reverse Polish notation puts the operator AFTER its operands: "3 4 +".
// No parentheses, no precedence rules, and evaluating it is a stack and
// nothing else -- twenty lines with no grammar and no recursion.
//
// That is not a coincidence. RPN is what an expression looks like once the
// structure is already resolved; example 7 converts infix to it.

type evalError struct {
	msg string
	tok int
}

func (e *evalError) Error() string { return fmt.Sprintf("token %d: %s", e.tok, e.msg) }

// evalRPN: push numbers, and on an operator pop TWO, apply, push the result.
func evalRPN(tokens []string) (float64, error) {
	var stack []float64

	for i, tok := range tokens {
		switch tok {
		case "+", "-", "*", "/":
			if len(stack) < 2 {
				return 0, &evalError{fmt.Sprintf("operator %q needs 2 operands, stack has %d", tok, len(stack)), i}
			}
			// ORDER MATTERS: the second pop is the LEFT operand.
			b := stack[len(stack)-1]
			a := stack[len(stack)-2]
			stack = stack[:len(stack)-2]

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
					return 0, &evalError{"division by zero", i}
				}
				v = a / b
			}
			stack = append(stack, v)

		default:
			f, err := strconv.ParseFloat(tok, 64)
			if err != nil {
				return 0, &evalError{fmt.Sprintf("not a number: %q", tok), i}
			}
			stack = append(stack, f)
		}
	}

	if len(stack) != 1 {
		return 0, &evalError{fmt.Sprintf("expression left %d values on the stack, want 1", len(stack)), len(tokens)}
	}
	return stack[0], nil
}

// trace shows the stack after every token, which makes the algorithm obvious.
func trace(expr string) {
	tokens := strings.Fields(expr)
	var stack []float64
	fmt.Printf("  %-8s %s\n", "token", "stack after")
	for _, tok := range tokens {
		switch tok {
		case "+", "-", "*", "/":
			b := stack[len(stack)-1]
			a := stack[len(stack)-2]
			stack = stack[:len(stack)-2]
			var v float64
			switch tok {
			case "+":
				v = a + b
			case "-":
				v = a - b
			case "*":
				v = a * b
			case "/":
				v = a / b
			}
			stack = append(stack, v)
		default:
			f, _ := strconv.ParseFloat(tok, 64)
			stack = append(stack, f)
		}
		fmt.Printf("  %-8s %v\n", tok, stack)
	}
}

func main() {
	fmt.Println("evaluating \"3 4 + 2 *\"  (which is (3+4)*2):")
	fmt.Println()
	trace("3 4 + 2 *")

	fmt.Println()
	fmt.Println("more, including the ones that catch a wrong operand order:")
	fmt.Println()
	fmt.Printf("  %-28s %-12s %s\n", "expression", "result", "infix")
	for _, c := range []struct{ expr, infix string }{
		{"3 4 +", "3 + 4"},
		{"3 4 + 2 *", "(3 + 4) * 2"},
		{"3 4 2 * +", "3 + 4 * 2"},
		{"10 2 -", "10 - 2"},
		{"2 10 -", "2 - 10"},
		{"100 5 / 2 /", "100 / 5 / 2"},
		{"5 1 2 + 4 * + 3 -", "5 + ((1+2)*4) - 3"},
	} {
		v, err := evalRPN(strings.Fields(c.expr))
		if err != nil {
			fmt.Printf("  %-28s %-12s %s\n", c.expr, "ERR", err)
			continue
		}
		fmt.Printf("  %-28s %-12g %s\n", c.expr, v, c.infix)
	}
	fmt.Println()
	fmt.Println("  rows 4 and 5 are why the operand order matters: the SECOND pop is")
	fmt.Println("  the left operand. Swap them and subtraction and division silently")
	fmt.Println("  give the wrong sign -- and addition still looks fine, so a test")
	fmt.Println("  suite of only + and * would never notice.")

	fmt.Println()
	fmt.Println("malformed input is rejected, not crashed on:")
	fmt.Println()
	for _, expr := range []string{"3 +", "+", "3 4", "3 0 /", "3 x +", ""} {
		_, err := evalRPN(strings.Fields(expr))
		fmt.Printf("  %-12q -> %v\n", expr, err)
	}

	fmt.Println()
	fmt.Println("why RPN needs no parentheses and no precedence:")
	fmt.Println()
	fmt.Println("  the ORDER of the tokens already encodes the tree. \"3 4 2 * +\"")
	fmt.Println("  can only mean 3 + (4*2), because the * consumes the two values")
	fmt.Println("  immediately below it. There is nothing left to disambiguate.")
	fmt.Println()
	fmt.Println("  so evaluation is one linear pass with a stack, and no grammar --")
	fmt.Println("  which is exactly why compilers and virtual machines use a")
	fmt.Println("  stack-based intermediate form. Example 7 does the conversion.")
}
```

**Output:**

```
evaluating "3 4 + 2 *"  (which is (3+4)*2):

  token    stack after
  3        [3]
  4        [3 4]
  +        [7]
  2        [7 2]
  *        [14]

more, including the ones that catch a wrong operand order:

  expression                   result       infix
  3 4 +                        7            3 + 4
  3 4 + 2 *                    14           (3 + 4) * 2
  3 4 2 * +                    11           3 + 4 * 2
  10 2 -                       8            10 - 2
  2 10 -                       -8           2 - 10
  100 5 / 2 /                  10           100 / 5 / 2
  5 1 2 + 4 * + 3 -            14           5 + ((1+2)*4) - 3

  rows 4 and 5 are why the operand order matters: the SECOND pop is
  the left operand. Swap them and subtraction and division silently
  give the wrong sign -- and addition still looks fine, so a test
  suite of only + and * would never notice.

malformed input is rejected, not crashed on:

  "3 +"        -> token 1: operator "+" needs 2 operands, stack has 1
  "+"          -> token 0: operator "+" needs 2 operands, stack has 0
  "3 4"        -> token 2: expression left 2 values on the stack, want 1
  "3 0 /"      -> token 2: division by zero
  "3 x +"      -> token 1: not a number: "x"
  ""           -> token 0: expression left 0 values on the stack, want 1

why RPN needs no parentheses and no precedence:

  the ORDER of the tokens already encodes the tree. "3 4 2 * +"
  can only mean 3 + (4*2), because the * consumes the two values
  immediately below it. There is nothing left to disambiguate.

  so evaluation is one linear pass with a stack, and no grammar --
  which is exactly why compilers and virtual machines use a
  stack-based intermediate form. Example 7 does the conversion.
```

**Complexity:** Θ(n) time, Θ(depth) space · the token ORDER already encodes the tree, which is why compilers use a stack-based intermediate form

---

## 5. Undo/redo

`🟢 easy` · *Two stacks and one rule*

The construction is easy; the **rule** is what people get wrong. A new action must clear the
redo stack, or you can redo your way into a state that never existed.

**Steps:**

1. Store commands and their inverses, not snapshots.
2. Undo pops one stack and pushes the other.
3. Then do a new action after an undo and check what happens to redo.

```go
package main

import (
	"fmt"
	"strings"
)

// Undo/redo is two stacks and one rule, and it is worth seeing because the RULE
// is the part people get wrong.
//
//	undo stack   commands already applied, newest on top
//	redo stack   commands undone, newest on top
//
//	do(cmd)   apply, push onto undo, and CLEAR redo
//	undo()    pop undo, invert, push onto redo
//	redo()    pop redo, apply, push onto undo
//
// Clearing redo on a new action is the rule. Skip it and you can redo your way
// into a document state that never existed.

type command interface {
	apply(*strings.Builder)
	invert() command
	String() string
}

type insertCmd struct {
	at   int
	text string
}

func (c insertCmd) apply(b *strings.Builder) {
	s := b.String()
	b.Reset()
	b.WriteString(s[:c.at] + c.text + s[c.at:])
}
func (c insertCmd) invert() command { return deleteCmd{c.at, c.text} }
func (c insertCmd) String() string  { return fmt.Sprintf("insert %q at %d", c.text, c.at) }

type deleteCmd struct {
	at   int
	text string
}

func (c deleteCmd) apply(b *strings.Builder) {
	s := b.String()
	b.Reset()
	b.WriteString(s[:c.at] + s[c.at+len(c.text):])
}
func (c deleteCmd) invert() command { return insertCmd{c.at, c.text} }
func (c deleteCmd) String() string  { return fmt.Sprintf("delete %q at %d", c.text, c.at) }

type editor struct {
	buf  strings.Builder
	undo []command
	redo []command
}

func (e *editor) Do(c command) {
	c.apply(&e.buf)
	e.undo = append(e.undo, c)
	e.redo = e.redo[:0] // THE RULE: a new action invalidates the redo history
}

func (e *editor) Undo() bool {
	if len(e.undo) == 0 {
		return false
	}
	c := e.undo[len(e.undo)-1]
	e.undo = e.undo[:len(e.undo)-1]
	c.invert().apply(&e.buf)
	e.redo = append(e.redo, c)
	return true
}

func (e *editor) Redo() bool {
	if len(e.redo) == 0 {
		return false
	}
	c := e.redo[len(e.redo)-1]
	e.redo = e.redo[:len(e.redo)-1]
	c.apply(&e.buf)
	e.undo = append(e.undo, c)
	return true
}

func (e *editor) show(label string) {
	fmt.Printf("  %-22s %-14q undo=%d redo=%d\n", label, e.buf.String(), len(e.undo), len(e.redo))
}

func main() {
	var e editor
	e.show("start")

	e.Do(insertCmd{0, "hello"})
	e.show(`Do insert "hello"`)
	e.Do(insertCmd{5, " world"})
	e.show(`Do insert " world"`)
	e.Do(deleteCmd{0, "hello"})
	e.show(`Do delete "hello"`)

	fmt.Println()
	e.Undo()
	e.show("Undo")
	e.Undo()
	e.show("Undo")

	fmt.Println()
	e.Redo()
	e.show("Redo")
	e.Redo()
	e.show("Redo")
	fmt.Printf("  %-22s %v\n", "Redo again ->", e.Redo())

	fmt.Println()
	fmt.Println("now THE RULE -- a new action after an undo clears the redo stack:")
	fmt.Println()
	var f editor
	f.Do(insertCmd{0, "abc"})
	f.Do(insertCmd{3, "def"})
	f.show("two inserts")
	f.Undo()
	f.show("Undo")
	f.Do(insertCmd{3, "XYZ"})
	f.show("Do a NEW action")
	fmt.Printf("  %-22s %v   <- the old redo is gone, correctly\n", "Redo ->", f.Redo())

	fmt.Println()
	fmt.Println("  without that clear, redo would replay \"def\" on top of \"abcXYZ\"")
	fmt.Println("  and produce \"abcXYZdef\" -- a state the user never created. Editors")
	fmt.Println("  that get this wrong corrupt documents.")

	fmt.Println()
	fmt.Println("why two STACKS and not two queues or a list:")
	fmt.Println()
	fmt.Println("  undo always means 'the most recent thing I did' -- the definition")
	fmt.Println("  of LIFO. Redo means 'the most recent thing I undid'. Both are")
	fmt.Println("  most-recent-first, so both are stacks (example 3's rule).")
	fmt.Println()
	fmt.Println("  and note what each stack stores: the COMMAND, not the document.")
	fmt.Println("  Storing snapshots would be Theta(document) per step; storing the")
	fmt.Println("  command plus its inverse is Theta(edit). That is the command")
	fmt.Println("  pattern, and it is what makes undo affordable at all.")

	fmt.Println()
	fmt.Println("the one design decision left: bounding the undo stack. A long session")
	fmt.Println("grows it without limit, so real editors cap it -- which needs removal")
	fmt.Println("from the BOTTOM as well as the top, and that is a deque (lesson 09).")
}
```

**Output:**

```
  start                  ""             undo=0 redo=0
  Do insert "hello"      "hello"        undo=1 redo=0
  Do insert " world"     "hello world"  undo=2 redo=0
  Do delete "hello"      " world"       undo=3 redo=0

  Undo                   "hello world"  undo=2 redo=1
  Undo                   "hello"        undo=1 redo=2

  Redo                   "hello world"  undo=2 redo=1
  Redo                   " world"       undo=3 redo=0
  Redo again ->          false

now THE RULE -- a new action after an undo clears the redo stack:

  two inserts            "abcdef"       undo=2 redo=0
  Undo                   "abc"          undo=1 redo=1
  Do a NEW action        "abcXYZ"       undo=2 redo=0
  Redo ->                false   <- the old redo is gone, correctly

  without that clear, redo would replay "def" on top of "abcXYZ"
  and produce "abcXYZdef" -- a state the user never created. Editors
  that get this wrong corrupt documents.

why two STACKS and not two queues or a list:

  undo always means 'the most recent thing I did' -- the definition
  of LIFO. Redo means 'the most recent thing I undid'. Both are
  most-recent-first, so both are stacks (example 3's rule).

  and note what each stack stores: the COMMAND, not the document.
  Storing snapshots would be Theta(document) per step; storing the
  command plus its inverse is Theta(edit). That is the command
  pattern, and it is what makes undo affordable at all.

the one design decision left: bounding the undo stack. A long session
grows it without limit, so real editors cap it -- which needs removal
from the BOTTOM as well as the top, and that is a deque (lesson 09).
```

**Complexity:** all three operations Θ(1) · Θ(edit) per step instead of Θ(document), which is what makes undo affordable

---

## 6. Iterative DFS, and the one-line switch to BFS

`🟢 easy` · *The container decides the algorithm*

For a graph traversal the stack is not a workaround — it IS the algorithm. Swap it for a queue
and you get a completely different traversal.

**Steps:**

1. Write DFS recursively, then with an explicit stack.
2. Note the two details: reverse push, and mark `seen` on **pop**.
3. Then change one line and watch it become BFS.

```go
package main

import "fmt"

// Lesson 05 converted recursion to a loop plus an explicit stack, and framed it
// as a way to survive deep input. Here it is again from the other side: for a
// graph traversal the stack is not a workaround, it IS the algorithm --
// and swapping it for a queue changes the algorithm entirely.

type graph map[string][]string

var g = graph{
	"A": {"B", "C"},
	"B": {"D", "E"},
	"C": {"F"},
	"D": {},
	"E": {"F"},
	"F": {},
}

// dfsRecursive: the call stack does the remembering.
func dfsRecursive(g graph, at string, seen map[string]bool, out *[]string) {
	if seen[at] {
		return
	}
	seen[at] = true
	*out = append(*out, at)
	for _, next := range g[at] {
		dfsRecursive(g, next, seen, out)
	}
}

// dfsIterative: you do the remembering. Note the two differences from the
// recursive version, both of which matter:
//
//  1. push the neighbours in REVERSE, because a stack reverses them again
//  2. mark seen when you POP, not when you push -- a node can be pushed several
//     times before it is first visited
func dfsIterative(g graph, start string) []string {
	var out []string
	seen := map[string]bool{}
	stack := []string{start}

	for len(stack) > 0 {
		at := stack[len(stack)-1]
		stack = stack[:len(stack)-1]

		if seen[at] {
			continue
		}
		seen[at] = true
		out = append(out, at)

		ns := g[at]
		for i := len(ns) - 1; i >= 0; i-- { // reverse push
			if !seen[ns[i]] {
				stack = append(stack, ns[i])
			}
		}
	}
	return out
}

// bfs is the SAME CODE with the stack replaced by a queue. That one substitution
// turns depth-first into breadth-first -- which is lesson 09's whole point.
func bfs(g graph, start string) []string {
	var out []string
	seen := map[string]bool{start: true}
	queue := []string{start}

	for len(queue) > 0 {
		at := queue[0]    // FRONT, not back
		queue = queue[1:] // the only difference
		out = append(out, at)

		for _, next := range g[at] {
			if !seen[next] {
				seen[next] = true
				queue = append(queue, next)
			}
		}
	}
	return out
}

// dfsWithPath keeps the path on the stack alongside the node, which is how you
// answer "how did I get here?" without a parent map.
func dfsWithPath(g graph, start, target string) []string {
	type frame struct {
		at   string
		path []string
	}
	seen := map[string]bool{}
	stack := []frame{{start, []string{start}}}

	for len(stack) > 0 {
		f := stack[len(stack)-1]
		stack = stack[:len(stack)-1]

		if f.at == target {
			return f.path
		}
		if seen[f.at] {
			continue
		}
		seen[f.at] = true

		ns := g[f.at]
		for i := len(ns) - 1; i >= 0; i-- {
			next := ns[i]
			if !seen[next] {
				// COPY the path: every branch needs its own (lesson 05, example 10)
				p := make([]string, len(f.path), len(f.path)+1)
				copy(p, f.path)
				stack = append(stack, frame{next, append(p, next)})
			}
		}
	}
	return nil
}

func main() {
	fmt.Println("the graph:")
	fmt.Println()
	fmt.Println("      A")
	fmt.Println("     / \\")
	fmt.Println("    B   C")
	fmt.Println("   / \\   \\")
	fmt.Println("  D   E - F")
	fmt.Println()

	var rec []string
	dfsRecursive(g, "A", map[string]bool{}, &rec)
	fmt.Printf("  DFS recursive  %v\n", rec)
	fmt.Printf("  DFS iterative  %v\n", dfsIterative(g, "A"))
	fmt.Printf("  BFS (a queue)  %v\n", bfs(g, "A"))

	fmt.Println()
	fmt.Println("the two versions of DFS agree -- but only because of two details:")
	fmt.Println()
	fmt.Println("  1. neighbours are pushed in REVERSE order. A stack hands them back")
	fmt.Println("     reversed, so pushing forwards would visit them right-to-left.")
	fmt.Println()
	fmt.Println("  2. `seen` is set when a node is POPPED, not when it is pushed. A")
	fmt.Println("     node reachable by two paths gets pushed twice before either is")
	fmt.Println("     visited, so the check has to happen at pop time.")
	fmt.Println()
	fmt.Println("     (BFS is the opposite: it marks on PUSH, because otherwise a node")
	fmt.Println("      can enter the queue many times and the shortest-path property")
	fmt.Println("      breaks. Same structure, different rule -- lesson 22.)")

	fmt.Println()
	fmt.Println("the substitution that changes the algorithm:")
	fmt.Println()
	fmt.Println("      stack:  at := xs[len(xs)-1]; xs = xs[:len(xs)-1]   -> DEPTH first")
	fmt.Println("      queue:  at := xs[0];         xs = xs[1:]            -> BREADTH first")
	fmt.Println()
	fmt.Println("  one line. Everything else is identical. That is worth internalising:")
	fmt.Println("  DFS and BFS are the same traversal with a different container, and")
	fmt.Println("  the container is what decides the order you meet the graph in.")

	fmt.Println()
	fmt.Printf("carrying the path on the stack: A -> F is %v\n", dfsWithPath(g, "A", "F"))
	fmt.Println()
	fmt.Println("  note the path is COPIED into each frame. Sharing one slice across")
	fmt.Println("  branches is lesson 05's example 10 bug, and it produces answers")
	fmt.Println("  that look plausible and are wrong.")

	fmt.Println()
	fmt.Println("why prefer the explicit stack here at all?")
	fmt.Println()
	fmt.Println("  the recursive DFS is shorter and clearer, and for a bounded graph")
	fmt.Println("  it is the right choice. The explicit version wins when the depth")
	fmt.Println("  can grow with the input -- a chain-shaped graph is n frames deep")
	fmt.Println("  (lesson 05, example 15), and an explicit stack lives on the heap.")
}
```

**Output:**

```
the graph:

      A
     / \
    B   C
   / \   \
  D   E - F

  DFS recursive  [A B D E F C]
  DFS iterative  [A B D E F C]
  BFS (a queue)  [A B C D E F]

the two versions of DFS agree -- but only because of two details:

  1. neighbours are pushed in REVERSE order. A stack hands them back
     reversed, so pushing forwards would visit them right-to-left.

  2. `seen` is set when a node is POPPED, not when it is pushed. A
     node reachable by two paths gets pushed twice before either is
     visited, so the check has to happen at pop time.

     (BFS is the opposite: it marks on PUSH, because otherwise a node
      can enter the queue many times and the shortest-path property
      breaks. Same structure, different rule -- lesson 22.)

the substitution that changes the algorithm:

      stack:  at := xs[len(xs)-1]; xs = xs[:len(xs)-1]   -> DEPTH first
      queue:  at := xs[0];         xs = xs[1:]            -> BREADTH first

  one line. Everything else is identical. That is worth internalising:
  DFS and BFS are the same traversal with a different container, and
  the container is what decides the order you meet the graph in.

carrying the path on the stack: A -> F is [A B E F]

  note the path is COPIED into each frame. Sharing one slice across
  branches is lesson 05's example 10 bug, and it produces answers
  that look plausible and are wrong.

why prefer the explicit stack here at all?

  the recursive DFS is shorter and clearer, and for a bounded graph
  it is the right choice. The explicit version wins when the depth
  can grow with the input -- a chain-shaped graph is n frames deep
  (lesson 05, example 15), and an explicit stack lives on the heap.
```

**Complexity:** Θ(V+E) either way · the difference between `xs[len-1]` and `xs[0]` is the entire difference between depth-first and breadth-first

---

> Next tier: [🟡 medium](2-medium.md).

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*