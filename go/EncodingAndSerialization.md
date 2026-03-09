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
