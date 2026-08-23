---
title: "Why Top-Down DP Feels Easier to Derive"
date: 2026-08-22 00:00:00 +0530
categories: [Algorithms, Dynamic Programming]
tags: [dynamic-programming, top-down, bottom-up, memoization, tabulation, lcs]
description: "Why top-down DP solutions are often easier to derive and remember, and where bottom-up still wins through control and optimization."
image:
  path: /assets/img/posts/top-down-vs-bottom-up-dp-cover-v3.png
  alt: A programmer easily retracing a top-down solution while struggling to derive a bottom-up DP table
---

You solve a dynamic programming problem, understand the bottom-up table, and implement it successfully.

A few days later, you return to the same problem. You remember that there was a table. You vaguely remember its dimensions. But deriving it again from scratch feels surprisingly difficult.

Top-down can be easier to reconstruct later because the implementation often preserves the reasoning that led to the recurrence. This is not because recursion is inherently better.

A useful way to describe the intuition is:

> **Top-down asks the question. Bottom-up builds the answer.**

---

## Top-Down Stays Close to the Question

Consider **Longest Common Subsequence**.

Given two strings, we want the length of the longest sequence of characters that appears in both strings in the same relative order.

```text
text1 = "abcde"
text2 = "ace"

answer = 3
```

A top-down state can be written as:

```go
solve(i, j)
```

It asks:

> What is the LCS of the two strings starting at positions `i` and `j`?

If the current characters match, include that character and advance both positions:

```go
if text1[i] == text2[j] {
    return 1 + solve(i+1, j+1)
}
```

If they do not match, skip one character from either string and keep the better result:

```go
return max(
    solve(i+1, j),
    solve(i, j+1),
)
```

The state and recurrence follow directly from the question:

```text
Do the current characters match?

Yes → count the character and move forward in both strings
No  → do not count either character yet; move forward in one string at a time and keep the better answer
```

The recursive solution will calculate the same `(i, j)` states more than once, so the next step is to store their answers.

The memoization table also follows naturally from the state:

```text
memo[i][j] = answer to solve(i, j)
```

The state has two changing values: `i`, a position in `text1`, and `j`, a position in `text2`. So the memo table obviously needs two dimensions:

```go
memo := make([][]int, len(text1))
for i := range memo {
    memo[i] = make([]int, len(text2))
    for j := range memo[i] {
        memo[i][j] = -1
    }
}
```

This is a two-dimensional DP problem because every unique subproblem is identified by the pair `(i, j)`. The table dimensions were not invented separately; they came directly from the question represented by `solve(i, j)`.

After evaluating `solve(0, 0)` for `"abcde"` and `"ace"`, the memo table looks like this:

<img src="/assets/img/posts/lcs-top-down-memo-table.png" alt="Hand-drawn top-down LCS memoization table" width="500">

Each number is the answer for `solve(i, j)`. A `-` means that state was never requested, so it was never computed. The table is only storage for questions the recursion has already asked; it does not determine how the solution is derived.

### Complexity

Let `m` and `n` be the lengths of `text1` and `text2`.

```text
Time:  O(m × n)
Space: O(m × n) for memoization + O(m + n) recursion stack
```

Each `(i, j)` state is calculated only once. The maximum recursion depth is `m + n`.

The implementation is easier to reconstruct because it still looks like the reasoning that produced it.

---

## What If We Start With Bottom-Up?

We just went through the top-down solution. From `solve(i, j)`, we already know that each subproblem is identified by two positions, so a two-dimensional table now feels familiar.

Bottom-up can use that exact same state:

```text
dp[i][j] = LCS of the strings starting at positions i and j
```

For the same example, a completed bottom-up solution is often presented as this table:

<img src="/assets/img/posts/lcs-bottom-up-table.png" alt="Hand-drawn bottom-up LCS table for abcde and ace" width="500">

Because the top-down solution has already revealed the state and its dimensions, this table is easy to recognize. But if we had started by trying to solve the problem bottom-up, arriving at this table would require answering several questions first:

- why an extra row and column are needed
- what values represent the empty-string base cases
- which neighboring cells each state depends on
- which cells must be available before computing `dp[i][j]`
- which direction the table should be filled
- where the final answer is stored

