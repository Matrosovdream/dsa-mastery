# Step 08 — Stacks · 🟡 Medium

Examples **7–11**: parsing, augmentation, and the amortized argument.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 7, 9 and 10 are deterministic. Examples 8 and 11 measure — sample output from an Apple M4, Go 1.26.3.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

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

**Rust:**

```rust
use std::fmt;

// Example 4 evaluated RPN with one stack. This is where RPN comes from.
//
// The SHUNTING-YARD algorithm (Dijkstra, 1961) converts infix to postfix using
// one stack of operators, and it encodes precedence and associativity as two
// comparisons rather than a grammar. Lesson 05's recursive-descent parser solved
// the same problem with the CALL stack; this solves it with an explicit one.
//
// Two ways to parse an expression, and the difference between them is exactly
// the difference between example 6's recursive and iterative DFS.

#[derive(Clone, Copy)]
struct Op {
    prec: u8,
    right_assoc: bool,
}

// Go keeps this in a package-level `map[string]op`. Rust has no mutable (or
// lazily built) globals without `LazyLock`, and a five-row table does not need
// hashing: a const array of (symbol, Op) rows is still "grammar as data", and
// adding an operator is still one new row.
const OPS: [(&str, Op); 5] = [
    (
        "+",
        Op {
            prec: 1,
            right_assoc: false,
        },
    ),
    (
        "-",
        Op {
            prec: 1,
            right_assoc: false,
        },
    ),
    (
        "*",
        Op {
            prec: 2,
            right_assoc: false,
        },
    ),
    (
        "/",
        Op {
            prec: 2,
            right_assoc: false,
        },
    ),
    // right-associative: 2^3^2 is 2^(3^2)
    (
        "^",
        Op {
            prec: 3,
            right_assoc: true,
        },
    ),
];

fn op_of(tok: &str) -> Option<Op> {
    OPS.iter().find(|(sym, _)| *sym == tok).map(|&(_, op)| op)
}

// Go returns `error` built with fmt.Errorf. Rust spells the failure modes as an
// enum, so a caller can match on them instead of parsing a message.
enum Error {
    UnmatchedClose,
    UnmatchedOpen,
    NotANumber(String),
    NeedsTwoOperands(String),
    DivisionByZero,
    Malformed,
}

impl fmt::Display for Error {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Error::UnmatchedClose => write!(f, "unmatched )"),
            Error::UnmatchedOpen => write!(f, "unmatched ("),
            Error::NotANumber(tok) => write!(f, "not a number: {tok:?}"),
            Error::NeedsTwoOperands(tok) => write!(f, "operator {tok:?} needs two operands"),
            Error::DivisionByZero => write!(f, "division by zero"),
            Error::Malformed => write!(f, "malformed expression"),
        }
    }
}

fn tokenize(s: &str) -> Vec<String> {
    let mut s = s.to_string();
    for sym in ["+", "-", "*", "/", "^", "(", ")"] {
        s = s.replace(sym, &format!(" {sym} "));
    }
    s.split_whitespace().map(str::to_string).collect()
}

/// Converts infix to RPN.
///
/// The rule for an incoming operator: pop everything already on the stack that
/// binds at least as tightly, THEN push. "At least as tightly" is where
/// associativity lives -- for a right-associative operator the test is strict.
///
/// The output tokens are *borrowed* from the input (elided lifetime: the
/// returned `&str`s live as long as `tokens`): nothing is copied, and the
/// borrow checker guarantees `tokens` outlives the result.
fn to_postfix(tokens: &[String]) -> Result<Vec<&str>, Error> {
    let mut out: Vec<&str> = Vec::new();
    let mut stack: Vec<&str> = Vec::new();

    for tok in tokens {
        let tok = tok.as_str();
        if tok == "(" {
            stack.push(tok);
        } else if tok == ")" {
            while let Some(&top) = stack.last() {
                if top == "(" {
                    break;
                }
                out.push(top);
                stack.pop();
            }
            // pop() is None only if the stack ran dry without finding "("
            if stack.pop().is_none() {
                return Err(Error::UnmatchedClose); // otherwise: discard the "("
            }
        } else if let Some(o) = op_of(tok) {
            while let Some(&top) = stack.last() {
                let Some(t) = op_of(top) else { break }; // "(" is not an operator
                // pop while the top binds tighter, or equally tight and we are
                // left-associative
                if t.prec > o.prec || (t.prec == o.prec && !o.right_assoc) {
                    out.push(top);
                    stack.pop();
                } else {
                    break;
                }
            }
            stack.push(tok);
        } else {
            if tok.parse::<f64>().is_err() {
                return Err(Error::NotANumber(tok.to_string()));
            }
            out.push(tok);
        }
    }

    while let Some(top) = stack.pop() {
        if top == "(" {
            return Err(Error::UnmatchedOpen);
        }
        out.push(top);
    }
    Ok(out)
}

/// evalRPN, unchanged from example 4 apart from ^.
fn eval_rpn(tokens: &[&str]) -> Result<f64, Error> {
    let mut st: Vec<f64> = Vec::new();
    for &tok in tokens {
        if op_of(tok).is_none() {
            let f = tok
                .parse::<f64>()
                .map_err(|_| Error::NotANumber(tok.to_string()))?;
            st.push(f);
            continue;
        }
        // `let ... else` pops both operands or bails out: no length check, no
        // index arithmetic (Go: `b, a := st[len-1], st[len-2]`)
        let (Some(b), Some(a)) = (st.pop(), st.pop()) else {
            return Err(Error::NeedsTwoOperands(tok.to_string()));
        };
        let v = match tok {
            "+" => a + b,
            "-" => a - b,
            "*" => a * b,
            "/" => {
                if b == 0.0 {
                    return Err(Error::DivisionByZero);
                }
                a / b
            }
            // Go multiplies in a loop; for the non-negative integer exponents used here
            // powi gives the same repeated multiplication
            _ => a.powi(b as i32),
        };
        st.push(v);
    }
    match st[..] {
        [v] => Ok(v),
        _ => Err(Error::Malformed),
    }
}

fn main() {
    println!("infix -> postfix -> value, with one operator stack:");
    println!();
    println!("  {:<24} {:<24} value", "infix", "postfix (RPN)");

    for expr in [
        "3 + 4",
        "3 + 4 * 2",
        "(3 + 4) * 2",
        "10 - 2 - 3",
        "2 ^ 3 ^ 2",
        "(1 + 2) * (3 + 4)",
        "100 / 5 / 2",
    ] {
        let tokens = tokenize(expr);
        let post = match to_postfix(&tokens) {
            Ok(p) => p,
            Err(e) => {
                println!("  {expr:<24} ERR {e}");
                continue;
            }
        };
        match eval_rpn(&post) {
            Ok(v) => println!("  {expr:<24} {:<24} {v}", post.join(" ")),
            Err(e) => println!("  {expr:<24} {:<24} ERR {e}", post.join(" ")),
        }
    }

    println!();
    println!("the two rows that prove the rule is right:");
    println!();
    println!("  \"10 - 2 - 3\"  -> \"10 2 - 3 -\"  = 5, not 11");
    println!("     LEFT-associative: the second - pops the first before pushing.");
    println!();
    println!("  \"2 ^ 3 ^ 2\"   -> \"2 3 2 ^ ^\"   = 512, not 64");
    println!("     RIGHT-associative: the second ^ does NOT pop the first, so the");
    println!("     right side is evaluated first. That is the single `!o.right_assoc`");
    println!("     in the comparison, and it is the whole of associativity.");

    println!();
    println!("errors:");
    for expr in ["(1 + 2", "1 + 2)", "1 + x"] {
        if let Err(e) = to_postfix(&tokenize(expr)) {
            println!("  {:<12} -> {e}", format!("{expr:?}"));
        }
    }

    println!();
    println!("two ways to parse the same grammar:");
    println!();
    println!("  {:<22} {:<22} shunting yard", "", "recursive descent");
    for r in [
        ["the stack is", "the CALL stack", "an explicit Vec"],
        ["precedence from", "the call graph", "a number you compare"],
        [
            "associativity from",
            "loop vs recursion",
            "strict vs non-strict test",
        ],
        ["depth limit", "the thread's stack", "available memory"],
        ["adding an operator", "a new function", "a row in a table"],
    ] {
        println!("  {:<22} {:<22} {}", r[0], r[1], r[2]);
    }
    println!();
    println!("  the last row is the practical difference. Shunting yard puts the");
    println!("  grammar in DATA, so a new operator is one table row. Recursive");
    println!("  descent puts it in code, which is easier to read and harder to");
    println!("  extend at runtime. Both are correct; they trade the same way");
    println!("  example 6's two DFS versions do.");
}
```

