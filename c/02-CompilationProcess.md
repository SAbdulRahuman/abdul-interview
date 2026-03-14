# Compilation Process

---

## Stages

```
Source (.c) → Preprocessor → Compiler → Assembler → Linker → Executable

Step-by-step:
1. Preprocessing:  gcc -E file.c -o file.i    (macro expansion, #include, #ifdef)
2. Compilation:    gcc -S file.i -o file.s    (C → assembly)
3. Assembly:       gcc -c file.s -o file.o    (assembly → object code)
4. Linking:        gcc file.o -o file         (resolves symbols, links libraries)
```

## Preprocessing

```
- Expands #include directives (copies header file contents)
- Performs macro substitution (#define)
- Evaluates conditional compilation (#ifdef, #ifndef, #if)
- Removes comments
- Output: translation unit (.i file)
```

## Compilation

```
- Parses the preprocessed source code
- Performs syntax and semantic analysis
- Generates assembly code (.s file)
- Applies optimizations (-O1, -O2, -O3)
```

## Assembly

```
- Converts assembly code to machine code
- Produces object file (.o / .obj)
- Contains machine instructions + relocation info
```

## Linking

```
- Resolves external symbol references
- Combines multiple object files
- Links with static (.a / .lib) or dynamic (.so / .dll) libraries
- Produces final executable (ELF on Linux, PE on Windows, Mach-O on macOS)
```

---

## Interview Q&A

**Q: What is the difference between a compiler and an interpreter?**

```
Compiler: Translates entire source code to machine code before execution (C, C++).
Interpreter: Translates and executes line-by-line at runtime (Python, Ruby).
```

**Q: What is a translation unit?**

```
A source file after preprocessing – all #include files expanded, macros substituted.
Each .c file becomes one translation unit, compiled independently.
```

**Q: What is the difference between a declaration and a definition?**

```
Declaration: tells the compiler about the type/signature (no memory allocated).
  extern int x;         // variable declaration
  int add(int, int);    // function declaration (prototype)

Definition: allocates storage or provides function body.
  int x = 10;           // variable definition
  int add(int a, int b) { return a + b; }  // function definition
```

---
