# Object-Oriented Patterns in C

---

## "Inheritance" via Struct Embedding

```c
typedef struct {
    int x, y;
} Shape;

typedef struct {
    Shape base;   // "inherits" from Shape
    int radius;
} Circle;

// Can cast Circle* to Shape* safely (first member rule)
Circle c = {{10, 20}, 5};
Shape *s = (Shape *)&c;
printf("x=%d\n", s->x);  // 10
```

## "Polymorphism" via Function Pointers (VTable)

```c
typedef struct {
    void (*draw)(void *self);
    double (*area)(void *self);
} ShapeVTable;

typedef struct {
    ShapeVTable *vtable;
    int x, y;
} ShapeObj;

// Usage:
shape->vtable->draw(shape);
shape->vtable->area(shape);
```

## "Constructor" / "Destructor" Pattern

```c
typedef struct {
    int *data;
    int size;
} MyObj;

MyObj *myobj_create(int size) {
    MyObj *obj = malloc(sizeof(MyObj));
    obj->data = calloc(size, sizeof(int));
    obj->size = size;
    return obj;
}

void myobj_destroy(MyObj *obj) {
    free(obj->data);
    free(obj);
}
```

---

## Interview Q&A

**Q: How does GObject (GNOME) achieve OOP in C?**

```
Uses struct embedding for inheritance, function pointer tables (vtables)
for polymorphism, and reference counting for memory management.
GTK, GStreamer, and many GNOME libraries use this pattern.
```

---
