# SQL Injection & Prepared Statements

## What is SQL Injection?
Attackers insert malicious SQL code into user inputs to manipulate database queries.

**Vulnerable Code (Go):**
```go
// BAD - Never do this!
userInput := "admin'; DROP TABLE users; --"
query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", userInput)
// Results in: SELECT * FROM users WHERE username = 'admin'; DROP TABLE users; --'
```

**Attack Examples:**
```sql
-- Data theft: ' OR '1'='1
-- Auth bypass: admin'--
```

## Prepared Statements
Separate SQL code from data. Database compiles structure first, then safely inserts data.

**Safe Code (Go):**
```go
// GOOD - Using prepared statements
stmt, err := db.Prepare("SELECT * FROM users WHERE username = ? AND password = ?")
defer stmt.Close()
rows, err := stmt.Query(username, password)
```

## Why It Works - Mad Libs Analogy
**Mad Libs:** "The [ADJECTIVE] cat [VERB] over the [NOUN]"
- Structure is fixed
- You can only fill blanks with data
- Can't change the sentence structure

**Vulnerable:**
```go
username := "admin'; DROP TABLE users; --"
query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", username)
// Database sees TWO commands!
```

**Safe:**
```go
// Step 1: Structure compiled
stmt, _ := db.Prepare("SELECT * FROM users WHERE name = ?")
// Step 2: Data inserted safely
rows, _ := stmt.Query("admin'; DROP TABLE users; --")
// Malicious string treated as DATA only
```

## Go Examples

**Basic Usage:**
```go
func getUser(db *sql.DB, userID int) error {
    stmt, err := db.Prepare("SELECT name, email FROM users WHERE id = ?")
    if err != nil {
        return err
    }
    defer stmt.Close()
    
    var name, email string
    err = stmt.QueryRow(userID).Scan(&name, &email)
    return err
}
```

**Input Validation:**
```go
func validateEmail(email string) bool {
    pattern := `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
    matched, _ := regexp.MatchString(pattern, email)
    return matched
}
```

## Key Protection
1. Always use prepared statements
2. Validate inputs
3. Use least privilege database users
4. Never concatenate user input into SQL

**Remember:** Once SQL structure is compiled, it's locked. User input can't change it into different commands.
