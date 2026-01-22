### Part 1: The Lists (Reference Guide)

Before we dive in, here is the cheat sheet of terms we will use.

#### The Problems (The Bugs)

1. **Dirty Read:** Reading data that hasn't been saved (committed) yet.
2. **Non-Repeatable Read:** Reading a row, and then finding that the data *inside that row* changed when you read it again.
3. **Phantom Read:** Reading a list of rows (like a search result), and finding that a *new row* appeared or disappeared when you searched again.
4. **Serialization Anomaly (Write Skew):** Two transactions follow valid logic individually, but when run together, they create a logical error.

#### The Isolation Levels (The Solutions)

1. **Read Uncommitted:** The "I don't care" level. No protection.
2. **Read Committed:** The "Standard" level. You only see saved data.
3. **Repeatable Read:** The "Stable" level. Data you are looking at stays frozen for you.
4. **Serializable:** The "Strict" level. Everything happens as if there is only one user at a time.

---

### Part 2: The Deep Dive (Problem + Example + Solution)

We will use an **Online Store (Amazon-style)** for all examples.
**Table:** `Product`
**Row:** `ID: 1, Name: iPhone, Stock: 10`

---

### Problem 1: The Dirty Read

**Concept:** You trust data that might not be real.

#### The Practical Example

Imagine two transactions happening at once:

* **Transaction A (The Manager):** Updates the iPhone stock from **10** to **20**... but hasn't clicked "Save" yet.
* **Transaction B ( The Customer):** Reads the stock level to see if they can buy 15 phones.

**The Disaster:**

1. Manager updates stock to 20 (but **does not commit**).
2. Customer reads stock: sees **20**.
3. Customer orders 15 phones.
4. Manager's computer crashes! The database performs a **Rollback**, reverting stock back to **10**.
5. **Result:** You just sold 15 phones, but you only have 10. You have a shipping disaster.

#### The Solution: Read Committed

This is the specific fix for Dirty Reads.

* **How it works:** The database hides any "uncommitted" data from you. If the Manager has updated the row to 20 but hasn't committed, the Customer will still see **10** (the last saved version).
* **The Outcome:** The Customer sees 10, realizes they can't buy 15, and the system works correctly.

---

### Problem 2: The Non-Repeatable Read

**Concept:** You are doing math on a number, but someone changes that number while you are working.

#### The Practical Example

* **Transaction A (The Accountant):** Wants to generate a monthly report.
* **Transaction B (Sales System):** Is processing a sale.

**The Disaster:**

1. Accountant reads the iPhone stock: **10**.
2. Accountant turns away to calculate the value ($1000 * 10 = $10,000).
3. Sales System sells 5 iPhones and **Commits**. Stock is now **5**.
4. Accountant decides to double-check the stock for the report. Reads again: **5**.
5. **Result:** The Accountant is confused. "I just saw 10, now it's 5?" The math in the report is now inconsistent (Revenue calculated on 10 units, but inventory count says 5).

#### The Solution: Repeatable Read

This is the specific fix for Non-Repeatable Reads.

* **How it works:** When the Accountant reads the row (Stock: 10), the database effectively hands them a "personal snapshot" or places a "lock" on that row.
* Even if the Sales System updates the stock to 5 and commits, when the Accountant looks at the row again, **they still see 10**.
* **The Outcome:** The Accountant finishes their report with consistent numbers. They see the "world" as it existed when they started working.

---

### Problem 3: The Phantom Read

**Concept:** This is subtle. *Non-Repeatable Read* is about a specific row changing. *Phantom Read* is about the **result of a search query** changing (rows appearing/disappearing).

#### The Practical Example

* **Transaction A (Shipping Dept):** Wants to print labels for all orders currently "Pending".
* **Transaction B (New Customer):** Is placing a new order.

**The Disaster:**

