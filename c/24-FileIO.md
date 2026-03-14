# File I/O

---

## Standard I/O (stdio.h)

```c
FILE *fp = fopen("file.txt", "r");   // modes: r, w, a, r+, w+, a+, rb, wb, ab
if (fp == NULL) { perror("fopen"); return -1; }

// Character I/O
int ch = fgetc(fp);       // returns int (to distinguish EOF from valid char)
fputc('A', fp);

// String I/O
char buf[256];
fgets(buf, sizeof(buf), fp);   // reads line (includes \n)
fputs("hello", fp);

// Formatted I/O
fprintf(fp, "Value: %d\n", 42);
fscanf(fp, "%d", &val);

// Block I/O
size_t n = fread(buf, sizeof(char), 256, fp);   // returns items read
fwrite(buf, sizeof(char), n, fp);

// Position
fseek(fp, offset, SEEK_SET);  // SEEK_SET, SEEK_CUR, SEEK_END
long pos = ftell(fp);
rewind(fp);                   // equivalent to fseek(fp, 0, SEEK_SET)

fclose(fp);
```

## Low-Level (POSIX) I/O

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("file.txt", O_RDONLY);
ssize_t bytes = read(fd, buf, sizeof(buf));
write(fd, data, len);
lseek(fd, 0, SEEK_SET);
close(fd);

// Difference: FILE* is buffered (user-space), fd is unbuffered (kernel calls)
```

---

## Interview Q&A

**Q: What is buffering in I/O?**

```
Three types:
  Fully buffered (_IOFBF): flushed when buffer is full (default for files)
  Line buffered  (_IOLBF): flushed on newline (default for stdout when terminal)
  Unbuffered     (_IONBF): each write is a system call (stderr)

setvbuf(fp, buf, _IOFBF, BUFSIZ);  // set buffering mode
```

**Q: Difference between `fgets` and `gets`?**

```
gets(): reads until newline, NO bounds checking → buffer overflow vulnerability.
        REMOVED from C11 standard.
fgets(): takes buffer size parameter → safe.
```

**Q: Why does `fgetc` return `int` instead of `char`?**

```
To distinguish EOF (-1) from valid characters.
If it returned char, EOF could be confused with a valid char value (e.g., 0xFF).
```

**Q: What is the difference between `fopen` and `open`?**

```
fopen: C standard library, returns FILE*, buffered I/O, portable.
open:  POSIX system call, returns int (file descriptor), unbuffered, not portable.
```

---
