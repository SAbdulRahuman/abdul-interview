# Frequently Asked Code Questions

---

## 1. Reverse a String In-Place

```c
void reverse_str(char *s) {
    int len = strlen(s);
    for (int i = 0; i < len / 2; i++) {
        char tmp = s[i];
        s[i] = s[len - 1 - i];
        s[len - 1 - i] = tmp;
    }
}
```

## 2. Check if a String is a Palindrome

```c
int is_palindrome(const char *s) {
    int l = 0, r = strlen(s) - 1;
    while (l < r) {
        if (s[l++] != s[r--]) return 0;
    }
    return 1;
}
```

## 3. Implement atoi

```c
int my_atoi(const char *s) {
    int sign = 1, result = 0;
    while (*s == ' ') s++;
    if (*s == '-' || *s == '+') { sign = (*s == '-') ? -1 : 1; s++; }
    while (*s >= '0' && *s <= '9') {
        if (result > (INT_MAX - (*s - '0')) / 10)
            return (sign == 1) ? INT_MAX : INT_MIN;
        result = result * 10 + (*s - '0');
        s++;
    }
    return sign * result;
}
```

## 4. Implement memcpy

```c
void *my_memcpy(void *dst, const void *src, size_t n) {
    char *d = dst;
    const char *s = src;
    while (n--) *d++ = *s++;
    return dst;
}
```

## 5. Implement memmove (Handles Overlapping Regions)

```c
void *my_memmove(void *dst, const void *src, size_t n) {
    char *d = dst;
    const char *s = src;
    if (d < s) {
        while (n--) *d++ = *s++;
    } else {
        d += n; s += n;
        while (n--) *--d = *--s;
    }
    return dst;
}
```

## 6. Detect Cycle in Linked List (Floyd's Algorithm)

```c
int has_cycle(Node *head) {
    Node *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return 1;
    }
    return 0;
}
```

## 7. Implement strlen

```c
size_t my_strlen(const char *s) {
    const char *p = s;
    while (*p) p++;
    return p - s;
}
```

## 8. Implement strcmp

```c
int my_strcmp(const char *s1, const char *s2) {
    while (*s1 && *s1 == *s2) { s1++; s2++; }
    return *(unsigned char *)s1 - *(unsigned char *)s2;
}
```

## 9. Find Middle of Linked List

```c
Node *find_middle(Node *head) {
    Node *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;
}
```

## 10. Merge Two Sorted Linked Lists

```c
Node *merge(Node *a, Node *b) {
    if (!a) return b;
    if (!b) return a;
    if (a->data <= b->data) {
        a->next = merge(a->next, b);
        return a;
    } else {
        b->next = merge(a, b->next);
        return b;
    }
}
```

---