1. Shipping runs a query: `SELECT * FROM Orders WHERE Status = 'Pending'`.
2. Database finds **50 orders**.
3. Shipping starts printing 50 labels.
4. New Customer clicks "Buy". A **new row** is inserted into the table with `Status = 'Pending'`.
5. Shipping runs the query again to mark them as "Shipped". Now they see **51 orders**.
6. **Result:** "Where did this extra order come from?" They printed 50 labels, but now the system says there are 51 orders to process. One order gets left behind in limbo.

#### The Solution: Serializable (or Snapshot Isolation)

* **Note:** Standard "Repeatable Read" often protects *existing* rows, but it doesn't always stop *new* rows from being added (Phantoms).
* **How Serializable works:** It locks the "Range". If you ask for "Pending Orders," the database might say, "Nobody is allowed to add a NEW pending order until Transaction A is finished."
* **The Outcome:** The New Customer's transaction is blocked (paused) until the Shipping Department finishes printing their labels.

---

### Problem 4: Write Skew (The Serialization Anomaly)

**Concept:** This is the most complex one. It happens when two people make a decision based on the *same* data, but update *different* data, crossing wires.

#### The Practical Example

The "Black Friday Voucher" problem.
**Rule:** We can give out a $50 discount voucher, but we must never give out more vouchers than we have budget for.
**Budget:** We have budget for **1** more voucher.
**Current Vouchers Claimed:** 9. (Limit is 10).

* **Transaction A (User Alice):** Checks count.
* **Transaction B (User Bob):** Checks count.

**The Disaster:**

1. Alice reads `Count`: Sees **9**. Logic: "9 < 10, so I can claim one."
2. Bob reads `Count`: Sees **9**. Logic: "9 < 10, so I can claim one."
3. Alice adds her voucher. New count is 10.
4. Bob adds his voucher. New count is 11.
5. **Result:** You have exceeded your budget. Both transactions were valid on their own, but together they broke the rules.

*Why didn't "Repeatable Read" stop this?* Because Alice didn't touch Bob's data, and Bob didn't touch Alice's data. They both read consistent data (9), but the *combination* of their actions was wrong.

#### The Solution: Serializable

* **How it works:** The database detects that both transactions are reading the *same* data (`Count`) to make a decision about writing data. It forces them to run one after the other.
* **The Outcome:**
1. Alice checks count (9).
2. Bob tries to check count, but the database makes him wait because Alice is busy with that data.
3. Alice claims voucher. Count becomes 10. Alice finishes.
4. Bob is allowed to proceed. He checks count... now he sees **10**.
5. Bob's Logic: "10 is not < 10. Sorry, no voucher."



---

### Summary Table

Here is how the levels map to the problems:

| Isolation Level | Solves Dirty Read? | Solves Non-Repeatable Read? | Solves Phantom Read? | Solves Write Skew? |
| --- | --- | --- | --- | --- |
| **Read Uncommitted** | No | No | No | No |
| **Read Committed** | YES | No | No | No |
| **Repeatable Read** | YES | YES | Sometimes | No |
| **Serializable** | YES | YES | YES | YES |

### Next Step

Here is the step-by-step SQL simulation of the **Dirty Read** anomaly.

To understand this deeply, imagine you have two open terminal windows (Connection A and Connection B) talking to the same database.

### The Setup

First, let's create our `Product` table with our "iPhone" example.

```sql
-- Initial Setup (Run this once)
CREATE TABLE Product (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    stock INT
);

INSERT INTO Product VALUES (1, 'iPhone', 10);

```

**Current State:** iPhone Stock = 10.

---

### Scenario 1: The Bug (Read Uncommitted)

In this scenario, we set the database to the lowest safety level (`READ UNCOMMITTED`) to demonstrate the danger.

**The Story:** The Manager (Transaction A) updates stock to 20 but hasn't saved. The Customer (Transaction B) sees 20, but then the Manager cancels the update.

