Here's a structured summary of the `copier` library behavior:

## 1. Deep Copy Behavior
```go
copier.Copy(&payroll, &employee)
```
- Creates a **deep copy** from `employee` to `payroll`
- Changes to `employee` after copying **will not affect** `payroll`

## 2. Unexported Fields Handling
```go
type A struct {
    secret int  // unexported (lowercase)
}
```
- **Can read** unexported values
- **Cannot write** to unexported fields
- Result: Unexported fields in destination will be **zero value** (e.g., `0` for `int`)

## 3. Field Tags (Target Struct)
```go
type Target struct {
    ID int `copier:"must"`  // Tag in destination field
}
```
- Tags belong in the **target** struct
- `copier:"must"` ensures the field must be present in source

## 4. Type Conversion Matrix

| Source Type | Destination Type | Works? | Notes |
|-------------|----------------|--------|-------|
| `int` | `string` | ❌ No | No auto-conversion |
| `string` | `int` | ❌ No | No auto-conversion |
| `int` | `int64` | ✅ Yes | Numeric conversion |
| `float64` | `int` | ✅ Yes | Numeric conversion |
| `bool` | `bool` | ✅ Yes | Direct match |
| `string` | `[]byte` | ✅ Yes | Special case |
| `time.Time` | `string` | ❌ No | Requires custom converter |

> **Key Principle**: `copier` does NOT attempt automatic conversion between incompatible types, with a few basic exceptions (numeric types, string→bytes, bool→bool).
