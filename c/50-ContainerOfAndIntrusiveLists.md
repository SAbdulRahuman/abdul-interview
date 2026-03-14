# container_of & Intrusive Lists

---

## container_of Macro (Linux Kernel Style)

```c
#include <stddef.h>

#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))

struct list_node { struct list_node *next, *prev; };

struct my_struct {
    int data;
    struct list_node node;
};

// Given a pointer to 'node' member, get pointer to containing struct:
struct list_node *n = get_next();
struct my_struct *s = container_of(n, struct my_struct, node);
```

## Intrusive Linked List

```c
// List node embedded in data structures (no separate allocation for list nodes)
typedef struct list_head {
    struct list_head *next, *prev;
} list_head;

void list_init(list_head *head) {
    head->next = head;
    head->prev = head;
}

void list_add(list_head *new, list_head *head) {
    new->next = head->next;
    new->prev = head;
    head->next->prev = new;
    head->next = new;
}

// Iterate:
#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)

// Get containing struct while iterating:
#define list_for_each_entry(pos, head, member) \
    for (pos = container_of((head)->next, typeof(*pos), member); \
         &pos->member != (head); \
         pos = container_of(pos->member.next, typeof(*pos), member))
```

---

## Interview Q&A

**Q: Why use intrusive lists over non-intrusive lists?**

```
1. No extra allocation for list nodes (embedded in the struct)
2. O(1) removal if you have a pointer to the element
3. An element can be in multiple lists simultaneously
4. Better cache locality
Used extensively in the Linux kernel.
```

**Q: How does container_of work?**

```
It subtracts the offset of the member within the struct from the member's address,
giving the address of the containing struct.
Uses offsetof() from <stddef.h> to compute the member offset at compile time.
```

---
