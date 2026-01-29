## Step 1: Understanding the Problem (Why Atomicity Exists)

Imagine you're transferring $100 from Account A to Account B. This requires two operations:
1. Subtract $100 from Account A
2. Add $100 to Account B

**What if the system crashes after step 1 but before step 2?** You'd lose $100! This is exactly the problem atomicity solves.

## Step 2: What is Atomicity?

**Atomicity** means a transaction is **"all or nothing"**. Either:
- ALL operations in the transaction complete successfully, OR
- NONE of them take effect (the database rolls back to its previous state)

The transaction is treated as a single, indivisible unit (an "atom").

## Step 3: Basic PostgreSQL Setup

Let's create a simple banking example:

```sql
-- Create accounts table
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    balance DECIMAL(10, 2)
);

-- Insert sample data
INSERT INTO accounts (name, balance) VALUES 
    ('Alice', 1000.00),
    ('Bob', 500.00);

-- Check initial state
SELECT * FROM accounts;
```

**Output:**
```
 id | name  | balance  
----+-------+----------
  1 | Alice | 1000.00
  2 | Bob   |  500.00
```

## Step 4: Transaction Without Atomicity (The Problem)

If we run operations separately without a transaction:

```sql
-- Deduct from Alice
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';

-- Imagine system crashes here! 💥

-- Add to Bob (never executes)
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
```

**Result:** Alice loses $100, Bob gains nothing. Money disappeared!

## Step 5: Transaction With Atomicity (The Solution)

```sql
-- Start transaction
BEGIN;

-- Deduct from Alice
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';

-- Add to Bob
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';

-- Commit the transaction
COMMIT;
```

Now it's atomic: Both updates happen together, or neither happens.

## Step 6: Demonstrating Rollback

Let's see what happens when something goes wrong:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- Alice's balance is temporarily 900

UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
-- Bob's balance is temporarily 600

-- Oops! We changed our mind or an error occurred
ROLLBACK;

-- Check balances
SELECT * FROM accounts;
```

**Result:** Both accounts return to original values (1000, 500). The transaction was rolled back atomically.

## Step 7: Automatic Rollback on Error

PostgreSQL automatically rolls back if an error occurs:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';

-- This will cause an error (division by zero)
UPDATE accounts SET balance = balance / 0 WHERE name = 'Bob';

COMMIT; -- This won't execute due to the error

-- Check state
SELECT * FROM accounts;
```

**Result:** Alice's balance is unchanged. The entire transaction was rolled back automatically.

## Step 8: Real-World Example - E-commerce Order

```sql
-- Create tables
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    stock INT
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INT,
    quantity INT,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, stock) VALUES ('Laptop', 10);

-- Atomic order placement
BEGIN;

-- Check if enough stock
DO $$
DECLARE
    current_stock INT;
BEGIN
    SELECT stock INTO current_stock FROM products WHERE id = 1;
    
    IF current_stock < 2 THEN
        RAISE EXCEPTION 'Insufficient stock';
    END IF;
END $$;

-- Decrease stock
UPDATE products SET stock = stock - 2 WHERE id = 1;

-- Create order
INSERT INTO orders (product_id, quantity) VALUES (1, 2);

COMMIT;

-- Verify
SELECT * FROM products;
SELECT * FROM orders;
```

Both the stock update and order creation happen atomically.

## Step 9: Savepoints (Partial Rollback)

PostgreSQL allows you to rollback to specific points within a transaction:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 50 WHERE name = 'Alice';
-- Alice: 950

SAVEPOINT my_savepoint;

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- Alice: 850

-- Oops, that was too much. Roll back to savepoint
ROLLBACK TO SAVEPOINT my_savepoint;
-- Alice: 950 again

UPDATE accounts SET balance = balance - 25 WHERE name = 'Alice';
-- Alice: 925

COMMIT;

SELECT * FROM accounts WHERE name = 'Alice';
```

**Result:** Alice has 925 (only the 50 and 25 deductions applied).

## Step 10: How PostgreSQL Implements Atomicity

PostgreSQL uses **Write-Ahead Logging (WAL)**:

1. **Before making changes**: PostgreSQL writes what it's about to do to a log file
2. **Makes changes**: Updates the actual data
3. **On COMMIT**: Marks the transaction as complete in the log
4. **On ROLLBACK or crash**: Uses the log to undo uncommitted changes

You can see WAL activity:

```sql
-- Check current WAL location
SELECT pg_current_wal_lsn();

-- View transaction status
SELECT * FROM pg_stat_activity WHERE state = 'active';
```

## Step 11: Testing Atomicity with Constraints

```sql
CREATE TABLE transfers (
    id SERIAL PRIMARY KEY,
    from_account VARCHAR(50),
    to_account VARCHAR(50),
    amount DECIMAL(10, 2),
    CHECK (amount > 0)  -- Constraint
);

BEGIN;

INSERT INTO transfers (from_account, to_account, amount) 
VALUES ('Alice', 'Bob', 100);

-- This violates the constraint
INSERT INTO transfers (from_account, to_account, amount) 
VALUES ('Bob', 'Charlie', -50);

COMMIT;

-- Check results
SELECT * FROM transfers;
```

**Result:** Empty table. The entire transaction rolled back because the second INSERT violated the CHECK constraint.

## Step 12: Isolation Levels and Atomicity

Atomicity works at all isolation levels:

```sql
-- Check current isolation level
SHOW transaction_isolation;

-- Set isolation level
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';

COMMIT;
```

Regardless of isolation level, the transaction remains atomic.

---

## Key Takeaways

1. **Atomicity = All or Nothing**: Every transaction either completes fully or has no effect
2. **Use BEGIN/COMMIT**: Wrap related operations in transactions
3. **ROLLBACK**: Manually undo changes when needed
4. **Automatic rollback**: Errors automatically trigger rollback
5. **Savepoints**: Enable partial rollback within transactions
6. **WAL**: PostgreSQL's mechanism for ensuring atomicity


