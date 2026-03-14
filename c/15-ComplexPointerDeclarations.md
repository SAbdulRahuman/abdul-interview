# Complex Pointer Declarations

---

## Clockwise/Spiral Rule

Read declarations starting from the variable name, going clockwise/spiraling outward.

```c
int *p;                  // pointer to int
int **p;                 // pointer to pointer to int
int (*p)[10];            // pointer to array of 10 ints
int *p[10];              // array of 10 pointers to int
int (*p)(int);           // pointer to function taking int, returning int
int (*p[5])(int);        // array of 5 pointers to functions taking int, returning int
```

## Const with Pointers

```c
const int *p;            // pointer to const int (can't modify *p)
int const *p;            // same as above
int * const p;           // const pointer to int (can't modify p)
const int * const p;     // const pointer to const int
```

## Reading Complex Declarations

```c
// Example: void (*signal(int, void (*)(int)))(int);
// 
// signal is a function that takes:
//   - an int
//   - a pointer to a function taking int and returning void
// and returns:
//   - a pointer to a function taking int and returning void

// Simplified with typedef:
typedef void (*SigHandler)(int);
SigHandler signal(int sig, SigHandler handler);
```

---

## Interview Q&A

**Q: What is `cdecl` utility?**

```
A tool that translates between C declarations and English:
$ cdecl
cdecl> explain int (*(*fp)(int))[10]
→ fp is pointer to function (int) returning pointer to array 10 of int
```

**Q: How to read `const` placement?**

```
Read right-to-left:
  const int *p     → p is a pointer to an int that is const
  int * const p    → p is a const pointer to an int
  const int * const p → p is a const pointer to a const int
```

---
