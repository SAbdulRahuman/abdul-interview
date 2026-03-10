# Chapter 20 — Encoding & Serialization

Go's standard library provides robust encoding support for multiple formats. The most commonly used is `encoding/json` for web APIs. All encoders use struct tags for field mapping.

```
┌──────────────────────────────────────────────────────────┐
│           Encoding Formats Comparison                   │
│                                                          │
│  Format    │ Package          │ Human    │ Use Case      │
│  ──────────┼──────────────────┼──────────┼──────────     │
│  JSON      │ encoding/json    │ ✓ Yes    │ Web APIs      │
│  XML       │ encoding/xml     │ ✓ Yes    │ SOAP, configs │
│  CSV       │ encoding/csv     │ ✓ Yes    │ Data export   │
│  Gob       │ encoding/gob     │ ✗ No     │ Go-to-Go RPC  │
│  Protobuf  │ google.golang.org│ ✗ No     │ gRPC, perf    │
│                                                          │
│  JSON Struct Tags:                                      │
│  ┌──────────────────────────────────────────────────┐    │
│  │ `json:"name"`         → field name in JSON       │   │
│  │ `json:"name,omitempty"` → omit if zero value     │   │
│  │ `json:"-"`            → always exclude           │    │
│  │ `json:",string"`      → encode int as JSON string│   │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  Marshal: Go struct ──► []byte (JSON/XML/etc.)          │
│  Unmarshal: []byte ──► Go struct                        │
│  Encoder: Go struct ──► io.Writer (streaming)           │
│  Decoder: io.Reader ──► Go struct (streaming)           │
└──────────────────────────────────────────────────────────┘
```

## encoding/json

### Marshal and Unmarshal

**Tutorial: JSON Marshal and Unmarshal with Struct Tags**

This example shows the two core JSON operations: `Marshal` (struct → JSON bytes) and `Unmarshal` (JSON bytes → struct). Struct tags like `json:"name"` control the JSON field names, `omitempty` skips zero-valued fields, and `json:"-"` permanently excludes a field (useful for secrets). Notice how `Password` never appears in output despite being set on the struct.

```
┌──────────────────────────────────────────────────────────┐
│                   json.Marshal                           │
│                                                          │
│  User struct (Go memory)         JSON []byte             │
│  ┌─────────────────────┐         ┌──────────────────┐    │
│  │ Name: "Alice"       │──tag──► │ "name":"Alice"   │    │
│  │ Email: "alice@.."   │──tag──► │ "email":"alice@"│    │
│  │ Age: 30             │──tag──► │ "age":30         │    │
│  │ Password: "secret"  │──"-"──► │  (excluded)      │    │
│  │ Tags: ["admin"]     │──tag──► │ "tags":["admin"]│    │
│  └─────────────────────┘         └──────────────────┘    │
│                                                          │
│                  json.Unmarshal                           │
│  JSON []byte ────────────────────► User struct           │
│  (fields not in struct are silently ignored)              │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    Name     string   `json:"name"`
    Email    string   `json:"email,omitempty"` // omit if empty
    Age      int      `json:"age"`
    Password string   `json:"-"`               // always omit
    Tags     []string `json:"tags,omitempty"`
}

func main() {
    // Marshal — struct to JSON
    user := User{
        Name:     "Alice",
        Email:    "alice@test.com",
        Age:      30,
        Password: "secret123",
        Tags:     []string{"admin", "user"},
    }

    data, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println(string(data))
    // {"name":"Alice","email":"alice@test.com","age":30,"tags":["admin","user"]}
    // Note: Password is omitted (json:"-")

    // Pretty print
    prettyData, _ := json.MarshalIndent(user, "", "  ")
    fmt.Println(string(prettyData))

    // Unmarshal — JSON to struct
    jsonStr := `{"name":"Bob","age":25,"tags":["developer"]}`
    var user2 User
    err = json.Unmarshal([]byte(jsonStr), &user2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("%+v\n", user2)
    // {Name:Bob Email: Age:25 Password: Tags:[developer]}
}
```

