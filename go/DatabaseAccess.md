# Chapter 34 — Database Access in Go

Go's standard library includes `database/sql` — a generic interface for SQL databases. It provides connection pooling, prepared statements, transactions, and a driver-agnostic API. Understanding `database/sql` is essential for backend Go interviews.

```
┌──────────────────────────────────────────────────────────────┐
│                  database/sql Architecture                    │
│                                                              │
│  Your Code                                                   │
│     │                                                        │
│     ▼                                                        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  database/sql  (standard library)                    │    │
│  │  ┌────────────────────────────────────────────────┐  │    │
│  │  │  sql.DB — connection pool manager               │  │    │
│  │  │  • Opens/closes connections automatically       │  │    │
│  │  │  • Reuses idle connections                      │  │    │
│  │  │  • Thread-safe (goroutine-safe)                 │  │    │
│  │  └────────────────────────────────────────────────┘  │    │
│  └──────────────┬───────────────────────────────────────┘    │
│                 │  driver.Driver interface                    │
│                 ▼                                            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Driver (third-party)                                │    │
│  │  • github.com/lib/pq           (PostgreSQL)          │    │
│  │  • github.com/go-sql-driver/mysql (MySQL)            │    │
│  │  • github.com/mattn/go-sqlite3  (SQLite)             │    │
│  │  • github.com/jackc/pgx/v5     (PostgreSQL, modern)  │    │
│  └──────────────────────────────────────────────────────┘    │
│                 │                                            │
│                 ▼                                            │
│           Database Server                                    │
└──────────────────────────────────────────────────────────────┘
```

## Opening a Database Connection

**Tutorial: sql.DB is a Pool, Not a Single Connection**

`sql.Open` doesn't actually open a connection — it creates a `sql.DB` which manages a pool of connections. Connections are opened lazily when needed. Always call `db.Ping()` to verify connectivity.

```
┌────────────────────────────────────────────────────────────┐
│  sql.Open("postgres", connStr)                             │
│     │                                                      │
│     ▼                                                      │
│  sql.DB (connection pool — no connections yet!)             │
│     │                                                      │
│     ▼  db.Ping()                                           │
│  Opens first connection, verifies server reachable         │
│     │                                                      │
│     ▼  db.Query(...) / db.Exec(...)                        │
│  Gets connection from pool (or opens new one)              │
│  Executes query                                            │
│  Returns connection to pool                                │
│                                                            │
│  Pool settings:                                            │
│  db.SetMaxOpenConns(25)    ← max simultaneous connections  │
│  db.SetMaxIdleConns(5)     ← max idle connections in pool  │
│  db.SetConnMaxLifetime(5m) ← max time a conn can be reused │
│  db.SetConnMaxIdleTime(1m) ← max time conn can sit idle    │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "github.com/lib/pq" // PostgreSQL driver — blank import registers it
)

func main() {
	// sql.Open does NOT open a connection — it creates a pool
	connStr := "host=localhost port=5432 user=myuser password=mypass dbname=mydb sslmode=disable"
	db, err := sql.Open("postgres", connStr)
	if err != nil {
		log.Fatal("Failed to create DB pool:", err)
	}
	defer db.Close() // Close pool when main exits

	// Configure connection pool
	db.SetMaxOpenConns(25)                 // max open connections
	db.SetMaxIdleConns(5)                  // max idle connections
	db.SetConnMaxLifetime(5 * time.Minute) // max lifetime per connection
	db.SetConnMaxIdleTime(1 * time.Minute) // close idle conns after 1 min

	// Ping verifies connectivity (opens actual connection)
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := db.PingContext(ctx); err != nil {
		log.Fatal("Cannot reach database:", err)
	}

	fmt.Println("Connected to database!")

	// Check pool stats
	stats := db.Stats()
	fmt.Printf("Open conns: %d, In use: %d, Idle: %d\n",
		stats.OpenConnections, stats.InUse, stats.Idle)
}
```

---

## Queries: QueryRow, Query, Exec

**Tutorial: The Three Query Methods**

