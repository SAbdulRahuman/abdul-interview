# Queue (Circular Buffer)

---

## Array-Based Circular Queue

```c
#define MAX 1000
typedef struct {
    int data[MAX];
    int front, rear, count;
} Queue;

void init(Queue *q) { q->front = 0; q->rear = -1; q->count = 0; }
int  is_empty(Queue *q) { return q->count == 0; }
int  is_full(Queue *q)  { return q->count == MAX; }

void enqueue(Queue *q, int val) {
    q->rear = (q->rear + 1) % MAX;
    q->data[q->rear] = val;
    q->count++;
}
int dequeue(Queue *q) {
    int val = q->data[q->front];
    q->front = (q->front + 1) % MAX;
    q->count--;
    return val;
}
```

## Linked List-Based Queue

```c
typedef struct QNode {
    int data;
    struct QNode *next;
} QNode;

typedef struct {
    QNode *front, *rear;
} Queue;

void enqueue(Queue *q, int val) {
    QNode *n = malloc(sizeof(QNode));
    n->data = val;
    n->next = NULL;
    if (q->rear) q->rear->next = n;
    else q->front = n;
    q->rear = n;
}

int dequeue(Queue *q) {
    QNode *tmp = q->front;
    int val = tmp->data;
    q->front = tmp->next;
    if (!q->front) q->rear = NULL;
    free(tmp);
    return val;
}
```

---

## Interview Q&A

**Q: Why use a circular buffer?**

```
Avoids wasting space. In a linear array queue, dequeued slots are wasted.
Circular buffer wraps around using modulo: (index + 1) % MAX.
```

**Q: What are common uses of queues?**

```
- BFS traversal
- Task scheduling
- Producer-consumer pattern
- Print spooling
- Message passing between threads
```

---