### Custom Marshaling

**Tutorial: Implementing the json.Marshaler and json.Unmarshaler Interfaces**

When the default JSON encoding isn't sufficient—such as formatting `time.Time` as `"2006-01-02"` instead of the default RFC 3339—you can implement `MarshalJSON()` and `UnmarshalJSON()` on your type. The alias-type trick (`type Alias Event`) prevents infinite recursion when calling `json.Marshal` inside your custom marshaler. Watch how the auxiliary struct overlays the `Date` field with a string while keeping all other fields via embedding.

```
┌─────────────────────────────────────────────────────────────┐
│           Custom Marshal Flow (Alias Trick)                 │
│                                                             │
│  Event.MarshalJSON() called by json.Marshal                 │
│  ┌───────────────────┐                                      │
│  │ 1. type Alias Event│◄── prevents recursion               │
│  │ 2. Build aux struct│                                     │
│  │    ┌──────────────────────────┐                          │
│  │    │ Alias (embedded)        │  ◄── all fields except   │
│  │    │ Date string (override)  │  ◄── Date as string      │
│  │    └──────────────────────────┘                          │
│  │ 3. json.Marshal(aux)          │                          │
│  └───────────────┬───────────────┘                          │
│                  ▼                                          │
│  {"name":"Conference","date":"2026-03-09"}                 │
└─────────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

type Event struct {
    Name string    `json:"name"`
    Date time.Time `json:"date"`
}

// Custom marshal — format time differently
func (e Event) MarshalJSON() ([]byte, error) {
    type Alias Event
    return json.Marshal(&struct {
        Alias
        Date string `json:"date"`
    }{
        Alias: Alias(e),
        Date:  e.Date.Format("2006-01-02"),
    })
}

// Custom unmarshal
func (e *Event) UnmarshalJSON(data []byte) error {
    type Alias Event
    aux := &struct {
        *Alias
        Date string `json:"date"`
    }{Alias: (*Alias)(e)}

    if err := json.Unmarshal(data, aux); err != nil {
        return err
    }

    t, err := time.Parse("2006-01-02", aux.Date)
    if err != nil {
        return err
    }
    e.Date = t
    return nil
}

func main() {
    event := Event{Name: "Conference", Date: time.Now()}
    data, _ := json.Marshal(event)
    fmt.Println(string(data))
    // {"name":"Conference","date":"2026-03-09"}
}
```

### Streaming and Dynamic JSON

**Tutorial: Streaming Encoders/Decoders, RawMessage, and Dynamic JSON**

Instead of buffering entire payloads in memory with `Marshal`/`Unmarshal`, `json.NewEncoder` and `json.NewDecoder` stream directly to/from `io.Writer`/`io.Reader`—ideal for HTTP handlers and large files. `json.RawMessage` lets you defer parsing of a sub-field until you know its type. For fully dynamic structures, `map[string]any` works as a schemaless container.