`database/sql` has three primary methods for executing SQL:
- `QueryRow` — returns a single row (SELECT one)
- `Query` — returns multiple rows (SELECT many)
- `Exec` — for INSERT, UPDATE, DELETE (no rows returned)

```
┌────────────────────────────────────────────────────────────┐
│  Three Query Methods                                       │
│                                                            │
│  db.QueryRowContext(ctx, sql, args...)                      │
│  → *sql.Row — single row, scan directly                    │
│  → row.Scan(&col1, &col2, ...)                             │
│  → Returns sql.ErrNoRows if no match                       │
│                                                            │
│  db.QueryContext(ctx, sql, args...)                         │
│  → *sql.Rows — multiple rows, iterate with rows.Next()     │
│  → MUST close: defer rows.Close()                          │
│  → rows.Scan(&col1, &col2, ...) per row                    │
│                                                            │
│  db.ExecContext(ctx, sql, args...)                          │
│  → sql.Result — no rows returned                           │
│  → result.RowsAffected() → int64                           │
│  → result.LastInsertId() → int64 (driver-dependent)        │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"
)

type User struct {
	ID        int
	Name      string
	Email     string
	CreatedAt time.Time
}

// QueryRow — single row
func getUserByID(ctx context.Context, db *sql.DB, id int) (*User, error) {
	var u User
	err := db.QueryRowContext(ctx,
		"SELECT id, name, email, created_at FROM users WHERE id = $1", id,
	).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)

	if err == sql.ErrNoRows {
		return nil, fmt.Errorf("user %d not found", id)
	}
	if err != nil {
		return nil, fmt.Errorf("query user: %w", err)
	}
	return &u, nil
}

// Query — multiple rows
func getActiveUsers(ctx context.Context, db *sql.DB) ([]User, error) {
	rows, err := db.QueryContext(ctx,
		"SELECT id, name, email, created_at FROM users WHERE active = true ORDER BY name",
	)
	if err != nil {
		return nil, fmt.Errorf("query users: %w", err)
	}
	defer rows.Close() // CRITICAL: always close rows

	var users []User
	for rows.Next() {
		var u User
		if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt); err != nil {
			return nil, fmt.Errorf("scan user: %w", err)
		}
		users = append(users, u)
	}

	// Check for errors during iteration
	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("rows error: %w", err)
	}

	return users, nil
}

// Exec — INSERT
func createUser(ctx context.Context, db *sql.DB, name, email string) (int, error) {
	var id int
	// PostgreSQL RETURNING — use QueryRow for INSERT...RETURNING
	err := db.QueryRowContext(ctx,
		"INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
		name, email,
	).Scan(&id)
	if err != nil {
		return 0, fmt.Errorf("insert user: %w", err)
	}
	return id, nil
}

// Exec — UPDATE
func updateUserEmail(ctx context.Context, db *sql.DB, id int, email string) error {
	result, err := db.ExecContext(ctx,
		"UPDATE users SET email = $1 WHERE id = $2",
		email, id,
	)
	if err != nil {
		return fmt.Errorf("update user: %w", err)
	}

	rowsAffected, _ := result.RowsAffected()
	if rowsAffected == 0 {
		return fmt.Errorf("user %d not found", id)
	}
	return nil
}

// Exec — DELETE
func deleteUser(ctx context.Context, db *sql.DB, id int) error {
	result, err := db.ExecContext(ctx,
		"DELETE FROM users WHERE id = $1", id,
	)
	if err != nil {
		return fmt.Errorf("delete user: %w", err)
	}
	affected, _ := result.RowsAffected()
	fmt.Printf("Deleted %d row(s)\n", affected)
	return nil
}

func main() {
	// This is illustrative — requires a running database
	log.Println("Database CRUD operations defined (requires live DB to run)")
}
```

---

## Parameterized Queries (SQL Injection Prevention)

**Tutorial: Always Use Placeholders, Never String Concatenation**

**NEVER** build SQL with `fmt.Sprintf` or string concatenation. Always use parameterized queries with `$1, $2` (PostgreSQL) or `?, ?` (MySQL/SQLite). The driver handles escaping, preventing SQL injection.

