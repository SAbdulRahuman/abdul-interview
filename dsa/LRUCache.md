# LRU (Least Recently Used) Cache

## Overview
An LRU Cache is a data structure that stores key-value pairs with bounded capacity. When the cache is full and a new item is added, the least recently used item is evicted.

**Key Property**: Combines O(1) access, insertion, and deletion by using a HashMap + Doubly Linked List.

---

## Why LRU Cache?
- **Real-world use**: CPU caches, database queries, CDN content caching
- **Problem**: Need fast lookups (O(1)) + ability to track usage order
- **Naive approach**: HashMap alone doesn't maintain order; Linked List alone has O(n) lookup

---

## Design Pattern

### Components
1. **HashMap**: Maps key → Node (for O(1) access)
2. **Doubly Linked List**: Maintains order of usage
   - Most recently used → right/tail
   - Least recently used → left/head

### Operations
| Operation | Time | Details |
|-----------|------|---------|
| `get(key)` | O(1) | Lookup + move to head (most recent) |
| `put(key, val)` | O(1) | Insert/update + move to head |
| Eviction | O(1) | Remove tail node when full |

---

## Implementation

### Python with Collections
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        
        self.cache[key] = value
        
        # Evict least recently used (first item)
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)

# Usage
cache = LRUCache(2)
cache.put(1, 1)      # cache: {1:1}
cache.put(2, 2)      # cache: {1:1, 2:2}
print(cache.get(1))  # 1 (now becomes most recent)
cache.put(3, 3)      # cache: {1:1, 3:3} (2 evicted)
print(cache.get(2))  # -1 (2 was evicted)
```

### Python with Doubly Linked List
```python
class Node:
    def __init__(self, key: int, val: int):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> Node
        
        # Dummy nodes for easier edge case handling
        self.head = Node(0, 0)  # least recently used
        self.tail = Node(0, 0)  # most recently used
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _remove(self, node: Node) -> None:
        """Remove node from linked list"""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node
    
    def _add_to_tail(self, node: Node) -> None:
        """Add node to end (most recent)"""
        prev_node = self.tail.prev
        prev_node.next = node
        node.prev = prev_node
        node.next = self.tail
        self.tail.prev = node
    
    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        
        node = self.cache[key]
        self._remove(node)
        self._add_to_tail(node)
        return node.val
    
    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self._remove(self.cache[key])
        
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_tail(node)
        
        # Evict if capacity exceeded
        if len(self.cache) > self.capacity:
            lru_node = self.head.next  # least recently used
            self._remove(lru_node)
            del self.cache[lru_node.key]
```

### Java
```java
class LRUCache {
    private class Node {
        int key, val;
        Node prev, next;
        Node(int key, int val) {
            this.key = key;
            this.val = val;
        }
    }
    
    private int capacity;
    private Map<Integer, Node> cache = new HashMap<>();
    private Node head, tail;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.head = new Node(0, 0);
        this.tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }
    
    private void remove(Node node) {
        Node prev = node.prev;
        Node next = node.next;
        prev.next = next;
        next.prev = prev;
    }
    
    private void addToTail(Node node) {
        Node prev = tail.prev;
        prev.next = node;
        node.prev = prev;
        node.next = tail;
        tail.prev = node;
    }
    
    public int get(int key) {
        if (!cache.containsKey(key)) return -1;
        
        Node node = cache.get(key);
        remove(node);
        addToTail(node);
        return node.val;
    }
    
    public void put(int key, int value) {
        if (cache.containsKey(key)) {
            remove(cache.get(key));
        }
        
        Node node = new Node(key, value);
        cache.put(key, node);
        addToTail(node);
        
        if (cache.size() > capacity) {
            Node lru = head.next;
            remove(lru);
            cache.remove(lru.key);
        }
    }
}
```

### Go
```go
type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node
    tail     *Node
}

type Node struct {
    key  int
    val  int
    prev *Node
    next *Node
}

func Constructor(capacity int) LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     head,
        tail:     tail,
    }
}

func (lru *LRUCache) remove(node *Node) {
    prev := node.prev
    next := node.next
    prev.next = next
    next.prev = prev
}

func (lru *LRUCache) addToTail(node *Node) {
    prev := lru.tail.prev
    prev.next = node
    node.prev = prev
    node.next = lru.tail
    lru.tail.prev = node
}

func (lru *LRUCache) Get(key int) int {
    if node, exists := lru.cache[key]; exists {
        lru.remove(node)
        lru.addToTail(node)
        return node.val
    }
    return -1
}

func (lru *LRUCache) Put(key int, value int) {
    if node, exists := lru.cache[key]; exists {
        lru.remove(node)
    }
    
    newNode := &Node{key: key, val: value}
    lru.cache[key] = newNode
    lru.addToTail(newNode)
    
    if len(lru.cache) > lru.capacity {
        lru_node := lru.head.next
        lru.remove(lru_node)
        delete(lru.cache, lru_node.key)
    }
}
```

---

## Complexity Analysis

| Operation | Time | Space |
|-----------|------|-------|
| `get()` | O(1) | O(1) |
| `put()` | O(1) | O(1) |
| **Overall** | **O(1)** | **O(capacity)** |

**Why O(1)?**
- Hash map lookup: O(1)
- Doubly linked list operations (add/remove): O(1)
- No sorting or searching needed

---

## Interview Tips

### Common Variations
1. **LFU Cache** (Least Frequently Used)
   - Track frequency of access instead of recency
   - More complex: need frequency map + priority queue

2. **Time-window Cache**
   - Invalidate items after TTL (time-to-live)
   - Use timestamp + priority queue

3. **Multi-tier Cache**
   - L1 (fast, small) + L2 (slow, large)
   - Promote on hit

### Edge Cases
- Capacity = 1
- Multiple `get()` on same key
- `put()` with existing key (update + move to front)
- Thread safety (use locks if needed)

### Optimization Notes
- **Dummy nodes**: Eliminate edge case handling for empty/full lists
- **Bidirectional access**: HashMap + DLL provides both O(1) forward and backward
- **Python's OrderedDict**: Simplifies implementation if allowed

---

## Practical Usage
```python
# Web server using LRU cache
cache = LRUCache(capacity=1000)

def fetch_page(url):
    if cached := cache.get(url):
        return cached  # O(1) hit
    
    content = download(url)  # I/O bound
    cache.put(url, content)
    return content
```

---

## Related Topics
- [DoublyLinkedList](DoublyLinkedList.md)
- [HashFunctions](HashFunctions.md)
- [Memoization](Memoization.md)
- [CompositeDataStructureDesign](CompositeDataStructureDesign.md)
