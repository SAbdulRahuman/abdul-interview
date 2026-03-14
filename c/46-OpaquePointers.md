# Opaque Pointers (Information Hiding)

---

## Pattern

```c
// stack.h – public interface
typedef struct Stack Stack;          // forward declaration (opaque)
Stack *stack_create(int capacity);
void   stack_push(Stack *s, int val);
int    stack_pop(Stack *s);
void   stack_destroy(Stack *s);

// stack.c – private implementation
struct Stack {
    int *data;
    int top;
    int capacity;
};

Stack *stack_create(int capacity) {
    Stack *s = malloc(sizeof(Stack));
    s->data = malloc(capacity * sizeof(int));
    s->top = -1;
    s->capacity = capacity;
    return s;
}

void stack_destroy(Stack *s) {
    free(s->data);
    free(s);
}
// Users cannot access struct members directly → encapsulation in C
```

---

## Interview Q&A

**Q: What is an opaque pointer?**

```
A pointer to an incomplete type (forward-declared struct).
Users can pass it around but cannot access its members.
The struct definition is hidden in the .c file → information hiding.
Used extensively in C APIs: FILE*, pthread_t, etc.
```

**Q: What are the advantages?**

```
1. Encapsulation – internal details are hidden
2. Binary compatibility – struct can change without recompiling callers
3. Prevents accidental misuse of internal fields
```

---