```
┌────────────────────────────────────────────────────────────┐
│  ❌ WRONG — SQL injection vulnerability:                    │
│  query := fmt.Sprintf("SELECT * FROM users WHERE name='%s'", name)│
│  db.Query(query)                                           │
│  → name = "'; DROP TABLE users; --"  💀                    │
│                                                            │
│  ✅ RIGHT — parameterized query:                            │
│  db.Query("SELECT * FROM users WHERE name = $1", name)     │
│  → Driver escapes the value, injection impossible          │
│                                                            │
│  Placeholder syntax by driver:                             │
│  ┌────────────┬──────────────┐                             │
│  │ PostgreSQL │ $1, $2, $3   │                             │
│  │ MySQL      │ ?, ?, ?      │                             │
│  │ SQLite     │ ?, ?, ?      │                             │
│  │ SQL Server │ @p1, @p2     │                             │
│  └────────────┴──────────────┘                             │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
)

func searchUsers(ctx context.Context, db *sql.DB, namePattern string, minAge int) ([]string, error) {
	// ✅ SAFE: parameterized query
	rows, err := db.QueryContext(ctx,
		"SELECT name FROM users WHERE name LIKE $1 AND age >= $2",
		"%"+namePattern+"%", // parameter 1
		minAge,              // parameter 2
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var names []string
	for rows.Next() {
		var name string
		if err := rows.Scan(&name); err != nil {
			return nil, err
		}
		names = append(names, name)
	}
	return names, rows.Err()
}

// ❌ NEVER DO THIS — vulnerable to SQL injection
func searchUsersUNSAFE(ctx context.Context, db *sql.DB, name string) {
	// This allows SQL injection!
	query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name)
	_ = query // DON'T
}
```

---

## Prepared Statements

**Tutorial: Precompiling SQL for Repeated Execution**

Prepared statements parse the SQL once and execute it many times with different parameters. This improves performance for repeated queries and provides an additional security layer.

```
┌────────────────────────────────────────────────────────────┐
│  Prepared Statement Lifecycle                              │
│                                                            │
│  stmt, err := db.PrepareContext(ctx, sql)                  │
│       │                                                    │
│       ▼  SQL parsed once by database                       │
│  stmt.QueryRowContext(ctx, arg1, arg2)   ← execute many    │
│  stmt.QueryRowContext(ctx, arg3, arg4)   ← times           │
│  stmt.ExecContext(ctx, arg5, arg6)                         │
│       │                                                    │
│       ▼                                                    │
│  defer stmt.Close()                                        │
│                                                            │
│  ⚠️ Prepared statements are tied to a connection           │
│  database/sql re-prepares on different connections          │
│  automatically (transparent to you)                        │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
)

func batchInsertUsers(ctx context.Context, db *sql.DB, users []struct{ Name, Email string }) error {
	// Prepare once
	stmt, err := db.PrepareContext(ctx,
		"INSERT INTO users (name, email) VALUES ($1, $2)",
	)
	if err != nil {
		return fmt.Errorf("prepare: %w", err)
	}
	defer stmt.Close()

	// Execute many times with different args
	for _, u := range users {
		_, err := stmt.ExecContext(ctx, u.Name, u.Email)
		if err != nil {
			return fmt.Errorf("insert %s: %w", u.Name, err)
		}
	}
	return nil
}

func main() {
	log.Println("Prepared statement batch insert (requires live DB)")
}
```

---

## Transactions

**Tutorial: Atomic Operations with Begin/Commit/Rollback**

Transactions group multiple operations so they either all succeed (commit) or all fail (rollback). Use `db.BeginTx` to start, `tx.Commit()` to finalize, and `tx.Rollback()` on error. A common pattern is `defer tx.Rollback()` — it's a no-op after `Commit()`.

