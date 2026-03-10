# Chapter 37 — gRPC & Protocol Buffers in Go

gRPC is a high-performance, open-source RPC (Remote Procedure Call) framework originally developed by Google. It uses **Protocol Buffers (Protobuf)** as its interface definition language and serialization format. Go is a first-class citizen in the gRPC ecosystem — many gRPC tools and the etcd, Kubernetes, and CockroachDB projects use gRPC extensively.

## Why gRPC?

```
┌───────────────────────────────────────────────────────────────────┐
│  REST vs gRPC Comparison                                         │
│                                                                   │
│  Aspect          │ REST (JSON/HTTP)       │ gRPC (Protobuf/HTTP2) │
│  ────────────────┼────────────────────────┼─────────────────────  │
│  Transport       │ HTTP/1.1 (text)        │ HTTP/2 (binary)       │
│  Serialization   │ JSON (text, ~10x)      │ Protobuf (binary, 1x) │
│  Contract        │ OpenAPI (optional)     │ .proto (required)     │
│  Code generation │ Optional               │ Built-in              │
│  Streaming       │ Not native (WebSocket) │ Native (4 patterns)   │
│  Browser support │ Native                 │ Needs grpc-web proxy  │
│  Human readable  │ Yes (JSON)             │ No (binary)           │
│  Latency         │ Higher                 │ Lower (~10x faster)   │
│  Type safety     │ Runtime validation     │ Compile-time types    │
│                                                                   │
│  When to use gRPC:                                               │
│  ✓ Microservice-to-microservice communication                    │
│  ✓ High-throughput, low-latency internal APIs                    │
│  ✓ Streaming data (real-time feeds, file transfers)              │
│  ✓ Polyglot environments (Go ↔ Java ↔ Python ↔ Rust)            │
│                                                                   │
│  When to use REST:                                               │
│  ✓ Public-facing APIs (browser consumption)                      │
│  ✓ Simple CRUD operations                                        │
│  ✓ When human readability matters                                │
│  ✓ Third-party integrations (most SDKs expect REST)              │
└───────────────────────────────────────────────────────────────────┘
```

---

## Protocol Buffers (Protobuf) — The Foundation

### Proto3 Syntax

```protobuf
// user.proto
syntax = "proto3";

package userservice;

// Go-specific options
option go_package = "github.com/myapp/proto/userpb";

// Message definition — like a struct
message User {
    // field_type field_name = field_number;
    string id = 1;             // field number 1
    string email = 2;          // field number 2
    string name = 3;           // field number 3
    int32 age = 4;             // field number 4
    UserRole role = 5;         // enum field
    Address address = 6;       // nested message
    repeated string tags = 7;  // list/slice
    optional string bio = 8;   // explicitly optional (has presence)

    // Timestamp from well-known types
    google.protobuf.Timestamp created_at = 9;
}

// Enum definition
enum UserRole {
    USER_ROLE_UNSPECIFIED = 0;  // zero value MUST be first
    USER_ROLE_ADMIN = 1;
    USER_ROLE_EDITOR = 2;
    USER_ROLE_VIEWER = 3;
}

// Nested message
message Address {
    string street = 1;
    string city = 2;
    string country = 3;
    string zip_code = 4;
}

// Oneof — only one field can be set
message Notification {
    string id = 1;
    oneof channel {
        EmailNotification email = 2;
        SMSNotification sms = 3;
        PushNotification push = 4;
    }
}

message EmailNotification { string to = 1; string subject = 2; }
message SMSNotification { string phone = 1; }
message PushNotification { string device_id = 1; string title = 2; }

// Map fields
message Config {
    map<string, string> settings = 1;   // map[string]string in Go
    map<string, int32> limits = 2;      // map[string]int32 in Go
}
```

```
┌────────────────────────────────────────────────────────────────┐
│  Protobuf Field Numbers — Critical Rules                      │
│                                                                │
│  Field numbers are the WIRE IDENTITY of each field.           │
│  They are encoded in the binary format, NOT the field name.   │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Numbers 1-15:   1 byte encoding  → use for frequent    │  │
│  │  Numbers 16-2047: 2 byte encoding → use for less common │  │
│  │  Numbers 2048+:   3+ bytes        → rarely needed       │  │
│  │                                                          │  │
│  │  Reserved: 19000-19999 (protobuf internal)               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  NEVER reuse or change field numbers after deployment!        │
│  This breaks backward compatibility.                          │
│                                                                │
│  Proto3 default values (zero values — not sent on wire):      │
│  string → ""    int32 → 0    bool → false                     │
│  bytes → empty  enum → 0     message → nil                    │
│                                                                │
│  optional keyword: adds has_* presence tracking               │
│  Without optional: can't distinguish "zero" from "not set"    │
└────────────────────────────────────────────────────────────────┘
```

### Proto3 Scalar Types → Go Types

```
┌──────────────────────────────────────────────────────────────┐
│  Proto3 Type   │ Go Type    │ Default │ Notes               │
│  ──────────────┼────────────┼─────────┼───────────────────   │
│  double        │ float64    │ 0       │                     │
│  float         │ float32    │ 0       │                     │
│  int32         │ int32      │ 0       │ variable-length     │
│  int64         │ int64      │ 0       │ variable-length     │
│  uint32        │ uint32     │ 0       │ variable-length     │
│  uint64        │ uint64     │ 0       │ variable-length     │
│  sint32        │ int32      │ 0       │ zigzag (neg-heavy)  │
│  sint64        │ int64      │ 0       │ zigzag (neg-heavy)  │
│  fixed32       │ uint32     │ 0       │ always 4 bytes      │
│  fixed64       │ uint64     │ 0       │ always 8 bytes      │
│  sfixed32      │ int32      │ 0       │ always 4 bytes      │
│  sfixed64      │ int64      │ 0       │ always 8 bytes      │
│  bool          │ bool       │ false   │                     │
│  string        │ string     │ ""      │ UTF-8               │
│  bytes         │ []byte     │ nil     │ arbitrary bytes     │
│                                                              │
│  When to use which int type:                                 │
│  • int32/int64: general purpose (most common)                │
│  • sint32/sint64: frequently negative numbers                │
│  • fixed32/fixed64: frequently > 2^28 or > 2^56             │
│  • uint32/uint64: never negative                             │
└──────────────────────────────────────────────────────────────┘
```

---

## Code Generation & Tooling