These decisions are not hidden requirements of bottom-up. A careful derivation can define the state, inspect its dependencies, establish the base cases, and then choose a valid evaluation order. The problem is that many tutorial explanations begin only after these choices have already been made. Explaining that finished table is much easier than deriving it from the original question.

The top-down recurrence gives us the dependencies, but bottom-up still requires turning those dependencies into a global evaluation order where every required state is ready first:

```go
for i := m - 1; i >= 0; i-- {
    for j := n - 1; j >= 0; j-- {
        if text1[i] == text2[j] {
            dp[i][j] = 1 + dp[i+1][j+1]
        } else {
            dp[i][j] = max(dp[i+1][j], dp[i][j+1])
        }
    }
}
```

### Complexity

```text
Time:  O(m × n)
Space: O(m × n)
```

Every cell in the table is computed once, and the full table remains in memory.

Understanding this completed table is not the same as knowing how to derive it. That is why a bottom-up solution can make perfect sense during an explanation and still be difficult to reproduce later.

Top-down asks:

```text
What answer do I need from these positions?
```

Bottom-up takes the same question and asks:

```text
In what order must every answer be computed?
```

Both solve the same dependency structure. Top-down explores the states it needs on demand; bottom-up chooses an explicit order in which to evaluate the required states.

---

## Why Bottom-Up Is Still Worth Learning

Optimization is one important answer, but it is not the only one. Bottom-up makes the evaluation order explicit. It also avoids function-call overhead, provides predictable memory access, and often improves locality when most or all states must be computed.

Top-down uses the program's call stack, so sufficiently deep state transitions can hit recursion-depth or stack limits. Bottom-up avoids that class of problem by evaluating the states iteratively.

The explicit table also makes space optimization easier to see.

Look again at the LCS transition. To compute `dp[i][j]`, we need only three cells:

```text
dp[i+1][j+1]  next row
dp[i+1][j]    next row
dp[i][j+1]    current row
```

Nothing depends on rows below `i+1`. After completing a row, every older row can be discarded.

Let `m` and `n` be the lengths of `text1` and `text2`. Instead of storing the full table:

```text
O(m × n) space
```

we can keep only the next suffix row and the row currently being built:

### Space-Optimized Table

During the final iteration, when `i = 0`, the two rows in memory are:

<img src="/assets/img/posts/lcs-space-optimized-table.png" alt="Hand-drawn two-row space-optimized LCS table" width="560">

`curr[j]` uses values from `next` and the value immediately to its right in `curr`. Once `curr` is complete, it becomes the next row used by the following iteration. The old `next` row can be overwritten because no future state needs it.

```go
next := make([]int, n+1)
curr := make([]int, n+1)

for i := m - 1; i >= 0; i-- {
    for j := n - 1; j >= 0; j-- {
        if text1[i] == text2[j] {
            curr[j] = 1 + next[j+1]
        } else {
            curr[j] = max(next[j], curr[j+1])
        }
    }

    next, curr = curr, next
}

return next[0]
```

### Complexity

```text
Time:  O(m × n)
Space: O(n)
```

The same number of states is computed, but only two rows remain in memory. If the shorter string is used for the columns, the space can be reduced to `O(min(m, n))`.

This optimization follows naturally from the bottom-up table because the table shows when each state is used and when it is no longer needed. That same control also removes the recursion stack and gives predictable iteration and memory access.

The table may not be the most natural place to discover the solution, but it is often the better place to optimize it.

---

## The Takeaway

A top-down solution is often easier to arrive at because it begins with the question the problem is asking. When the state maps cleanly to the original question, turning that question into a recursive function often makes the recurrence emerge naturally. Because the code preserves that reasoning, the solution is also easier to reconstruct later.

A bottom-up table is different. Once the table and transition are explained, the solution may be easy to understand. Deriving that table in the first place is the difficult part. Without remembering a similar solution, it may not be obvious what the table should represent, how large it should be, how it should be initialized, or in which direction it should be filled.

Bottom-up becomes valuable when explicit evaluation order matters: avoiding recursion limits, reducing call overhead, improving memory locality, computing all states efficiently, or compressing the table.

Derive the solution top-down when the recursive state follows naturally from the question. Choose bottom-up when stack safety, explicit evaluation order, memory optimization, or constant-factor performance matters.
