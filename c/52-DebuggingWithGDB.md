# Debugging with GDB

---

## Setup & Basic Commands

```bash
gcc -g program.c -o program
gdb ./program

# Common commands:
break main         # set breakpoint at function
break file.c:42    # set breakpoint at line
run                # start program
run arg1 arg2      # start with arguments
next (n)           # step over (execute line, don't enter functions)
step (s)           # step into (enter function calls)
continue (c)       # continue to next breakpoint
finish             # run until current function returns
```

## Inspecting State

```bash
print var          # print variable value
print *ptr         # dereference pointer
print arr[0]@10    # print 10 elements of array
display var        # print var at every stop
info locals        # show all local variables
info args          # show function arguments
backtrace (bt)     # show call stack
frame N            # switch to stack frame N
```

## Watchpoints & Catchpoints

```bash
watch var          # break when var changes (hardware watchpoint)
rwatch var         # break on read
awatch var         # break on read or write
catch throw        # break on C++ exception
```

## Memory Examination

```bash
x/10x ptr          # examine 10 hex words at ptr
x/10s ptr          # examine as strings
x/10i func         # examine as instructions (disassemble)
```

---

## Interview Q&A

**Q: How to debug a segfault with GDB?**

```bash
$ gdb ./program
(gdb) run
# ... program crashes ...
(gdb) backtrace        # see where it crashed
(gdb) frame 0          # go to crash frame
(gdb) info locals      # see local variables
(gdb) print ptr        # check if pointer is NULL
```

**Q: How to debug a core dump?**

```bash
ulimit -c unlimited    # enable core dumps
./program              # crashes, produces core file
gdb ./program core     # load core dump
(gdb) backtrace        # see crash location
```

---