```
┌────────────────────────────────────────────────────────────┐
│  Transaction Pattern                                       │
│                                                            │
│  tx, err := db.BeginTx(ctx, nil)                           │
│  if err != nil { return err }                              │
│  defer tx.Rollback() // no-op if committed                 │
│       │                                                    │
│       ▼                                                    │
│  tx.ExecContext(ctx, "UPDATE accounts SET balance = ...")   │
│  tx.ExecContext(ctx, "INSERT INTO transfers ...")           │
│       │                                                    │
│       ▼  all succeeded?                                    │
│  return tx.Commit()                                        │
│                                                            │
│  If any step fails → return err → defer runs Rollback()    │
│  If Commit() succeeds → defer Rollback() is a no-op       │
│                                                            │
│  Isolation Levels:                                         │
│  sql.LevelReadCommitted (default for most DBs)             │
│  sql.LevelRepeatableRead                                   │
│  sql.LevelSerializable                                     │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
)

// TransferFunds atomically moves money between accounts
func TransferFunds(ctx context.Context, db *sql.DB, fromID, toID int, amount float64) error {
	// Start transaction
	tx, err := db.BeginTx(ctx, &sql.TxOptions{
		Isolation: sql.LevelSerializable, // strongest isolation
	})
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback() // no-op after successful commit

	// Check sender balance
	var balance float64
	err = tx.QueryRowContext(ctx,
		"SELECT balance FROM accounts WHERE id = $1 FOR UPDATE", fromID,
	).Scan(&balance)
	if err != nil {
		return fmt.Errorf("get balance: %w", err)
	}

	if balance < amount {
		return fmt.Errorf("insufficient funds: have %.2f, need %.2f", balance, amount)
	}

	// Debit sender
	_, err = tx.ExecContext(ctx,
		"UPDATE accounts SET balance = balance - $1 WHERE id = $2",
		amount, fromID,
	)
	if err != nil {
		return fmt.Errorf("debit: %w", err)
	}

	// Credit receiver
	_, err = tx.ExecContext(ctx,
		"UPDATE accounts SET balance = balance + $1 WHERE id = $2",
		amount, toID,
	)
	if err != nil {
		return fmt.Errorf("credit: %w", err)
	}

	// Record transfer
	_, err = tx.ExecContext(ctx,
		"INSERT INTO transfers (from_id, to_id, amount) VALUES ($1, $2, $3)",
		fromID, toID, amount,
	)
	if err != nil {
		return fmt.Errorf("record transfer: %w", err)
	}

	// Commit — all or nothing
	return tx.Commit()
}
```

---

## Handling NULL Values

**Tutorial: sql.NullString, sql.NullInt64, and Alternatives**

SQL NULL doesn't map directly to Go types. Use `sql.NullString`, `sql.NullInt64`, etc., or use pointer types (`*string`, `*int`). The `Null*` types have a `Valid` field indicating whether the value is NULL.