| Step | Time | **Transaction A (The Manager)** | **Transaction B (The Customer)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;` | `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;` | We lower the shields. |
| 2 | 00:02 | `BEGIN TRANSACTION;` |  | Manager starts work. |
| 3 | 00:03 | `UPDATE Product SET stock = 20 WHERE id = 1;` |  | Stock is now 20 in memory, **but not saved to disk.** |
| 4 | 00:04 |  | `BEGIN TRANSACTION;` | Customer logs in. |
| 5 | 00:05 |  | `SELECT stock FROM Product WHERE id = 1;` | **DIRTY READ!**<br>

<br>The database returns **20**. The customer thinks there are 20 phones. |
| 6 | 00:06 | `ROLLBACK;` |  | Manager realizes a mistake and cancels (Rollback). Stock reverts to **10**. |
| 7 | 00:07 |  | *Customer Logic...* | The customer is now trying to buy based on a stock of 20, which **never technically existed**. |

**The Result:** The system is now inconsistent. Transaction B has "hallucinated" data.

---

### Scenario 2: The Fix (Read Committed)

Now, let's see how the standard isolation level (`READ COMMITTED`) prevents this. This is the default setting for PostgreSQL, Oracle, and SQL Server.

**The Story:** The Manager updates stock to 20. The Customer tries to look, but the database refuses to show the unsaved "20". It shows the safe "10" instead.

| Step | Time | **Transaction A (The Manager)** | **Transaction B (The Customer)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` | Shields are up. |
| 2 | 00:02 | `BEGIN TRANSACTION;` |  | Manager starts work. |
| 3 | 00:03 | `UPDATE Product SET stock = 20 WHERE id = 1;` |  | Stock is 20 in memory, not saved. |
| 4 | 00:04 |  | `BEGIN TRANSACTION;` | Customer logs in. |
| 5 | 00:05 |  | `SELECT stock FROM Product WHERE id = 1;` | **PROTECTION!**<br>

<br>The DB returns **10** (the last committed value).<br>

<br>*(Note: In some DBs like SQL Server, this query might simply PAUSE and wait until Tx A finishes).* |
| 6 | 00:06 | `COMMIT;` |  | Manager saves the work. Stock is officially 20. |
| 7 | 00:07 |  | `SELECT stock FROM Product WHERE id = 1;` | Now the customer sees **20**. |

**The Result:** Transaction B never saw the "maybe" data. It only saw the "definite" data.

---

### A Critical Detail: "Blocking" vs "Old Data"

In Step 5 of the "Fix" above, different databases behave slightly differently:

1. **Blocking (Pessimistic - e.g., Standard SQL Server):** Transaction B tries to read, sees that Transaction A is busy with that row, and **waits** (the spinner spins). It freezes until Transaction A commits.
2. **Old Data (MVCC - e.g., PostgreSQL, Oracle, MySQL InnoDB):** Transaction B reads the row, sees that A is working on it, and ignores A's work. It reads the **previous version** (10) instantly without waiting.

Both methods successfully solve the Dirty Read!

### Next Step?

Here is the step-by-step SQL simulation of the **Non-Repeatable Read** anomaly.

**The Distinction:**

* **Dirty Read:** You saw data that *wasn't saved*.
* **Non-Repeatable Read:** You see data that *was saved*, but it changed while you were still looking at it.

This bug happens frequently because the default isolation level for most databases (PostgreSQL, SQL Server, Oracle) is `READ COMMITTED`, which **allows** this to happen.

### The Setup

We use the same table: `Product` (iPhone, Stock: 10).

---

### Scenario 1: The Bug (Read Committed)

**The Story:** An **Accountant (Transaction A)** is creating a report. They read the stock (10). Meanwhile, a **Sales System (Transaction B)** sells 5 phones and commits the change. The Accountant checks the stock again and gets a different number.

**Note:** This is the default behavior in many databases.