```
┌───────────────────────────────────────────────────────────┐
│           Streaming vs In-Memory                          │
│                                                           │
│  In-Memory (Marshal/Unmarshal):                           │
│    struct ──► []byte ──► io.Writer                        │
│    io.Reader ──► []byte ──► struct                        │
│                  ▲ full copy in RAM                        │
│                                                           │
│  Streaming (Encoder/Decoder):                             │
│    struct ──────────────────► io.Writer (os.Stdout, HTTP) │
│    io.Reader ────────────────► struct   (no []byte copy)  │
│                                                           │
│  json.RawMessage — delayed parsing:                       │
│  ┌──────────────────────────────────┐                     │
│  │ {"type":"config", "data": ... } │                     │
│  │                          ▲       │                     │
│  │              kept as raw bytes   │                     │
│  │         until you decide how to  │                     │
│  │              unmarshal it        │                     │
│  └──────────────────────────────────┘                     │
└───────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "strings"
)

func main() {
    // Streaming with Encoder/Decoder
    encoder := json.NewEncoder(os.Stdout)
    encoder.SetIndent("", "  ")
    encoder.Encode(map[string]int{"a": 1, "b": 2})

    // Decoder from reader
    jsonReader := strings.NewReader(`{"name":"Alice","age":30}`)
    decoder := json.NewDecoder(jsonReader)
    var result map[string]any
    decoder.Decode(&result)
    fmt.Println(result) // map[age:30 name:Alice]

    // json.RawMessage — delay parsing
    raw := json.RawMessage(`{"nested": true}`)
    envelope := struct {
        Type string          `json:"type"`
        Data json.RawMessage `json:"data"`
    }{
        Type: "config",
        Data: raw,
    }
    data, _ := json.Marshal(envelope)
    fmt.Println(string(data))
    // {"type":"config","data":{"nested":true}}

    // Dynamic JSON with map[string]any
    dynamic := map[string]any{
        "name":   "Alice",
        "age":    30,
        "scores": []int{90, 85, 95},
        "address": map[string]any{
            "city":    "NYC",
            "country": "USA",
        },
    }
    output, _ := json.MarshalIndent(dynamic, "", "  ")
    fmt.Println(string(output))
}
```

---

## encoding/xml

**Tutorial: XML Marshaling with Struct Tags for Attributes and Nesting**

Go's `encoding/xml` mirrors the JSON API but adds XML-specific struct tags: `xml:"name,attr"` maps a field to an XML attribute, and `xml:"parent>child"` produces nested elements without declaring an intermediate struct. The special `XMLName xml.Name` field controls the root element name. Unmarshal reverses the process, parsing XML back into structs.

```
┌──────────────────────────────────────────────────────────┐
│            XML Struct Tag Mapping                        │
│                                                          │
│  Go Struct                     XML Output                │
│  ┌──────────────────────┐      ┌──────────────────────┐  │
│  │ XMLName: "person"    │─────►│ <person age="30">    │  │
│  │ Name: "Alice"        │─────►│   <name>Alice</name> │  │
│  │ Age: 30  (attr)      │──┘   │   <contact>          │  │
│  │ Email: "alice@.."    │─────►│     <email>alice@..  │  │
│  │  (contact>email)     │      │     </email>         │  │
│  └──────────────────────┘      │   </contact>         │  │
│                                │ </person>            │  │
│                                └──────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/xml"
    "fmt"
)

type Person struct {
    XMLName xml.Name `xml:"person"`
    Name    string   `xml:"name"`
    Age     int      `xml:"age,attr"`
    Email   string   `xml:"contact>email"` // nested element
}

func main() {
    p := Person{Name: "Alice", Age: 30, Email: "alice@test.com"}

    data, _ := xml.MarshalIndent(p, "", "  ")
    fmt.Println(string(data))
    // <person age="30">
    //   <name>Alice</name>
    //   <contact>
    //     <email>alice@test.com</email>
    //   </contact>
    // </person>

    // Unmarshal
    xmlData := `<person age="25"><name>Bob</name><contact><email>bob@test.com</email></contact></person>`
    var p2 Person
    xml.Unmarshal([]byte(xmlData), &p2)
    fmt.Printf("%+v\n", p2)
}
```

---

## encoding/csv

**Tutorial: Reading and Writing CSV Data with encoding/csv**

The `csv.Writer` writes records (string slices) into comma-separated rows, handling quoting and escaping automatically. Always call `Flush()` after writing to ensure buffered data is emitted. The `csv.Reader` parses CSV back into `[][]string` via `ReadAll()` (all at once) or `Read()` (one record at a time for large files). Both reader and writer support custom delimiters via the `Comma` field.