**Rust output:**

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
     right side is evaluated first. That is the single `!o.right_assoc`
     in the comparison, and it is the whole of associativity.

errors:
  "(1 + 2"     -> unmatched (
  "1 + 2)"     -> unmatched )
  "1 + x"      -> not a number: "x"

two ways to parse the same grammar:

                         recursive descent      shunting yard
  the stack is           the CALL stack         an explicit Vec
  precedence from        the call graph         a number you compare
  associativity from     loop vs recursion      strict vs non-strict test
  depth limit            the thread's stack     available memory
  adding an operator     a new function         a row in a table

  the last row is the practical difference. Shunting yard puts the
  grammar in DATA, so a new operator is one table row. Recursive
  descent puts it in code, which is easier to read and harder to
  extend at runtime. Both are correct; they trade the same way
  example 6's two DFS versions do.
```

> **Go vs Rust:** Same seven rows, same RPN, same values as Go, byte for byte except four lines that name Go-specific things (`rightAssoc`, "slice", "goroutine stack", "map entry"). The operator table is a `const` array searched linearly because a `HashMap` cannot be built in a `const` (Go has a package-level `map`), and `to_postfix` returns `Vec<&str>` *borrowed* from the token list, so the compiler enforces that the tokens outlive the RPN; failures are an `enum Error`, not `fmt.Errorf` strings.

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

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// "Give me the minimum, in Theta(1), at any point" sounds like it needs a heap
// (lesson 20). On a STACK it does not -- because a stack's history is a strict
// nesting, and that lets you carry the answer along with each element.
//
// Three designs, in increasing order of cleverness. Go writes each one as its
// own type with the same three methods; Rust writes that down once, as a trait,
// which lets the benchmark below be ONE generic function instead of three
// copies. `Min()` returns Option<i64> (None when empty) where Go returns
// `(int, bool)`.
trait Stack: Default {
    fn push(&mut self, v: i64);
    fn pop(&mut self) -> Option<i64>;
    fn min(&self) -> Option<i64>;
}

// --- 1. scan on demand: Theta(n) per Min ---

#[derive(Default)]
struct ScanStack {
    xs: Vec<i64>,
}

impl Stack for ScanStack {
    fn push(&mut self, v: i64) {
        self.xs.push(v);
    }
    fn pop(&mut self) -> Option<i64> {
        self.xs.pop()
    }
    fn min(&self) -> Option<i64> {
        self.xs.iter().copied().min() // Theta(n), every call; None when empty
    }
}

// --- 2. pair each element with the min AT THAT MOMENT: Theta(1), 2x memory ---
//
// The key insight: when v is on the stack, the minimum of everything below it
// can never change until v is popped. So it is safe to freeze it alongside v.

#[derive(Default)]
struct PairStack {
    xs: Vec<(i64, i64)>, // (val, min of everything up to and including val)
}

impl Stack for PairStack {
    fn push(&mut self, v: i64) {
        let m = self.xs.last().map_or(v, |&(_, below)| below.min(v));
        self.xs.push((v, m));
    }
    fn pop(&mut self) -> Option<i64> {
        self.xs.pop().map(|(val, _)| val)
    }
    fn min(&self) -> Option<i64> {
        self.xs.last().map(|&(_, m)| m)
    }
}

// --- 3. a second stack, pushed only when the minimum CHANGES ---
//
// Same Theta(1), and it only stores a min when one is actually new. On data
// with few new minima -- which is most data -- the auxiliary stack stays tiny.

#[derive(Default)]
struct MinStack {
    xs: Vec<i64>,
    mins: Vec<i64>,
}

impl Stack for MinStack {
    fn push(&mut self, v: i64) {
        self.xs.push(v);
        if self.mins.last().is_none_or(|&m| v <= m) {
            // <= matters: see below
            self.mins.push(v);
        }
    }
    fn pop(&mut self) -> Option<i64> {
        let v = self.xs.pop()?; // None on an empty stack, like Go's (0, false)
        if self.mins.last() == Some(&v) {
            self.mins.pop();
        }
        Some(v)
    }
    fn min(&self) -> Option<i64> {
        self.mins.last().copied()
    }
}

impl MinStack {
    fn aux_len(&self) -> usize {
        self.mins.len()
    }
}

/// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
/// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

/// The number, or the word None: Go's `%d` of a failed lookup prints a fake 0.
fn opt(m: Option<i64>) -> String {
    m.map_or_else(|| "None".to_string(), |v| v.to_string())
}

fn show(label: &str, a: &ScanStack, b: &PairStack, c: &MinStack) {
    println!(
        "  {label:<12} {:<10} {:<10} {:<10}",
        opt(a.min()),
        opt(b.min()),
        opt(c.min())
    );
}

// ---- the kit's ns/op harness: stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

/// One generic benchmark instead of Go's three copy-pasted closures: push every
/// value and ask for the minimum after each.
fn bench<S: Stack>(vals: &[i64]) -> f64 {
    ns_per_op(|| {
        let mut s = S::default();
        for &v in vals {
            s.push(v);
            black_box(s.min());
        }
    })
}

// ---- Pcg: a port of Go's math/rand/v2 PCG-DXSM and its IntN ----
// Go's example seeds rand.NewPCG(1, 2); with the same generator and seed this
// program sees the very same 100,000 values, so the auxiliary-stack sizes
// below are the same numbers, not just similar ones.
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

fn main() {
    println!("all three agree, step by step:");
    println!();
    println!(
        "  {:<12} {:<10} {:<10} {:<10}",
        "op", "scan", "pair", "two-stack"
    );

    let mut a = ScanStack::default();
    let mut b = PairStack::default();
    let mut c = MinStack::default();
    for v in [5, 3, 7, 3, 8, 1] {
        a.push(v);
        b.push(v);
        c.push(v);
        show(&format!("push {v}"), &a, &b, &c);
    }
    for _ in 0..4 {
        a.pop();
        b.pop();
        c.pop();
        show("pop", &a, &b, &c);
    }

    println!();
    println!("the subtle bug in design 3, and why the test is <= and not <:");
    println!();
    println!("  push 3, then push 3 again. With a strict <, the second 3 does not");
    println!("  go on the min stack -- and then popping ONE of them removes the");
    println!("  only recorded 3, leaving Min() with nothing to report while a 3 is");
    println!("  still on the stack.");
    println!();
    let mut wrong = MinStack::default();
    for v in [3, 3] {
        wrong.xs.push(v);
        if wrong.mins.last().is_none_or(|&m| v < m) {
            // the BUG: <
            wrong.mins.push(v);
        }
    }
    wrong.pop();
    println!(
        "  with `<`:  push 3, push 3, pop -> Min()={}, and the stack is {}",
        opt(wrong.min()),
        go_slice(&wrong.xs)
    );
    let mut right = MinStack::default();
    right.push(3);
    right.push(3);
    right.pop();
    println!(
        "  with `<=`: push 3, push 3, pop -> Min()={}   correct",
        opt(right.min())
    );
    println!();
    println!("  Go's Min() returned (0, false) and the program printed the 0, a made-up");
    println!("  minimum; here the Option prints None, and nobody can mistake it for a value.");
    println!();
    println!("  duplicates are exactly the case a hand-written test forgets, and");
    println!("  exactly the case a model oracle finds immediately (example 16).");

    println!();
    println!("cost, over 100,000 pushes with a Min() after each:");
    println!();
    const N: usize = 100_000;
    let mut rng = Pcg::new(1, 2);
    let vals: Vec<i64> = (0..N).map(|_| rng.int_n(1 << 20) as i64).collect();

    let head = black_box(&vals[..2000]); // only 2000: the scan is quadratic
    let t_scan = bench::<ScanStack>(head);
    let t_pair = bench::<PairStack>(head);
    let t_two = bench::<MinStack>(head);
    println!(
        "  {:<26} {:>14} {:>12}",
        "design (2000 elements)", "ns/op", "vs pair"
    );
    println!(
        "  {:<26} {:>14.0} {:>11.1}x",
        "1. scan on demand",
        t_scan,
        t_scan / t_pair
    );
    println!(
        "  {:<26} {:>14.0} {:>11.1}x",
        "2. pair (val, min)", t_pair, 1.0
    );
    println!(
        "  {:<26} {:>14.0} {:>11.1}x",
        "3. two stacks",
        t_two,
        t_two / t_pair
    );

    // How big does the auxiliary stack actually get?
    let mut full = MinStack::default();
    for &v in &vals {
        full.push(v);
    }
    println!(
        "\n  after {N} random pushes the auxiliary min stack holds {} entries",
        full.aux_len()
    );
    println!(
        "  -- {:.2}% of the data, because a new minimum gets rarer as you go.",
        100.0 * full.aux_len() as f64 / N as f64
    );
    println!();
    println!("  worst case is still Theta(n): a strictly decreasing input pushes a");
    println!("  new minimum every time, and design 3 degenerates to design 2.");

    let mut dec = MinStack::default();
    for i in (1..=N as i64).rev() {
        dec.push(i);
    }
    println!(
        "  with a strictly DECREASING input the aux stack holds {} entries.",
        dec.aux_len()
    );

    println!();
    println!("what makes this possible at all:");
    println!();
    println!("  a stack's contents are strictly nested -- what is below an element");
    println!("  cannot change while that element is present. So an answer computed");
    println!("  at push time stays valid for exactly as long as the element does.");
    println!();
    println!("  that argument works for ANY associative query, not just min: max,");
    println!("  sum, gcd, running product. It does NOT work for a queue, where the");
    println!("  far end changes underneath you -- which is why a min-QUEUE needs");
    println!("  the monotonic deque of lesson 09.");
}
```

