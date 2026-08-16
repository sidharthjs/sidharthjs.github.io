---
title: Fibonacci in Every Wrong Way Possible
date: 2026-08-13 00:00:00 +0530
categories: [Algorithms, Dynamic Programming]
tags: [fibonacci, dynamic-programming, go, recursion, iteration]
description: "A deliberately overengineered tour of Fibonacci that separates top-down and bottom-up thinking from recursion and iteration."
---


> **Top-Down Iteration and Bottom-Up Recursion: Crimes Against Dynamic
> Programming**

I picked Fibonacci as a deliberately simple playground for exploring a
larger idea.

The recurrence is familiar enough to stay out of the way, but flexible
enough to demonstrate recursion, iteration, dynamic programming,
explicit stacks, matrix algebra, floating-point arithmetic, goroutines,
channels, and the questionable things a programmer can do when nobody
takes the keyboard away.

This is not a guide to the *best* way to calculate Fibonacci.

That would be boring.

Instead, we're going to calculate the same number in increasingly
questionable ways and use the journey to uncover one useful idea:

> **Top-down vs. bottom-up and recursion vs. iteration are orthogonal
> concepts.**

We usually pair top-down with recursion and bottom-up with iteration
because those combinations are natural.

They are not inseparable.

Let's commit some crimes.

------------------------------------------------------------------------

## 1. Naive Recursion: The Innocent Beginning

We begin with the most obvious definition of Fibonacci:

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    return fib(n-1) + fib(n-2)
}
```

Beautiful.

Simple.

Mathematically faithful.

And horribly inefficient.

The recurrence is:

``` text
F(n) = F(n-1) + F(n-2)
```

The problem is that the same values are calculated again and again.

For example, calculating `fib(5)` causes `fib(3)` to be calculated more
than once, and the duplication gets dramatically worse as `n` grows.

The time complexity is exponential, approximately:

``` text
O(φ^n)
```

where `φ` is the golden ratio.

So naturally, I submitted it to LeetCode expecting a Time Limit
Exceeded.

It got accepted.

Why?

Because LeetCode's Fibonacci problem has a tiny constraint: `n <= 30`.

Apparently, even an exponentially terrible algorithm can survive if you
give it a small enough input.

And that leads us to the obvious question:

**Can we stop recalculating the same things?**

------------------------------------------------------------------------

## 2. Top-Down DP: Memoization

The recursive solution already tells us something important:

> `fib(n)` depends on `fib(n-1)` and `fib(n-2)`.

So instead of throwing away every result after calculating it, let's
remember it.

``` go
func fib(n int) int {
    memo := make(map[int]int)
    return recurse(n, memo)
}

func recurse(n int, memo map[int]int) int {
    if n <= 1 {
        return n
    }

    if v, ok := memo[n]; ok {
        return v
    }

    memo[n] = recurse(n-1, memo) + recurse(n-2, memo)
    return memo[n]
}
```

This is **top-down dynamic programming**, also called **memoization**.

We start with the state we actually want:

``` text
fib(n)
```

Then recursively descend into its dependencies:

``` text
fib(n)
  ↓
fib(n-1), fib(n-2)
  ↓
smaller states
  ↓
base cases
```

Once a state has been calculated, we store it.

Now every state from `0` through `n` is calculated at most once.

### Complexity

``` text
Time:  O(n)
Space: O(n)
```

The `O(n)` space comes from both the memoization table and the recursion
stack.

This is the sensible solution.

Surely we're done.

Right?

------------------------------------------------------------------------

## 3. Bottom-Up DP: The Textbook Approach

Instead of starting at `fib(n)` and recursively going down, why not
start at the smallest states and build our way up?

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    dp := make([]int, n+1)

    dp[0] = 0
    dp[1] = 1

    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }

    return dp[n]
}
```

This is **bottom-up DP**.

We solve:

``` text
fib(0)
fib(1)
fib(2)
fib(3)
...
fib(n)
```

Each state is ready before the state that depends on it.

### Complexity

``` text
Time:  O(n)
Space: O(n)
```

This is probably the version you'd expect to see in a DP tutorial.

