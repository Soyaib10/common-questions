Here is the **restructured, clean version** of your document.  

I have grouped **each anomaly together with**:

- its definition  
- practical business example  
- disaster/consequence explanation  
- the corresponding isolation level that solves it  
- how the solution works  
- all related step-by-step table scenarios (bug + fix)  
- important database nuances mentioned for that anomaly  
- any extra talking points that appeared in the original text about it  

Everything stays at full length — no shortening, no removal of explanations, stories, warnings, or details.

# Database Isolation Levels – A Complete Practical Guide

## Cheat Sheet – Quick Reference

### The Problems (The Bugs / Anomalies)
1. **Dirty Read**  
   Reading data that hasn't been saved (committed) yet.

2. **Non-Repeatable Read**  
   Reading a row, then finding that the **data inside that row** changed when you read it again.

3. **Phantom Read**  
   Reading a list of rows (like a search result), then finding that a **new row** appeared or disappeared when you searched again.

4. **Serialization Anomaly (Write Skew)**  
   Two transactions follow valid logic individually, but when run concurrently they create a logical/business error.

### The Isolation Levels (The Solutions)
1. **Read Uncommitted** → The "I don't care" level. No protection.  
2. **Read Committed** → The "Standard" level. You only see saved data.  
3. **Repeatable Read** → The "Stable" level. Data you are looking at stays frozen for you.  
4. **Serializable** → The "Strict" level. Everything happens as if there is only one user at a time.

## Online Store Example Used Throughout
**Table:** `Product`  
**Example row:** `ID: 1, Name: iPhone, Stock: 10`

**Other tables (used later):**  
- `Orders` (for Phantom Read)  
- `Doctors` or voucher `Count` table (for Write Skew)

---

## Problem 1: Dirty Read + Solution: Read Committed

**Concept:** You trust (and act on) data that might not be real / might never become permanent.

### Practical Example – Online Store Stock
- **Transaction A (Manager):** Updates iPhone stock from **10 → 20**… but hasn’t committed yet.  
- **Transaction B (Customer):** Wants to buy 15 phones → reads stock level.

**The Disaster:**
1. Manager updates stock to 20 (uncommitted).  
2. Customer reads → sees **20**.  
3. Customer places order for 15 phones.  
4. Manager’s computer crashes → database **rollbacks** → stock reverts to **10**.  
**Result:** You sold 15 phones you don’t have → shipping disaster.

### Solution: Read Committed
This level **specifically prevents Dirty Reads**.

**How it works:**  
The database **hides uncommitted changes**. If the manager updated to 20 but didn’t commit, the customer still sees the last **committed** value (10).

**Outcome:** Customer sees 10 → cannot buy 15 → correct behavior.

### Scenario 1 – The Bug (Read Uncommitted)
**Setup (run once):**
```sql
CREATE TABLE Product (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    stock INT
);
INSERT INTO Product VALUES (1, 'iPhone', 10);
```