**Rust sample output:**

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
  only recorded 3, leaving Min() with nothing to report while a 3 is
  still on the stack.

  with `<`:  push 3, push 3, pop -> Min()=None, and the stack is [3]
  with `<=`: push 3, push 3, pop -> Min()=3   correct

  Go's Min() returned (0, false) and the program printed the 0, a made-up
  minimum; here the Option prints None, and nobody can mistake it for a value.

  duplicates are exactly the case a hand-written test forgets, and
  exactly the case a model oracle finds immediately (example 16).

cost, over 100,000 pushes with a Min() after each:

  design (2000 elements)              ns/op      vs pair
  1. scan on demand                  248790        56.4x
  2. pair (val, min)                   4415         1.0x
  3. two stacks                        2188         0.5x

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

> **Go vs Rust:** Same step table, same `<` vs `<=` bug, and the same 15 and 100000 auxiliary-stack sizes as Go, because `Pcg` is the port of Go's PCG(1, 2) from example 14 of the introduction lesson. One line differs: Go printed `Min()=0` for the buggy stack because `Min()` returned `(0, false)` and the program ignored the `false`; the `Option` prints `None`. Timings over four runs: scan 237-251 us, 54-57x the pair stack (Go: 61-95x in the runs I saw), pair 4.4 us, two-stack 2.1-2.2 us, i.e. consistently 0.5x the pair stack where Go measured 0.8-1.2x (the half-size push is a plausible reason, not measured).

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