```
┌────────────────────────────────────────────────────────────┐
│  SQL NULL Handling                                         │
│                                                            │
│  Option 1: sql.Null* types                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ sql.NullString{String: "hello", Valid: true}  → "hello"│  │
│  │ sql.NullString{String: "", Valid: false}       → NULL │  │
│  │ sql.NullInt64{Int64: 42, Valid: true}          → 42   │  │
│  │ sql.NullInt64{Int64: 0, Valid: false}          → NULL │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Option 2: Pointer types                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ var name *string                                     │  │
│  │ name = nil        → NULL                              │  │
│  │ name = &"hello"   → "hello"                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Go 1.22+: sql.Null[T] generic type                       │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
)

// Using sql.Null* types
type UserWithNull struct {
	ID    int
	Name  string
	Bio   sql.NullString // might be NULL
	Age   sql.NullInt64  // might be NULL
}

func getUserNullable(ctx context.Context, db *sql.DB, id int) (*UserWithNull, error) {
	var u UserWithNull
	err := db.QueryRowContext(ctx,
		"SELECT id, name, bio, age FROM users WHERE id = $1", id,
	).Scan(&u.ID, &u.Name, &u.Bio, &u.Age)
	if err != nil {
		return nil, err
	}
	return &u, nil
}

// Using pointer types (cleaner with JSON)
type UserWithPointers struct {
	ID   int     `json:"id"`
	Name string  `json:"name"`
	Bio  *string `json:"bio"`  // nil = NULL, *string = value
	Age  *int    `json:"age"`  // nil = NULL
}

func getUserPointers(ctx context.Context, db *sql.DB, id int) (*UserWithPointers, error) {
	var u UserWithPointers
	err := db.QueryRowContext(ctx,
		"SELECT id, name, bio, age FROM users WHERE id = $1", id,
	).Scan(&u.ID, &u.Name, &u.Bio, &u.Age)
	if err != nil {
		return nil, err
	}
	return &u, nil
}

func main() {
	// Demonstrate sql.NullString
	withValue := sql.NullString{String: "Hello", Valid: true}
	withNull := sql.NullString{String: "", Valid: false}

	fmt.Printf("With value: %q (valid=%t)\n", withValue.String, withValue.Valid)
	fmt.Printf("With null:  %q (valid=%t)\n", withNull.String, withNull.Valid)

	// Pointer pattern works better with JSON
	name := "Alice"
	user := UserWithPointers{
		ID:   1,
		Name: "Alice",
		Bio:  &name, // has value
		Age:  nil,   // NULL
	}
	data, _ := json.MarshalIndent(user, "", "  ")
	fmt.Println(string(data))
	// {
	//   "id": 1,
	//   "name": "Alice",
	//   "bio": "Alice",
	//   "age": null
	// }
}
```

---

## Connection Pool Tuning

**Tutorial: Configuring the Pool for Production**

The default `sql.DB` pool settings are often suboptimal. Tune them based on your workload and database server limits.

```
┌────────────────────────────────────────────────────────────┐
│  Pool Configuration Parameters                             │
│                                                            │
│  db.SetMaxOpenConns(n)                                     │
│  ├── Default: 0 (unlimited!) ← dangerous in production    │
│  ├── Set to: DB max_connections / num_app_instances         │
│  └── Too high: DB overwhelmed. Too low: requests queue.    │
│                                                            │
│  db.SetMaxIdleConns(n)                                     │
│  ├── Default: 2                                            │
│  ├── Set to: ~25-50% of MaxOpenConns                       │
│  └── Too low: frequent reconnects. Too high: wasted RAM.   │
│                                                            │
│  db.SetConnMaxLifetime(d)                                  │
│  ├── Default: 0 (unlimited — connections live forever)     │
│  ├── Set to: 5-15 minutes                                  │
│  └── Prevents stale connections after DB restarts/failover │
│                                                            │
│  db.SetConnMaxIdleTime(d)                                  │
│  ├── Default: 0 (idle connections never expire)            │
│  ├── Set to: 1-5 minutes                                   │
│  └── Closes idle connections to free DB resources          │
│                                                            │
│  Monitoring: db.Stats() returns pool health metrics        │
└────────────────────────────────────────────────────────────┘
```

```go
package main

import (
	"database/sql"
	"fmt"
	"time"
)

func configurePool(db *sql.DB) {
	// Production-ready settings for a typical web app
	db.SetMaxOpenConns(25)                  // match DB capacity
	db.SetMaxIdleConns(10)                  // keep warm connections
	db.SetConnMaxLifetime(5 * time.Minute)  // rotate connections
	db.SetConnMaxIdleTime(1 * time.Minute)  // close idle ones

	// Monitor pool health
	stats := db.Stats()
	fmt.Printf("Pool stats:\n")
	fmt.Printf("  Open connections:   %d\n", stats.OpenConnections)
	fmt.Printf("  In use:             %d\n", stats.InUse)
	fmt.Printf("  Idle:               %d\n", stats.Idle)
	fmt.Printf("  Wait count:         %d\n", stats.WaitCount)         // # of times waited for conn
	fmt.Printf("  Wait duration:      %v\n", stats.WaitDuration)     // total wait time
	fmt.Printf("  Max idle closed:    %d\n", stats.MaxIdleClosed)    // closed for being idle
	fmt.Printf("  Max lifetime closed:%d\n", stats.MaxLifetimeClosed) // closed for max lifetime
}
```

