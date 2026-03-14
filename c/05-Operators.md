# Operators

---

## All Operators

```
Arithmetic:    +  -  *  /  %
Relational:    ==  !=  <  >  <=  >=
Logical:       &&  ||  !   (short-circuit evaluation)
Bitwise:       &  |  ^  ~  <<  >>
Assignment:    =  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=
Increment:     ++  --  (prefix vs postfix)
Conditional:   ? :
Comma:         ,
Sizeof:        sizeof
Pointer:       *  &
Member access: .  ->
Cast:          (type)
```

## Operator Precedence (High → Low)

```
1.  () [] -> .                    (postfix)
2.  ! ~ ++ -- + - * & (type) sizeof  (unary / prefix)
3.  * / %                        (multiplicative)
4.  + -                          (additive)
5.  << >>                        (shift)
6.  < <= > >=                    (relational)
7.  == !=                        (equality)
8.  &                            (bitwise AND)
9.  ^                            (bitwise XOR)
10. |                            (bitwise OR)
11. &&                           (logical AND)
12. ||                           (logical OR)
13. ?:                           (ternary)
14. = += -= etc.                 (assignment)
15. ,                            (comma)
```

---

## Interview Q&A

**Q: What is the difference between `++i` and `i++`?**

```
++i (prefix): increments i, then returns the new value.
i++ (postfix): returns the current value of i, then increments.
In standalone statements (i++; vs ++i;) there is no difference.
```

**Q: What is short-circuit evaluation?**

```
In && : if left operand is false (0), right operand is NOT evaluated.
In || : if left operand is true (non-zero), right operand is NOT evaluated.
Used for guarding: if (ptr != NULL && ptr->val > 0)
```

**Q: What does the comma operator do?**

```
Evaluates left operand, discards result, evaluates and returns right operand.
int x = (1, 2, 3);  // x = 3
```

**Q: What is the difference between `=` and `==`?**

```
= is assignment operator.
== is equality comparison operator.
Common bug: if (x = 5) always true (assigns 5 to x).
```

---