**Rust:**

```rust
use std::collections::VecDeque;

// A queue from two stacks. It is a classic interview question, and the
// interesting part is not the construction -- it is the AMORTIZED ARGUMENT,
// which is lesson 03's aggregate method applied to something less obvious than
// push.
//
//	input   everything pushed since the last transfer, newest on top
//	out     everything ready to leave, OLDEST on top
//
// (Go calls the first one `in`; that is a keyword in Rust.)
//
// Dequeue takes from `out`. When `out` is empty, pour ALL of `input` into it --
// which reverses the order, turning LIFO into FIFO.

#[derive(Default)]
struct Queue {
    input: Vec<i64>,
    out: Vec<i64>,
    moves: usize, // count every element transfer, so we can check the bound
}

impl Queue {
    fn len(&self) -> usize {
        self.input.len() + self.out.len()
    }

    /// Enqueue is always Theta(1).
    fn enqueue(&mut self, v: i64) {
        self.input.push(v);
    }

    /// Pour `input` into `out`, but only when `out` is empty. Go writes this loop
    /// out twice (Dequeue and Peek); one private method does it here.
    fn refill(&mut self) {
        if self.out.is_empty() {
            while let Some(v) = self.input.pop() {
                // pour input -> out, reversing
                self.out.push(v);
                self.moves += 1;
            }
        }
    }

    /// Dequeue is Theta(n) on the transfer and Theta(1) otherwise -- amortized
    /// Theta(1), because each element is transferred exactly ONCE in its lifetime.
    fn dequeue(&mut self) -> Option<i64> {
        self.refill();
        self.out.pop()
    }

    /// Takes `&mut self` even though it removes nothing: peeking may pour, and
    /// Rust makes that visible in the signature (Go's Peek hides it behind `*Queue`).
    fn peek(&mut self) -> Option<i64> {
        self.refill();
        self.out.last().copied()
    }

    fn show(&self, label: &str) {
        println!(
            "  {label:<18} in={} out={}",
            go_slice(&self.input),
            go_slice(&self.out)
        );
    }
}

/// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
/// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

// --- and the mirror: a stack from two queues, which is genuinely worse ---
//
// Go uses a slice as a queue (`s.a = s.a[1:]`). Rust has a real queue in std,
// VecDeque, so the two queues here are two VecDeques.

#[derive(Default)]
struct StackFromQueues {
    a: VecDeque<i64>,
    b: VecDeque<i64>,
    moves: usize,
}

impl StackFromQueues {
    /// Push is Theta(n): enqueue onto the empty queue, then move everything else
    /// behind it so the newest element is at the FRONT.
    fn push(&mut self, v: i64) {
        self.b.push_back(v);
        while let Some(x) = self.a.pop_front() {
            self.b.push_back(x);
            self.moves += 1;
        }
        std::mem::swap(&mut self.a, &mut self.b);
    }

    fn pop(&mut self) -> Option<i64> {
        self.a.pop_front()
    }
}

fn main() {
    let mut q = Queue::default();
    println!("a FIFO queue built from two LIFO stacks:");
    println!();
    for v in [1, 2, 3] {
        q.enqueue(v);
        q.show(&format!("enqueue {v}"));
    }

    println!();
    let v = q.dequeue().unwrap();
    println!("  dequeue -> {v}      <- POURED in into out, reversing it");
    q.show("");

    let v = q.dequeue().unwrap();
    println!("  dequeue -> {v}      <- no pour needed, out is not empty");
    q.show("");

    println!();
    q.enqueue(4);
    q.show("enqueue 4");
    let v = q.dequeue().unwrap();
    println!("  dequeue -> {v}      <- still 3, because out is drained FIRST");
    let v = q.dequeue().unwrap();
    println!("  dequeue -> {v}      <- now the pour happens");
    println!("  dequeue -> {:?}", q.dequeue());
    // the queue is empty now: len() and peek() agree with the line above
    assert_eq!((q.len(), q.peek()), (0, None));

    println!();
    println!("the amortized argument, counted rather than asserted:");
    println!();
    println!(
        "  {:>10} {:>14} {:>16} worst single dequeue",
        "n", "transfers", "per operation"
    );
    for n in [100, 1_000, 10_000, 100_000] {
        let mut r = Queue::default();
        let mut worst = 0;
        for i in 0..n {
            r.enqueue(i);
        }
        for _ in 0..n {
            let before = r.moves;
            r.dequeue();
            worst = worst.max(r.moves - before);
        }
        println!(
            "  {n:>10} {:>14} {:>16.2} {worst}",
            r.moves,
            r.moves as f64 / (2 * n) as f64
        );
    }
    println!();
    println!("  transfers = n exactly, for 2n operations. Each element moves from");
    println!("  `input` to `out` ONCE and never goes back, so the total is bounded by");
    println!("  the number of elements -- amortized Theta(1) per operation, by");
    println!("  exactly the aggregate argument lesson 03 used for push.");
    println!();
    println!("  and note the last column: one unlucky dequeue moves n elements.");
    println!("  Amortized Theta(1) and worst-case Theta(n), simultaneously true --");
    println!("  lesson 03, example 8. If you have a latency SLO, this structure");
    println!("  has a tail.");

    println!();
    println!("interleaving is the case that breaks a naive analysis:");
    println!();
    let mut alt = Queue::default();
    for i in 0..5 {
        alt.enqueue(i);
        alt.dequeue();
    }
    println!(
        "  5 x (enqueue then dequeue): {} transfers -- one per element, still",
        alt.moves
    );
    println!("  (each pour finds exactly one waiting element, so it is Theta(1) each)");

    println!();
    println!("the mirror image -- a stack from two queues -- is genuinely worse:");
    println!();
    for n in [100, 1000] {
        let mut t = StackFromQueues::default();
        for i in 0..n {
            t.push(i);
        }
        println!(
            "  {n:>5} pushes -> {} element moves  (that is n^2/2)",
            t.moves
        );
        // sanity check that it really is a stack: the newest element pops first
        assert_eq!(t.pop(), Some(n - 1));
    }
    println!();
    println!("  Push must move the ENTIRE queue every time, and no amortization");
    println!("  saves it: elements move again and again. Theta(n) per push, always.");
    println!();
    println!("  the asymmetry is worth noticing. Two stacks make a fine queue");
    println!("  because pouring REVERSES, and reversing twice is the identity.");
    println!("  Two queues cannot reverse anything, so the trick has no mirror.");

    println!();
    println!("would you ship this? Almost never -- lesson 09's ring buffer is a");
    println!("better queue in every way (and std's VecDeque is one). Learn it for");
    println!("the amortized argument, which generalises far beyond queues.");
}
```

