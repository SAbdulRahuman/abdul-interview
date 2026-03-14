# Tricky Output Questions

---

## Q1: Argument Evaluation Order

```c
int x = 5;
printf("%d %d %d\n", x, x++, ++x);
// Answer: UNDEFINED BEHAVIOR
// Order of argument evaluation is unspecified in C
```

## Q2: Pointer Increment with Dereference

```c
char *p = "Hello";
printf("%c\n", *p++);     // 'H' (dereference, then increment pointer)
printf("%c\n", *p);       // 'e'
```

## Q3: sizeof Array Parameter

```c
void foo(int arr[100]) {
    printf("%zu\n", sizeof(arr));  // 8 (pointer size, NOT 400!)
}
// Array parameter decays to pointer
```

## Q4: String Literal Comparison

```c
char *a = "hello";
char *b = "hello";
printf("%d\n", a == b);  // may print 1 (string pooling) — implementation-defined
```

## Q5: Comma Operator

```c
int x = (1, 2, 3);        // x = 3 (comma evaluates all, returns last)
```

## Q6: Maximal Munch

```c
int a = 1, b = 2;
int c = a+++b;              // parsed as: (a++) + b = 1 + 2 = 3; a becomes 2
```

## Q7: sizeof String

```c
printf("%zu\n", sizeof("Hello"));  // 6 (includes '\0')
printf("%zu\n", strlen("Hello"));  // 5 (excludes '\0')
```

## Q8: Negative Modulus

```c
printf("%d\n", -7 % 3);   // -1 (in C99+, result has sign of dividend)
printf("%d\n", 7 % -3);   //  1
```

## Q9: Struct Size

```c
struct S { char c; int i; char d; };
printf("%zu\n", sizeof(struct S));  // 12 (not 6!) due to padding
```

## Q10: Pointer to Array vs Array of Pointers

```c
int (*p)[5];    // pointer to array of 5 ints
int *q[5];      // array of 5 pointers to int
printf("%zu %zu\n", sizeof(p), sizeof(q));  // 8, 40 (on 64-bit)
```

---
