# Recursion

---

## Basics

```c
// Factorial
int factorial(int n) {
    if (n <= 1) return 1;       // base case
    return n * factorial(n - 1); // recursive case
}

// Fibonacci
int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);  // O(2^n) – very slow!
}
```

## Tail Recursion

```c
// Tail recursion: recursive call is the LAST operation
int factorial_tail(int n, int acc) {
    if (n <= 1) return acc;
    return factorial_tail(n - 1, n * acc);  // compiler can optimize to loop
}
// Call: factorial_tail(5, 1)

// GCC can optimize tail calls with -O2
```

## Types of Recursion

```
Direct recursion:   function calls itself
Indirect recursion: function A calls B, B calls A
Linear recursion:   one recursive call per invocation
Tree recursion:     multiple recursive calls per invocation (e.g., Fibonacci)
Tail recursion:     recursive call is the last operation
```

---

## Interview Q&A

**Q: What is stack overflow in recursion?**

```
Each recursive call pushes a new stack frame. Without a proper base case or with
very deep recursion, the stack space is exhausted → segmentation fault / stack overflow.
Default stack size is typically 1-8 MB (OS-dependent).
```

**Q: How to convert recursion to iteration?**

```
1. Use an explicit stack (data structure) to simulate call stack
2. Transform tail recursion to a loop
3. Use memoization to avoid redundant calls (dynamic programming)
```

**Q: What is the time complexity of naive recursive Fibonacci?**

```
O(2^n) – exponential, because of redundant re-computation.
Can be reduced to O(n) with memoization or iterative approach.
```

---
