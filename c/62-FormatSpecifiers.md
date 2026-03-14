# Format Specifiers Reference

---

## printf / scanf Specifiers

```
%d   – int (signed decimal)
%u   – unsigned int
%ld  – long int
%lu  – unsigned long
%lld – long long int
%llu – unsigned long long
%f   – float / double (printf promotes float to double)
%e   – scientific notation
%g   – shorter of %f and %e
%c   – character
%s   – string (char *)
%p   – pointer (void *)
%x   – unsigned hex (lowercase)
%X   – unsigned hex (uppercase)
%o   – unsigned octal
%zu  – size_t
%td  – ptrdiff_t
%%   – literal percent sign
%n   – number of chars written so far (DANGEROUS – format string attacks!)
```

## Width & Precision

```c
printf("%10d", 42);       // right-aligned in 10-char field: "        42"
printf("%-10d", 42);      // left-aligned:  "42        "
printf("%05d", 42);       // zero-padded:   "00042"
printf("%.3f", 3.14159);  // precision:     "3.142"
printf("%.5s", "Hello World"); // max 5 chars: "Hello"
printf("%*d", width, val);     // width from argument
```

## scanf Specifiers

```c
int x;     scanf("%d", &x);
float f;   scanf("%f", &f);
double d;  scanf("%lf", &d);  // note: %lf for double in scanf (not %f)
char c;    scanf(" %c", &c);  // space before %c skips whitespace
char s[50]; scanf("%49s", s); // limit width to prevent overflow
```

---

## Interview Q&A

**Q: Why is `%n` dangerous?**

```
%n writes the number of characters printed so far to a pointer argument.
In format string attacks, an attacker can use %n to write arbitrary values
to arbitrary memory addresses → code execution vulnerability.
Never use %n with user-controlled format strings.
```

**Q: Difference between `%f` for printf and scanf?**

```
printf: %f works for both float and double (float is promoted to double).
scanf:  %f reads float, %lf reads double. Using wrong one → undefined behavior.
```

---