---

## Repository Pattern with database/sql

**Tutorial: Clean Data Access Layer Structure**

Wrap `database/sql` operations in a repository struct. This separates SQL logic from business logic and makes testing easier.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

type User struct {
	ID        int
	Name      string
	Email     string
	CreatedAt time.Time
}

type UserRepository struct {
	db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
	return &UserRepository{db: db}
}

func (r *UserRepository) GetByID(ctx context.Context, id int) (*User, error) {
	var u User
	err := r.db.QueryRowContext(ctx,
		`SELECT id, name, email, created_at FROM users WHERE id = $1`, id,
	).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
	if err == sql.ErrNoRows {
		return nil, nil // not found
	}
	return &u, err
}

func (r *UserRepository) List(ctx context.Context, limit, offset int) ([]User, error) {
	rows, err := r.db.QueryContext(ctx,
		`SELECT id, name, email, created_at FROM users ORDER BY id LIMIT $1 OFFSET $2`,
		limit, offset,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var users []User
	for rows.Next() {
		var u User
		if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt); err != nil {
			return nil, err
		}
		users = append(users, u)
	}
	return users, rows.Err()
}

func (r *UserRepository) Create(ctx context.Context, name, email string) (*User, error) {
	var u User
	err := r.db.QueryRowContext(ctx,
		`INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email, created_at`,
		name, email,
	).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
	return &u, err
}

func (r *UserRepository) Update(ctx context.Context, id int, name, email string) error {
	result, err := r.db.ExecContext(ctx,
		`UPDATE users SET name = $1, email = $2 WHERE id = $3`,
		name, email, id,
	)
	if err != nil {
		return err
	}
	rows, _ := result.RowsAffected()
	if rows == 0 {
		return fmt.Errorf("user %d not found", id)
	}
	return nil
}

func (r *UserRepository) Delete(ctx context.Context, id int) error {
	_, err := r.db.ExecContext(ctx,
		`DELETE FROM users WHERE id = $1`, id,
	)
	return err
}

// Transaction helper — passes *sql.Tx to a function
func (r *UserRepository) WithTx(ctx context.Context, fn func(*sql.Tx) error) error {
	tx, err := r.db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	if err := fn(tx); err != nil {
		return err
	}
	return tx.Commit()
}
```

---

## Testing with sqlmock

**Tutorial: Unit Testing Database Code Without a Real Database**

The `github.com/DATA-DOG/go-sqlmock` library creates a mock `sql.DB` that lets you set expectations on queries without connecting to a real database.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	// In real tests:
	// "github.com/DATA-DOG/go-sqlmock"
)

// Demonstrating the pattern (pseudo-code for the mock parts)
func ExampleTestGetByID() {
	// db, mock, err := sqlmock.New()
	// if err != nil { t.Fatal(err) }
	// defer db.Close()

	// Set expectations
	// rows := sqlmock.NewRows([]string{"id", "name", "email", "created_at"}).
	//     AddRow(1, "Alice", "alice@test.com", time.Now())
	// mock.ExpectQuery("SELECT .+ FROM users WHERE id = \\$1").
	//     WithArgs(1).
	//     WillReturnRows(rows)

	// Execute
	// repo := NewUserRepository(db)
	// user, err := repo.GetByID(context.Background(), 1)

	// Assert
	// assert.NoError(t, err)
	// assert.Equal(t, "Alice", user.Name)
	// assert.NoError(t, mock.ExpectationsWereMet())

	fmt.Println("sqlmock enables testing DB code without a real database")
}

// Interface-based approach for testing
type UserStore interface {
	GetByID(ctx context.Context, id int) (*User, error)
	List(ctx context.Context, limit, offset int) ([]User, error)
	Create(ctx context.Context, name, email string) (*User, error)
}

// Mock implementation for tests
type MockUserStore struct {
	Users map[int]*User
}

func (m *MockUserStore) GetByID(ctx context.Context, id int) (*User, error) {
	u, ok := m.Users[id]
	if !ok {
		return nil, nil
	}
	return u, nil
}

func (m *MockUserStore) List(ctx context.Context, limit, offset int) ([]User, error) {
	var result []User
	for _, u := range m.Users {
		result = append(result, *u)
	}
	return result, nil
}

func (m *MockUserStore) Create(ctx context.Context, name, email string) (*User, error) {
	u := &User{ID: len(m.Users) + 1, Name: name, Email: email, CreatedAt: time.Now()}
	m.Users[u.ID] = u
	return u, nil
}

type User2 = User // alias to avoid redeclaration in same file

func main() {
	// In tests, inject MockUserStore instead of real UserRepository
	mock := &MockUserStore{
		Users: map[int]*User{
			1: {ID: 1, Name: "Alice", Email: "alice@test.com"},
		},
	}

	user, _ := mock.GetByID(context.Background(), 1)
	fmt.Println("Mock user:", user.Name) // Alice
}
```