```
┌────────────────────────────────────────────────────────────────┐
│  Protobuf → Go Code Generation Pipeline                       │
│                                                                │
│  .proto file                                                   │
│       │                                                        │
│       ▼                                                        │
│  ┌──────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  protoc   │───▶│ protoc-gen-go    │───▶│ *.pb.go          │  │
│  │ compiler  │    │ (message types)  │    │ (structs, enums) │  │
│  └──────────┘    └──────────────────┘    └──────────────────┘  │
│       │                                                        │
│       │          ┌──────────────────┐    ┌──────────────────┐  │
│       └─────────▶│protoc-gen-go-grpc│───▶│ *_grpc.pb.go     │  │
│                  │ (service stubs)  │    │ (client/server)  │  │
│                  └──────────────────┘    └──────────────────┘  │
│                                                                │
│  Two separate plugins — two separate output files.             │
└────────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# Install protobuf compiler
# macOS:
brew install protobuf

# Linux:
apt install -y protobuf-compiler

# Install Go plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Verify
protoc --version
which protoc-gen-go
which protoc-gen-go-grpc
```

### Generation Command

```bash
# Generate Go code from .proto files
protoc \
    --go_out=. \
    --go_opt=paths=source_relative \
    --go-grpc_out=. \
    --go-grpc_opt=paths=source_relative \
    proto/user.proto

# Generates:
#   proto/user.pb.go       ← message types, enums
#   proto/user_grpc.pb.go  ← gRPC client/server stubs
```

```
┌──────────────────────────────────────────────────────────────┐
│  Generated Code: What You Get                                │
│                                                              │
│  From message User { ... }:                                  │
│  ┌──────────────────────────────────────────────────┐        │
│  │  type User struct {                              │        │
│  │      Id    string    `protobuf:"bytes,1,opt...`  │        │
│  │      Email string    `protobuf:"bytes,2,opt...`  │        │
│  │      Name  string    `protobuf:"bytes,3,opt...`  │        │
│  │      Age   int32     `protobuf:"varint,4,..."`   │        │
│  │      Role  UserRole  `protobuf:"varint,5,..."`   │        │
│  │      Tags  []string  `protobuf:"bytes,7,rep...`  │        │
│  │      Bio   *string   `protobuf:"bytes,8,opt...`  │        │
│  │  }                                               │        │
│  │                                                  │        │
│  │  func (*User) ProtoReflect()                     │        │
│  │  func (*User) Reset()                            │        │
│  │  func (*User) String() string                    │        │
│  │  func (*User) GetId() string      ← getter       │        │
│  │  func (*User) GetEmail() string   ← getter       │        │
│  │  func (*User) GetBio() string     ← nil-safe     │        │
│  └──────────────────────────────────────────────────┘        │
│                                                              │
│  Getters are nil-safe: user.GetBio() returns ""              │
│  even if user is nil. This prevents nil pointer panics.      │
│                                                              │
│  optional fields become pointers (*string).                  │
│  Use user.Bio != nil to check presence.                      │
└──────────────────────────────────────────────────────────────┘
```

---

## Protobuf Serialization in Go

```go
package main

import (
    "fmt"
    "log"

    "google.golang.org/protobuf/proto"
    userpb "github.com/myapp/proto/userpb"
)