```
┌──────────────────────────────────────────────────────────┐
│              CSV Write / Read Flow                       │
│                                                          │
│  Write:                                                  │
│  []string{"Name","Age","City"}                           │
│       │                                                  │
│       ▼                                                  │
│  csv.Writer ──► strings.Builder                          │
│       │              │                                   │
│  .Write(record)      │    Name,Age,City                  │
│  .Write(record)      │    Alice,30,NYC                   │
│  .Flush()            │    Bob,25,LA                      │
│                      ▼                                   │
│  Read:          csv.Reader                               │
│                      │                                   │
│                .ReadAll()                                │
│                      ▼                                   │
│              [][]string (rows × cols)                    │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "encoding/csv"
    "fmt"
    "strings"
)

func main() {
    // Write CSV
    var buf strings.Builder
    writer := csv.NewWriter(&buf)
    writer.Write([]string{"Name", "Age", "City"})
    writer.Write([]string{"Alice", "30", "NYC"})
    writer.Write([]string{"Bob", "25", "LA"})
    writer.Flush()
    fmt.Print(buf.String())

    // Read CSV
    reader := csv.NewReader(strings.NewReader(buf.String()))
    records, _ := reader.ReadAll()
    for _, record := range records {
        fmt.Println(record)
    }
    // [Name Age City]
    // [Alice 30 NYC]
    // [Bob 25 LA]
}
```

---

## encoding/gob

**Tutorial: Go-Native Binary Encoding with encoding/gob**

Gob is a Go-specific binary format designed for efficient Go-to-Go communication (e.g., `net/rpc`). It is self-describing—type information is transmitted with the data—so the encoder and decoder don't need identical struct definitions, just compatible ones. Gob is faster and more compact than JSON for Go-only systems, but it is not cross-language compatible. Encoding and decoding both operate on `io.Writer`/`io.Reader` via `gob.NewEncoder` and `gob.NewDecoder`.

```
┌──────────────────────────────────────────────────────────┐
│               Gob Encode / Decode                        │
│                                                          │
│  Go Struct          bytes.Buffer           Go Struct     │
│  ┌──────────┐      ┌────────────┐      ┌──────────┐     │
│  │ Sender:  │      │ binary gob │      │ Sender:  │     │
│  │  "Alice" │─Enc─►│ [type info │─Dec─►│  "Alice" │     │
│  │ Content: │      │  + field   │      │ Content: │     │
│  │  "Hello"│      │  values]   │      │  "Hello"│     │
│  │ ID: 1   │      │            │      │ ID: 1   │     │
│  └──────────┘      └────────────┘      └──────────┘     │
│                     compact binary                       │
│                  (not human-readable)                     │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "bytes"
    "encoding/gob"
    "fmt"
)

type Message struct {
    Sender  string
    Content string
    ID      int
}

func main() {
    // gob is Go-specific binary encoding
    // Very efficient for Go-to-Go communication

    // Encode
    var buf bytes.Buffer
    encoder := gob.NewEncoder(&buf)
    msg := Message{Sender: "Alice", Content: "Hello!", ID: 1}
    encoder.Encode(msg)
    fmt.Printf("Encoded %d bytes\n", buf.Len())

    // Decode
    decoder := gob.NewDecoder(&buf)
    var decoded Message
    decoder.Decode(&decoded)
    fmt.Printf("Decoded: %+v\n", decoded)
    // Decoded: {Sender:Alice Content:Hello! ID:1}
}
```

---

## encoding/binary

**Tutorial: Low-Level Binary Encoding with Byte Order Control**

The `encoding/binary` package reads and writes fixed-size numeric values (`uint32`, `float64`, etc.) with explicit byte order—`BigEndian` (network byte order, MSB first) or `LittleEndian` (x86 native, LSB first). This is essential for binary protocols, file formats, and network packets where exact byte layout matters. `binary.Write` serializes a value into an `io.Writer`, and `binary.Read` deserializes it back.

