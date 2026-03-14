# X-Macros

---

## Pattern

```c
// Define list of items once, use in multiple contexts
#define COLORS(X) \
    X(RED,   "red",   0xFF0000) \
    X(GREEN, "green", 0x00FF00) \
    X(BLUE,  "blue",  0x0000FF)

// Generate enum
#define ENUM_ENTRY(name, str, hex) name,
enum Color { COLORS(ENUM_ENTRY) };

// Generate string array
#define STR_ENTRY(name, str, hex) [name] = str,
const char *color_names[] = { COLORS(STR_ENTRY) };

// Generate hex array
#define HEX_ENTRY(name, str, hex) [name] = hex,
int color_hex[] = { COLORS(HEX_ENTRY) };
```

## Benefits

```
- Single source of truth – add/remove items in one place
- Keeps enum, strings, and other representations in sync
- Eliminates bugs from mismatched arrays
- Used in: error code tables, command dispatchers, protocol definitions
```

---

## Interview Q&A

**Q: What problem do X-macros solve?**

```
Keeping parallel data structures (enum + string array + handlers) in sync.
Without X-macros, adding a new enum value requires updating multiple places,
which is error-prone.
```

---
