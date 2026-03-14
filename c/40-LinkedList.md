# Linked List

---

## Implementation

```c
typedef struct Node {
    int data;
    struct Node *next;
} Node;

Node *create_node(int val) {
    Node *n = malloc(sizeof(Node));
    if (!n) return NULL;
    n->data = val;
    n->next = NULL;
    return n;
}

// Insert at head
void push(Node **head, int val) {
    Node *n = create_node(val);
    n->next = *head;
    *head = n;
}

// Insert at tail
void append(Node **head, int val) {
    Node *n = create_node(val);
    if (!*head) { *head = n; return; }
    Node *curr = *head;
    while (curr->next) curr = curr->next;
    curr->next = n;
}

// Delete a node by value
void delete_node(Node **head, int val) {
    Node *curr = *head, *prev = NULL;
    while (curr && curr->data != val) {
        prev = curr;
        curr = curr->next;
    }
    if (!curr) return;
    if (!prev) *head = curr->next;
    else prev->next = curr->next;
    free(curr);
}

// Reverse a linked list
Node *reverse(Node *head) {
    Node *prev = NULL, *curr = head, *next;
    while (curr) {
        next = curr->next;
        curr->next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// Free list
void free_list(Node *head) {
    while (head) {
        Node *tmp = head;
        head = head->next;
        free(tmp);
    }
}
```

---

## Interview Q&A

**Q: Why pass `Node **head` instead of `Node *head`?**

```
To modify the head pointer itself (e.g., insert at head, delete head).
Node *head passes the pointer by value → changes to head are local.
Node **head passes the pointer by reference → can update caller's head.
```

**Q: How to detect a cycle in a linked list?**

```c
int has_cycle(Node *head) {
    Node *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return 1;  // cycle detected
    }
    return 0;
}
// Floyd's tortoise and hare algorithm – O(n) time, O(1) space
```

---