**Rust output:**

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
  dequeue -> None

the amortized argument, counted rather than asserted:

           n      transfers    per operation worst single dequeue
         100            100             0.50 100
        1000           1000             0.50 1000
       10000          10000             0.50 10000
      100000         100000             0.50 100000

  transfers = n exactly, for 2n operations. Each element moves from
  `input` to `out` ONCE and never goes back, so the total is bounded by
  the number of elements -- amortized Theta(1) per operation, by
  exactly the aggregate argument lesson 03 used for push.

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
better queue in every way (and std's VecDeque is one). Learn it for
the amortized argument, which generalises far beyond queues.
```

> **Go vs Rust:** Same transfer counts as Go (100 / 1000 / 10000 / 100000, worst single dequeue n, 4950 and 499500 for the mirror); the only output differences are `dequeue -> None` (Go printed `ok=false`) and two prose words. `in` is a Rust keyword so the field is `input`, `peek` has to take `&mut self` because it may pour (the signature shows what Go's pointer receiver hides), and the stack-from-two-queues uses `VecDeque` where Go reslices (`s.a = s.a[1:]`).

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

**Rust:**

```rust
// Lesson 05 said "an explicit stack is what the call stack was doing for you."
// This is that sentence made literal: a recursive function converted into a
// loop by writing down the FRAME the compiler would have built.
//
// The technique matters when the recursion has work AFTER the recursive call,
// because then a simple stack of arguments is not enough -- you also have to
// remember WHERE you were.

// --- the recursion we are going to dismantle ---

// Go: `left, right *tree`, with nil for "no child". Rust has no null pointers
// in safe code: the absent child is `None`, the present one is an owned `Box`.
//
// One more place the call stack hides in Rust: the Drop the compiler generates
// for this type is itself recursive, so dropping a very deep chain of Boxes can
// overflow the stack (not run here, because it would crash). Example 11's linked
// stack avoids that with an iterative Drop, which is this example's technique.
struct Tree {
    val: i64,
    left: Option<Box<Tree>>,
    right: Option<Box<Tree>>,
}

fn sum_rec(t: Option<&Tree>) -> i64 {
    match t {
        None => 0,
        Some(t) => t.val + sum_rec(t.left.as_deref()) + sum_rec(t.right.as_deref()),
    }
}

// --- attempt 1: a stack of nodes. Works, because + is associative. ---
//
// The stack holds `&Tree`: borrowed references into the tree, not copies and
// not owners. The borrow checker keeps the tree alive and unchanged until the
// stack is gone.

fn sum_stack(t: Option<&Tree>) -> i64 {
    let mut total = 0;
    let mut stack: Vec<&Tree> = t.into_iter().collect(); // empty if t is None
    while let Some(n) = stack.pop() {
        total += n.val;
        if let Some(r) = n.right.as_deref() {
            stack.push(r);
        }
        if let Some(l) = n.left.as_deref() {
            stack.push(l);
        }
    }
    total
}

// --- attempt 2: when the order of the work matters, you need the STATE ---
//
// Post-order visits a node AFTER both children, so a popped node is not
// finished -- it must be pushed back with a marker saying "children done".
// That marker IS the program counter the CPU would have saved.

// Go: `type stage int` plus two iota constants. A Rust enum cannot hold a
// third, invalid value, and `match` must handle every variant.
#[derive(Clone, Copy)]
enum Stage {
    Enter, // first arrival: schedule the children
    Exit,  // children finished: do the work
}

struct Frame<'a> {
    node: &'a Tree,
    stage: Stage,
}