| Step | Time | **Transaction A (The Accountant)** | **Transaction B (The Sales System)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` | `SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` | Standard isolation. |
| 2 | 00:02 | `BEGIN TRANSACTION;` |  | Accountant starts the report. |
| 3 | 00:03 | `SELECT stock FROM Product WHERE id = 1;` |  | Accountant sees **10**. |
| 4 | 00:04 |  | `BEGIN TRANSACTION;` | Sales System starts. |
| 5 | 00:05 |  | `UPDATE Product SET stock = 5 WHERE id = 1;` | Sales System updates stock to 5. |
| 6 | 00:06 |  | `COMMIT;` | **CRITICAL MOMENT:** The change is saved permanently. |
| 7 | 00:07 | `SELECT stock FROM Product WHERE id = 1;` |  | **NON-REPEATABLE READ!**<br>

<br>The Accountant sees **5**. |
| 8 | 00:08 | *Accountant Logic...* |  | The Accountant is confused. "I just calculated the report based on 10 units... why does the database now say 5?" |

**The Result:** Your report calculates `Value = $1000 * 10` but prints `Stock Count = 5`. The numbers don't match.

---

### Scenario 2: The Fix (Repeatable Read)

Now we upgrade to `REPEATABLE READ`. This is the default in MySQL (InnoDB).

**The Story:** The Accountant takes a "Snapshot" of the database at the start. Even though the Sales System changes the reality of the database, the Accountant continues to see their own private version of history.

| Step | Time | **Transaction A (The Accountant)** | **Transaction B (The Sales System)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` | Shields upgraded. |
| 2 | 00:02 | `BEGIN TRANSACTION;` |  | Accountant starts. |
| 3 | 00:03 | `SELECT stock FROM Product WHERE id = 1;` |  | Accountant sees **10**. **Snapshot Created!** |
| 4 | 00:04 |  | `BEGIN TRANSACTION;` | Sales System starts. |
| 5 | 00:05 |  | `UPDATE Product SET stock = 5 WHERE id = 1;` | Sales System updates stock to 5. |
| 6 | 00:06 |  | `COMMIT;` | Sales System saves. The "Real" database now says 5. |
| 7 | 00:07 | `SELECT stock FROM Product WHERE id = 1;` |  | **PROTECTION!**<br>

<br>The Accountant *still* sees **10**. |
| 8 | 00:08 | `COMMIT;` |  | Accountant finishes. |
| 9 | 00:09 | `SELECT stock FROM Product WHERE id = 1;` |  | *Now* (in a new transaction) the Accountant would see 5. |

**The Result:** The report is consistent. It represents the state of the world at exactly 00:02, ignoring the chaos that happened at 00:06.

---

### Key Takeaway: The "Snapshot"

In modern databases (PostgreSQL, MySQL), `REPEATABLE READ` works by giving Transaction A a frozen view of the data.

* **Transaction B:** "I just changed this row to 5!"
* **Database to Transaction A:** "That's nice for him. But here is the version (10) that existed when *you* started. Keep using this one."

### Next Step
The **Phantom Read** is the trickiest of the three classic anomalies. It is often confused with the Non-Repeatable Read, but there is a key difference:

* **Non-Repeatable Read:** One **specific row** you already read changes its internal value.
* **Phantom Read:** The **number of rows** that match your search query changes because someone added or deleted a record.

---

### The Setup

We have an `Orders` table.
`| id | customer | amount | status    |`
`|----|----------|--------|-----------|`
`| 1  | Alice    | 100    | 'PENDING' |`
`| 2  | Bob      | 150    | 'PENDING' |`

---

### Scenario 1: The Bug (Repeatable Read)

**The Story:** A **Shipping Manager (Transaction A)** is counting how many pending orders need to be packed today. While they are counting, a **Customer (Transaction B)** places a brand new order.

**Note:** In some databases (like standard SQL implementations), the `REPEATABLE READ` level protects existing rows you’ve already touched, but it doesn't "lock" the gaps between rows where new data can be inserted.