But then I noticed something.

When calculating:

``` go
dp[i] = dp[i-1] + dp[i-2]
```

do I actually need the entire array?

No.

I only need the previous two values.

------------------------------------------------------------------------

## 4. Top-Down DP With an Explicit Stack: "I Don't Trust the Runtime"

Here's where things start getting weird.

We've been treating **top-down** and **recursion** as if they were the
same thing.

They're not.

Top-down simply means:

> Start from the target and explore its dependencies.

Recursion is just one way to implement that traversal.

Normally, recursion gives us an implicit call stack.

So what happens if we remove recursion and build the stack ourselves?

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    memo := map[int]int{
        0: 0,
        1: 1,
    }

    stack := []int{n}

    for len(stack) > 0 {
        x := stack[len(stack)-1]

        if _, ok := memo[x]; ok {
            stack = stack[:len(stack)-1]
            continue
        }

        _, leftReady := memo[x-1]
        _, rightReady := memo[x-2]

        if leftReady && rightReady {
            memo[x] = memo[x-1] + memo[x-2]
            stack = stack[:len(stack)-1]
            continue
        }

        if !rightReady {
            stack = append(stack, x-2)
        }

        if !leftReady {
            stack = append(stack, x-1)
        }
    }

    return memo[n]
}
```

This is still top-down.

We start with:

``` text
fib(n)
```

and descend into unresolved dependencies.

The difference is that the runtime is no longer maintaining the call
stack.

**I am now maintaining the call stack.**

I didn't change the algorithm's direction.

I just fired the runtime.

### The realization

Recursion is not magic.

A recursive function is, among other things, a convenient way to manage
a stack of unfinished work.

Once you see that, a lot of "recursive vs. iterative" distinctions start
looking less absolute.

------------------------------------------------------------------------

## 5. Bottom-Up DP With Recursion: "Loops Are for Mortals"

Now let's commit the opposite crime.

Can bottom-up DP be recursive?

Yes.

The important thing is that **bottom-up describes the order in which
states are solved**, not whether a `for` loop exists.

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    dp := make([]int, n+1)
    dp[0] = 0
    dp[1] = 1

    fill(2, n, dp)

    return dp[n]
}

func fill(i, n int, dp []int) {
    if i > n {
        return
    }

    dp[i] = dp[i-1] + dp[i-2]
    fill(i+1, n, dp)
}
```

Look carefully.

We're still computing:

``` text
dp[0]
dp[1]
dp[2]
dp[3]
...
dp[n]
```

in increasing order.

That's bottom-up.

The fact that a recursive function happens to drive that progression
doesn't magically turn it into top-down.

We're just using recursion as a very expensive replacement for a loop.

### Complexity

``` text
Time:  O(n)
Space: O(n)
```

The DP array is `O(n)` and the recursive call stack is also `O(n)`.

At this point, we're beginning to lose the plot.

Good.

------------------------------------------------------------------------

## 6. Tail-Recursive Bottom-Up With O(1) DP State: "Go Won't Optimize This, But I Will"

We already discovered that the entire DP array isn't necessary.

Only the previous two values matter.

So let's combine:

-   bottom-up evaluation
-   recursion
-   two-value state

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    return solve(2, n, 0, 1)
}

