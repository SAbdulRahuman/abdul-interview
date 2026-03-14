# Stack

---

## Array-Based Stack

```c
#define MAX 1000
typedef struct {
    int data[MAX];
    int top;
} Stack;

void init(Stack *s)          { s->top = -1; }
int  is_empty(Stack *s)      { return s->top == -1; }
int  is_full(Stack *s)       { return s->top == MAX - 1; }
void push(Stack *s, int val) { s->data[++(s->top)] = val; }
int  pop(Stack *s)           { return s->data[(s->top)--]; }
int  peek(Stack *s)          { return s->data[s->top]; }
```

## Linked List-Based Stack

```c
typedef struct StackNode {
    int data;
    struct StackNode *next;
} StackNode;

void push(StackNode **top, int val) {
    StackNode *n = malloc(sizeof(StackNode));
    n->data = val;
    n->next = *top;
    *top = n;
}

int pop(StackNode **top) {
    StackNode *tmp = *top;
    int val = tmp->data;
    *top = tmp->next;
    free(tmp);
    return val;
}
```

---

## Interview Q&A

**Q: What are common uses of stacks?**

```
- Function call stack (recursion)
- Expression evaluation (infix to postfix)
- Parenthesis matching
- Undo/redo operations
- DFS traversal
- Browser back button
```

---