| Step | Time | **Transaction A (Shipping Manager)** | **Transaction B (New Customer)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` | `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;` | Level is set to Repeatable Read. |
| 2 | 00:02 | `BEGIN;` |  | Manager starts the count. |
| 3 | 00:03 | `SELECT COUNT(*) FROM Orders WHERE status = 'PENDING';` |  | Manager sees **2**. |
| 4 | 00:04 |  | `BEGIN;` | New Customer starts. |
| 5 | 00:05 |  | `INSERT INTO Orders VALUES (3, 'Charlie', 200, 'PENDING');` | New order added. |
| 6 | 00:06 |  | `COMMIT;` | The new order is now permanent. |
| 7 | 00:07 | `SELECT COUNT(*) FROM Orders WHERE status = 'PENDING';` |  | **PHANTOM READ!**<br>

<br>Manager sees **3**. |
| 8 | 00:08 | `UPDATE Orders SET status = 'SHIPPED' WHERE status = 'PENDING';` |  | **Confusion:** The Manager's code thought it was updating 2 orders, but it actually updated 3. |

**The Result:** A "Phantom" order appeared mid-transaction. If the manager was printing shipping labels based on the first count, they would be short by one label.

---

### Scenario 2: The Fix (Serializable)

Now we upgrade to `SERIALIZABLE`, the highest level of isolation.

**The Story:** The database ensures that no one can "mess with the range" that the Manager is looking at.

| Step | Time | **Transaction A (Shipping Manager)** | **Transaction B (New Customer)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` | `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` | Highest level. |
| 2 | 00:02 | `BEGIN;` |  | Manager starts. |
| 3 | 00:03 | `SELECT COUNT(*) FROM Orders WHERE status = 'PENDING';` |  | Manager sees **2**. The DB "locks" the range of pending orders. |
| 4 | 00:04 |  | `BEGIN;` |  |
| 5 | 00:05 |  | `INSERT INTO Orders ... 'PENDING';` | **BLOCKING!**<br>

<br>Transaction B **waits** (freezes). The DB won't let a new 'PENDING' order be added because it would change A's results. |
| 6 | 00:06 | `COMMIT;` |  | Manager finishes. |
| 7 | 00:07 |  | *(System unfreezes B)*<br>

<br>`COMMIT;` | Now the customer's order is finally saved. |

**The Result:** Total consistency. The manager sees 2, finishes their work, and *only then* is the 3rd order allowed to exist.

---

### Important Nuance: PostgreSQL vs. Others

In **PostgreSQL**, the `REPEATABLE READ` level is actually much stronger than the SQL standard requires. In Postgres, `REPEATABLE READ` **does** prevent Phantoms using a technology called Snapshot Isolation.

In **MySQL (InnoDB)**, `REPEATABLE READ` also prevents phantoms for plain `SELECT` queries but can still see them if you use "locking reads" (like `SELECT ... FOR UPDATE`).

This is why `SERIALIZABLE` is the only 100% guarantee across all database types.

---

### We've covered the 3 main "Bugs."

You now know:

1. **Dirty Read** (Reading uncommitted junk) → Fix: **Read Committed**.
2. **Non-Repeatable Read** (A row changing under your feet) → Fix: **Repeatable Read**.
3. **Phantom Read** (New rows popping up) → Fix: **Serializable**.

Let's tackle **Write Skew** first. This is the "final boss" of database isolation.

It is the most dangerous because **Repeatable Read** (the level that solves all our previous problems) cannot stop it. Only **Serializable** can.

### The Problem: Write Skew (The "Two Doctors" Problem)

**The Rule:** A hospital must always have at least **one** doctor on call.
**The Situation:** There are currently two doctors on call: Alice and Bob. Both are feeling sick and want to go home.

#### The Practical Example (The Bug)

