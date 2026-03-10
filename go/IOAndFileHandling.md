# Chapter 19 — I/O & File Handling

Go's I/O system is built around two simple interfaces: `io.Reader` and `io.Writer`. Everything — files, network connections, compressed streams, HTTP bodies — implements these interfaces, making them composable and interchangeable.

```
┌──────────────────────────────────────────────────────────┐
│            io.Reader / io.Writer Ecosystem              │
│                                                          │
│  io.Reader: Read(p []byte) (n int, err error)           │
│  io.Writer: Write(p []byte) (n int, err error)          │
│                                                          │
│  Common Readers:           Common Writers:               │
│  ├── os.File               ├── os.File                  │
│  ├── strings.Reader        ├── strings.Builder          │
│  ├── bytes.Buffer          ├── bytes.Buffer             │
│  ├── http.Request.Body     ├── http.ResponseWriter      │
│  ├── gzip.Reader           ├── gzip.Writer              │
│  └── bufio.Reader          └── bufio.Writer             │
│                                                          │
│  Composable:                                            │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐              │
│  │ os.File  │──►│ gzip.     │──►│ bufio.  │              │
│  │ (Reader) │  │ Reader    │  │ Reader  │              │
│  └──────────┘  └───────────┘  └──────────┘              │
│  Read compressed file into buffered reader              │
│                                                          │
│  Key functions:                                         │
│  io.Copy(dst, src)      → stream from reader to writer  │
│  io.ReadAll(r)          → read all bytes from reader    │
│  io.TeeReader(r, w)     → read + write simultaneously  │
│  io.MultiReader(r1, r2) → concatenate readers           │
│  io.MultiWriter(w1, w2) → duplicate writes              │
└──────────────────────────────────────────────────────────┘
```

## io.Reader and io.Writer Interfaces

**Tutorial: Reading and Writing Byte Streams**

This example demonstrates the two foundational I/O interfaces in Go. A `strings.NewReader` is used as an `io.Reader`, reading data in fixed-size chunks until `io.EOF` is returned. A `strings.Builder` serves as an `io.Writer`, accumulating bytes written to it. Notice how `Read` returns the number of bytes actually read — you must use `buf[:n]`, not the entire buffer.

```
┌─────────────────────────────────────────────┐
│         io.Reader loop pattern              │
│                                             │
│  strings.Reader("Hello, Reader!")           │
│       │                                     │
│       ▼                                     │
│  ┌──────────┐  Read(buf)  ┌────────────┐    │
│  │  Reader   │───────────►│ buf[0..4]  │    │
│  │ pos=0..14 │  n=5       │ "Hello"    │    │
│  └──────────┘             └────────────┘    │
│       │         Read(buf)                   │
│       ├──────────────────►│ ", Rea"    │    │
│       │         Read(buf)                   │
│       ├──────────────────►│ "der!"     │    │
│       │         Read(buf)                   │
│       └──────────────────► n=0, io.EOF      │
│                                             │
│  io.Writer accumulation:                    │
│  Builder ◄── Write("Hello, ")               │
│  Builder ◄── Write("Writer!")               │
│  Builder.String() ──► "Hello, Writer!"      │
└─────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

func main() {
    // io.Reader: Read(p []byte) (n int, err error)
    // io.Writer: Write(p []byte) (n int, err error)

    // strings.NewReader implements io.Reader
    reader := strings.NewReader("Hello, Reader!")
    buf := make([]byte, 5)

    for {
        n, err := reader.Read(buf)
        if n > 0 {
            fmt.Print(string(buf[:n]))
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            fmt.Println("Error:", err)
            break
        }
    }
    fmt.Println()
    // Output: Hello, Reader!

    // strings.Builder implements io.Writer
    var writer strings.Builder
    writer.Write([]byte("Hello, "))
    writer.Write([]byte("Writer!"))
    fmt.Println(writer.String()) // Hello, Writer!
}
```

---

## io.Copy, io.TeeReader, io.MultiReader, io.MultiWriter