func main() {
    // Create a protobuf message
    user := &userpb.User{
        Id:    "user-123",
        Email: "alice@example.com",
        Name:  "Alice",
        Age:   30,
        Role:  userpb.UserRole_USER_ROLE_ADMIN,
        Tags:  []string{"go", "grpc"},
    }

    // Marshal (serialize to bytes)
    data, err := proto.Marshal(user)
    if err != nil {
        log.Fatalf("marshal error: %v", err)
    }
    fmt.Printf("Serialized: %d bytes\n", len(data)) // ~50 bytes vs ~150 JSON

    // Unmarshal (deserialize from bytes)
    user2 := &userpb.User{}
    if err := proto.Unmarshal(data, user2); err != nil {
        log.Fatalf("unmarshal error: %v", err)
    }
    fmt.Printf("Deserialized: %s (%s)\n", user2.GetName(), user2.GetEmail())

    // Compare messages
    if proto.Equal(user, user2) {
        fmt.Println("Messages are equal")
    }

    // Clone (deep copy)
    user3 := proto.Clone(user).(*userpb.User)
    user3.Name = "Bob" // doesn't affect user
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  Protobuf Binary Encoding (simplified)                       │
│                                                              │
│  User { id: "abc", age: 30 }                                │
│                                                              │
│  Wire format:                                                │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐          │
│  │ 0x0A │ 0x03 │  a   │  b   │  c   │ 0x20 │ 0x1E │          │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘          │
│     │      │    └─────────┘     │      │                     │
│     │      │    "abc" UTF-8     │      30 (varint)           │
│     │      3 bytes              field 4 (age), varint type   │
│     field 1 (id), length-delimited type                      │
│                                                              │
│  Key insight: field NAMES not in binary — only field NUMBERS │
│  This is why field numbers must NEVER change.                │
│                                                              │
│  Wire types:                                                 │
│  0 = Varint (int32, bool, enum)                              │
│  1 = 64-bit (fixed64, double)                                │
│  2 = Length-delimited (string, bytes, messages, repeated)    │
│  5 = 32-bit (fixed32, float)                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Backward Compatibility Rules

```
┌──────────────────────────────────────────────────────────────────┐
│  Protobuf Backward Compatibility                                 │
│                                                                  │
│  ✅ SAFE changes (won't break existing clients/servers):         │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  • Add new fields (with new field numbers)                │   │
│  │  • Remove fields (old number not reused)                  │   │
│  │  • Rename fields (wire uses numbers, not names)           │   │
│  │  • Add new enum values                                    │   │
│  │  • Add new RPC methods to a service                       │   │
│  │  • Change int32 ↔ int64 (compatible encoding)             │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ❌ BREAKING changes (will cause data corruption/crashes):       │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │  • Change field numbers                                   │   │
│  │  • Change field types (incompatible wire type)            │   │
│  │  • Reuse field numbers of deleted fields                  │   │
│  │  • Change enum value numbers                              │   │
│  │  • Remove/rename a service or RPC method                  │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Best practice: Use `reserved` for deleted fields                │
│  message User {                                                  │
│      reserved 6, 9 to 11;              // reserved numbers       │
│      reserved "phone", "address";      // reserved names         │
│      // protoc will error if anyone tries to use these           │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## gRPC Service Definition

```protobuf
// service.proto
syntax = "proto3";
package userservice;
option go_package = "github.com/myapp/proto/userpb";

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";

// Service definition
service UserService {
    // Unary RPC — one request, one response
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
    rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
    rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);

    // Server streaming — one request, stream of responses
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client streaming — stream of requests, one response
    rpc UploadUsers(stream User) returns (UploadSummary);

    // Bidirectional streaming — stream both directions
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

// Request/Response messages
message GetUserRequest {
    string id = 1;
}

message GetUserResponse {
    User user = 1;
}

message CreateUserRequest {
    string email = 1;
    string name = 2;
    UserRole role = 3;
}

message CreateUserResponse {
    User user = 1;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
    string filter = 3;  // e.g., "role=ADMIN"
}

message UpdateUserRequest {
    string id = 1;
    string name = 2;
    string email = 3;
    // Use field mask to specify which fields to update
    google.protobuf.FieldMask update_mask = 4;
}

message UpdateUserResponse {
    User user = 1;
}

message DeleteUserRequest {
    string id = 1;
}

message UploadSummary {
    int32 total_received = 1;
    int32 total_created = 2;
    int32 total_failed = 3;
}

message ChatMessage {
    string sender = 1;
    string text = 2;
    google.protobuf.Timestamp sent_at = 3;
}
```

```
┌────────────────────────────────────────────────────────────────────┐
│  4 gRPC Communication Patterns                                     │
│                                                                    │
│  1. Unary RPC (most common)                                        │
│     Client ──── Request ────▶ Server                               │
│     Client ◀─── Response ─── Server                                │
│     Like a normal function call.                                   │
│                                                                    │
│  2. Server Streaming                                               │
│     Client ──── Request ────▶ Server                               │
│     Client ◀─── Response 1 ─ Server                               │
│     Client ◀─── Response 2 ─ Server                               │
│     Client ◀─── Response N ─ Server                               │
│     Use: list results, real-time feeds, log tailing.               │
│                                                                    │
│  3. Client Streaming                                               │
│     Client ──── Request 1 ──▶ Server                               │
│     Client ──── Request 2 ──▶ Server                               │
│     Client ──── Request N ──▶ Server                               │
│     Client ◀─── Response ─── Server                                │
│     Use: file upload, batch operations.                            │
│                                                                    │
│  4. Bidirectional Streaming                                        │
│     Client ──── Request 1 ──▶ Server                               │
│     Client ◀─── Response 1 ─ Server                                │
│     Client ──── Request 2 ──▶ Server                               │
│     Client ◀─── Response 2 ─ Server                                │
│     Independent streams, can interleave.                           │
│     Use: chat, game state sync, live dashboards.                   │
└────────────────────────────────────────────────────────────────────┘
```

---

## gRPC Server Implementation

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net"
    "sync"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/emptypb"

    userpb "github.com/myapp/proto/userpb"
)

// server implements the generated UserServiceServer interface.
type server struct {
    userpb.UnimplementedUserServiceServer // embed for forward compatibility
    mu    sync.RWMutex
    users map[string]*userpb.User
}

func newServer() *server {
    return &server{users: make(map[string]*userpb.User)}
}

// Unary RPC — GetUser
func (s *server) GetUser(ctx context.Context, req *userpb.GetUserRequest) (*userpb.GetUserResponse, error) {
    // Always validate input
    if req.GetId() == "" {
        return nil, status.Errorf(codes.InvalidArgument, "user id is required")
    }

    s.mu.RLock()
    user, ok := s.users[req.GetId()]
    s.mu.RUnlock()

    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %q not found", req.GetId())
    }

    return &userpb.GetUserResponse{User: user}, nil
}

// Unary RPC — CreateUser
func (s *server) CreateUser(ctx context.Context, req *userpb.CreateUserRequest) (*userpb.CreateUserResponse, error) {
    if req.GetEmail() == "" {
        return nil, status.Errorf(codes.InvalidArgument, "email is required")
    }

    user := &userpb.User{
        Id:    fmt.Sprintf("user-%d", len(s.users)+1),
        Email: req.GetEmail(),
        Name:  req.GetName(),
        Role:  req.GetRole(),
    }

    s.mu.Lock()
    s.users[user.Id] = user
    s.mu.Unlock()

    return &userpb.CreateUserResponse{User: user}, nil
}

// Unary RPC — DeleteUser
func (s *server) DeleteUser(ctx context.Context, req *userpb.DeleteUserRequest) (*emptypb.Empty, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.users[req.GetId()]; !ok {
        return nil, status.Errorf(codes.NotFound, "user %q not found", req.GetId())
    }
    delete(s.users, req.GetId())
    return &emptypb.Empty{}, nil
}

// Server Streaming — ListUsers
func (s *server) ListUsers(req *userpb.ListUsersRequest, stream userpb.UserService_ListUsersServer) error {
    s.mu.RLock()
    defer s.mu.RUnlock()

    for _, user := range s.users {
        // Check if client cancelled
        if err := stream.Context().Err(); err != nil {
            return status.Errorf(codes.Canceled, "client cancelled")
        }

        if err := stream.Send(user); err != nil {
            return err
        }
    }
    return nil
}

// Client Streaming — UploadUsers
func (s *server) UploadUsers(stream userpb.UserService_UploadUsersServer) error {
    var received, created, failed int32

    for {
        user, err := stream.Recv()
        if err != nil {
            // io.EOF means client is done sending
            if err.Error() == "EOF" {
                break
            }
            return err
        }
        received++

        s.mu.Lock()
        if _, exists := s.users[user.GetId()]; exists {
            failed++ // duplicate
        } else {
            s.users[user.GetId()] = user
            created++
        }
        s.mu.Unlock()
    }

    // Send single response after receiving all
    return stream.SendAndClose(&userpb.UploadSummary{
        TotalReceived: received,
        TotalCreated:  created,
        TotalFailed:   failed,
    })
}

// Bidirectional Streaming — Chat
func (s *server) Chat(stream userpb.UserService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err != nil {
            return err // includes io.EOF
        }

        // Echo back with server prefix
        reply := &userpb.ChatMessage{
            Sender: "server",
            Text:   fmt.Sprintf("Echo: %s", msg.GetText()),
        }
        if err := stream.Send(reply); err != nil {
            return err
        }
    }
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    grpcServer := grpc.NewServer()
    userpb.RegisterUserServiceServer(grpcServer, newServer())

    log.Println("gRPC server listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

```
┌────────────────────────────────────────────────────────────────┐
│  UnimplementedXxxServer — Forward Compatibility               │
│                                                                │
│  type server struct {                                          │
│      userpb.UnimplementedUserServiceServer  ← MUST embed this │
│  }                                                             │
│                                                                │
│  Why?                                                          │
│  • Generated interface has ALL methods                         │
│  • If you add a new RPC to .proto and regenerate,              │
│    without this embedding, your code won't compile             │
│  • UnimplementedXxxServer returns "Unimplemented" for          │
│    any method you haven't overridden                           │
│  • This is forward compatibility — you can add RPCs            │
│    without breaking existing server code                       │
│                                                                │
│  Alternative: UnsafeXxxServer (opt out of forward compat)      │
│  Use when you want compile errors for missing methods.         │
└────────────────────────────────────────────────────────────────┘
```

---

## gRPC Client Implementation

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    userpb "github.com/myapp/proto/userpb"
)

func main() {
    // Establish connection
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()

    client := userpb.NewUserServiceClient(conn)

    // Context with timeout — ALWAYS use timeouts
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // --- Unary RPC ---
    createResp, err := client.CreateUser(ctx, &userpb.CreateUserRequest{
        Email: "alice@example.com",
        Name:  "Alice",
        Role:  userpb.UserRole_USER_ROLE_ADMIN,
    })
    if err != nil {
        log.Fatalf("CreateUser failed: %v", err)
    }
    fmt.Printf("Created: %s (ID: %s)\n", createResp.User.GetName(), createResp.User.GetId())

    // --- Unary RPC ---
    getResp, err := client.GetUser(ctx, &userpb.GetUserRequest{
        Id: createResp.User.GetId(),
    })
    if err != nil {
        log.Fatalf("GetUser failed: %v", err)
    }
    fmt.Printf("Got: %s\n", getResp.User.GetName())

    // --- Server Streaming ---
    stream, err := client.ListUsers(ctx, &userpb.ListUsersRequest{})
    if err != nil {
        log.Fatalf("ListUsers failed: %v", err)
    }
    for {
        user, err := stream.Recv()
        if err == io.EOF {
            break // server done sending
        }
        if err != nil {
            log.Fatalf("stream error: %v", err)
        }
        fmt.Printf("User: %s (%s)\n", user.GetName(), user.GetEmail())
    }
}
```

```
┌────────────────────────────────────────────────────────────────┐
│  gRPC Client Best Practices                                    │
│                                                                │
│  1. Always use context with timeout/deadline                   │
│     ctx, cancel := context.WithTimeout(ctx, 5*time.Second)    │
│     defer cancel()                                             │
│                                                                │
│  2. Reuse connections — grpc.ClientConn is thread-safe         │
│     One conn per service, shared across goroutines.            │
│                                                                │
│  3. Connection lifecycle                                       │
│     ┌───────────┐                                              │
│     │   IDLE    │ ← initial state                              │
│     └─────┬─────┘                                              │
│           │ RPC call                                           │
│     ┌─────▼─────┐                                              │
│     │ CONNECTING│ ← establishing connection                    │
│     └─────┬─────┘                                              │
│           │ success                                            │
│     ┌─────▼─────┐                                              │
│     │   READY   │ ← can send RPCs                              │
│     └─────┬─────┘                                              │
│           │ failure                                             │
│     ┌─────▼─────────────┐                                      │
│     │ TRANSIENT_FAILURE │ ← auto-reconnect with backoff        │
│     └───────────────────┘                                      │
│                                                                │
│  4. Handle errors with gRPC status codes:                     │
│     st, ok := status.FromError(err)                            │
│     st.Code()    → codes.NotFound, codes.Internal, etc.       │
│     st.Message() → human-readable error message               │
└────────────────────────────────────────────────────────────────┘
```

---

## gRPC Error Handling — Status Codes

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Server-side: Return proper status errors
func (s *server) GetUser(ctx context.Context, req *userpb.GetUserRequest) (*userpb.GetUserResponse, error) {
    if req.GetId() == "" {
        return nil, status.Errorf(codes.InvalidArgument, "id is required")
    }

    user, ok := s.users[req.GetId()]
    if !ok {
        return nil, status.Errorf(codes.NotFound, "user %q not found", req.GetId())
    }

    return &userpb.GetUserResponse{User: user}, nil
}

// Client-side: Check status codes
resp, err := client.GetUser(ctx, req)
if err != nil {
    st, ok := status.FromError(err)
    if !ok {
        // Not a gRPC error
        log.Fatalf("unexpected error: %v", err)
    }

    switch st.Code() {
    case codes.NotFound:
        fmt.Printf("User not found: %s\n", st.Message())
    case codes.InvalidArgument:
        fmt.Printf("Bad request: %s\n", st.Message())
    case codes.Unavailable:
        fmt.Println("Service unavailable, retrying...")
    case codes.DeadlineExceeded:
        fmt.Println("Request timed out")
    default:
        fmt.Printf("RPC error: %s (%s)\n", st.Code(), st.Message())
    }
}
```

```
┌────────────────────────────────────────────────────────────────┐
│  gRPC Status Codes (commonly used)                             │
│                                                                │
│  Code               │ HTTP Equiv │ When to Use                │
│  ───────────────────┼────────────┼──────────────────────────   │
│  OK (0)             │ 200        │ Success                    │
│  Canceled (1)       │ 499        │ Client cancelled           │
│  InvalidArgument(3) │ 400        │ Bad request data           │
│  NotFound (5)       │ 404        │ Resource doesn't exist     │
│  AlreadyExists (6)  │ 409        │ Duplicate creation         │
│  PermissionDenied(7)│ 403        │ Auth ok, not authorized    │
│  Unauthenticated(16)│ 401        │ Missing/bad credentials    │
│  ResourceExhausted  │ 429        │ Rate limit, quota          │
│  FailedPrecondition │ 400        │ System not in right state  │
│  Unimplemented (12) │ 501        │ Method not implemented     │
│  Internal (13)      │ 500        │ Server bug                 │
│  Unavailable (14)   │ 503        │ Transient failure          │
│  DeadlineExceeded(4)│ 504        │ Timeout                    │
│                                                                │
│  Rule of thumb:                                                │
│  • Unavailable → client should retry (with backoff)            │
│  • Internal → client should NOT retry (server bug)             │
│  • DeadlineExceeded → might retry with longer timeout          │
│  • InvalidArgument → fix the request, don't retry              │
└────────────────────────────────────────────────────────────────┘
```

---

## gRPC Interceptors (Middleware)

Interceptors are gRPC's equivalent of HTTP middleware. They intercept RPCs before/after the handler.

```
┌──────────────────────────────────────────────────────────────────┐
│  Interceptor Types                                               │
│                                                                  │
│  ┌──────────────────┬───────────────────────────────────────┐    │
│  │ Type             │ When to use                           │    │
│  ├──────────────────┼───────────────────────────────────────┤    │
│  │ UnaryInterceptor │ For unary RPCs (request → response)   │    │
│  │ StreamInterceptor│ For streaming RPCs                     │    │
│  ├──────────────────┼───────────────────────────────────────┤    │
│  │ Server-side      │ grpc.UnaryInterceptor()               │    │
│  │ Client-side      │ grpc.WithUnaryInterceptor()           │    │
│  └──────────────────┴───────────────────────────────────────┘    │
│                                                                  │
│  Execution flow:                                                 │
│  Client → [Client Interceptors] → Network → [Server Interceptors]│
│         → Handler → [Server Interceptors] → Network              │
│         → [Client Interceptors] → Client                         │
└──────────────────────────────────────────────────────────────────┘
```

### Server-Side Interceptors

```go
import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// Logging interceptor
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // Call the actual handler
    resp, err := handler(ctx, req)

    // Log after execution
    duration := time.Since(start)
    code := codes.OK
    if err != nil {
        if st, ok := status.FromError(err); ok {
            code = st.Code()
        }
    }
    log.Printf("[gRPC] %s | %s | %v", info.FullMethod, code, duration)

    return resp, err
}