fn postorder_stack(t: Option<&Tree>) -> Vec<i64> {
    let mut out = Vec::new();
    let mut stack: Vec<Frame> = t
        .into_iter()
        .map(|node| Frame {
            node,
            stage: Stage::Enter,
        })
        .collect();

    while let Some(f) = stack.pop() {
        match f.stage {
            Stage::Enter => {
                // Re-push OURSELVES for later, then the children on top.
                stack.push(Frame {
                    node: f.node,
                    stage: Stage::Exit,
                });
                if let Some(r) = f.node.right.as_deref() {
                    stack.push(Frame {
                        node: r,
                        stage: Stage::Enter,
                    });
                }
                if let Some(l) = f.node.left.as_deref() {
                    stack.push(Frame {
                        node: l,
                        stage: Stage::Enter,
                    });
                }
            }
            Stage::Exit => out.push(f.node.val), // the work AFTER both calls
        }
    }
    out
}

// Go threads `out *[]int` through the recursion; Rust threads `&mut Vec`.
fn postorder_rec(t: Option<&Tree>, out: &mut Vec<i64>) {
    let Some(t) = t else { return };
    postorder_rec(t.left.as_deref(), out);
    postorder_rec(t.right.as_deref(), out);
    out.push(t.val);
}

// --- the same technique on a non-tree recursion: fib with an explicit stack ---

#[derive(Clone, Copy)]
enum FibStage {
    Left,    // compute n-1
    Right,   // compute n-2
    Combine, // add
}

struct FibFrame {
    n: u64,
    stage: FibStage,
    a: u64, // saved locals: the result of fib(n-1)...
    b: u64, // ...and of fib(n-2)
}

impl FibFrame {
    fn new(n: u64) -> Self {
        FibFrame {
            n,
            stage: FibStage::Left,
            a: 0,
            b: 0,
        }
    }
}

fn fib_stack(n: u64) -> u64 {
    let mut result = 0;
    let mut stack = vec![FibFrame::new(n)];

    // `f` is a &mut into the top of the stack. Go does the same with
    // `f := &stack[len(stack)-1]` and appends in the same branch. Writing
    // `stack.push(FibFrame::new(f.n - 1))` here is error[E0499] (checked): the
    // push may reallocate the Vec, which would leave `f` dangling, and `f.n` is
    // still being read as the push begins. So every branch copies what it needs
    // out of `f` BEFORE it pushes, and never touches `f` afterwards.
    while let Some(f) = stack.last_mut() {
        if f.n < 2 {
            result = f.n;
            stack.pop();
            continue;
        }

        match f.stage {
            FibStage::Left => {
                f.stage = FibStage::Right;
                let n = f.n;
                stack.push(FibFrame::new(n - 1));
            }
            FibStage::Right => {
                f.a = result;
                f.stage = FibStage::Combine;
                let n = f.n;
                stack.push(FibFrame::new(n - 2));
            }
            FibStage::Combine => {
                f.b = result;
                result = f.a + f.b;
                stack.pop();
            }
        }
    }
    result
}

fn fib_rec(n: u64) -> u64 {
    if n < 2 {
        return n;
    }
    fib_rec(n - 1) + fib_rec(n - 2)
}

fn build() -> Tree {
    //        1
    //      /   \
    //     2     3
    //    / \
    //   4   5
    let leaf = |val| {
        Some(Box::new(Tree {
            val,
            left: None,
            right: None,
        }))
    };
    Tree {
        val: 1,
        left: Some(Box::new(Tree {
            val: 2,
            left: leaf(4),
            right: leaf(5),
        })),
        right: leaf(3),
    }
}

/// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
/// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    let t = build();

    println!("attempt 1 -- a plain stack of nodes, for a commutative fold:");
    println!("  sum_rec   = {}", sum_rec(Some(&t)));
    println!("  sum_stack = {}", sum_stack(Some(&t)));
    println!();
    println!("  this works because addition does not care about order. Nothing");
    println!("  happens 'after' the recursive calls that a plain stack cannot do.");

    println!();
    println!("attempt 2 -- post-order, where the work happens AFTER both calls:");
    let mut rec = Vec::new();
    postorder_rec(Some(&t), &mut rec);
    println!("  recursive {}", go_slice(&rec));
    println!("  explicit  {}", go_slice(&postorder_stack(Some(&t))));
    println!();
    println!("  a popped node is only HALF done: its children have not run yet.");
    println!("  So it is pushed back with stage=exit, underneath its children.");
    println!("  When it surfaces again, the children really are finished.");
    println!();
    println!("  that `stage` field is the RETURN ADDRESS. The CPU saves one on");
    println!("  every call so it knows where to resume; converting a recursion by");
    println!("  hand means writing it down yourself.");

    println!();
    println!("the same on a non-tree recursion -- fib has TWO resume points:");
    println!();
    println!("  {:>5} {:>12} {:>12}", "n", "recursive", "explicit");
    for n in [0, 1, 5, 10, 20] {
        println!("  {n:>5} {:>12} {:>12}", fib_rec(n), fib_stack(n));
    }
    println!();
    println!("  three stages: compute fib(n-1), compute fib(n-2), then add. Each");
    println!("  one is a place the recursion would have resumed at, so each one");
    println!("  needs its own case -- and `a` and `b` are the saved locals.");

    println!();
    println!("what this tells you about when to bother:");
    println!();
    println!("  the conversion is MECHANICAL but not free -- you are hand-writing");
    println!("  what the compiler generates. Do it when the depth can grow with");
    println!("  the input (lesson 05, example 15), and leave the recursion alone");
    println!("  when it cannot.");
    println!();
    println!("  the tell for how hard the conversion will be: count the RESUME");
    println!("  POINTS. One recursive call at the very end (a tail call) needs no");
    println!("  stack at all -- it is a loop. Work after one call needs a stage");
    println!("  field. Work between two calls needs saved locals as well.");
}
```

**Rust output:**

```
attempt 1 -- a plain stack of nodes, for a commutative fold:
  sum_rec   = 15
  sum_stack = 15

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

