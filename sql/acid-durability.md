# ACID - Durability: The Ultimate Guide

## What is Durability?

**Definition:** Once a transaction has been successfully committed, it will remain committed permanently.

Think of it as the **"Save Button" promise**. If the database tells you, "Success, I saved your data," that data must survive:
- A software crash
- A server restart  
- A complete power failure (pulling the plug)

**Example:** If you deposit money at an ATM and the bank's server loses power 10 milliseconds later, your balance must still reflect that deposit when the power comes back on. If it doesn't, the database lacks durability.

## The Physical Problem (RAM vs. Disk)
### The Conflict:
- Databases want to be **fast**, so they do all their processing in **RAM**
- To be **durable**, they must write to the **Disk**
- Writing to disk for every single tiny change is incredibly slow and would paralyze the database

**Question:** How do databases get RAM-like speed with Disk-like safety?

## The Solution — Write-Ahead Logging (WAL)
This is the single most important concept in database durability.

Instead of writing the actual data to the complex database files immediately (which requires slow, random movements on the disk), the database appends a simple record of the change to a separate file called the **Write-Ahead Log (WAL)** or **Transaction Log**.

### How the WAL works:
1. **The Request:** You send an UPDATE command to change a user's balance from $100 to $200
2. **The Log Write:** The database creates a small log entry: "Transaction #101 changed Page 5 from 100 to 200"
3. **Append Only:** It writes this log entry to the **end** of the log file on the disk. This is very fast because it is a **sequential write** (the disk head doesn't have to jump around)
4. **The Promise:** Only after this log entry is safely on the disk does the database tell you "Commit Success"
5. **The Data Write (Later):** The actual main database file (the "data page") is updated in RAM, but it is not written to the disk yet. It is **"dirty"** (changed in RAM, old on disk). The database will write this to the disk lazily, perhaps minutes later

### Why is this durable?
If the power fails right after step 4, the data file on the disk still says "$100" (old data). However, when the database restarts, it reads the WAL. It sees the entry for Transaction #101, realizes the data file is outdated, and **"replays"** the change to make the balance $200.

## The OS Lie (Fsync and Buffers)
Just writing to a file isn't enough. When a database code says `write(data)`, the OS is lazy. It puts that data into a **Page Cache** (RAM belonging to the OS) and says "Okay, I wrote it!" to the database.

**The Danger:** If the power fails now, the data is in the OS RAM, not the magnetic disk platter. The data is lost, even though the database thought it was safe.

### The Fix: `fsync()`
To guarantee durability, databases use a specific system command called `fsync` (on UNIX/Linux).

**Without fsync:** The OS keeps data in memory to write later.

**With fsync:** 
- The OS actually does **not** use `fsync()` on its own. It is a tool provided by the OS for applications (like a database) to use
- By default, the OS uses a strategy called **Write-Back Caching**. If an application writes data, the OS stores it in the "Page Cache" (RAM) and waits for a "convenient" time to write it to the disk (usually every 5–30 seconds) to save power and improve performance
- The database engine (PostgreSQL, MySQL, etc.) cannot wait for the OS to be "ready." It uses `fsync()` at a very specific moment: **The Commit**

### The Commit Process with fsync():
1. You send `COMMIT`
2. The DB writes the transaction to the WAL file
3. The DB immediately calls `fsync()` on that file
4. The DB **stops and waits**
5. The OS is **forced** to stop what it's doing and push that specific data to the disk hardware
6. Only after the hardware says "Done," the DB tells you "Transaction Successful"

**Note:** This is why database servers often have high "I/O Wait" metrics. They are constantly waiting for the physical disk to confirm the fsync.

## The Hardware Lie (Disk Caches)
We aren't done yet. Even the hard drive lies.

Modern Hard Drives and SSDs have their own internal RAM caches to speed things up. When the OS sends data to the disk, the disk might put it in its internal RAM buffer and say "I got it!" before actually writing it to the magnetic platter or flash storage.

**If power fails now**, the data in the disk's tiny RAM buffer is lost.

### The Solution: Battery-Backed Write Caches (BBWC)
Enterprise databases and hardware controllers use **Battery-Backed Write Caches (BBWC)**.

- The RAID controller (the hardware managing the disks) has a physical battery attached to it
- If power is cut, the battery keeps the cache chip alive for a few days
- When power is restored, the controller flushes the cache to the disk

## Summary of the Durability Pipeline
For a transaction to be truly durable, it must pass through these gates:

1. **Application:** User sends `COMMIT`
2. **Database Engine:** Writes change to the WAL buffer
3. **OS Kernel:** Database calls `fsync`, forcing data from OS Cache to Disk Controller
4. **Disk Controller:** Writes data to physical media (or battery-backed cache)
5. **Confirmation:** The success message travels all the way back up to the User

---

# Checkpoints

## Step 1: The Problem (Why do we need Checkpoints?)
- Changes are written to the WAL (disk) immediately
- Changes are applied to the main data files (RAM) but not saved to the main disk files yet

This creates two major problems over time:

1. **Infinite Log Growth:** If we never clear the WAL, the log file will eventually fill up the entire hard drive
2. **Recovery Time:** If the database runs for 6 months and then crashes, the database would have to replay 6 months of transaction logs to get the data back to a consistent state. This could take days

We need a way to synchronize the RAM (dirty pages) with the Disk (main data files) so we can delete old logs.

## Step 2: What is a Checkpoint?
A **Checkpoint** is a database event that forces all the **"dirty" pages** in memory (modified data that hasn't been saved to the main disk file) to be flushed to the permanent data files on the disk.

## Step 3: The Checkpoint Process (Step-by-Step)
Here is what happens during a standard checkpoint operation:

1. **Trigger:** The database decides it's time to checkpoint (usually based on a timer, e.g., every 5 minutes, or log size, e.g., WAL > 1GB)
2. **Identify Dirty Pages:** The database looks at its **"Buffer Pool"** (RAM) to see which data pages have been modified but not written to the main disk files
3. **Flush:** It writes these dirty pages to the main data files on the disk. This involves **"random I/O"** (moving the disk head around), which is why it can be slow
4. **Sync:** It calls `fsync()` to ensure these writes hit the physical platter
5. **Log the Event:** It writes a special **Checkpoint Record** into the WAL. This record essentially says: "As of this point in the log, all data prior to this is safely stored in the main files"
6. **Truncate:** The database can now safely delete or archive all WAL files created before this checkpoint. They are no longer needed for recovery

## Step 4: Crash Recovery with Checkpoints
Checkpoints drastically speed up recovery after a crash.

### Scenario:
- **12:00 PM:** Database starts
- **12:30 PM:** Checkpoint completes (Log Sequence Number 500)
- **12:45 PM:** Power failure

### Recovery Steps:
1. **Restart:** The database boots up and enters "Recovery Mode"
2. **Locate Checkpoint:** It scans the WAL specifically looking for the last successful Checkpoint record. It finds the one from 12:30 PM
3. **Skip Old Logs:** It knows everything before 12:30 PM is already safe on the disk. It ignores those logs
4. **Replay (Redo):** It only replays the log entries from 12:30 PM to 12:45 PM
5. **Open:** The database is now consistent and ready for users. Recovery took seconds instead of hours

## Step 5: Fuzzy vs. Sharp Checkpoints
There is a trade-off in how checkpoints are implemented.

### 1. Sharp Checkpoints (Blocking)
The database **"Freezes."** It stops accepting new write transactions, flushes everything to disk, and then resumes.

**Pro:** Very simple to implement. Recovery is very fast  
**Con:** Latency Spikes. Users might see the database "hang" for a few seconds (or minutes) every time a checkpoint happens. This is unacceptable for high-performance apps

### 2. Fuzzy Checkpoints (Non-Blocking)
Most modern databases (PostgreSQL, MySQL/InnoDB, Oracle) use this. The database continues to accept new writes while it is writing old dirty pages to the disk in the background.

**Pro:** No "stop-the-world" pauses. Performance is smooth  
**Con:** Complex to implement. The checkpoint record in the log doesn't mean "everything is saved," it means "everything started to save." Recovery is slightly more complex because the start and end of the checkpoint might overlap with new transactions

---

# Distributed Durability

## Step 1: The New Problem (Single Point of Failure)
Even with WAL and Checkpoints, your data lives on **one physical machine**.
- If the motherboard fries? You are fine (move the disk to a new server)
- If the disk corrupted? Data is lost forever

To achieve high durability, we must copy the data to a different machine, ideally in a different geographic location. This is called **Replication**.

## Step 2: The Mechanism (Log Shipping)
How do we copy data efficiently? We don't send the whole database file every time a user changes a row.

We reuse the tool we already learned about: **The WAL**.

### The Process:
1. **The Leader (Primary):** This is the main server accepting writes. It writes the transaction to its local WAL
2. **Log Shipping:** The Leader sends that specific WAL entry (e.g., "Set ID 500 to Active") over the network to another server
3. **The Follower (Replica):** This server receives the log entry and **"replays"** it, just like it was recovering from a crash. It updates its own data to match the Leader

## Step 3: The Durability Trade-Off (Sync vs. Async)
Here is the critical decision every database engineer must make. When does the Leader tell the user "Success"?

### Option A: Asynchronous Durability (The "Fast but Risky" Way)
1. Leader writes to local disk
2. Leader tells User: "Success!"
3. Leader then sends the log to the Follower in the background

**Pro:** Extremely fast. The user doesn't wait for network lag  
**Con:** Data Loss is possible. If the Leader crashes 10ms after telling the user "Success" but before sending the log to the Follower, that transaction is lost forever. The Follower becomes the new Leader, but it doesn't have that last piece of data

### Option B: Synchronous Durability (The "Slow but Safe" Way)
1. Leader writes to local disk
2. Leader sends log to Follower
3. Leader waits for Follower to reply "I wrote it to my disk too"
4. Leader tells User: "Success!"

**Pro:** Zero Data Loss. If the Leader dies, the Follower is guaranteed to have the data  
**Con:** Latency. Every transaction is as slow as the network connection between the servers. If the Follower goes offline, the Leader cannot accept writes (system halts)

## Step 4: Quorums (The "Middle Ground")
What if you have 5 followers? Do you wait for all of them (too slow) or none of them (too risky)?

We use a **Quorum**.

A **Quorum** is the minimum number of votes required to consider a transaction "Durable." Usually, this is a majority ($N/2 + 1$).

### Example with 3 Servers:
- You set the Durability requirement to **Quorum (2/3)**
- The Leader writes to itself (1 vote)
- It sends logs to Follower A and Follower B
- As soon as **one of them replies**, the Leader has 2 votes (Majority)
- It tells the user "Success!" without waiting for the slowest server

This is how systems like Cassandra, CockroachDB, and Google Spanner work. They tolerate the failure of a minority of nodes without losing data or stopping availability.

---

# Interview Questions on Durability

## Junior Level: The Basics

### Q1: If a database crashes before a transaction is written to the data file, but after it has been committed, is the data lost?
**Answer:** No. Thanks to the Write-Ahead Log (WAL), the transaction is considered durable once the log record is flushed to the disk. Upon restart, the database performs a "Redo" operation, reading the WAL and applying those changes to the data files.

### Q2: What is the difference between a "Dirty Page" and a "WAL entry"?
**Answer:** A Dirty Page is a data page in RAM that has been modified but not yet written to the main database file on disk. A WAL entry is a small, sequential record of that modification stored on disk. Durability relies on the WAL entry, not the Dirty Page.

## Intermediate Level: Mechanics & Performance

### Q3: How does a Checkpoint affect the recovery time (MTTR) of a database?
**Answer:** It reduces it. Without checkpoints, the database would have to replay the WAL from the beginning of time. A checkpoint ensures all changes up to a specific LSN are hardened in the main data files, allowing the database to safely discard old logs and only replay the "tail" of the log during recovery.

### Q4: Why is fsync() expensive, and what happens if a database skips it?
**Answer:** fsync() is expensive because it forces a synchronous "flush" to the physical hardware, making the CPU wait for the slow disk. If a database skips it, the OS might keep data in its RAM cache. If the power fails, the OS cache vanishes, and even though the DB thought the data was "saved," it is lost—violating the Durability guarantee.

## Senior Level: Distributed & Architecture

### Q5: In a Leader-Follower setup, what is the "Durability Gap" in asynchronous replication?
**Answer:** The Durability Gap is the window of time between a transaction being committed on the Leader and it being received/persisted by the Follower. If the Leader suffers a hardware failure during this window, the data is lost because the Follower (the new Leader) never saw it.

### Q6: Explain the "Double Write Buffer" (or Why we need it for Durability).
**Answer:** Most filesystems write in 4KB chunks, but databases use 8KB or 16KB pages. If the system crashes halfway through writing a page, you get a "Torn Page" (half old, half new), which is corrupted. To ensure durability, databases like MySQL write pages to a "Double Write Buffer" first. If a crash happens, they can recover the clean page from this buffer instead of the corrupted data file.

## The "Scenario" Question (The Most Common)

### Q7: "Our database is too slow. A developer suggests turning off the WAL or setting synchronous_commit to 'off' to increase speed. What are the risks?"
**Answer:** Turning off these features increases throughput by removing disk I/O bottlenecks, but it breaks the "D" in ACID.

**Risk:** In the event of a crash, you may suffer "Partial Durability" or "Data Loss."

**Counter-proposal:** Instead of breaking ACID, I would suggest moving to faster hardware (NVMe SSDs), using a Battery-Backed Write Cache, or tuning the Checkpoint frequency to reduce I/O spikes.

---

# Summary Table for Quick Revision

| Keyword | Why it matters for the Interview |
|---------|----------------------------------|
| **WAL** | The "Source of Truth" for durability |
| **LSN** | The "Version Number" that prevents replaying data twice |
| **Fsync** | The bridge between Software RAM and Physical Disk |
| **Quorum** | Durability in the cloud (saving to multiple servers) |
| **Torn Page** | Hardware-level corruption during the write process |

---

# Advanced Topics (Would you like to go deeper?)

We have covered the software side of durability. The final frontier is the Hardware.

## Potential Next Topics:

### 1. Bit Rot & Checksums
How databases detect when a single bit flips on the hard drive due to cosmic rays or magnetic decay.

### 2. SSD Wear Leveling
How modern drives physically degrade and how databases try not to kill your SSDs.

### 3. Erasure Coding
Going beyond simple replication to achieve durability with less storage overhead.

### 4. Consensus Algorithms (Raft, Paxos)
How distributed databases achieve durability and consistency across many nodes.