| Step | Time | **Transaction A (Alice)** | **Transaction B (Bob)** | **Explanation** |
| --- | --- | --- | --- | --- |
| 1 | 00:01 | `BEGIN;` | `BEGIN;` | Both doctors open the app at the same time. |
| 2 | 00:02 | `SELECT COUNT(*) FROM Doctors WHERE on_call = true;` |  | Alice sees **2**. She thinks: "Since 2 > 1, I can leave." |
| 3 | 00:03 |  | `SELECT COUNT(*) FROM Doctors WHERE on_call = true;` | Bob sees **2**. He thinks: "Since 2 > 1, I can leave." |
| 4 | 00:04 | `UPDATE Doctors SET on_call = false WHERE name = 'Alice';` |  | Alice updates her status. |
| 5 | 00:05 |  | `UPDATE Doctors SET on_call = false WHERE name = 'Bob';` | Bob updates his status. |
| 6 | 00:06 | `COMMIT;` | `COMMIT;` | Both transactions are saved. |

**The Disaster:**
Both Alice and Bob have left. The `COUNT` is now **0**. The hospital rule is broken.

**Why did "Repeatable Read" fail here?**

* **Dirty Read?** No. They both read committed data (the count was 2).
* **Non-Repeatable Read?** No. For Alice, the count stayed 2 the whole time she was looking. For Bob, the count stayed 2.
* **Phantom Read?** No. No new rows were added or deleted.

The problem is that they made a decision based on a **premise** (the count of doctors), but then they modified **different rows**. Because they didn't try to update the *same* row, the database didn't see a conflict.

---

### The Solution: Serializable

Under `SERIALIZABLE` isolation, the database acts like a strict supervisor.

**How it works (The Fix):**

1. Alice reads the count (2). The database takes a note: "Alice's decision depends on the fact that the count of on-call doctors is 2."
2. Bob reads the count (2). The database takes a note: "Bob's decision also depends on that same count."
3. Alice updates her status.
4. Bob tries to update his status.
5. **The Catch:** The database realizes that if Bob updates his status, it would change the "premise" Alice used for her update. To prevent a "skewed" result, the database will **force one transaction to fail.**

**The Outcome:** Alice’s transaction succeeds. Bob’s transaction is aborted with a "Serialization Error." Bob has to try again, and this time, he will see the count is only 1, so he won't be allowed to leave.

---

### Part 3: How does the DB do this without being slow? (MVCC)

If the database had to lock everything to keep things safe, it would be incredibly slow. Imagine if Amazon froze every time one person looked at an item!

To solve this, modern databases use **MVCC (Multi-Version Concurrency Control)**.

#### The "Versioning" Concept

Instead of overwriting data, the database keeps **versions** of rows. Think of it like a "Timeline."

When you look at a row, the database doesn't just give you the "latest" value. It looks at your **Transaction ID** and says, "This transaction started at 10:00 AM. I will show you the version of the data that was valid at 10:00 AM, even if someone changed it at 10:01 AM."

**Key Benefits of MVCC:**

1. **Readers never block Writers:** You can read a row while someone else is updating it. You just see the "old" version.
2. **Writers never block Readers:** Someone can update a row without stopping people from reading it.
3. **Efficiency:** The only time people actually "wait" is if two people try to update the **exact same row** at the exact same time.

#### How it handles a Delete:

In MVCC, a `DELETE` doesn't actually remove the data immediately. It just marks that version as "deleted at Time X." Old transactions can still see it; new ones can't. Later, a process called **Vacuuming** (in Postgres) or **Purging** (in MySQL) cleans up the old, unneeded versions.

---

### Summary Checklist:

* **Read Committed:** Use this for 90% of apps. It's fast and prevents "Dirty" (fake) data.
* **Repeatable Read:** Use this for reports or math where you need the numbers to stay still while you calculate.
* **Serializable:** Use this for complex logic (like the Doctors or Bank Vouchers) where "rules" span across multiple rows.
* **MVCC:** The engine under the hood that makes all of this possible by keeping "snapshots" of time.