func solve(i, n, prev2, prev1 int) int {
    if i > n {
        return prev1
    }

    return solve(i+1, n, prev1, prev1+prev2)
}
```

The DP state is now constant-sized.

``` text
prev2 = F(i-2)
prev1 = F(i-1)
```

Then we recursively advance to the next state.

From an algorithmic-state perspective:

``` text
Space: O(1)
```

But there is an important Go-specific catch.

Go does **not** perform general tail-call optimization.

So although the algorithm only carries two Fibonacci values, every
recursive call still adds a stack frame.

Therefore:

``` text
DP state:       O(1)
Actual stack:   O(n)
```

We optimized the state.

Then voluntarily put the growth back into the call stack.

Outstanding work.

------------------------------------------------------------------------

## 7. Matrix Exponentiation: "Linear Time Is Too Mainstream"

At this point, `O(n)` has started to feel embarrassing.

Fibonacci has another mathematical representation:

``` text
[ F(n+1) ]   [ 1  1 ]^n [ 1 ]
[ F(n)   ] = [ 1  0 ]   [ 0 ]
```

So instead of calculating Fibonacci numbers one by one, we can raise a
matrix to the `n`th power.

The important trick is **exponentiation by squaring**.

Instead of multiplying the matrix `n` times:

``` text
A × A × A × A × ... × A
```

we repeatedly square:

``` text
A
A²
A⁴
A⁸
A¹⁶
...
```

and combine the powers we need.

That brings the complexity down to:

``` text
Time:  O(log n)
Space: O(1)
```

We have officially left the comfortable neighborhood of basic DP and
entered:

> "What if Fibonacci were linear algebra?"

------------------------------------------------------------------------

## 8. Fast Doubling: "O(log n), Because Why Not?"

Matrix exponentiation isn't the only way to get logarithmic time.

Fibonacci has identities that let us calculate:

``` text
F(2k)
F(2k+1)
```

directly from:

``` text
F(k)
F(k+1)
```

The key formulas are:

``` text
F(2k)   = F(k) × [2F(k+1) − F(k)]

F(2k+1) = F(k+1)² + F(k)²
```

This lets us recursively halve the problem size.

So instead of:

``` text
n → n-1 → n-2 → ...
```

we get something more like:

``` text
n
↓
n/2
↓
n/4
↓
n/8
↓
...
```

giving:

``` text
Time:  O(log n)
```

This is **fast doubling**.

At this point, Fibonacci is no longer a DP exercise.

It's become a mathematical arms race.

------------------------------------------------------------------------

## 9. Binet's Formula: "Floating-Point Errors Are Just a Social Construct"

Or...

What if we don't calculate the sequence at all?

There is a closed-form expression for Fibonacci:

``` text
F(n) = (φⁿ − ψⁿ) / √5
```

where:

``` text
φ = (1 + √5) / 2
ψ = (1 − √5) / 2
```

In Go, a naive implementation could look like:

``` go
func fib(n int) int {
    phi := (1 + math.Sqrt(5)) / 2
    psi := (1 - math.Sqrt(5)) / 2

    return int(math.Round((math.Pow(phi, float64(n)) -
        math.Pow(psi, float64(n))) / math.Sqrt(5)))
}
```

On paper, this looks ridiculous in the best possible way.

No recursion.

No DP table.

No loop.

Just math.

But there is a catch.

Computers don't have infinite-precision floating-point numbers.

For sufficiently large `n`, rounding errors become a problem.

So this is a beautiful mathematical formula, but not a robust
general-purpose integer algorithm for arbitrarily large Fibonacci
numbers.

Still:

**We have earned the right to say we solved Fibonacci without actually
traversing the Fibonacci sequence.**

------------------------------------------------------------------------

## 10. Compile-Time Fibonacci: "Why Compute at Runtime?"

Now let's ask an even more questionable question:

> Why calculate Fibonacci when the program runs?

The specific input is not known at compile time, but the entire input
range is. LeetCode constrains `n` to `0 <= n <= 30`, which gives us only
31 possible inputs. We can generate the answer for every valid case
ahead of time and cover the whole problem with a lookup.

So yes, this is effectively a hardcoded approach. The only distinction
is that a generator can produce the constants and switch cases for us
instead of making us type them by hand.

Go does not have C++-style general `constexpr` function evaluation, so
there isn't a magic recursive Fibonacci function that the Go compiler
evaluates for us.

We can generate Fibonacci constants as source code:

``` go
const (
    Fib0 = 0
    Fib1 = 1
    Fib2 = 1
    Fib3 = 2
    Fib4 = 3
    Fib5 = 5
    Fib6 = 8
    // ...
)
```

Then:

``` go
func fib(n int) int {
    switch n {
    case 0:
        return Fib0
    case 1:
        return Fib1
    case 2:
        return Fib2
    case 3:
        return Fib3
    case 4:
        return Fib4
    case 5:
        return Fib5
    case 6:
        return Fib6
    // ...
    default:
        panic("unsupported n")
    }
}
```

For the LeetCode constraints, the omitted cases continue through
`Fib30`, so the `default` branch is only for inputs outside the allowed
range.

Technically, we've moved the computation outside runtime execution.

The runtime complexity of the lookup is effectively constant.

But let's be honest:

**We didn't make Fibonacci faster.**

We just moved the problem somewhere else.

This is the programming equivalent of hiding the body under the carpet.

------------------------------------------------------------------------

## 11. Concurrent Fibonacci With Goroutines: "Let's Make It Slower in Parallel"

Naive Fibonacci already does a huge amount of redundant work.

So naturally, the next optimization is to perform the redundant work
**concurrently**.

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    leftCh := make(chan int)
    rightCh := make(chan int)

    go func() {
        leftCh <- fib(n - 1)
    }()

    go func() {
        rightCh <- fib(n - 2)
    }()

    return <-leftCh + <-rightCh
}
```