```
┌──────────────────────────────────────────────────────────┐
│         binary.Write — Big Endian Layout                 │
│                                                          │
│  uint32(42)   = 0x0000002A                               │
│  ┌──────┬──────┬──────┬──────┐                           │
│  │ 0x00 │ 0x00 │ 0x00 │ 0x2A │  ◄── Big Endian (MSB)    │
│  └──────┴──────┴──────┴──────┘                           │
│  byte[0]                byte[3]                          │
│                                                          │
│  float64(3.14) = 8 bytes IEEE 754                        │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬────┐ │
│  │ 0x40 │ 0x09 │ 0x1E │ 0xB8 │ 0x51 │ 0xEB │ 0x85 │0x1F│ │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴────┘ │
│                                                          │
│  binary.Read reverses the process:                       │
│  []byte ──► reader ──► binary.Read ──► uint32 / float64  │
└──────────────────────────────────────────────────────────┘
```

```go
package main

import (
    "bytes"
    "encoding/binary"
    "fmt"
)

func main() {
    // Write binary data with specified byte order
    var buf bytes.Buffer

    // Big Endian (network byte order)
    binary.Write(&buf, binary.BigEndian, uint32(42))
    binary.Write(&buf, binary.BigEndian, float64(3.14))

    fmt.Printf("Binary data: % x\n", buf.Bytes())

    // Read binary data back
    var num uint32
    var f float64
    reader := bytes.NewReader(buf.Bytes())
    binary.Read(reader, binary.BigEndian, &num)
    binary.Read(reader, binary.BigEndian, &f)

    fmt.Println("Number:", num) // 42
    fmt.Println("Float:", f)   // 3.14
}
```

---

## Interview Questions

1. **How does JSON encoding/decoding work in Go?**
   - `json.Marshal(v)` converts to `[]byte`, `json.Unmarshal(data, &v)` parses JSON. For streams, use `json.NewEncoder(w)` / `json.NewDecoder(r)`. Struct tags (`json:"name,omitempty"`) control field mapping.

2. **What is the difference between `json.Marshal` and `json.NewEncoder`?**
   - `Marshal` produces a `[]byte` in memory. `Encoder` writes directly to an `io.Writer` (streaming). Use `Encoder` for HTTP responses and file writing; `Marshal` for in-memory operations.

3. **How do you handle unknown JSON fields?**
   - Use `map[string]interface{}` or `map[string]json.RawMessage` for dynamic JSON. `json.RawMessage` delays parsing. `Decoder.DisallowUnknownFields()` rejects unexpected fields.

4. **What does `omitempty` do in struct tags?**
   - Omits the field from JSON output if it has its zero value (0, "", nil, false, empty slice/map). Note: `0` and `false` are omitted too, which can be undesirable—use pointer types to distinguish.

5. **How do you implement custom JSON marshaling?**
   - Implement `json.Marshaler` (`MarshalJSON() ([]byte, error)`) and `json.Unmarshaler` (`UnmarshalJSON([]byte) error`). Common for custom date formats, enums, or complex serialization logic.

6. **What is `encoding/gob` used for?**
   - Go-specific binary encoding for Go-to-Go communication. More efficient than JSON. Used by `net/rpc`. Self-describing format but not cross-language compatible.

7. **How do you encode/decode XML in Go?**
   - `encoding/xml` works similarly to JSON: `xml.Marshal`/`xml.Unmarshal` with struct tags like `xml:"name,attr"`. Supports attributes, namespaces, and nested elements.

8. **What is Protocol Buffers (protobuf) and how is it used in Go?**
   - A language-neutral binary serialization format by Google. Define schemas in `.proto` files, generate Go code with `protoc-gen-go`. Smaller and faster than JSON. Used with gRPC.

9. **How do you handle CSV files in Go?**
   - `encoding/csv` provides `csv.NewReader(r)` and `csv.NewWriter(w)`. `ReadAll()` reads everything, `Read()` reads one record at a time. Handles quoting, custom delimiters, and comments.

10. **What is `encoding.TextMarshaler` / `encoding.TextUnmarshaler`?**
    - Interfaces for custom text-based encoding. Types that implement these are automatically used by JSON (as keys in maps) and other text-based encoders. Example: `time.Time` implements `TextMarshaler`.