---

## Common Third-Party Libraries

```
┌────────────────────────────────────────────────────────────┐
│  Popular Go Database Libraries                             │
│                                                            │
│  Drivers:                                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ lib/pq         — PostgreSQL (mature, stable)         │  │
│  │ jackc/pgx      — PostgreSQL (faster, more features)  │  │
│  │ go-sql-driver  — MySQL                               │  │
│  │ mattn/sqlite3  — SQLite (requires CGo)               │  │
│  │ modernc/sqlite — SQLite (pure Go, no CGo)            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Higher-level:                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ sqlx          — extensions to database/sql            │  │
│  │                 StructScan, NamedExec, Get, Select    │  │
│  │ GORM          — full ORM (migrations, relations)     │  │
│  │ sqlc          — generates type-safe Go from SQL       │  │
│  │ ent           — Facebook's entity framework for Go   │  │
│  │ goose/migrate — database migration tools             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  sqlx example:                                             │
│  db.Get(&user, "SELECT * FROM users WHERE id=$1", 1)       │
│  db.Select(&users, "SELECT * FROM users")                  │
│  db.NamedExec("INSERT INTO users (name) VALUES (:name)",   │
│               map[string]any{"name": "Alice"})             │
└────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **What is `sql.DB` in Go?**
   - It's a connection pool, not a single connection. It manages opening, closing, and reusing connections. It's safe for concurrent use from multiple goroutines. `sql.Open` creates the pool but doesn't connect — `Ping` verifies connectivity.

2. **What's the difference between `Query`, `QueryRow`, and `Exec`?**
   - `Query` returns `*sql.Rows` (multiple rows, must close). `QueryRow` returns `*sql.Row` (single row, returns `ErrNoRows` if empty). `Exec` returns `sql.Result` (for INSERT/UPDATE/DELETE — affected rows, last insert ID).

3. **How do you prevent SQL injection in Go?**
   - Use parameterized queries: `db.Query("SELECT * FROM users WHERE id = $1", id)`. Never concatenate user input into SQL strings. The driver handles escaping.

4. **How do you handle transactions?**
   - `tx, err := db.BeginTx(ctx, opts)`, then `tx.ExecContext(...)`, `tx.QueryContext(...)`. Commit with `tx.Commit()`. Use `defer tx.Rollback()` — it's a no-op after commit. All operations in a transaction use the same connection.

5. **How do you handle NULL columns?**
   - Use `sql.NullString`/`sql.NullInt64` (has `.Valid` field), or pointer types (`*string`, `*int`). Pointers work better with JSON marshaling (nil becomes `null`).

6. **What connection pool settings should you configure?**
   - `SetMaxOpenConns` (default unlimited — always set this!), `SetMaxIdleConns`, `SetConnMaxLifetime` (5-15 min to handle DB failovers), `SetConnMaxIdleTime`. Monitor with `db.Stats()`.

7. **Why must you always close `*sql.Rows`?**
   - `rows.Close()` returns the connection to the pool. Without closing, the connection is leaked — eventually the pool runs out and queries block. Always `defer rows.Close()` immediately after `Query`.

8. **What is `sql.ErrNoRows`?**
   - Returned by `QueryRow().Scan()` when the query matches zero rows. Not returned by `Query()` — that returns empty `Rows` with `Next()` returning false. Check with `errors.Is(err, sql.ErrNoRows)`.

---

## Interview Problems & Solutions

### Problem 1 — Build a Safe CRUD Service

**Problem:** Implement a complete user CRUD service with proper error handling, context usage, connection pool configuration, and transaction support for a "transfer credits" operation.

```go
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"time"
)