There is something deeply satisfying about this.

The original algorithm says:

> "I have too much work."

The concurrent version says:

> "What if I had thousands of goroutines doing the same unnecessary
> work?"

This is almost certainly worse for practical performance.

For large `n`, the number of goroutines and synchronization operations
can explode.

Concurrency doesn't magically fix a bad algorithm.

Sometimes it just lets you make the bad algorithm more complicated.

------------------------------------------------------------------------

## 12. Infinite Fibonacci Channel: "A Lazy Stream With a Goroutine"

What if we don't even ask for a particular Fibonacci number?

What if we create an infinite stream?

``` go
func fibonacci() <-chan int {
    ch := make(chan int)

    go func() {
        a, b := 0, 1

        for {
            ch <- a
            a, b = b, a+b
        }
    }()

    return ch
}
```

Now we can consume Fibonacci numbers forever:

``` go
ch := fibonacci()

for i := 0; i < 10; i++ {
    fmt.Println(<-ch)
}
```

Output:

``` text
0
1
1
2
3
5
8
13
21
34
```

This isn't really an efficient way to calculate a particular Fibonacci
number.

We don't need the first `n` numbers if all we want is `F(n)`.

But it introduces a different programming idea:

> Instead of returning a collection, produce values lazily as they're
> requested.

And because we're doing it with a Go channel and a goroutine, we've
successfully turned Fibonacci into a tiny streaming system.

For a sequence containing two integers.

------------------------------------------------------------------------

## 13. Mutual Recursion: "One Function Wasn't Enough"

One recursive function was apparently too conventional.

Let's split Fibonacci between two functions.

``` go
func fibA(n int) int {
    if n <= 1 {
        return n
    }

    return fibB(n-1) + fibA(n-2)
}

func fibB(n int) int {
    if n <= 1 {
        return n
    }

    return fibA(n-1) + fibB(n-2)
}
```

The functions call each other recursively.

This is called **mutual recursion**.

Does it help?

No.

Does it improve performance?

Absolutely not.

Does it make the control flow harder to understand?

Yes.

Was that the point?

Exactly.

The Fibonacci recurrence didn't change.

We just added organizational overhead for the sake of the blog.

------------------------------------------------------------------------

## 14. Top-Down Iteration + Bottom-Up Recursion: "I Have Become Stack, Destroyer of Loops"

And now we reach the final boss.

We've spent this entire article casually pairing:

``` text
Top-down    → recursion
Bottom-up   → iteration
```

But now we've seen that those are independent dimensions.

So let's deliberately cross the streams.

### Top-down + iteration

Use an explicit stack.

The stack simulates the recursive call stack while still starting from
the target and descending into dependencies.

### Bottom-up + recursion

Use recursion to advance through the states in increasing order.

The recursion is merely acting as a loop.

Put those ideas together and the final lesson becomes obvious:

``` text
                    Recursion       Iteration

Top-down                ✓               ✓

Bottom-up               ✓               ✓
```

All four combinations are possible.

The difference is how natural they are.

The conventional combinations:

``` text
Top-down + recursion
Bottom-up + iteration
```

