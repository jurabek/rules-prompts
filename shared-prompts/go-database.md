# Go Database Best Practices

## Database Libraries

### SQL Operations
- Use **sqlx** for enhanced SQL operations.
- Use **standard sql package** as foundation.
- Leverage `sqlx` named, queryx, operations instead of parameterized queries for complex struct mappings.
- Define `DbExecutor` global interface in db layer for all DB operation that is compatible with `*sqlx.DB` and `*sqlx.Tx`, and use this interface on each repository methods.
- Use always lower case SQL syntax for better readability. e.g `select`, `from`, `where` etc.
- Avoid SQL functions and triggers in favor of application logic.

### Guide to sqlx
- Always use `QueryxContext`, `SelectContext`, `GetContext`, `NamedExecContext` methods instead of their non-context counterparts.
- Use `PrepareNamedContext` for prepared statements with named parameters.
- Use `sqlx.In` for queries with `IN` clause and slice parameters.
- Always use StructScan for mapping query results to structs, and do not use `rows.Scan` by passing each field separately, unless it is necessary.
```go
      type Place struct {
        Country       string
        City          sql.NullString
        TelephoneCode int `db:"telcode"`
    }
     
    rows, err := db.QueryxContext(ctx, "select * from place") // or sqlx.NamedQueryContext(ctx, dbExec, query, args) for named queries
    for rows.Next() {
        var p Place
        err = rows.StructScan(&p)
    }
```
- If querying a single row, use `GetContext` or `QueryRowContext` for better error handling and struct mapping.
```go
	var p Place
	err := db.GetContext(ctx, &p, "select * from place where country=$1", country)

	// or 
	var p Place
    err := db.QueryRowx("SELECT city, telcode FROM place LIMIT 1").StructScan(&p)
	
```
- If there need for mapping json/jsonb columns to struct/map fields, implement `sql.Scanner` and `driver.Valuer` interfaces for custom types.
```go
	type JSONB map[string]interface{}

	func (j *JSONB) Scan(value any) error {
		switch v := val.(type) {
		case []byte:
			json.Unmarshal(v, &j)
			return nil
		case string:
			json.Unmarshal([]byte(v), &j)
			return nil
		default:
			return fmt.Errorf("unsupported type: %T", v)
		}
	}

	func (j *JSONB) Value() (driver.Value, error) {
		return json.Marshal(j)
	}
```
- When executing Update, Insert or Delete statements, use `NamedExecContext` for better readability and maintainability, if there only single parameter just use $1, no need to use named parameters.
```go
type User struct {
	ID    string `db:"id"`
	Name  string `db:"name"`
	Email string `db:"email"`
}

// named update or insert
func UpdateUser(ctx context.Context, dbEx db.DbExecutor, user User) error {
	updateUserQuery := `update users set name = :name, email = :email where id = :id`
	_, err := dbEx.NamedExecContext(ctx, updateUserQuery, user)
	if err != nil {
		return fmt.Errorf("failed to update user: %w", err)
	}
	return nil
}

```
### Example Usage
```go

// database/db.go
var (
	_ DbExecutor = (*sqlx.DB)(nil)
	_ DbExecutor = (*sqlx.Tx)(nil)
)

type DbExecutor interface {
	sqlx.ExtContext
	GetContext(ctx context.Context, dest any, query string, args ...any) error
	SelectContext(ctx context.Context, dest any, query string, args ...any) error
	NamedExecContext(ctx context.Context, query string, arg any) (sql.Result, error)
	PrepareNamedContext(ctx context.Context, query string) (*sqlx.NamedStmt, error)
	QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
	Rebind(query string) string
	Select(dest any, query string, args ...any) error
}

// repositories/repo.go
type Repository struct {
    db db.DbExecutor
}

func (r *Repository) GetUser(ctx context.Context, dbExec db.DbExecutor, id string) (*User, error) {
    return GetUser(ctx, dbExec, id)
}

func GetUser(ctx context.Context, dbExec db.DbExecutor, id string) (*User, error) {
    var user User
    query := `select id, name, email from users where id = $1`
    
    err := dbExec.GetContext(ctx, &user, query, id)
    if err != nil {
        return nil, fmt.Errorf("get user: %w", err)
    }
    
    return &user, nil
}
```

## Database Migrations

### Migration Tool
- Use **goose** for creating and applying migrations e.g `goose create migration_name` 
- Store migrations in version control.
- Keep migrations idempotent when possible.