**Tutorial: Composing Readers and Writers**

This example shows four powerful I/O combinators. `io.Copy` streams data from a Reader to a Writer without loading everything into memory. `io.TeeReader` splits a read stream so data is simultaneously consumed and logged. `io.MultiReader` concatenates multiple readers into one sequential stream, while `io.MultiWriter` fans out writes to multiple destinations at once. These combinators make complex pipelines simple without custom glue code.

```
┌──────────────────────────────────────────────────┐
│  io.Copy           io.TeeReader                  │
│  ┌────────┐        ┌────────┐                    │
│  │ Reader │──Copy──►│ Writer │                    │
│  └────────┘        └────────┘                    │
│                                                  │
│  ┌────────┐  read  ┌────────────┐  write         │
│  │Original│───────►│ TeeReader  │───────►logBuf  │
│  └────────┘        └─────┬──────┘                │
│                          ▼                       │
│                     consumer reads               │
│                                                  │
│  io.MultiReader        io.MultiWriter            │
│  ┌────┐ ┌────┐         ┌────────────┐            │
│  │ r1 │►│ r2 │──►read  │ MultiWrite │            │
│  └────┘ └────┘         └──┬───┬──┬──┘            │
│  sequential               ▼   ▼  ▼              │
│                         buf1 buf2 stdout         │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "io"
    "os"
    "strings"
)

func main() {
    // io.Copy — copies from Reader to Writer
    src := strings.NewReader("Copy this text")
    var dst strings.Builder
    n, _ := io.Copy(&dst, src)
    fmt.Printf("Copied %d bytes: %s\n", n, dst.String())

    // io.TeeReader — reads from r, writes to w (like Unix tee)
    original := strings.NewReader("Tee this data")
    var logBuf strings.Builder
    tee := io.TeeReader(original, &logBuf) // reads go to both tee and logBuf

    content, _ := io.ReadAll(tee)
    fmt.Println("Read:", string(content))     // Tee this data
    fmt.Println("Logged:", logBuf.String())   // Tee this data

    // io.MultiReader — concatenates multiple readers
    r1 := strings.NewReader("Hello, ")
    r2 := strings.NewReader("World!")
    multi := io.MultiReader(r1, r2)
    data, _ := io.ReadAll(multi)
    fmt.Println("Multi:", string(data)) // Hello, World!

    // io.MultiWriter — writes to multiple writers simultaneously
    var buf1, buf2 strings.Builder
    multiW := io.MultiWriter(&buf1, &buf2, os.Stdout)
    fmt.Fprint(multiW, "Goes everywhere!")
    // Prints to stdout AND writes to buf1 AND buf2
    fmt.Println()
    fmt.Println("buf1:", buf1.String()) // Goes everywhere!
    fmt.Println("buf2:", buf2.String()) // Goes everywhere!
}
```

---

## io.ReadAll, io.LimitReader, io.NopCloser

**Tutorial: Controlled Reading with Limits and Wrappers**

This example covers three utility functions from the `io` package. `io.ReadAll` consumes an entire reader into a `[]byte` — convenient but use carefully with large streams. `io.LimitReader` wraps a reader to cap the maximum bytes read, which is critical for preventing memory exhaustion on untrusted input. `io.NopCloser` adds a no-op `Close()` method, adapting an `io.Reader` into an `io.ReadCloser` when an API requires one.