// Recovery interceptor (catch panics)
func recoveryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("[PANIC] %s: %v", info.FullMethod, r)
            err = status.Errorf(codes.Internal, "internal server error")
        }
    }()
    return handler(ctx, req)
}

// Authentication interceptor
func authInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Skip auth for health check
    if info.FullMethod == "/grpc.health.v1.Health/Check" {
        return handler(ctx, req)
    }

    // Extract metadata (gRPC headers)
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Errorf(codes.Unauthenticated, "missing metadata")
    }

    // Get authorization token
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Errorf(codes.Unauthenticated, "missing auth token")
    }

    // Validate token (your auth logic here)
    userID, err := validateToken(tokens[0])
    if err != nil {
        return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
    }

    // Add user info to context for downstream handlers
    ctx = context.WithValue(ctx, "user_id", userID)
    return handler(ctx, req)
}

// Chain multiple interceptors
func main() {
    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            recoveryInterceptor,  // outermost — catches panics first
            loggingInterceptor,   // logs every call
            authInterceptor,      // checks auth
        ),
        grpc.ChainStreamInterceptor(
            // stream interceptors here...
        ),
    )
    // ...
}
```

### Client-Side Interceptors

```go
// Client-side: add auth token to every request
func clientAuthInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    // Attach token as metadata
    ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+getToken())
    return invoker(ctx, method, req, reply, cc, opts...)
}