### Migration Best Practices
- Write both up and down migrations.
- Test migrations in development before production.
- Keep migrations small and focused.
- Never modify existing migrations after deployment.
- Use descriptive migration names.
- When writing migrations, avoid complex logic; prefer simple SQL statements, and do not create SQL functions or triggers.

## Repository Pattern

### Interface Definition

Define repository interfaces on the consumer side, based on which methods are used on that particular consumer.
```go
// In service package
type userGetter interface {
    GetUser(ctx context.Context, id string) (*User, error)
}
```

### Context Propagation
- Always pass `context.Context` to database operations.
- Use context for:
  - Query cancellation
  - Timeout enforcement
  - Tracing propagation
  - Transaction management

## Transaction Management
- Apply transaction management where ACID operation is needed.

### Transaction Pattern
```go
type Transactioner struct {
	DB *sqlx.DB
}

func (tr *Transactioner) WithTx(ctx context.Context, opts *sql.TxOptions, fn func(s *sqlx.Tx) error) error {
	tx, err := tr.DB.BeginTxx(ctx, opts)
	if err != nil {
		return fmt.Errorf("begin transaction failed: %w", err)
	}

	defer func() {
		if p := recover(); p != nil {
			// a panic occurred, rollback and re-panic
			if rbErr := tx.Rollback(); rbErr != nil {
				fmt.Println("rollback err :%+v", err)
			}
			panic(p)
		}
	}()
	// we should pass nil into transaction beginner, to avoid starting new transaction within transaction
	err = fn(tx)
	if err != nil {
		if rbErr := tx.Rollback(); rbErr != nil {
			return fmt.Errorf("transaction failed: %w rollback failed: %w", err, rbErr)
		}
		return err
	}
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit failed: %w", err)
	}
	return nil
}
```

## Error Handling

### Database Errors
- Wrap database errors with context.
- Check for specific error types (e.g., `sql.ErrNoRows`).
- Provide meaningful error messages.

## Query Optimization

### Best Practices
- Use prepared statements for repeated queries.
- Implement proper indexing strategies.
- Use query parameters to prevent SQL injection.
- Avoid N+1 query problems.
- Use EXPLAIN ANALYZE to understand query performance.

### Efficient Queries
```go
// Good: Single query with JOIN and use StructScan by defining custom struct including fields from both tables
func (r *Repository) GetUsersWithOrders(ctx context.Context) ([]UserWithOrders, error) {
    query := `
        select u.id, u.name, o.id as order_id, o.total
        from users u
        left join orders o on u.id = o.user_id
    `
    // ... implementation
}

// Avoid: N+1 query pattern
// Don't fetch users then loop to fetch orders for each
```

## Connection Management

### Connection Pool
```go
func NewDB(connString string) (*sqlx.DB, error) {
    db, err := sqlx.Connect("postgres", connString)
    if err != nil {
        return nil, fmt.Errorf("connect to database: %w", err)
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)
    
    return db, nil
}
```

### Resource Cleanup
- Always close database connections properly.
- Use defer for cleanup when appropriate.
- Handle connection pool exhaustion gracefully.

## Observability

### Tracing Database Operations
- Use `github.com/XSAM/otelsql` and wrap the `sql.DB`
```go
    db := otelsql.OpenDB(stdlib.GetConnector(*pgxCfg),
		otelsql.WithAttributes(semconv.DBNamespace(componentName)),
		otelsql.WithSpanOptions(otelsql.SpanOptions{
			OmitConnResetSession: true,
			OmitRows:             true,
		}),
	)
	ddx := sqlx.NewDb(db, "pgx")

	err = otelsql.RegisterDBStatsMetrics(db, otelsql.WithAttributes(
		semconv.DBSystemPostgreSQL,
		semconv.DBNamespace(componentName),
	))
```

### Query Logging
- Log slow queries for performance monitoring.
- Include query parameters in logs (sanitized).
- Track query execution times.

## Testing

### Test Database Setup
- Use TestContainers for integration tests.
- Create isolated test databases.
- Reset database state between tests.

### Repository Testing
```go
func TestRepository_GetUser(t *testing.T) {
    // Setup TestContainer with PostgreSQL
    ctx := context.Background()
    container, db := setupTestDB(t, ctx)
    defer container.Terminate(ctx)
    
    repo := NewPostgresUserRepository(db)
    
    // Test implementation
}
```

### Transaction Testing
- Test transaction rollback scenarios.
- Verify isolation levels.
- Test concurrent access patterns.