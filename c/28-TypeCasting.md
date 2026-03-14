# Type Casting & Integer Promotions

---

## Implicit Conversions

```c
// Integer promotion: char, short → int in expressions
char c = 'A';
int i = c;        // char promoted to int

// Usual arithmetic conversions:
// If operands differ, the "smaller" type is converted to the "larger":
// int → unsigned int → long → unsigned long → long long → float → double → long double

int a = 5;
double d = a + 2.0;  // a is implicitly converted to double
```

## Explicit Casting

```c
double d = 3.14;
int i = (int)d;   // truncates to 3

// Pointer casting
void *vp = malloc(4);
int *ip = (int *)vp;

// Casting to avoid integer overflow
int a = 100000, b = 100000;
long long result = (long long)a * b;  // without cast: overflow!
```

## Common Pitfalls

```c
// Signed/unsigned comparison trap
int x = -1;
unsigned int y = 1;
if (x < y)  // FALSE! x is converted to unsigned → huge positive number
    printf("x < y\n");  // this does NOT print

// Truncation
int big = 300;
char c = big;  // c = 44 (300 % 256) – data lost

// Float to int truncation
int i = (int)3.9;  // i = 3, not 4
```

---

## Interview Q&A

**Q: What is integer promotion?**

```
When char or short are used in expressions, they are automatically promoted to int
(or unsigned int if int cannot represent all values).
This happens before any arithmetic operation.
```

**Q: What happens when you compare signed and unsigned integers?**

```
The signed value is implicitly converted to unsigned.
-1 becomes a very large positive number → counter-intuitive comparison results.
Always use same signedness for comparisons, or cast explicitly.
```

---