// Client-side: retry interceptor
func retryInterceptor(maxRetries int) grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        var lastErr error
        for i := 0; i <= maxRetries; i++ {
            lastErr = invoker(ctx, method, req, reply, cc, opts...)
            if lastErr == nil {
                return nil
            }
            st, ok := status.FromError(lastErr)
            if !ok || st.Code() != codes.Unavailable {
                return lastErr // don't retry non-transient errors
            }
            time.Sleep(time.Duration(i*100) * time.Millisecond) // backoff
        }
        return lastErr
    }
}

func main() {
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithChainUnaryInterceptor(
            clientAuthInterceptor,
            retryInterceptor(3),
        ),
    )
    // ...
}
```

---

## gRPC Metadata (Headers)

```go
import "google.golang.org/grpc/metadata"

// Client-side: send metadata
func sendWithMetadata(client userpb.UserServiceClient) {
    // Create metadata
    md := metadata.Pairs(
        "authorization", "Bearer my-token",
        "x-request-id", "req-12345",
    )
    ctx := metadata.NewOutgoingContext(context.Background(), md)

    // Or append to existing context
    ctx = metadata.AppendToOutgoingContext(ctx,
        "x-custom-header", "value1",
    )

    resp, err := client.GetUser(ctx, &userpb.GetUserRequest{Id: "1"})
    // ...
}

// Server-side: read metadata
func (s *server) GetUser(ctx context.Context, req *userpb.GetUserRequest) (*userpb.GetUserResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    if ok {
        if reqIDs := md.Get("x-request-id"); len(reqIDs) > 0 {
            fmt.Printf("Request ID: %s\n", reqIDs[0])
        }
    }
    // ...
}

