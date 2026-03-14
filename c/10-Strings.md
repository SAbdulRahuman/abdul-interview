# Strings

---

## String Basics

```c
// C strings are null-terminated char arrays
char str1[] = "Hello";          // {'H','e','l','l','o','\0'}  – mutable
char *str2 = "Hello";           // pointer to string literal   – immutable (UB if modified)
char str3[10] = "Hi";           // remaining chars are '\0'
```

## Common String Functions (string.h)

```c
strlen(s)                // length (excluding '\0')
strcpy(dst, src)         // copy (UNSAFE – no bounds check)
strncpy(dst, src, n)     // bounded copy (may NOT null-terminate!)
strcat(dst, src)         // concatenate
strncat(dst, src, n)     // bounded concatenate
strcmp(s1, s2)           // compare: 0 if equal, <0 or >0 otherwise
strncmp(s1, s2, n)      // bounded compare
strchr(s, c)             // find first occurrence of char
strrchr(s, c)            // find last occurrence of char
strstr(haystack, needle) // find substring
strtok(s, delim)         // tokenize (destructive, NOT thread-safe)
memcpy(dst, src, n)      // copy n bytes (no overlap)
memmove(dst, src, n)     // copy n bytes (handles overlap)
memset(ptr, val, n)      // fill n bytes with val
snprintf(buf, n, fmt, ...) // safe formatted string (C99)
```

## Safe String Practices

```c
// Always use snprintf instead of sprintf
snprintf(buf, sizeof(buf), "Value: %d", val);

// Always use strncat, strncpy with care
// strncpy may NOT null-terminate if src length >= n
strncpy(dst, src, n - 1);
dst[n - 1] = '\0';  // always null-terminate manually
```

---

## Interview Q&A

**Q: Difference between `char str[]` and `char *str`?**

```
char str[] = "Hello";  → array on stack, mutable, sizeof = 6
char *str = "Hello";   → pointer to read-only string literal, sizeof = 8 (64-bit)
Modifying string literal via pointer is undefined behavior.
```

**Q: What is the null terminator?**

```
'\0' (ASCII value 0). Marks the end of a C string.
All string functions rely on it. Without it, functions read past the buffer → UB.
```

**Q: Difference between `strlen` and `sizeof` on strings?**

```c
char str[] = "Hello";
strlen(str) → 5  (does not count '\0')
sizeof(str) → 6  (counts '\0', compile-time for arrays)

char *p = "Hello";
strlen(p) → 5
sizeof(p) → 8  (size of pointer on 64-bit)
```

---