var ErrNotFound = errors.New("not found")
var ErrInsufficientCredits = errors.New("insufficient credits")

type UserRecord struct {
	ID        int
	Name      string
	Email     string
	Credits   int
	CreatedAt time.Time
}

type UserService struct {
	db *sql.DB
}

func NewUserService(db *sql.DB) *UserService {
	// Configure pool
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(10)
	db.SetConnMaxLifetime(5 * time.Minute)
	return &UserService{db: db}
}

func (s *UserService) GetUser(ctx context.Context, id int) (*UserRecord, error) {
	var u UserRecord
	err := s.db.QueryRowContext(ctx,
		`SELECT id, name, email, credits, created_at FROM users WHERE id = $1`, id,
	).Scan(&u.ID, &u.Name, &u.Email, &u.Credits, &u.CreatedAt)

	if errors.Is(err, sql.ErrNoRows) {
		return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
	}
	if err != nil {
		return nil, fmt.Errorf("get user %d: %w", id, err)
	}
	return &u, nil
}

// TransferCredits atomically moves credits between users
func (s *UserService) TransferCredits(ctx context.Context, fromID, toID, amount int) error {
	if amount <= 0 {
		return fmt.Errorf("amount must be positive")
	}

	tx, err := s.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback()

	// Lock sender row and check balance
	var senderCredits int
	err = tx.QueryRowContext(ctx,
		`SELECT credits FROM users WHERE id = $1 FOR UPDATE`, fromID,
	).Scan(&senderCredits)
	if errors.Is(err, sql.ErrNoRows) {
		return fmt.Errorf("sender %d: %w", fromID, ErrNotFound)
	}
	if err != nil {
		return err
	}
	if senderCredits < amount {
		return fmt.Errorf("user %d has %d credits: %w", fromID, senderCredits, ErrInsufficientCredits)
	}

	// Debit sender
	_, err = tx.ExecContext(ctx,
		`UPDATE users SET credits = credits - $1 WHERE id = $2`, amount, fromID)
	if err != nil {
		return err
	}

	// Credit receiver
	result, err := tx.ExecContext(ctx,
		`UPDATE users SET credits = credits + $1 WHERE id = $2`, amount, toID)
	if err != nil {
		return err
	}
	affected, _ := result.RowsAffected()
	if affected == 0 {
		return fmt.Errorf("receiver %d: %w", toID, ErrNotFound)
	}

	// Record transaction
	_, err = tx.ExecContext(ctx,
		`INSERT INTO credit_transfers (from_id, to_id, amount, created_at) VALUES ($1, $2, $3, NOW())`,
		fromID, toID, amount)
	if err != nil {
		return err
	}

	return tx.Commit()
}

func main() {
	fmt.Println("UserService with transactions — requires live database")
	fmt.Println("Key patterns demonstrated:")
	fmt.Println("  1. Pool configuration in constructor")
	fmt.Println("  2. ErrNoRows → custom sentinel error")
	fmt.Println("  3. Serializable isolation for transfers")
	fmt.Println("  4. SELECT FOR UPDATE to lock rows")
	fmt.Println("  5. defer tx.Rollback() safety net")
}
```

**Key Points:**
- `SELECT FOR UPDATE` locks the row within the transaction to prevent concurrent modifications
- Sentinel errors (`ErrNotFound`, `ErrInsufficientCredits`) enable callers to handle specific cases with `errors.Is`
- `defer tx.Rollback()` is a safety net — no-op after commit, ensures rollback on any error path
- Context propagation allows request timeouts to cancel long transactions