```
┌─────────────────────────────────────────────────┐
│  io.ReadAll                                     │
│  ┌──────────┐  ReadAll  ┌───────────────────┐   │
│  │  Reader   │─────────►│ []byte (all data) │   │
│  └──────────┘           └───────────────────┘   │
│                                                 │
│  io.LimitReader                                 │
│  ┌──────────────────────────┐                   │
│  │ Reader: "Only read..."   │                   │
│  └──────────┬───────────────┘                   │
│             ▼                                   │
│  ┌──────────────────┐                           │
│  │ LimitReader(r,5) │  max 5 bytes              │
│  └────────┬─────────┘                           │
│           ▼                                     │
│   ReadAll ──► "Only " (5 bytes only)            │
│                                                 │
│  io.NopCloser                                   │
│  ┌────────┐  NopCloser  ┌──────────────┐        │
│  │ Reader │────────────►│ ReadCloser   │        │
│  └────────┘             │ Close()=noop │        │
│                         └──────────────┘        │
└─────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

func main() {
    // io.ReadAll — read everything from a reader
    r := strings.NewReader("Read all of this")
    data, err := io.ReadAll(r)
    if err != nil {
        fmt.Println("Error:", err)
    }
    fmt.Println(string(data)) // Read all of this

    // io.LimitReader — limits how many bytes to read
    r2 := strings.NewReader("Only read first 5 bytes")
    limited := io.LimitReader(r2, 5)
    data2, _ := io.ReadAll(limited)
    fmt.Println(string(data2)) // Only

    // io.NopCloser — wraps Reader to add a no-op Close method
    // Useful when you need an io.ReadCloser but only have io.Reader
    r3 := strings.NewReader("No close needed")
    rc := io.NopCloser(r3) // now implements io.ReadCloser
    defer rc.Close()       // Close does nothing
    data3, _ := io.ReadAll(rc)
    fmt.Println(string(data3)) // No close needed
}
```

---

## os Package — File Operations

**Tutorial: File CRUD with the os Package**

This example walks through the core file operations: writing an entire file with `os.WriteFile`, reading it back with `os.ReadFile`, inspecting metadata with `os.Stat`, and using lower-level `os.Open` / `os.Create` / `os.OpenFile` for fine-grained control. Note the flag constants (`os.O_APPEND|os.O_CREATE|os.O_WRONLY`) that control append-mode behavior. Always `defer file.Close()` immediately after opening to avoid resource leaks.

```
┌────────────────────────────────────────────────────┐
│  os File Operations                                │
│                                                    │
│  os.WriteFile("f.txt", data, 0644)                 │
│       │  creates/overwrites file                   │
│       ▼                                            │
│  ┌──────────┐                                      │
│  │ f.txt    │  ◄── on disk                         │
│  └──────────┘                                      │
│       │                                            │
│  os.ReadFile("f.txt") ──► []byte                   │
│  os.Stat("f.txt")     ──► FileInfo{Name,Size,...}  │
│                                                    │
│  os.Open("f.txt")     ──► *File (read-only)        │
│  os.Create("out.txt") ──► *File (write, truncate)  │
│  os.OpenFile("log.txt", flags, perm) ──► *File     │
│       │                                            │
│       │  flags: O_APPEND │ O_CREATE │ O_WRONLY     │
│       ▼                                            │
│  defer file.Close()   ◄── always close!            │
└────────────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.WriteFile — write entire file (Go 1.16+)
    err := os.WriteFile("example.txt", []byte("Hello, File!"), 0644)
    if err != nil {
        fmt.Println("Write error:", err)
        return
    }

    // os.ReadFile — read entire file (Go 1.16+)
    data, err := os.ReadFile("example.txt")
    if err != nil {
        fmt.Println("Read error:", err)
        return
    }
    fmt.Println("Content:", string(data)) // Hello, File!

    // os.Stat — get file info
    info, err := os.Stat("example.txt")
    if err != nil {
        if os.IsNotExist(err) {
            fmt.Println("File does not exist")
        }
        return
    }
    fmt.Println("Name:", info.Name())    // example.txt
    fmt.Println("Size:", info.Size())    // 12
    fmt.Println("IsDir:", info.IsDir())  // false

    // os.Open — open for reading
    file, err := os.Open("example.txt")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer file.Close()

    buf := make([]byte, 5)
    n, _ := file.Read(buf)
    fmt.Println("Read:", string(buf[:n])) // Hello

    // os.Create — create or truncate file for writing
    f2, _ := os.Create("output.txt")
    defer f2.Close()
    f2.WriteString("Created with os.Create")

    // os.OpenFile — full control over flags and permissions
    f3, _ := os.OpenFile("log.txt", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    defer f3.Close()
    f3.WriteString("Appended line\n")

    // Environment variables
    fmt.Println("HOME:", os.Getenv("HOME"))
    os.Setenv("MY_VAR", "hello")
    fmt.Println("MY_VAR:", os.Getenv("MY_VAR"))

    // Command-line args
    fmt.Println("Program:", os.Args[0])

    // Cleanup
    os.Remove("example.txt")
    os.Remove("output.txt")
    os.Remove("log.txt")
}
```