> **Go vs Rust:** Output is identical to Go apart from the labels `sum_rec`/`sum_stack`. The stack holds `&Tree` borrows rather than pointers, `Stage` is an enum with no invalid third value, and the `fib` conversion exposes a borrow-checker fact Go hides: pushing while holding `f = stack.last_mut()` is error[E0499] (checked), so each branch copies what it needs out of the top frame before it pushes.

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

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::LinkedList;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Lesson 07 measured a linked list losing almost every comparison to a slice.
// A stack is the cleanest possible test of that, because the two structures do
// EXACTLY the same operations here -- push and pop at one end -- with no
// indexing, no middle insertion, and no traversal to muddy the comparison.
//
// If a list is ever going to be competitive, it is here.
//
// Go writes one type per implementation, each with Push/Pop. Rust writes the
// two operations once, as a trait, so a single generic workload runs on every
// implementation -- and `Vec<i64>` and std's `LinkedList<i64>` can be plugged
// in directly, with no wrapper struct.
trait Stack {
    fn push(&mut self, v: i64);
    fn pop(&mut self) -> Option<i64>;
}

impl Stack for Vec<i64> {
    fn push(&mut self, v: i64) {
        Vec::push(self, v);
    }
    fn pop(&mut self) -> Option<i64> {
        Vec::pop(self)
    }
}

// std's doubly linked list: Go's container/list. The stack end is the front.
// Unlike container/list (which stores `any` and boxes every int), the
// element type is a generic parameter here, so the i64 sits inside the node.
impl Stack for LinkedList<i64> {
    fn push(&mut self, v: i64) {
        self.push_front(v);
    }
    fn pop(&mut self) -> Option<i64> {
        self.pop_front()
    }
}

struct Node {
    val: i64,
    next: Option<Box<Node>>,
}

/// Free a chain one node at a time. The Drop the compiler would generate for
/// `Option<Box<Node>>` recurses once per node, so a long chain can overflow the
/// stack when it is dropped (example 10's point, in the destructor). Go's GC
/// has no such problem. Not run here: the failure is a crash.
fn drop_chain(mut link: Option<Box<Node>>) {
    while let Some(mut node) = link {
        link = node.next.take();
    }
}

// ListStack pushes at the HEAD, which is a linked list's Theta(1) end.
#[derive(Default)]
struct ListStack {
    head: Option<Box<Node>>,
}

impl Stack for ListStack {
    fn push(&mut self, v: i64) {
        let next = self.head.take();
        self.head = Some(Box::new(Node { val: v, next }));
    }
    fn pop(&mut self) -> Option<i64> {
        let node = self.head.take()?;
        self.head = node.next; // the Box is freed here, as the node goes out of scope
        Some(node.val)
    }
}

impl Drop for ListStack {
    fn drop(&mut self) {
        drop_chain(self.head.take());
    }
}

// PooledListStack reuses removed nodes, removing the per-push allocation --
// the fairest possible version of a list-based stack.
#[derive(Default)]
struct PooledListStack {
    head: Option<Box<Node>>,
    free: Option<Box<Node>>,
}

impl Stack for PooledListStack {
    fn push(&mut self, v: i64) {
        let mut node = match self.free.take() {
            Some(mut n) => {
                self.free = n.next.take();
                n.val = v;
                n
            }
            None => Box::new(Node { val: v, next: None }),
        };
        node.next = self.head.take();
        self.head = Some(node);
    }
    fn pop(&mut self) -> Option<i64> {
        let mut node = self.head.take()?;
        self.head = node.next.take();
        node.next = self.free.take();
        let v = node.val;
        self.free = Some(node);
        Some(v)
    }
}

impl Drop for PooledListStack {
    fn drop(&mut self) {
        drop_chain(self.head.take());
        drop_chain(self.free.take());
    }
}

/// The workload: push 0..n, then pop until empty.
fn push_then_pop_all(s: &mut impl Stack, n: i64) {
    for i in 0..n {
        s.push(i);
    }
    while let Some(v) = s.pop() {
        black_box(v);
    }
}

// ---- the kit's allocation counter: stand-in for runtime.MemStats ----
// Go counts live heap objects before and after, with runtime.GC() around both
// readings, and needs a SIGNED delta because the GC can free more than the
// build allocated (the Go source says it once printed 18446744073709551613).
// Here there is no GC, and this counts allocation EVENTS: a counter that only
// goes up, so the difference of two readings cannot go negative.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);

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

fn allocs_of(f: impl FnOnce()) -> usize {
    let before = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - before
}

// ---- the kit's ns/op harness: stand-in for testing.Benchmark / b.Loop ----
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return el.as_nanos() as f64 / n as f64;
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