| Step | Time  | Transaction A (Manager)                                 | Transaction B (Customer)                                | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;`            | `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;`            | Lower the shields                                                           |
| 2    | 00:02 | `BEGIN TRANSACTION;`                                           |                                                                | Manager starts                                                              |
| 3    | 00:03 | `UPDATE Product SET stock = 20 WHERE id = 1;`                  |                                                                | Stock = 20 in memory, not committed                                         |
| 4    | 00:04 |                                                                | `BEGIN TRANSACTION;`                                           | Customer logs in                                                            |
| 5    | 00:05 |                                                                | `SELECT stock FROM Product WHERE id = 1;`                      | **DIRTY READ** → returns 20                                                 |
| 6    | 00:06 | `ROLLBACK;`                                                    |                                                                | Stock reverts to 10                                                         |
| 7    | 00:07 |                                                                | *Customer logic…*                                              | Uses data that never really existed                                         |

**Result:** Transaction B hallucinated data → inconsistency.

### Scenario 2 – The Fix (Read Committed)
**Default in:** PostgreSQL, Oracle, SQL Server

| Step | Time  | Transaction A (Manager)                                 | Transaction B (Customer)                                | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;`              | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;`              | Shields up                                                                  |
| 2    | 00:02 | `BEGIN TRANSACTION;`                                           |                                                                | Manager starts                                                              |
| 3    | 00:03 | `UPDATE Product SET stock = 20 WHERE id = 1;`                  |                                                                | Stock = 20 in memory, not committed                                         |
| 4    | 00:04 |                                                                | `BEGIN TRANSACTION;`                                           | Customer logs in                                                            |
| 5    | 00:05 |                                                                | `SELECT stock FROM Product WHERE id = 1;`                      | Returns **10** (last committed value)                                       |
| 6    | 00:06 | `COMMIT;`                                                      |                                                                | Stock officially becomes 20                                                 |
| 7    | 00:07 |                                                                | `SELECT stock FROM Product WHERE id = 1;`                      | Now sees **20**                                                             |

**Critical detail – Blocking vs Old Data (MVCC):**
- **Blocking** (pessimistic – e.g. classic SQL Server): Transaction B **waits** until A commits.  
- **MVCC** (PostgreSQL, Oracle, MySQL InnoDB): Transaction B instantly sees the **old committed version** (10) → no waiting.

Both approaches eliminate Dirty Reads.

---

## Problem 2: Non-Repeatable Read + Solution: Repeatable Read

**Concept:** You read a value, do some logic/calculation based on it, then read the same row again → the value changed (even though it was already committed).

### Practical Example – Accounting Report
- **Transaction A (Accountant):** Generating monthly report.  
- **Transaction B (Sales System):** Processing a sale.

**The Disaster:**
1. Accountant reads stock: **10** → calculates value `$1000 × 10 = $10,000`.  
2. Sales sells 5 phones → commits → stock = **5**.  
3. Accountant double-checks stock → sees **5**.  
**Result:** Report says value = $10,000 but stock count = 5 → inconsistent numbers.

### Solution: Repeatable Read
**How it works:**  
When you first read the row, the database gives you a **personal snapshot** (or locks it in a way that preserves your view). Even after committed changes happen, you keep seeing your original value.

**Outcome:** Accountant’s report stays internally consistent.

### Scenario 1 – The Bug (Read Committed – default in many DBs)

| Step | Time  | Transaction A (Accountant)                              | Transaction B (Sales)                                   | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;`              | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;`              | Standard level                                                              |
| 2    | 00:02 | `BEGIN TRANSACTION;`                                           |                                                                | Accountant starts report                                                    |
| 3    | 00:03 | `SELECT stock FROM Product WHERE id = 1;`                      |                                                                | Sees **10**                                                                 |
| 4    | 00:04 |                                                                | `BEGIN TRANSACTION;`                                           | Sale starts                                                                 |
| 5    | 00:05 |                                                                | `UPDATE Product SET stock = 5 WHERE id = 1;`                   | Stock → 5                                                                   |
| 6    | 00:06 |                                                                | `COMMIT;`                                                      | Change permanent                                                            |
| 7    | 00:07 | `SELECT stock FROM Product WHERE id = 1;`                      |                                                                | **NON-REPEATABLE READ** → sees **5**                                        |
| 8    | 00:08 | *Accountant logic…*                                            |                                                                | Confusion: calculated on 10, now stock = 5                                  |

### Scenario 2 – The Fix (Repeatable Read – default in MySQL InnoDB)

| Step | Time  | Transaction A (Accountant)                              | Transaction B (Sales)                                   | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;`             | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;`             | Snapshot mode                                                               |
| 2    | 00:02 | `BEGIN TRANSACTION;`                                           |                                                                | Accountant starts → **snapshot created**                                    |
| 3    | 00:03 | `SELECT stock FROM Product WHERE id = 1;`                      |                                                                | Sees **10**                                                                 |
| 4    | 00:04 |                                                                | `BEGIN TRANSACTION;`                                           | Sale starts                                                                 |
| 5    | 00:05 |                                                                | `UPDATE Product SET stock = 5 WHERE id = 1;`                   | Stock → 5                                                                   |
| 6    | 00:06 |                                                                | `COMMIT;`                                                      | Real DB now 5                                                               |
| 7    | 00:07 | `SELECT stock FROM Product WHERE id = 1;`                      |                                                                | Still sees **10** (snapshot)                                                |
| 8    | 00:08 | `COMMIT;`                                                      |                                                                | Accountant finishes                                                         |
| 9    | 00:09 | (new transaction) `SELECT stock …`                             |                                                                | Now sees **5**                                                              |

**Key takeaway – Snapshot concept:**  
In MVCC databases (PostgreSQL, MySQL InnoDB), Repeatable Read gives a frozen view from the start of your transaction.

---

## Problem 3: Phantom Read + Solution: Serializable

**Concept:** The **set of rows** matching your query changes because new rows appear (or disappear) — even though individual rows you already read didn’t change.

### Practical Example – Shipping Department
- **Transaction A (Shipping Manager):** Counting & printing labels for `PENDING` orders.  
- **Transaction B (New Customer):** Placing new order.

**The Disaster:**
1. Manager queries `PENDING` orders → **50** rows → starts printing 50 labels.  
2. New order inserted & committed → now 51 pending.  
3. Manager queries again → sees **51**.  
**Result:** Short one label → one order left in limbo.

### Solution: Serializable (or strong Snapshot Isolation)
**How it works:**  
The database protects the **range** / predicate. If you queried “all pending orders”, no one can insert/delete rows that would affect that result until you finish.

**Outcome:** New order is **blocked** until manager finishes.

### Scenario 1 – The Bug (Repeatable Read)

**Initial data:**
```sql
-- Orders table
-- id | customer | amount | status
-- 1  | Alice    | 100    | 'PENDING'
-- 2  | Bob      | 150    | 'PENDING'
```

| Step | Time  | Transaction A (Shipping Manager)                        | Transaction B (New Customer)                            | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;`             | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;`             |                                                                             |
| 2    | 00:02 | `BEGIN;`                                                       |                                                                |                                                                             |
| 3    | 00:03 | `SELECT COUNT(*) FROM Orders WHERE status = 'PENDING';`        |                                                                | Sees **2**                                                                  |
| 4    | 00:04 |                                                                | `BEGIN;`                                                       |                                                                             |
| 5    | 00:05 |                                                                | `INSERT INTO Orders VALUES (3, 'Charlie', 200, 'PENDING');`    | New order                                                                   |
| 6    | 00:06 |                                                                | `COMMIT;`                                                      |                                                                             |
| 7    | 00:07 | `SELECT COUNT(*) FROM Orders WHERE status = 'PENDING';`        |                                                                | **PHANTOM READ** → sees **3**                                               |
| 8    | 00:08 | `UPDATE Orders SET status = 'SHIPPED' WHERE status = 'PENDING';`|                                                                | Thought updating 2, actually updated 3                                      |

### Scenario 2 – The Fix (Serializable)

| Step | Time  | Transaction A (Shipping Manager)                        | Transaction B (New Customer)                            | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;`                | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;`                |                                                                             |
| 2    | 00:02 | `BEGIN;`                                                       |                                                                | Range lock on pending orders                                                |
| 3    | 00:03 | `SELECT COUNT(*) FROM Orders WHERE status = 'PENDING';`        |                                                                | Sees **2**                                                                  |
| 4    | 00:04 |                                                                | `BEGIN;`                                                       |                                                                             |
| 5    | 00:05 |                                                                | `INSERT … 'PENDING';`                                          | **Blocked / waits**                                                         |
| 6    | 00:06 | `COMMIT;`                                                      |                                                                | Range lock released                                                         |
| 7    | 00:07 |                                                                | (unblocked) `COMMIT;`                                          | Now allowed                                                                 |

**Important Nuance – Database Differences:**
- **PostgreSQL:** `REPEATABLE READ` already prevents phantoms (via strong Snapshot Isolation).  
- **MySQL InnoDB:** `REPEATABLE READ` prevents phantoms on plain `SELECT`, but can allow them with certain locking reads.  
- **SQL standard / cross-DB guarantee:** Only `SERIALIZABLE` is 100% reliable everywhere.

---

## Problem 4: Write Skew (Serialization Anomaly) + Solution: Serializable

**Concept:** Two transactions read the same premise, make independent valid decisions, update **different rows** → combined result violates business rule.

### Practical Examples
1. **Hospital – Doctors on call**  
   Rule: Always ≥ 1 doctor on call. Currently 2 (Alice + Bob). Both want to leave.

2. **Voucher / Budget**  
   Limit: max 10 vouchers. Currently claimed: 9.  
   Alice & Bob both see 9 → both claim → ends with 11.

### The Disaster – Doctors Example

| Step | Time  | Transaction A (Alice)                                   | Transaction B (Bob)                                     | Explanation                                                                 |
|------|-------|----------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------|
| 1    | 00:01 | `BEGIN;`                                                       | `BEGIN;`                                                       |                                                                             |
| 2    | 00:02 | `SELECT COUNT(*) FROM Doctors WHERE on_call = true;`           |                                                                | Sees **2** → "I can leave"                                                  |
| 3    | 00:03 |                                                                | `SELECT COUNT(*) FROM Doctors WHERE on_call = true;`           | Sees **2** → "I can leave"                                                  |
| 4    | 00:04 | `UPDATE Doctors SET on_call = false WHERE name = 'Alice';`     |                                                                |                                                                             |
| 5    | 00:05 |                                                                | `UPDATE Doctors SET on_call = false WHERE name = 'Bob';`       |                                                                             |
| 6    | 00:06 | `COMMIT;`                                                      | `COMMIT;`                                                      | Now **0** doctors on call – rule broken                                     |

**Why Repeatable Read fails:**  
No dirty read, no non-repeatable read, no phantom → but still wrong outcome.  
They read consistent data but modified **different** rows → no direct conflict detected.

### Solution: Serializable

**How it works (conceptual):**
Database detects that both transactions’ writes depend on the **same read predicate** (count of on-call doctors / count of vouchers).  
It forces **serial execution** semantics → one transaction will fail with **serialization error**.

**Outcome – Doctors Example:**
- Alice reads count (2) → proceeds  
- Bob reads count (2) → database detects conflict → **aborts Bob**  
- Bob retries → now sees count = 1 → cannot leave

Same logic applies to voucher example → only one person gets the last voucher.

---

## How Modern Databases Achieve This Without Being Slow – MVCC

**MVCC = Multi-Version Concurrency Control**

**Core idea:**  
Instead of overwriting rows, the database keeps **multiple versions** with timestamps/transaction IDs.

When a transaction reads:
- It gets the version that was visible **at the start of its transaction** (or according to its isolation rules).

**Key Benefits:**
- Readers never block writers  
- Writers never block readers  
- Only conflicting **writers** on the **same row** wait/block/abort  
- Deletes are soft (mark as deleted at time X) → old transactions still see the row  
- Old versions cleaned later by **vacuum** (PostgreSQL) / **purge** (MySQL)

This is why **Repeatable Read** and **Serializable** can be reasonably fast in PostgreSQL, MySQL InnoDB, Oracle, etc.

---

## Final Summary Table

| Isolation Level     | Prevents Dirty Read? | Prevents Non-Repeatable Read? | Prevents Phantom Read? | Prevents Write Skew? | Typical Use Case                              |
|---------------------|----------------------|-------------------------------|------------------------|----------------------|-----------------------------------------------|
| Read Uncommitted    | No                   | No                            | No                     | No                   | Rarely used – debugging only                  |
| Read Committed      | **YES**              | No                            | No                     | No                   | 90% of applications – good default            |
| Repeatable Read     | **YES**              | **YES**                       | **Sometimes / DB-dependent** | No              | Reports, calculations needing stable rows     |
| Serializable        | **YES**              | **YES**                       | **YES**                | **YES**              | Business rules across multiple rows (vouchers, doctors on call, etc.) |

This should now be a clean, self-contained, logically grouped reference document — everything in its proper home. Let me know if you'd like any section expanded, renamed, or converted into slides format!