---

## bufio Package — Buffered I/O

**Tutorial: Buffered Reading and Writing with bufio**

This example demonstrates three key `bufio` types. `bufio.Scanner` reads input token-by-token (lines by default, or words with `ScanWords`). `bufio.Reader` wraps a reader with an internal buffer enabling `ReadString` and `Peek` operations. `bufio.Writer` batches small writes into a buffer — you **must** call `Flush()` to push buffered data to the underlying writer, otherwise data is silently lost.

```
┌──────────────────────────────────────────────────┐
│  bufio.Scanner                                   │
│  ┌─────────────────────┐                         │
│  │ "Line 1\nLine 2\n" │                          │
│  └────────┬────────────┘                         │
│           ▼                                      │
│  ┌──────────────┐  Scan()   ┌──────────┐         │
│  │   Scanner    │──────────►│ "Line 1" │         │
│  │ Split=Lines  │  Scan()   │ "Line 2" │         │
│  │              │──────────►│ "Line 3" │         │
│  └──────────────┘  false    └──────────┘         │
│                                                  │
│  bufio.Writer — MUST Flush!                      │
│  ┌──────────┐  WriteString  ┌────────────┐       │
│  │  Writer  │◄──────────────│ "Buffered "│       │
│  │ [buffer] │◄──────────────│ "content"  │       │
│  └────┬─────┘               └────────────┘       │
│       │ Flush()                                  │
│       ▼                                          │
│  ┌──────────────────┐                            │
│  │ underlying Writer│ ◄── data written here      │
│  └──────────────────┘                            │
└──────────────────────────────────────────────────┘
```

```go
package main

import (
    "bufio"
    "fmt"
    "strings"
)

func main() {
    // bufio.Scanner — read line by line
    input := "Line 1\nLine 2\nLine 3\n"
    scanner := bufio.NewScanner(strings.NewReader(input))

    lineNum := 0
    for scanner.Scan() {
        lineNum++
        fmt.Printf("  Line %d: %s\n", lineNum, scanner.Text())
    }
    if err := scanner.Err(); err != nil {
        fmt.Println("Scanner error:", err)
    }

    // Scanner with custom split function (word by word)
    wordScanner := bufio.NewScanner(strings.NewReader("hello world foo bar"))
    wordScanner.Split(bufio.ScanWords)
    for wordScanner.Scan() {
        fmt.Print(wordScanner.Text(), " ")
    }
    fmt.Println() // hello world foo bar

    // bufio.Reader — buffered reading with peek/readline
    reader := bufio.NewReader(strings.NewReader("Hello\nWorld\n"))
    line, _ := reader.ReadString('\n')
    fmt.Print("First line: ", line) // Hello\n

    // bufio.Writer — buffered writing (must call Flush!)
    var output strings.Builder
    writer := bufio.NewWriter(&output)
    writer.WriteString("Buffered ")
    writer.WriteString("content")
    writer.Flush() // IMPORTANT: flush or data stays in buffer
    fmt.Println(output.String()) // Buffered content
}
```

---

## embed Package (Go 1.16+)

**Tutorial: Embedding Files into Go Binaries**

The `//go:embed` directive embeds file content into the compiled binary at build time, eliminating the need for external file dependencies at runtime. You can embed as a `string`, `[]byte` (for single files), or `embed.FS` (for directories). The embedded filesystem implements `fs.FS`, so you can use `ReadFile` and `ReadDir` on it. This is commonly used for bundling HTML templates, static assets, or default configuration.