// Server-side: send response metadata (headers + trailers)
func (s *server) GetUser(ctx context.Context, req *userpb.GetUserRequest) (*userpb.GetUserResponse, error) {
    // Send headers (before response)
    header := metadata.Pairs("x-cache", "miss")
    grpc.SendHeader(ctx, header)

    // Send trailers (after response)
    trailer := metadata.Pairs("x-processing-time", "42ms")
    grpc.SetTrailer(ctx, trailer)

    return &userpb.GetUserResponse{User: user}, nil
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  Metadata vs HTTP Headers                                    │
│                                                              │
│  gRPC metadata ≈ HTTP/2 headers                              │
│                                                              │
│  • Keys are lowercase strings                                │
│  • Values are strings (or binary with "-bin" suffix)         │
│  • Keys starting with "grpc-" are reserved                   │
│  • Multiple values per key allowed                           │
│                                                              │
│  Headers vs Trailers:                                        │
│  ┌────────────────────────────────────────────────────┐      │
│  │ Headers:  sent BEFORE response body                │      │
│  │           (auth tokens, request IDs, cache info)   │      │
│  │                                                    │      │
│  │ Trailers: sent AFTER response body                 │      │
│  │           (processing time, checksums, status)     │      │
│  │           Only possible with HTTP/2 (not HTTP/1.1) │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

---

## gRPC Health Checking

```go
import (
    "google.golang.org/grpc/health"
    "google.golang.org/grpc/health/grpc_health_v1"
)

func main() {
    grpcServer := grpc.NewServer()

    // Register your service
    userpb.RegisterUserServiceServer(grpcServer, newServer())

    // Register health service
    healthServer := health.NewServer()
    grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)

    // Set service health status
    healthServer.SetServingStatus("userservice.UserService",
        grpc_health_v1.HealthCheckResponse_SERVING)

    // Can update status dynamically
    // healthServer.SetServingStatus("...", grpc_health_v1.HealthCheckResponse_NOT_SERVING)
}

// Client-side health check
func checkHealth(conn *grpc.ClientConn) error {
    client := grpc_health_v1.NewHealthClient(conn)
    resp, err := client.Check(context.Background(),
        &grpc_health_v1.HealthCheckRequest{
            Service: "userservice.UserService",
        })
    if err != nil {
        return err
    }
    if resp.Status != grpc_health_v1.HealthCheckResponse_SERVING {
        return fmt.Errorf("service not healthy: %s", resp.Status)
    }
    return nil
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  gRPC Health Check Protocol                                  │
│                                                              │
│  Standard protocol: grpc.health.v1.Health                    │
│                                                              │
│  Statuses:                                                   │
│  • SERVING              → healthy, accepting requests        │
│  • NOT_SERVING          → unhealthy, reject requests         │
│  • SERVICE_UNKNOWN      → service not registered             │
│  • UNKNOWN              → not yet determined                 │
│                                                              │
│  Used by:                                                    │
│  • Kubernetes gRPC health probes (since K8s 1.24)            │
│  • Load balancers (Envoy, gRPC-LB)                           │
│  • Service mesh (Istio)                                      │
│                                                              │
│  Command-line testing:                                       │
│  grpcurl -plaintext localhost:50051 \                        │
│      grpc.health.v1.Health/Check                             │
└──────────────────────────────────────────────────────────────┘
```

---

## Testing gRPC Services

```go
package main

import (
    "context"
    "log"
    "net"
    "testing"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
    "google.golang.org/grpc/test/bufconn"

    userpb "github.com/myapp/proto/userpb"
)

const bufSize = 1024 * 1024

// bufconn: in-memory connection for testing (no real network)
func startTestServer(t *testing.T) userpb.UserServiceClient {
    t.Helper()

    lis := bufconn.Listen(bufSize)
    grpcServer := grpc.NewServer()
    userpb.RegisterUserServiceServer(grpcServer, newServer())

    go func() {
        if err := grpcServer.Serve(lis); err != nil {
            log.Printf("server exited: %v", err)
        }
    }()

    t.Cleanup(func() {
        grpcServer.Stop()
        lis.Close()
    })

    // Connect via bufconn (in-memory, no TCP)
    conn, err := grpc.NewClient("passthrough:///",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
            return lis.DialContext(ctx)
        }),
    )
    if err != nil {
        t.Fatalf("dial error: %v", err)
    }
    t.Cleanup(func() { conn.Close() })

    return userpb.NewUserServiceClient(conn)
}

func TestCreateAndGetUser(t *testing.T) {
    client := startTestServer(t)
    ctx := context.Background()

    // Create user
    createResp, err := client.CreateUser(ctx, &userpb.CreateUserRequest{
        Email: "test@example.com",
        Name:  "Test User",
    })
    if err != nil {
        t.Fatalf("CreateUser: %v", err)
    }

    if createResp.User.GetName() != "Test User" {
        t.Errorf("expected name 'Test User', got %q", createResp.User.GetName())
    }

    // Get user
    getResp, err := client.GetUser(ctx, &userpb.GetUserRequest{
        Id: createResp.User.GetId(),
    })
    if err != nil {
        t.Fatalf("GetUser: %v", err)
    }

    if getResp.User.GetEmail() != "test@example.com" {
        t.Errorf("expected email 'test@example.com', got %q", getResp.User.GetEmail())
    }
}

func TestGetUser_NotFound(t *testing.T) {
    client := startTestServer(t)
    ctx := context.Background()

    _, err := client.GetUser(ctx, &userpb.GetUserRequest{Id: "nonexistent"})
    if err == nil {
        t.Fatal("expected error, got nil")
    }

    st, ok := status.FromError(err)
    if !ok {
        t.Fatalf("expected gRPC status error, got %v", err)
    }
    if st.Code() != codes.NotFound {
        t.Errorf("expected NotFound, got %s", st.Code())
    }
}

func TestCreateUser_InvalidArgument(t *testing.T) {
    client := startTestServer(t)
    ctx := context.Background()

    _, err := client.CreateUser(ctx, &userpb.CreateUserRequest{
        // Missing required email
        Name: "No Email",
    })
    if err == nil {
        t.Fatal("expected error for missing email")
    }

    st, _ := status.FromError(err)
    if st.Code() != codes.InvalidArgument {
        t.Errorf("expected InvalidArgument, got %s", st.Code())
    }
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  gRPC Testing Strategies                                     │
│                                                              │
│  1. bufconn (in-memory connection)                           │
│     • No real TCP — fast, no port conflicts                  │
│     • Tests full gRPC stack (interceptors, serialization)    │
│     • Best for integration tests                             │
│                                                              │
│  2. Interface mocking                                        │
│     • Mock the generated client interface                    │
│     • Use mockgen to generate mocks                          │
│     • Best for unit testing client-side code                 │
│                                                              │
│  3. grpcurl (manual testing)                                 │
│     grpcurl -plaintext -d '{"id":"1"}' \                     │
│         localhost:50051 userservice.UserService/GetUser       │
│                                                              │
│  4. Test status codes                                        │
│     Always verify error codes, not just error != nil.        │
│     NotFound vs InvalidArgument vs Internal matters.         │
└──────────────────────────────────────────────────────────────┘
```

---

## gRPC with TLS

```go
import "google.golang.org/grpc/credentials"

// Server with TLS
func main() {
    creds, err := credentials.NewServerTLSFromFile("cert.pem", "key.pem")
    if err != nil {
        log.Fatalf("TLS setup failed: %v", err)
    }

    grpcServer := grpc.NewServer(grpc.Creds(creds))
    // register services...
}

// Client with TLS
func main() {
    creds, err := credentials.NewClientTLSFromFile("ca-cert.pem", "")
    if err != nil {
        log.Fatalf("TLS setup failed: %v", err)
    }

    conn, err := grpc.NewClient("server:50051",
        grpc.WithTransportCredentials(creds),
    )
    // ...
}

// Mutual TLS (mTLS) — both sides present certificates
func setupMTLS() credentials.TransportCredentials {
    cert, _ := tls.LoadX509KeyPair("client-cert.pem", "client-key.pem")
    caCert, _ := os.ReadFile("ca-cert.pem")
    pool := x509.NewCertPool()
    pool.AppendCertsFromPEM(caCert)

    return credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      pool,
    })
}
```

```
┌──────────────────────────────────────────────────────────────┐
│  TLS vs mTLS                                                 │
│                                                              │
│  TLS (one-way):                                              │
│  Client ──── verifies ────▶ Server's certificate             │
│  • Server proves its identity                                │
│  • Client is anonymous at TLS level                          │
│  • Standard for public APIs                                  │
│                                                              │
│  mTLS (mutual):                                              │
│  Client ◀─── verify ───── Server's certificate               │
│  Client ──── present ────▶ Client's certificate              │
│  • Both sides prove identity                                 │
│  • Standard for microservice-to-microservice                 │
│  • Used with service mesh (Istio does automatic mTLS)        │
└──────────────────────────────────────────────────────────────┘
```

---

## gRPC-Gateway (REST + gRPC)

```
┌──────────────────────────────────────────────────────────────────┐
│  gRPC-Gateway Architecture                                       │
│                                                                  │
│  External clients (browsers, mobile) use REST.                   │
│  Internal services use gRPC.                                     │
│  gRPC-Gateway bridges the two from a single .proto definition.   │
│                                                                  │
│  ┌──────────┐    REST/JSON     ┌──────────────────┐    gRPC      │
│  │  Browser  │ ──────────────▶ │  gRPC-Gateway    │ ──────────▶  │
│  │  Mobile   │ ◀────────────── │  (reverse proxy) │ ◀──────────  │
│  └──────────┘    HTTP/1.1      └──────────────────┘    HTTP/2    │
│                                         │                        │
│  ┌──────────┐    gRPC/HTTP2    ┌────────▼─────────┐              │
│  │  Service  │ ──────────────▶ │   gRPC Server    │              │
│  │  (Go)     │ ◀────────────── │   (same server)  │              │
│  └──────────┘                  └──────────────────┘              │
│                                                                  │
│  Single .proto defines both gRPC + REST endpoints.               │
└──────────────────────────────────────────────────────────────────┘
```

```protobuf
// With gRPC-Gateway annotations
import "google/api/annotations.proto";

service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse) {
        option (google.api.http) = {
            get: "/v1/users/{id}"
        };
    }

    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
        option (google.api.http) = {
            post: "/v1/users"
            body: "*"
        };
    }

    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {
        option (google.api.http) = {
            get: "/v1/users"
        };
    }
}

// This generates:
// GET  /v1/users/123     → GetUser({id: "123"})
// POST /v1/users         → CreateUser(body as JSON)
// GET  /v1/users?page_size=10 → ListUsers({page_size: 10})
```

---

## Protobuf Well-Known Types

```
┌──────────────────────────────────────────────────────────────┐
│  Commonly Used Well-Known Types                              │
│                                                              │
│  Import                          │ Go Type                   │
│  ────────────────────────────────┼─────────────────────────  │
│  google/protobuf/timestamp.proto │ *timestamppb.Timestamp    │
│  google/protobuf/duration.proto  │ *durationpb.Duration      │
│  google/protobuf/empty.proto     │ *emptypb.Empty            │
│  google/protobuf/wrappers.proto  │ *wrapperspb.StringValue   │
│  google/protobuf/struct.proto    │ *structpb.Struct          │
│  google/protobuf/any.proto       │ *anypb.Any                │
│  google/protobuf/field_mask.proto│ *fieldmaskpb.FieldMask    │
│                                                              │
│  Timestamp conversion:                                       │
│  ts := timestamppb.Now()                                     │
│  goTime := ts.AsTime()          // → time.Time               │
│  ts2 := timestamppb.New(goTime) // → *Timestamp              │
│                                                              │
│  Duration conversion:                                        │
│  d := durationpb.New(5 * time.Second)                        │
│  goDur := d.AsDuration()         // → time.Duration          │
│                                                              │
│  Wrappers (distinguish "not set" from "zero"):               │
│  message User {                                              │
│      google.protobuf.StringValue nickname = 1;               │
│  }                                                           │
│  // In Go: user.Nickname == nil → not set                    │
│  //        user.Nickname.Value == "" → set to empty string   │
└──────────────────────────────────────────────────────────────┘
```

---

## Performance Considerations

```
┌──────────────────────────────────────────────────────────────────┐
│  gRPC Performance Tuning                                         │
│                                                                  │
│  1. Connection management                                        │
│     • Reuse connections (one per service)                         │
│     • Enable keepalive pings for long-lived connections           │
│                                                                  │
│  2. Message size                                                  │
│     • Default max: 4MB per message                                │
│     • Override: grpc.MaxRecvMsgSize(16 * 1024 * 1024) // 16MB    │
│     • Better: use streaming for large data                       │
│                                                                  │
│  3. Keepalive settings                                           │
│     grpc.NewServer(                                              │
│         grpc.KeepaliveParams(keepalive.ServerParameters{         │
│             Time:    30 * time.Second,   // ping interval        │
│             Timeout: 10 * time.Second,   // ping timeout         │
│         }),                                                      │
│     )                                                            │
│                                                                  │
│  4. Load balancing                                               │
│     • Client-side: round_robin, pick_first (built-in)            │
│     • L7 proxy: Envoy, Traefik, nginx (with gRPC module)        │
│     • Service mesh: Istio (automatic L7 gRPC balancing)          │
│                                                                  │
│  5. Compression                                                  │
│     import "google.golang.org/grpc/encoding/gzip"                │
│     // Server: automatically decompresses                        │
│     // Client: grpc.UseCompressor(gzip.Name)                     │
│                                                                  │
│  Benchmark comparison (typical 1KB payload):                     │
│  ┌────────────────────┬───────────┬───────────┐                  │
│  │                    │ REST/JSON │ gRPC/Proto │                  │
│  ├────────────────────┼───────────┼───────────┤                  │
│  │ Serialized size    │ ~1200 B   │ ~400 B    │                  │
│  │ Serialization time │ ~15 μs    │ ~3 μs     │                  │
│  │ Deserialization    │ ~25 μs    │ ~5 μs     │                  │
│  │ Latency (local)    │ ~500 μs   │ ~100 μs   │                  │
│  └────────────────────┴───────────┴───────────┘                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What is gRPC and how does it differ from REST?**

gRPC is a high-performance RPC framework using HTTP/2 and Protocol Buffers. Key differences from REST: (1) Binary serialization (Protobuf) vs text (JSON) — 3-10x smaller, 5-10x faster serialization. (2) HTTP/2 vs HTTP/1.1 — multiplexing, header compression, bidirectional streaming. (3) Strongly typed contract (`.proto` file) vs loosely typed (OpenAPI is optional). (4) Code generation built-in vs optional. (5) Native streaming (4 patterns) vs no native streaming. REST is better for public-facing APIs and browser clients; gRPC is better for internal microservice communication.

**Q2: Explain the 4 gRPC communication patterns.**

(1) **Unary**: single request → single response. Most common, like a function call. (2) **Server streaming**: single request → stream of responses. Used for list operations, real-time feeds, log tailing. (3) **Client streaming**: stream of requests → single response. Used for file uploads, batch operations. (4) **Bidirectional streaming**: both sides stream independently. Used for chat, game state, live dashboards. All patterns use a single HTTP/2 connection with multiplexed streams.

**Q3: Why are field numbers important in Protobuf? What happens if you change them?**

Field numbers are the wire identity of each field — the binary format uses numbers, not names, to identify fields. Numbers 1-15 use 1 byte of encoding (use for frequent fields), 16-2047 use 2 bytes. **Never change or reuse field numbers** after deployment — changing them means old clients will read the wrong fields (data corruption), and reusing deleted numbers means old messages on the wire will be misinterpreted. Use `reserved` to prevent reuse of deleted field numbers and names.

**Q4: What is `UnimplementedXxxServer` and why must you embed it?**

`UnimplementedXxxServer` is a generated struct that provides default "Unimplemented" responses for all service methods. You must embed it in your server struct for **forward compatibility**: if you add a new RPC to the `.proto` file and regenerate, your existing server code still compiles — the new method returns `codes.Unimplemented` by default. Without it, adding a new RPC breaks compilation. Alternative: embed `UnsafeXxxServer` to opt out and get compile errors for missing methods.

**Q5: How do gRPC interceptors work? Give examples of common interceptors.**

Interceptors are gRPC middleware that run before/after the handler. Two types: `UnaryInterceptor` (for request-response RPCs) and `StreamInterceptor` (for streaming RPCs). Both exist for server-side and client-side. Common interceptors: (1) Logging — log method, duration, status code. (2) Authentication — extract and validate tokens from metadata. (3) Recovery — catch panics and return `codes.Internal`. (4) Rate limiting — track request counts per client. Chain multiple with `grpc.ChainUnaryInterceptor()`.

**Q6: How do you handle errors properly in gRPC?**

Use `google.golang.org/grpc/status` and `google.golang.org/grpc/codes`. Server returns `status.Errorf(codes.NotFound, "message")`. Client checks with `st, ok := status.FromError(err)` then switches on `st.Code()`. Important: `codes.Unavailable` means retry (transient), `codes.Internal` means don't retry (server bug), `codes.InvalidArgument` means fix the request. Never return raw Go errors — they become `codes.Unknown` which gives clients no information to act on.

**Q7: How do you test gRPC services in Go?**

Use `bufconn` (in-memory connection) for integration testing — it provides a full gRPC stack without real TCP, so tests are fast and don't need ports. Create a `bufconn.Listener`, start the gRPC server on it, connect via `grpc.WithContextDialer`. For client-side unit testing, mock the generated client interface using `mockgen`. For manual testing, use `grpcurl`. Always test error codes specifically (e.g., `codes.NotFound` vs `codes.InvalidArgument`), not just `err != nil`.

**Q8: What is gRPC metadata and how does it relate to HTTP headers?**

gRPC metadata is the equivalent of HTTP headers — key-value pairs sent alongside RPCs. Keys must be lowercase strings. Values are strings (or binary with `-bin` suffix). Server reads incoming metadata with `metadata.FromIncomingContext(ctx)`. Client sends metadata with `metadata.NewOutgoingContext()` or `metadata.AppendToOutgoingContext()`. Server can send response metadata as headers (before response) with `grpc.SendHeader()` or trailers (after response) with `grpc.SetTrailer()`. Common uses: auth tokens, request IDs, tracing context.

**Q9: Explain the difference between TLS and mTLS in gRPC.**

TLS (one-way): only the server presents a certificate, client verifies server identity. Standard for public APIs. mTLS (mutual TLS): both sides present certificates. Server verifies client's certificate too. Standard for microservice-to-microservice communication where both parties need to prove identity. In service meshes like Istio, mTLS is handled automatically (sidecar proxies manage certificates). In Go, TLS uses `credentials.NewServerTLSFromFile()`, mTLS requires loading CA cert pool and configuring `tls.Config` with client certificates.

**Q10: What is gRPC-Gateway and when would you use it?**

gRPC-Gateway is a reverse proxy that generates a REST API from gRPC service definitions. You annotate `.proto` files with HTTP path mappings (`option (google.api.http) = { get: "/v1/users/{id}" }`), and it generates a Go handler that translates REST requests into gRPC calls. Use it when: (1) you need to serve both REST clients (browsers) and gRPC clients (microservices), (2) you want a single source of truth (`.proto` file) for both APIs, (3) migration path from REST to gRPC.

**Q11: What are Protobuf well-known types and when do you use them?**

Well-known types are standard message types provided by Google for common patterns: `Timestamp` (time.Time equivalent), `Duration` (time.Duration equivalent), `Empty` (void return), `FieldMask` (partial updates — specify which fields to update), `Struct` (dynamic JSON-like data), `Any` (polymorphic messages), wrapper types like `StringValue` (distinguish "not set" from empty string). Use them instead of reinventing — they have proper serialization, JSON mapping, and cross-language support built in.

**Q12: How does gRPC load balancing differ from REST load balancing?**

REST uses HTTP/1.1 where each request is a separate TCP connection — a simple L4 load balancer (TCP round-robin) works. gRPC uses HTTP/2 with multiplexed streams over a single long-lived connection — an L4 load balancer sends ALL requests to the same backend. Solutions: (1) Client-side LB with `grpc.WithDefaultServiceConfig()` using `round_robin` or `pick_first`. (2) L7 proxy (Envoy, Traefik) that understands HTTP/2 frames. (3) Service mesh (Istio) with sidecar proxies. In Kubernetes, always use headless services or L7 proxies for gRPC.

---

## Interview Problems

**Problem 1: Design a gRPC service for a URL shortener**

```
// Answer:

service URLShortener {
    rpc Shorten(ShortenRequest) returns (ShortenResponse);       // create short URL
    rpc Resolve(ResolveRequest) returns (ResolveResponse);       // redirect short → long
    rpc GetStats(GetStatsRequest) returns (GetStatsResponse);    // click analytics
    rpc ListURLs(ListURLsRequest) returns (stream URLEntry);     // server stream
}

message ShortenRequest {
    string long_url = 1;
    optional string custom_alias = 2;
    optional google.protobuf.Timestamp expires_at = 3;
}

message ShortenResponse {
    string short_code = 1;
    string short_url = 2;
}

Design decisions:
• Unary for Shorten/Resolve (simple request-response)
• Server streaming for ListURLs (could be large dataset)
• Interceptors: auth (who created), rate limit (prevent abuse)
• Error codes: AlreadyExists (custom alias taken),
  NotFound (short code invalid), InvalidArgument (bad URL)
• Cache layer: Redis for hot short codes
• Backward compat: use optional for new fields
```

**Problem 2: Implement a gRPC interceptor that adds request tracing**

```go
// Solution: Tracing interceptor that adds/propagates request IDs

func tracingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Extract or generate request ID
    requestID := ""
    if md, ok := metadata.FromIncomingContext(ctx); ok {
        if ids := md.Get("x-request-id"); len(ids) > 0 {
            requestID = ids[0]
        }
    }
    if requestID == "" {
        requestID = uuid.New().String()
    }

    // Add to context for downstream use
    ctx = context.WithValue(ctx, "request_id", requestID)

    // Set as response header so client gets it
    grpc.SetHeader(ctx, metadata.Pairs("x-request-id", requestID))

    // Log start
    start := time.Now()
    log.Printf("[%s] START %s", requestID, info.FullMethod)

    // Execute handler
    resp, err := handler(ctx, req)

    // Log end
    code := codes.OK
    if err != nil {
        if st, ok := status.FromError(err); ok {
            code = st.Code()
        }
    }
    log.Printf("[%s] END %s | %s | %v",
        requestID, info.FullMethod, code, time.Since(start))

    return resp, err
}
```

**Problem 3: Compare Protobuf schema evolution — which changes are safe?**

```
// Answer with analysis:

Version 1:
message Order {
    string id = 1;
    string product = 2;
    int32 quantity = 3;
    double price = 4;
}

Version 2 — SAFE changes:
message Order {
    string id = 1;
    string product = 2;
    int32 quantity = 3;
    double price = 4;
    string customer_id = 5;     // ✅ new field, new number
    optional string notes = 6;  // ✅ new optional field
    repeated string tags = 7;   // ✅ new repeated field
    reserved 8, 9;              // ✅ reserve for future
}

Why safe:
• Old clients ignore unknown field numbers (5, 6, 7)
• New clients see zero values for missing fields
• Field names can change (wire uses numbers)

UNSAFE changes (never do):
• Change field 3 from int32 to string — wire type mismatch
• Change field 2's number from 2 to 10 — data misread
• Remove field 1 and add new field reusing number 1
• Change enum values' numbers
```
