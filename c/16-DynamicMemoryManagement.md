# Dynamic Memory Management

---

## Memory Allocation Functions

```c
#include <stdlib.h>

// malloc: allocate uninitialized memory
int *p = (int *)malloc(10 * sizeof(int));  // cast optional in C, required in C++

// calloc: allocate zero-initialized memory
int *q = (int *)calloc(10, sizeof(int));

// realloc: resize previously allocated memory
p = (int *)realloc(p, 20 * sizeof(int));
// If realloc fails, returns NULL but original block is NOT freed → memory leak risk
// Solution:
int *tmp = realloc(p, new_size);
if (tmp == NULL) { /* handle error */ }
else { p = tmp; }

// free: deallocate memory
free(p);
p = NULL;  // prevent dangling pointer
```

## Common Memory Bugs

```
1. Memory leak:       malloc without corresponding free
2. Double free:       calling free() twice on same pointer → UB
3. Use after free:    accessing memory after free → UB
4. Buffer overflow:   writing beyond allocated bounds → UB
5. Uninitialized read: reading from malloc'd memory without writing
6. Dangling pointer:  using pointer after the pointed-to memory is freed
7. Null dereference:  dereferencing NULL pointer
```

## Dynamic 2D Array

```c
// Method 1: Array of pointers
int **matrix = malloc(rows * sizeof(int *));
for (int i = 0; i < rows; i++)
    matrix[i] = malloc(cols * sizeof(int));

// Free
for (int i = 0; i < rows; i++) free(matrix[i]);
free(matrix);

// Method 2: Single contiguous block (better cache performance)
int *matrix = malloc(rows * cols * sizeof(int));
// Access: matrix[i * cols + j]
free(matrix);

// Method 3: Pointer to VLA (C99)
int (*matrix)[cols] = malloc(rows * sizeof(*matrix));
matrix[i][j] = val;
free(matrix);
```

---

## Interview Q&A

**Q: Difference between `malloc` and `calloc`?**

```
malloc(size): allocates uninitialized memory. One argument (total bytes).
calloc(count, size): allocates zero-initialized memory. Two arguments.
calloc also checks for multiplication overflow internally.
```

**Q: How to detect memory leaks?**

```
Tools: Valgrind (memcheck), AddressSanitizer (ASan), LeakSanitizer
  gcc -fsanitize=address -g program.c -o program
  valgrind --leak-check=full ./program
```

**Q: Can you `free(NULL)`?**

```
Yes. free(NULL) is safe – it is a no-op (does nothing).
Defined by the C standard.
```

**Q: What happens if `realloc` fails?**

```
Returns NULL, but the original memory block is NOT freed.
You lose the original pointer if you assign directly:
  p = realloc(p, new_size);  // BUG: if NULL, original p is lost → memory leak
Always use a temporary pointer.
```

---