```
┌────────────────────────────────────────────────┐
│  //go:embed at compile time                    │
│                                                │
│  Source Files            Compiled Binary        │
│  ┌────────────┐         ┌──────────────────┐   │
│  │config.json │──embed──►│ var configData   │   │
│  └────────────┘         │     string       │   │
│  ┌────────────┐         │                  │   │
│  │ logo.png   │──embed──►│ var logoBytes    │   │
│  └────────────┘         │     []byte       │   │
│  ┌────────────┐         │                  │   │
│  │templates/  │──embed──►│ var templateFS   │   │
│  │ ├─index.   │         │     embed.FS     │   │
│  │ └─about.   │         └──────────────────┘   │
│  └────────────┘                                │
│                                                │
│  At runtime:                                   │
│  templateFS.ReadFile("templates/index.html")   │
│  templateFS.ReadDir("templates")               │
└────────────────────────────────────────────────┘
```

```go
package main

import (
    "embed"
    "fmt"
)

// Embed a single file as a string
//go:embed config.json
var configData string

// Embed a single file as bytes
//go:embed logo.png
var logoBytes []byte

// Embed entire directory
//go:embed templates/*
var templateFS embed.FS

func main() {
    // Use embedded file content
    fmt.Println("Config:", configData)

    fmt.Printf("Logo size: %d bytes\n", len(logoBytes))

    // Read from embedded filesystem
    data, err := templateFS.ReadFile("templates/index.html")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Template:", string(data))

    // Walk embedded directory
    entries, _ := templateFS.ReadDir("templates")
    for _, entry := range entries {
        fmt.Println("  File:", entry.Name())
    }
}
```

---

## os/exec — Running External Commands

**Tutorial: Spawning External Processes**

This example shows how to run shell commands from Go using `os/exec`. `exec.Command` creates a `Cmd` struct, and `.Output()` runs it and captures stdout. `exec.LookPath` checks whether a command exists in `$PATH` before attempting execution. `CombinedOutput()` captures both stdout and stderr in a single byte slice. Always check the returned `error` — a non-zero exit code produces an `*exec.ExitError`.

```
┌───────────────────────────────────────────────┐
│  exec.Command flow                            │
│                                               │
│  Go Process                                   │
│  ┌──────────────────────┐                     │
│  │ cmd := exec.Command  │                     │
│  │  ("echo", "Hello")   │                     │
│  └──────────┬───────────┘                     │
│             │ .Output()                       │
│             ▼                                 │
│  ┌──────────────────────┐                     │
│  │  OS spawns child     │                     │
│  │  process: echo Hello │                     │
│  └──────────┬───────────┘                     │
│             │ stdout                          │
│             ▼                                 │
│  ┌──────────────────────┐                     │
│  │ []byte("Hello...\n") │ ◄── returned        │
│  └──────────────────────┘                     │
│                                               │
│  exec.LookPath("go")                          │
│   ──► searches $PATH ──► "/usr/local/go/..." │
└───────────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "os/exec"
)

func main() {
    // Run a command and get output
    out, err := exec.Command("echo", "Hello from exec!").Output()
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Print(string(out)) // Hello from exec!

    // Check if command exists
    path, err := exec.LookPath("go")
    if err != nil {
        fmt.Println("go not found")
    } else {
        fmt.Println("go found at:", path)
    }

    // Run with combined stdout+stderr
    cmd := exec.Command("go", "version")
    output, err := cmd.CombinedOutput()
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Print(string(output)) // go version go1.23...
}
```

---

## io/fs Package (Go 1.16+)

**Tutorial: Walking Directory Trees with io/fs**

This example uses `fs.WalkDir` to recursively traverse a directory tree rooted at `"."`. The `os.DirFS(".")` call creates an `fs.FS` from the real filesystem, and the callback receives each entry with its path, `DirEntry` metadata, and any error. This pattern is preferred over the older `filepath.Walk` because it avoids an extra `os.Stat` call per file — `DirEntry` can lazily fetch `FileInfo` via `.Info()` only when needed.