fn main() {
    const N: i64 = 10_000;

    println!("push {N} then pop {N}, six implementations:\n");

    let t_vec = ns_per_op(|| push_then_pop_all(&mut Vec::new(), black_box(N)));
    let t_vec_res = ns_per_op(|| {
        push_then_pop_all(&mut Vec::with_capacity(N as usize), black_box(N));
    });
    let t_list = ns_per_op(|| push_then_pop_all(&mut ListStack::default(), black_box(N)));
    // A fresh pool every run, as Go's benchmark does: all N pushes happen
    // before any pop, so the free list is empty for every push and every node
    // is allocated anyway. The pool only pays off on a LATER run.
    let t_pooled_cold =
        ns_per_op(|| push_then_pop_all(&mut PooledListStack::default(), black_box(N)));
    // The same stack kept across runs: after the first run the pool holds N
    // nodes, so no push allocates. This is the pool actually being used.
    let mut warm = PooledListStack::default();
    let t_pooled_warm = ns_per_op(|| push_then_pop_all(&mut warm, black_box(N)));
    let t_std_list = ns_per_op(|| push_then_pop_all(&mut LinkedList::new(), black_box(N)));

    println!("  {:<34} {:>14} {:>12}", "", "ns/op", "vs Vec");
    let rows = [
        ("Vec<i64>, grown by push", t_vec),
        ("Vec<i64>, preallocated", t_vec_res),
        ("linked list (Box per node)", t_list),
        ("linked list + free pool, cold", t_pooled_cold),
        ("linked list + free pool, warm", t_pooled_warm),
        ("std LinkedList<i64>", t_std_list),
    ];
    for (name, t) in rows {
        println!("  {name:<34} {t:>14.0} {:>11.1}x", t / t_vec_res);
    }

    // Allocation events for ONE push-N-pop-N run of each (the warm pool has
    // already been filled by the timing loop above).
    let a_vec = allocs_of(|| push_then_pop_all(&mut Vec::new(), N));
    let a_vec_res = allocs_of(|| push_then_pop_all(&mut Vec::with_capacity(N as usize), N));
    let a_list = allocs_of(|| push_then_pop_all(&mut ListStack::default(), N));
    let a_cold = allocs_of(|| push_then_pop_all(&mut PooledListStack::default(), N));
    let a_warm = allocs_of(|| push_then_pop_all(&mut warm, N));
    let a_std = allocs_of(|| push_then_pop_all(&mut LinkedList::new(), N));

    println!();
    println!("  allocation events for {N} elements (one push-then-pop run):");
    println!("    Vec grown {a_vec}, Vec preallocated {a_vec_res}, list {a_list},");
    println!("    pool cold {a_cold}, pool warm {a_warm}, std LinkedList {a_std}");

    println!(
        "    one element is {} bytes in the Vec, {} in a list node (payload + next pointer)",
        size_of::<i64>(),
        size_of::<Node>()
    );

    println!();
    println!("this is the comparison most favourable to a linked list in the whole");
    println!("plan -- both structures do only Theta(1) operations at their best end,");
    println!("with no indexing and no middle insertion -- and it still loses, by far:");
    println!("tens of times behind the preallocated Vec, in every run.");
    println!();
    println!("  the plain list pays one allocation per push ({a_list}) and one free per pop;");
    println!("    the Vec pays {a_vec_res} in total ({a_vec} when it grows by push)");
    println!("  the POOLED list only removes that if the pool outlives the run: cold, a");
    println!("    fresh pool every run, it never finds a node to reuse (same {a_cold}");
    println!("    allocations as the plain list, and just as far behind); warm, it makes {a_warm}");
    println!("    allocations and falls to roughly twice the Vec. That remainder is what");
    println!("    is left once malloc/free are gone: a dependent pointer load per");
    println!("    element and twice the bytes (a plausible reading, not measured)");
    println!("  std's LinkedList<i64> does not box the integer -- the element type is a");
    println!("    generic parameter, not `any` -- so it makes the same {a_std}");
    println!("    allocations as the plain list and lands in the same range of cost");

    println!();
    println!("so the recommendation for a stack in Rust is as unambiguous as in Go,");
    println!("which is rare enough to be worth stating plainly:");
    println!();
    println!("      use a Vec. Preallocate it if you know the depth.");
    println!();
    println!("there is no workload in this lesson where a linked stack wins.");
    println!("The one thing a list is good at -- Theta(1) splice at a held handle");
    println!("(lesson 07) -- is an operation a stack does not have.");
}
```

**Rust sample output:**

```
push 10000 then pop 10000, six implementations:

                                              ns/op       vs Vec
  Vec<i64>, grown by push                     13499         1.5x
  Vec<i64>, preallocated                       8807         1.0x
  linked list (Box per node)                 317530        36.1x
  linked list + free pool, cold              387448        44.0x
  linked list + free pool, warm               21487         2.4x
  std LinkedList<i64>                        566696        64.3x

  allocation events for 10000 elements (one push-then-pop run):
    Vec grown 13, Vec preallocated 1, list 10000,
    pool cold 10000, pool warm 0, std LinkedList 10000
    one element is 8 bytes in the Vec, 16 in a list node (payload + next pointer)

this is the comparison most favourable to a linked list in the whole
plan -- both structures do only Theta(1) operations at their best end,
with no indexing and no middle insertion -- and it still loses, by far:
tens of times behind the preallocated Vec, in every run.

  the plain list pays one allocation per push (10000) and one free per pop;
    the Vec pays 1 in total (13 when it grows by push)
  the POOLED list only removes that if the pool outlives the run: cold, a
    fresh pool every run, it never finds a node to reuse (same 10000
    allocations as the plain list, and just as far behind); warm, it makes 0
    allocations and falls to roughly twice the Vec. That remainder is what
    is left once malloc/free are gone: a dependent pointer load per
    element and twice the bytes (a plausible reading, not measured)
  std's LinkedList<i64> does not box the integer -- the element type is a
    generic parameter, not `any` -- so it makes the same 10000
    allocations as the plain list and lands in the same range of cost

so the recommendation for a stack in Rust is as unambiguous as in Go,
which is rare enough to be worth stating plainly:

      use a Vec. Preallocate it if you know the depth.

there is no workload in this lesson where a linked stack wins.
The one thing a list is good at -- Theta(1) splice at a held handle
(lesson 07) -- is an operation a stack does not have.
```

> **Go vs Rust:** The gap is far wider in Rust: the plain linked stack is about 40-75x the preallocated `Vec` (260-440 us vs 5-7 us for 10,000 pushes and pops, about fifteen runs on a shared machine) where Go measured 2.1-3.3x (82-127 us vs 38-49 us), so the Vec got faster than Go's slice and the list slower than Go's (mechanism not measured). Go's "pooled" row never reuses a node, since its fresh stack pushes all 10,000 before popping any (checked with `testing.AllocsPerRun`: 10,000 allocations, same as the plain list); the cold row here reproduces that, and the extra warm row shows the pool working (0 allocations, 1.8-2.3x the Vec). `std::collections::LinkedList<i64>` does not box the integer the way `container/list` does, and was usually within 7% of the hand-written list (single noisy runs: 28% faster to 41% slower).

**Complexity:** all Θ(1) per operation · the constants decide, and they are not close

---

> Next tier: [🔴 hard](3-hard.md).

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*