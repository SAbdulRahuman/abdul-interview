# Hash Table

---

## Implementation (Chaining)

```c
#define TABLE_SIZE 256

typedef struct Entry {
    char *key;
    int value;
    struct Entry *next;
} Entry;

typedef struct {
    Entry *buckets[TABLE_SIZE];
} HashTable;

unsigned int hash(const char *key) {
    unsigned int h = 0;
    while (*key) h = h * 31 + *key++;
    return h % TABLE_SIZE;
}

void ht_set(HashTable *ht, const char *key, int value) {
    unsigned int idx = hash(key);
    Entry *e = ht->buckets[idx];
    while (e) {
        if (strcmp(e->key, key) == 0) { e->value = value; return; }
        e = e->next;
    }
    e = malloc(sizeof(Entry));
    e->key = strdup(key);
    e->value = value;
    e->next = ht->buckets[idx];
    ht->buckets[idx] = e;
}

int ht_get(HashTable *ht, const char *key, int *out) {
    unsigned int idx = hash(key);
    Entry *e = ht->buckets[idx];
    while (e) {
        if (strcmp(e->key, key) == 0) { *out = e->value; return 1; }
        e = e->next;
    }
    return 0;  // not found
}
```

## Collision Resolution Strategies

```
1. Chaining (separate chaining): each bucket is a linked list
2. Open addressing:
   - Linear probing: check next slot
   - Quadratic probing: check slot + 1², 2², 3², ...
   - Double hashing: use second hash function for step size
```

---

## Interview Q&A

**Q: What is a good hash function?**

```
- Distributes keys uniformly across buckets
- Fast to compute
- Deterministic (same key → same hash)
Common: djb2, FNV-1a, MurmurHash, CityHash
```

**Q: What is the time complexity of hash table operations?**

```
Average case: O(1) for insert, lookup, delete
Worst case: O(n) when all keys hash to the same bucket
Load factor = n/m (items/buckets). Resize when load factor > 0.7-0.8.
```

---