are convenient because they naturally match the way the algorithms are
expressed.

The other combinations require more machinery:

``` text
Top-down + iteration
→ explicit stack

Bottom-up + recursion
→ recursive loop simulation
```

But there is nothing fundamentally preventing them.

And that is the whole point of this ridiculous exercise.

------------------------------------------------------------------------

## What Did We Actually Learn?

After all of this nonsense, Fibonacci itself isn't the interesting part
anymore.

The interesting part is separating concepts that we normally learn
together.

### Top-down vs. bottom-up

This describes **the direction in which we traverse the dependency
graph**.

``` text
Top-down:

        F(n)
       /    \
   F(n-1)  F(n-2)
      ↓       ↓
   smaller  smaller
    states   states
```

Bottom-up reverses the direction:

``` text
F(0), F(1)
     ↓
   F(2)
     ↓
   F(3)
     ↓
   ...
     ↓
   F(n)
```

### Recursion vs. iteration

This describes **how we implement the traversal**.

Recursion uses the language/runtime's call stack.

Iteration normally uses loops and explicit state.

But you can replace the implicit stack with an explicit stack.

And you can replace a loop with recursive calls.

Therefore:

> **Top-down is not synonymous with recursion.**
>
> **Bottom-up is not synonymous with iteration.**

They're just natural pairings.

------------------------------------------------------------------------

## The Fibonacci Descent Into Madness

| # | Approach | Time | Space | Sanity |
|---:|---|---:|---:|:---:|
| 1 | Naive recursion | O(φⁿ) | O(n) | 😀 |
| 2 | Top-down DP | O(n) | O(n) | 🙂 |
| 3 | Bottom-up DP | O(n) | O(n) | 🙂 |
| 4 | Top-down + explicit stack | O(n) | O(n) | 🤨 |
| 5 | Bottom-up + recursion | O(n) | O(n) | 🤔 |
| 6 | Tail-recursive bottom-up | O(n) | O(n) stack in Go | 😬 |
| 7 | Matrix exponentiation | O(log n) | O(1) | 🧠 |
| 8 | Fast doubling | O(log n) | O(log n) recursive | 🚀 |
| 9 | Binet's formula | ~O(1)* | O(1) | ⚠️ |
| 10 | Compile-time/generated constants | O(1) lookup | O(1) runtime | 🤡 |
| 11 | Goroutines | Exponential work | Exponential-ish overhead | 💀 |
| 12 | Infinite channel | O(n) to consume n values | O(1) state | 🌀 |
| 13 | Mutual recursion | O(φⁿ) | O(n) | 🫠 |
| 14 | Top-down iteration + bottom-up recursion | Depends on implementation | Depends on implementation | ☠️ |

\* Binet's formula is not reliably exact for arbitrarily large integers
because of floating-point precision.

------------------------------------------------------------------------

## The Actual Takeaway

I chose Fibonacci because the problem is small and familiar enough to
let the implementation choices take center stage.

The point was never the sequence itself. It was to use one simple
recurrence to demonstrate something more interesting:

**An algorithmic concept and the mechanism used to implement it don't
have to be the same thing.**

Top-down and bottom-up describe **how we think about dependencies**.

Recursion and iteration describe **how we execute the computation**.

We usually put them together because it's convenient.

But once you separate those ideas, you can start asking much more
interesting questions:

> "What is recursion actually doing?"

> "Can I replace the call stack?"

> "Can a loop perform a top-down traversal?"

> "Can recursion perform a bottom-up traversal?"

> "How much of my DP state do I actually need?"

And eventually:

> "Can I make Fibonacci run at compile time?"

At that point, someone should probably take away my keyboard.

------------------------------------------------------------------------

## Epilogue

The funny thing is that none of these implementations were necessary.

For practical use, this is probably all you need:

``` go
func fib(n int) int {
    if n <= 1 {
        return n
    }

    a, b := 0, 1

    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }

    return b
}
```

But sometimes the fastest way to understand a concept isn't to stop at
the correct answer.

Sometimes you have to ask:

> **"Okay, but what happens if I do it the wrong way?"**

And then keep going.