```
┌───────────────────────────────────────────┐
│  fs.WalkDir traversal order               │
│                                           │
│  "." (root)                               │
│   ├── DIR: src/                           │
│   │    ├── FILE: main.go (1234 bytes)     │
│   │    └── FILE: util.go (567 bytes)      │
│   ├── DIR: tests/                         │
│   │    └── FILE: main_test.go (890 bytes) │
│   └── FILE: go.mod (45 bytes)             │
│                                           │
│  Callback receives:                       │
│  ┌──────────────────────────────────┐     │
│  │ path: "src/main.go"              │     │
│  │ d.IsDir() ──► false              │     │
│  │ d.Info()  ──► FileInfo{Size:...} │     │
│  │ return nil (continue walking)    │     │
│  └──────────────────────────────────┘     │
└───────────────────────────────────────────┘
```

```go
package main

import (
    "fmt"
    "io/fs"
    "os"
)

func main() {
    // fs.WalkDir — walk a directory tree
    err := fs.WalkDir(os.DirFS("."), ".", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        if d.IsDir() {
            fmt.Println("DIR:", path)
        } else {
            info, _ := d.Info()
            fmt.Printf("FILE: %s (%d bytes)\n", path, info.Size())
        }
        return nil
    })
    if err != nil {
        fmt.Println("Walk error:", err)
    }
}
```

---

## Interview Questions

1. **What are the `io.Reader` and `io.Writer` interfaces?**
   - `io.Reader` has `Read(p []byte) (n int, err error)`. `io.Writer` has `Write(p []byte) (n int, err error)`. They are the fundamental I/O abstractions in Go—files, network connections, buffers all implement them.

2. **What is the difference between `os.Open` and `os.Create`?**
   - `os.Open` opens a file read-only. `os.Create` creates or truncates a file for writing (mode 0666). For more control, use `os.OpenFile` with custom flags and permissions.

3. **How does `bufio` improve I/O performance?**
   - `bufio.Reader` and `bufio.Writer` wrap I/O with an internal buffer, reducing the number of system calls. `bufio.Scanner` provides convenient line-by-line or token-by-token reading.

4. **What is `io.Copy` and when would you use it?**
   - `io.Copy(dst, src)` copies from Reader to Writer until EOF. It uses an internal 32KB buffer. Use `io.CopyBuffer` for a custom buffer size. Efficient for streaming large data without loading into memory.

5. **How do you read an entire file into memory?**
   - `os.ReadFile(path)` reads the entire file into `[]byte`. For older Go: `ioutil.ReadFile()` (deprecated). For controlled reading, open with `os.Open` then use `io.ReadAll`.

6. **What is `//go:embed` used for?**
   - Embeds files into the Go binary at compile time. Use `embed.FS` for dirs or `string`/`[]byte` for single files. Useful for bundling templates, static assets, or config files.

7. **How do you handle file paths portably across OS?**
   - Use `filepath.Join()`, `filepath.Abs()`, `filepath.Dir()`, `filepath.Base()` from `path/filepath`. These handle OS-specific separators. The `path` package is for URL/slash-separated paths only.

8. **What is `defer file.Close()` and why is it important?**
   - `defer` ensures `Close()` is called when the function returns, preventing resource leaks. Always check the error from `Close()` on writes: `defer func() { err = file.Close() }()`.

9. **How does `io.Pipe` work?**
   - Creates a synchronous in-memory pipe: `pr, pw := io.Pipe()`. Writes to `pw` block until read from `pr`. Useful for connecting a writer to a reader without buffering (e.g., streaming HTTP request body).

10. **What is `io.TeeReader` used for?**
    - `io.TeeReader(r, w)` returns a Reader that writes everything read from `r` to `w`. Useful for logging, hashing, or duplicating an I/O stream while consuming it.
