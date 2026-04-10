# 📘 The Complete Guide to Worker Pools in Go
### From Manual Implementation to Mastering `pond/v2`

---

## Table of Contents

1. **What is a Worker Pool and Why It Exists**
2. **Manual Implementation — Approach 1 (Data via Channels)**
3. **Manual Implementation — Approach 2 (Functions via Channels)**
4. **Pond Basic — NewPool, Submit, StopAndWait**
5. **SubmitErr — Tasks That Can Fail**
6. **ResultPool — Tasks That Return Values**
7. **Task Groups — Wait for a Whole Batch at Once**
8. **Context & Cancellation — Stopping Tasks Early**
9. **Panic Recovery — Crashes Don't Kill the Pool**
10. **Queue Control — Bounded, Unbounded, No Queue**
11. **Dynamic Resize — Change Worker Count at Runtime**
12. **Subpools — Dedicated Lanes Inside the Pool**
13. **Default Pool — Zero Setup Global Pool**
14. **Metrics — Monitoring Your Pool**

---

## 1. What is a Worker Pool and Why It Exists

### The Problem

Imagine you need to process 10,000 tasks. In Go, the easiest way to do something concurrently is to launch a goroutine:

```go
for i := 0; i < 10000; i++ {
    go processTask(i)
}
```

**This works. But it's a disaster in production.**

Why? Because 10,000 goroutines means:
- 10,000 open network connections (if each calls a database or API)
- 10,000 things fighting for CPU time
- Memory pressure grows linearly
- The database or API you're calling might **rate-limit or ban you** for too many concurrent requests
- If each goroutine holds a file handle, you'll hit OS limits

### The Solution: Worker Pool

Instead of 10,000 workers, you use **10 workers**. You put all 10,000 tasks in a **queue** (conveyor belt). Workers grab tasks one by one until everything is done.

| Real World | Go Equivalent |
|---|---|
| 10,000 packages to deliver | 10,000 tasks |
| 10 delivery workers | 10 goroutines |
| Conveyor belt / queue | A channel |
| Worker picks package | Worker reads from channel |

### Why `pond` Exists

You *can* build this manually with Go's standard library (`sync.WaitGroup`, channels, goroutines). But every time you do, you write the same 30-50 lines of boilerplate. And you risk bugs:
- Forgetting to close the channel → **deadlock**
- Forgetting `wg.Add(1)` → program exits early
- Forgetting `defer wg.Done()` → `wg.Wait()` hangs forever
- No panic recovery → one crash kills everything

**`pond` is a package that gives you all of this in 3 lines of code.**

---

## 2. Manual Implementation — Approach 1 (Data via Channels)

### The Scenario: Food Delivery App

You're building the backend for a food delivery app (like Pathao Food or Foodpanda).

- 10 orders come in from customers
- Only 3 delivery riders are on duty
- Orders go into a queue
- Riders grab orders one by one
- Each rider picks up food from restaurant, delivers to customer

### The Code

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// A real delivery order
type Order struct {
	ID         int
	Restaurant string
	Customer   string
	Address    string
}

func main() {
	// A queue of orders waiting for riders
	orders := make(chan Order, 10)

	// We have 3 riders on duty
	var wg sync.WaitGroup
	numRiders := 3

	// Launch 3 riders (workers)
	for i := 1; i <= numRiders; i++ {
		wg.Add(1)

		go func(riderID int) {
			defer wg.Done()

			// Each rider keeps taking orders from the queue
			for order := range orders {
				fmt.Printf("🏍️ Rider %d → Pick up from %s → Deliver to %s at %s\n",
					riderID, order.Restaurant, order.Customer, order.Address)

				// Simulate delivery taking time
				time.Sleep(1 * time.Second)

				fmt.Printf("✅ Rider %d → Order #%d delivered!\n", riderID, order.ID)
			}
		}(i)
	}

	// 10 customer orders come in
	allOrders := []Order{
		{1, "KFC", "Rahim", "Gulshan-1"},
		{2, "Pizza Hut", "Karim", "Banani-2"},
		{3, "BFC", "Sumi", "Dhanmondi-27"},
		{4, "Madchef", "Nadia", "Uttara-10"},
		{5, "Takeout", "Fahim", "Mirpur-12"},
		{6, "Chillox", "Tania", "Bashundhara R/A"},
		{7, "Star Kabab", "Jisan", "Mohakhali"},
		{8, "Domino's", "Riya", "Niketon"},
		{9, "Burger King", "Arif", "Baridhara"},
		{10, "Subway", "Mila", "Nikunja-2"},
	}

	for _, order := range allOrders {
		orders <- order
	}

	// No more orders coming in
	close(orders)

	// Wait for all riders to finish all deliveries
	wg.Wait()

	fmt.Println("\n🎉 All orders delivered! Riders clocked out.")
}
```

### Line-by-Line Breakdown

| Line | What It Does | Why It Matters |
|---|---|---|
| `orders := make(chan Order, 10)` | Creates a buffered channel (queue) that holds `Order` structs. Capacity = 10. | This is your conveyor belt. Buffered means you can push up to 10 orders without a rider reading yet. |
| `var wg sync.WaitGroup` | Creates a counter to track active workers. | Without this, `main()` exits before riders finish. |
| `wg.Add(1)` | Increments the counter before each worker starts. | Says: "One more worker started, I need to wait for it." |
| `go func(riderID int) { ... }(i)` | Launches a goroutine (background thread). Passes `i` so each rider knows their identity. | This is the worker. `go` keyword means "run this concurrently." |
| `defer wg.Done()` | Decrements the counter when the worker exits. | Says: "I'm done, subtract 1 from the waiting count." |
| `for order := range orders` | Reads from the channel one order at a time. | **Most important line.** Blocks if empty. Exits loop when channel is closed. |
| `orders <- order` | Pushes an order into the channel. | Puts work onto the conveyor belt. |
| `close(orders)` | Closes the channel. | Signals: "No more orders coming." Without this, workers wait forever → **deadlock**. |
| `wg.Wait()` | Blocks until all workers call `Done()`. | Ensures program doesn't exit until all work is done. |

### Key Concept: Why a Channel, Not a Slice?

A **slice** is like a notice board — everyone can see all items. If 3 riders look at a slice, they all see Order #1 and might try to deliver it (duplicate work).

A **channel** is like a stack of letters — once you take the top one, it's gone. Go guarantees that only **one worker** ever receives a given item. No locks needed. No race conditions.

### The Problem With This Manual Approach

- You wrote **50 lines** for a basic pool.
- If you need this in 5 places in your app, you copy-paste 5 times.
- Every copy-paste risks bugs (forgetting `close`, forgetting `Add`, etc.).
- No built-in error handling. No metrics. No way to cancel mid-flight.

**This is why `pond` exists.**

---

## 3. Manual Implementation — Approach 2 (Functions via Channels)

### The Scenario: Same Food Delivery, Different Design

In Approach 1, the channel carried **data** (`Order` structs) and workers **knew what to do** with that data (deliver it).

In Approach 2, the channel carries **functions** — the actual work to do. Workers don't know what the work is. They just **execute whatever function they receive**.

### The Code

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	// Channel now carries FUNCTIONS, not data
	deliveries := make(chan func(), 10)

	var wg sync.WaitGroup
	numRiders := 3

	// Launch 3 riders (workers)
	for i := 1; i <= numRiders; i++ {
		wg.Add(1)

		go func(riderID int) {
			defer wg.Done()

			// Each rider just runs whatever delivery function they receive
			for delivery := range deliveries {
				fmt.Printf("🏍️ Rider %d → Starting a delivery...\n", riderID)
				delivery() // Execute the delivery function
				fmt.Printf("✅ Rider %d → Delivery completed!\n\n", riderID)
			}
		}(i)
	}

	// Order #1: KFC to Rahim
	deliveries <- func() {
		fmt.Println("   📦 Order #1: Pick up from KFC → Deliver to Rahim at Gulshan-1")
		time.Sleep(1 * time.Second)
	}

	// Order #2: Pizza Hut to Karim
	deliveries <- func() {
		fmt.Println("   📦 Order #2: Pick up from Pizza Hut → Deliver to Karim at Banani-2")
		time.Sleep(1 * time.Second)
	}

	// ... more orders ...

	// No more orders
	close(deliveries)
	wg.Wait()

	fmt.Println("🎉 All orders delivered! Riders clocked out.")
}
```

### What's Different From Approach 1?

| Approach 1 (Data) | Approach 2 (Functions) |
|---|---|
| `chan Order` | `chan func()` |
| Workers **know** they're delivering food | Workers don't care — they just call `delivery()` |
| Task = **what** to process | Task = **the work itself** |

### Why This Is Better

In Approach 1, if you want workers to do something different (send emails, resize images, etc.), you have to **rewrite the worker logic**.

In Approach 2, workers **never change**. You just feed them different functions. The workers are dumb — they just execute. This makes them **reusable for any kind of task**.

**This is exactly how `pond` works.** You give pond functions, and its internal workers just run them.

---

## 4. Pond Basic — NewPool, Submit, StopAndWait

### The Scenario: Same Food Delivery, Using `pond`

Replace 50 lines of manual boilerplate with **15 lines** of clean code.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	// Create a pool with max 3 workers (riders)
	pool := pond.NewPool(3)

	// Submit 10 orders as tasks
	for i := 1; i <= 10; i++ {
		orderNum := i // capture loop variable

		pool.Submit(func() {
			fmt.Printf("🏍️ Rider processing Order #%d\n", orderNum)
			time.Sleep(1 * time.Second) // Simulate delivery
			fmt.Printf("✅ Order #%d delivered!\n\n", orderNum)
		})
	}

	// Stop pool and wait for ALL submitted tasks to complete
	pool.StopAndWait()

	fmt.Println("🎉 All orders delivered! Riders clocked out.")
}
```

### Line-by-Line Comparison: Manual vs Pond

| What | Manual Code | Pond Code | Who Manages It? |
|---|---|---|---|
| Channel | `make(chan Order, 10)` | Hidden inside `NewPool` | Pond |
| WaitGroup | `var wg sync.WaitGroup` | Hidden inside pool | Pond |
| Launch goroutines | `for` loop with `go func() { ... }` | `NewPool(3)` | Pond |
| `wg.Add(1)` | Before each goroutine | Automatic | Pond |
| `defer wg.Done()` | Inside each goroutine | Automatic | Pond |
| Read from channel | `for order := range orders` | Automatic | Pond |
| Send to channel | `orders <- order` | `pool.Submit(func())` | You (but cleaner) |
| Close channel | `close(orders)` | Inside `StopAndWait()` | Pond |
| Wait for finish | `wg.Wait()` | `pool.StopAndWait()` | You (but same call does more) |

### Why This Matters

You trade "knowing every variable name" for "configuring behavior through options."

- **Channel name?** You don't need it. You use `Submit()`.
- **Channel size?** Default is unbounded. Change it with `pond.WithQueueSize(n)` if needed.
- **How many workers?** `NewPool(3)` — that's it.

### Real-World Use Case

Any time you have a batch of independent tasks and want to limit concurrency:
- Processing uploaded files
- Sending batch emails
- Scraping multiple web pages
- Updating database records in parallel

---

## 5. SubmitErr — Tasks That Can Fail

### Why It's Needed

In the manual way, if a task fails (errors out), you have to handle the error yourself inside the goroutine. There's no built-in way to get that error back to `main`.

With `pond`, `SubmitErr` lets you return an error, and pond captures it so you can check it later.

### The Code

```go
package main

import (
	"errors"
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	pool := pond.NewPool(3)

	// Submit a task that succeeds
	task1 := pool.SubmitErr(func() error {
		fmt.Println("📧 Sending welcome email...")
		time.Sleep(500 * time.Millisecond)
		return nil // Success
	})

	// Submit a task that fails
	task2 := pool.SubmitErr(func() error {
		fmt.Println("💳 Processing payment...")
		time.Sleep(300 * time.Millisecond)
		return errors.New("payment gateway timeout") // Error
	})

	// Submit another success
	task3 := pool.SubmitErr(func() error {
		fmt.Println("📊 Generating report...")
		time.Sleep(400 * time.Millisecond)
		return nil
	})

	// Wait for each task and check results
	err1 := task1.Wait()
	if err1 != nil {
		fmt.Printf("❌ Task 1 failed: %v\n", err1)
	} else {
		fmt.Println("✅ Task 1 succeeded")
	}

	err2 := task2.Wait()
	if err2 != nil {
		fmt.Printf("❌ Task 2 failed: %v\n", err2)
	} else {
		fmt.Println("✅ Task 2 succeeded")
	}

	err3 := task3.Wait()
	if err3 != nil {
		fmt.Printf("❌ Task 3 failed: %v\n", err3)
	} else {
		fmt.Println("✅ Task 3 succeeded")
	}

	pool.StopAndWait()
}
```

### Key Insight

- `pool.Submit(func())` — fire and forget. You don't care about the result.
- `pool.SubmitErr(func() error)` — you want to know if it failed later.

`SubmitErr` returns a `Task` object. You call `.Wait()` on it to block until that specific task finishes, and it gives you back the error (or `nil`).

### Real-World Use Case

- Payment processing (need to know if a transaction failed)
- Database writes (need to know if a row failed to insert)
- API calls (need to know if a request returned 500)

---

## 6. ResultPool — Tasks That Return Values

### Why It's Needed

Sometimes you don't just want to know if a task *failed*. You want the **result** it computed.

Like: "Calculate the delivery fee for this order" → you want the fee amount back.

In the manual way, to get a return value from a goroutine, you'd need another channel:

```go
// Manual way — messy
results := make(chan string)
go func() {
    result := calculate()
    results <- result  // send result back through another channel
}()
result := <-results    // receive it
```

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	// ResultPool[string] means tasks will return a string
	pool := pond.NewResultPool[string](3)

	// Submit tasks that return a result
	task1 := pool.Submit(func() string {
		fmt.Println("🧮 Calculating delivery fee for Order #1 (KFC)...")
		time.Sleep(500 * time.Millisecond)
		return "Fee: ৳50"
	})

	task2 := pool.Submit(func() string {
		fmt.Println("🧮 Calculating delivery fee for Order #2 (Pizza Hut)...")
		time.Sleep(300 * time.Millisecond)
		return "Fee: ৳75"
	})

	task3 := pool.Submit(func() string {
		fmt.Println("🧮 Calculating delivery fee for Order #3 (BFC)...")
		time.Sleep(400 * time.Millisecond)
		return "Fee: ৳60"
	})

	// Wait for each task and get the result
	result1, _ := task1.Wait()
	result2, _ := task2.Wait()
	result3, _ := task3.Wait()

	fmt.Println("\n📋 Results:")
	fmt.Println(result1)
	fmt.Println(result2)
	fmt.Println(result3)

	pool.StopAndWait()
}
```

### Key Insight

`pond.NewResultPool[string](3)` — The `[string]` part means "tasks in this pool return a `string`". You could use `[int]`, `[float64]`, or even a custom struct.

`result, err := task1.Wait()` — You get **both** the result and an error (error would be non-nil if a panic occurred inside the task).

### Real-World Use Case

- Calculating prices or fees in parallel
- Fetching user profiles from a cache/database
- Running parallel database queries and aggregating results

---

## 7. Task Groups — Wait for a Whole Batch at Once

### Why It's Needed

Previously, you called `task1.Wait()`, `task2.Wait()`, `task3.Wait()` separately. That's fine for 3 tasks. But what if you submit **20 tasks** and want to wait for **all of them**?

Manually tracking 20 task variables is tedious. `Task Groups` solve this.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	pool := pond.NewPool(3)

	// Create a group for a batch of tasks
	group := pool.NewGroup()

	// Submit 5 tasks to the group
	for i := 1; i <= 5; i++ {
		orderNum := i
		group.Submit(func() {
			fmt.Printf("🏍️ Delivering Order #%d\n", orderNum)
			time.Sleep(500 * time.Millisecond)
			fmt.Printf("✅ Order #%d done\n", orderNum)
		})
	}

	fmt.Println("⏳ Waiting for all 5 orders to complete...")

	// One call waits for EVERYTHING in the group
	err := group.Wait()

	if err != nil {
		fmt.Printf("❌ Some delivery failed: %v\n", err)
	} else {
		fmt.Println("🎉 All 5 orders delivered!")
	}

	pool.StopAndWait()
}
```

### What Changed

Before:
```go
task1 := pool.Submit(...)
task2 := pool.Submit(...)
task3 := pool.Submit(...)

task1.Wait()
task2.Wait()
task3.Wait()
```

With a group:
```go
group := pool.NewGroup()

for i := 1; i <= 5; i++ {
    group.Submit(func() { ... })
}

err := group.Wait()  // One call waits for ALL
```

### Why This Is Better Than the Manual Way

To do this manually, you'd need a `sync.WaitGroup` for every batch of tasks. You'd `Add(1)` before each task, `Done()` inside each task, and `Wait()` at the end. With pond, the group **is** that WaitGroup — built in.

### `group.Wait()` Waits for Tasks to FINISH

It blocks until **every task submitted to the group has completed** (or failed). It does **not** wait for them to start. The workers start tasks as soon as they're free.

### Real-World Use Case

- "Run 20 database migrations, then continue."
- "Scrape 50 product pages, then aggregate the data."
- "Send notifications to all users in a segment, then log completion."

---

## 8. Context & Cancellation — Stopping Tasks Early

### Why It's Needed

Sometimes you don't want to wait forever. Maybe a task is taking too long, or maybe one task failed and you want to cancel all the others.

`context` lets you say: **"Stop everything, I don't care about the rest."**

In the manual way, canceling goroutines is painful. You'd need to:
- Pass a `context.Context` into every goroutine
- Check `ctx.Done()` inside every loop
- Coordinate cancellation across all goroutines

With `pond`, you just create the group with `NewGroupContext(ctx)` and it propagates cancellation to all tasks automatically.

### The Code

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	pool := pond.NewPool(3)

	// Create a context that cancels after 2 seconds
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)

	// Create a group that is bound to this context
	group := pool.NewGroupContext(ctx)

	fmt.Println("📤 Submitting 6 tasks (each takes 1 second)...")

	// Submit 6 tasks. With 3 workers, this would normally take 2 rounds (2s).
	for i := 1; i <= 6; i++ {
		taskNum := i
		group.SubmitErr(func() error {
			fmt.Printf("  ▶️ Task %d started\n", taskNum)

			// Simulate work that checks for cancellation
			select {
			case <-time.After(1 * time.Second):
				fmt.Printf("  ✅ Task %d finished\n", taskNum)
				return nil
			case <-ctx.Done():
				fmt.Printf("  ❌ Task %d CANCELED (context timed out)\n", taskNum)
				return ctx.Err()
			}
		})
	}

	// Wait for all tasks — or for the context to timeout
	err := group.Wait()

	if err != nil {
		fmt.Printf("\n⚠️  group.Wait() returned: %v\n", err)
	} else {
		fmt.Println("\n🎉 All tasks completed!")
	}

	// Always call cancel() to release resources
	cancel()

	pool.StopAndWait()
}
```

### Two Ways to Cancel Tasks

| Method | When It Cancels |
|---|---|
| `context.WithTimeout(ctx, 2s)` | After 2 seconds, no matter what |
| `context.WithCancel(ctx)` + `cancel()` | Whenever you manually call `cancel()` |

### What Happens to Already-Finished Tasks?

If you had 10 tasks running, and the context timed out after 3 seconds, but 5 tasks had already finished — those 5 tasks **count as successful**. The timeout only affects tasks that are **still running** or **still waiting in the queue** when the timer hits.

### Real-World Use Case

- "Fetch data from 10 APIs, but give up after 5 seconds."
- "If one critical task fails, cancel all remaining tasks."
- "User closes the browser tab — cancel their in-progress background jobs."

---

## 9. Panic Recovery — Crashes Don't Kill the Pool

### Why It's Needed

In Go, if a goroutine panics and you don't recover it, **the whole program crashes**.

If you do this manually, you need `defer recover()` inside *every single goroutine*.

With `pond`, **panic recovery is automatic**. If a task panics, pond catches it and turns it into an error.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	pool := pond.NewPool(3)

	// Task 1: Normal
	task1 := pool.SubmitErr(func() error {
		fmt.Println("🟢 Task 1: Running normally...")
		time.Sleep(500 * time.Millisecond)
		fmt.Println("✅ Task 1: Done")
		return nil
	})

	// Task 2: PANICS!
	task2 := pool.SubmitErr(func() error {
		fmt.Println("🟡 Task 2: About to crash...")
		time.Sleep(300 * time.Millisecond)
		panic("database connection lost!") // Pond catches this
	})

	// Task 3: Normal
	task3 := pool.SubmitErr(func() error {
		fmt.Println("🔵 Task 3: I'm still running despite the crash!")
		time.Sleep(500 * time.Millisecond)
		fmt.Println("✅ Task 3: Done")
		return nil
	})

	// Check results
	fmt.Println("\n--- Checking Results ---")
	
	if err := task1.Wait(); err != nil {
		fmt.Printf("❌ Task 1 failed: %v\n", err)
	}

	if err := task2.Wait(); err != nil {
		fmt.Printf("❌ Task 2 panicked: %v\n", err)
	}

	if err := task3.Wait(); err != nil {
		fmt.Printf("❌ Task 3 failed: %v\n", err)
	}

	pool.StopAndWait()

	fmt.Println("\n🏁 Program survived! No crash.")
}
```

### Key Takeaway

- Task 2 panics.
- Without pond: **entire program crashes**.
- With pond: Panic is caught, converted to an error, and you get it back via `task2.Wait()`.

Other tasks (1 and 3) run fine. The pool keeps working.

### Disabling Panic Recovery

If you want panics to crash the program (sometimes you *want* that for critical bugs):

```go
pool := pond.NewPool(3, pond.WithoutPanicRecovery())
```

### Real-World Use Case

- Processing user-uploaded files (one malformed file shouldn't kill your server)
- Calling third-party APIs (one bad response shouldn't crash everything)
- Data pipelines (one bad record shouldn't stop the entire pipeline)

---

## 10. Queue Control — Bounded, Unbounded, No Queue

### Why It's Needed

When all workers are busy, new tasks sit in a **queue** (line) waiting for their turn.

By default, `pond`'s queue is **unbounded**. This means you can submit 1 million tasks, and pond will keep them all in memory until workers are free.

**The problem:** If tasks arrive faster than workers finish them, memory usage grows until the program crashes (Out Of Memory).

### Queue Types

| Queue Type | Configuration | Behavior |
|---|---|---|
| **Unbounded (Default)** | `pond.NewPool(1)` | Infinite queue. You can submit 1M tasks, memory fills up. |
| **Bounded** | `pond.NewPool(1, pond.WithQueueSize(5))` | Queue holds max 5 tasks. Extra tasks are handled based on submission method. |
| **No Queue** | `pond.NewPool(1, pond.WithQueueSize(0))` | Tasks must run immediately or be rejected. No backlog allowed. |

### `Submit` vs `TrySubmit`

| Method | What Happens When Queue Is Full |
|---|---|
| `pool.Submit(func())` | **Blocks** — waits in line until space opens up. |
| `pool.TrySubmit(func())` | **Fails fast** — returns `(Task, bool)`. If `false`, task was rejected. |

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	// 1 worker, max 2 tasks in the queue
	pool := pond.NewPool(1, pond.WithQueueSize(2))

	fmt.Println("📤 Submitting tasks to a pool with 1 worker and queue size 2...")

	// Task 1: Runs immediately on the worker
	pool.Submit(func() {
		fmt.Println("▶️ Task 1: Running immediately")
		time.Sleep(2 * time.Second) // Takes a long time
		fmt.Println("✅ Task 1: Done")
	})

	// Task 2 & 3: Sit in the queue
	pool.Submit(func() { fmt.Println("▶️ Task 2: Waiting in queue") })
	pool.Submit(func() { fmt.Println("▶️ Task 3: Waiting in queue") })

	// Task 4: Queue is full! TrySubmit returns (Task, bool)
	_, ok := pool.TrySubmit(func() { fmt.Println("▶️ Task 4: Trying to enter...") })
	if !ok {
		fmt.Println("❌ Task 4: REJECTED (Queue is full)")
	}

	_, ok2 := pool.TrySubmit(func() { fmt.Println("▶️ Task 5: Trying to enter...") })
	if !ok2 {
		fmt.Println("❌ Task 5: REJECTED (Queue is full)")
	}

	pool.StopAndWait()
	fmt.Println("🏁 All done.")
}
```

### Real-World Use Case: Notification System

Imagine your app sends SMS/Email. A bug causes 50,000 notifications to queue at once.

- **Unbounded:** Memory fills up → server crashes → everything dies.
- **Bounded (1,000) + TrySubmit:** Extra 49,000 are rejected. You can log them, retry later, or drop them safely. Your server stays alive.

### It's a Tradeoff

Do you want to **wait** (`Submit`), or **fail fast** (`TrySubmit`)?

---

## 11. Dynamic Resize — Change Worker Count at Runtime

### Why It's Needed

Sometimes workloads change. Maybe you normally need 2 workers, but during rush hour you need 10.

In the manual way, worker count is **hardcoded**. To add more workers:
- You'd have to stop the program
- Change `numWorkers = 3`
- Relaunch goroutines

With `pond`, you just call `pool.Resize(n)` and it happens automatically.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	// Start with only 1 worker
	pool := pond.NewPool(1)

	fmt.Println("🚀 Pool started with 1 worker.")

	// Submit 6 slow tasks.
	// With 1 worker, each takes 1 second. Total time: 6 seconds.
	for i := 1; i <= 6; i++ {
		taskNum := i
		pool.Submit(func() {
			fmt.Printf("  ▶️ Task %d started\n", taskNum)
			time.Sleep(1 * time.Second)
			fmt.Printf("  ✅ Task %d done\n", taskNum)
		})
	}

	// Wait 2 seconds... (2 tasks will have finished by now)
	time.Sleep(2 * time.Second)

	fmt.Println("\n📈 Resizing pool to 3 workers...")
	pool.Resize(3)

	fmt.Println("⏳ Now 3 tasks run in parallel. The rest will finish faster!")

	// Wait for everything to finish
	pool.StopAndWait()

	fmt.Println("\n🏁 All tasks done. No restart needed!")
}
```

### What Happens

1. **First 2 seconds:** 1 worker handles tasks 1 & 2. Tasks 3-6 wait in queue.
2. **You call `pool.Resize(3)`:** Two more workers spin up instantly.
3. **Tasks 3, 4, 5 start immediately** (running in parallel now).
4. **Task 6** starts as soon as one of them finishes.

**Total time:** Instead of 6 seconds (1 worker), it finishes much faster (~3-4 seconds).

### Key Insight

If you have 5 tasks waiting, and you call `pool.Resize(100)`, only **5 workers** actually run. Pond creates workers **on demand** — one per task. It doesn't waste memory creating 100 idle threads.

### Real-World Use Case

- **Low traffic (night):** `pool.Resize(2)`
- **High traffic (sale event):** `pool.Resize(20)`
- **Back to normal:** `pool.Resize(2)`

You scale up/down without restarting anything.

---

## 12. Subpools — Dedicated Lanes Inside the Pool

### Why It's Needed

Imagine a delivery hub with 10 total riders.

- 5 riders handle regular food orders.
- 3 riders handle "Express" orders.
- 2 riders handle "Fragile" orders.

They all share the same total pool of 10 riders. If the Express team has no work, their "slots" can be used by regular orders. But if Express gets busy, they can use up to 3 of the total 10 riders.

**This is a Subpool.** It shares the parent's worker capacity, but limits how much of it *this specific type of work* can use.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	// Main pool has max 4 workers total
	pool := pond.NewPool(4)

	// Express subpool can use up to 2 workers
	// (They share the parent's 4 workers)
	express := pool.NewSubpool(2)

	fmt.Println("🚀 Submitting 4 Regular tasks and 3 Express tasks...")

	// Submit 3 Express tasks
	for i := 1; i <= 3; i++ {
		taskNum := i
		express.Submit(func() {
			fmt.Printf("🚄 Express Task %d started\n", taskNum)
			time.Sleep(1 * time.Second)
			fmt.Printf("✅ Express Task %d done\n", taskNum)
		})
	}

	// Submit 4 Regular tasks
	for i := 1; i <= 4; i++ {
		taskNum := i
		pool.Submit(func() {
			fmt.Printf("📦 Regular Task %d started\n", taskNum)
			time.Sleep(1 * time.Second)
			fmt.Printf("✅ Regular Task %d done\n", taskNum)
		})
	}

	// Wait for everything
	pool.StopAndWait()

	fmt.Println("🏁 All tasks complete!")
}
```

### Important: Workers Are Shared, Not Separated

There aren't "2 express workers" and "2 main workers" sitting in different rooms.

Think of it like a **hotel with 4 rooms** (Total Workers = 4).

- **Express Subpool:** Max 2 rooms allowed.
- **Regular Pool:** Max 4 rooms allowed.

**Scenario A:** Hotel is empty. 2 VIPs arrive → They take 2 rooms. (Total used: 2)  
8 Regular guests arrive → They take the other 2 rooms. (Total used: 4)

**Scenario B:** Hotel is full with **4 Regular guests**. A VIP arrives.  
**Does the VIP get a room?** **NO.** All 4 workers are busy. The subpool limit (2) doesn't mean "reserved". It means "capped".

### The Rule

The Subpool **consumes from the Parent's total capacity**. It doesn't get its own separate workers.

### Why Use a Subpool?

Imagine your server has 4 CPU cores (`NewPool(4)`).

You want to:
1.  **Process Videos** (heavy work)
2.  **Handle API Requests** (light work)

If you give all 4 workers to Video Processing, your API requests will hang. The API will time out.

With a **Subpool**:
- Main pool: 4 workers total.
- Video subpool: Max 2 workers.
- API tasks: Direct to main pool (can use remaining 2+ workers when Video is idle).

This prevents the heavy work (Videos) from starving the critical work (API).

---

## 13. Default Pool — Zero Setup Global Pool

### Why It's Needed

You don't even need to create a pool if you don't want to. `pond` provides a **global default pool** that you can use immediately.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	fmt.Println("🚀 Using the default global pool...")

	// Submit tasks directly to the package, no "pool" variable needed
	task1 := pond.Submit(func() {
		fmt.Println("  ▶️ Task 1 (Default Pool)")
		time.Sleep(1 * time.Second)
	})

	task2 := pond.Submit(func() {
		fmt.Println("  ▶️ Task 2 (Default Pool)")
		time.Sleep(1 * time.Second)
	})

	task3 := pond.Submit(func() {
		fmt.Println("  ▶️ Task 3 (Default Pool)")
		time.Sleep(1 * time.Second)
	})

	// Wait for tasks to finish
	task3.Wait()

	fmt.Println("🏁 All tasks done!")
}
```

### What Makes This Special?

**No initialization.** You didn't call `NewPool`. You didn't pick a worker count. You just started sending work.

### How It Works

- `pond` creates a pool in the background the first time you call `Submit`.
- It has **no worker limit** (it scales automatically based on demand).
- The queue is **unbounded**.

### When to Use It

- Simple scripts.
- Background tasks in a web app where you just want "fire and forget" and don't care about limiting concurrency strictly.

### When NOT to Use It

- When you need to strictly limit connections (e.g., "Max 5 database connections").
- When you need to monitor specific pools separately (Metrics).

### The Danger

The default pool is **auto-scaling**. If you throw 10,000 tasks at it instantly, it will spin up **10,000 workers** (goroutines).

Even though Go goroutines are cheap (starting at ~2KB stack), 10,000 of them add up:
- **Memory:** 10,000 * 2KB = ~20MB just for stacks. Plus whatever data each task holds.
- **CPU:** The CPU tries to switch between 10,000 threads. It spends more time **switching context** than actually doing work.
- **Crash:** If your tasks are heavy (e.g., loading images), you will hit **Out Of Memory (OOM)** and the OS will kill your process.

**In production, you almost always use `NewPool(N)` with a limit.**

---

## 14. Metrics — Monitoring Your Pool

### Why It's Needed

In a real backend, you can't just "run and hope." You need to know:
- How many workers are busy?
- How many tasks are waiting?
- Did any fail?

`pond` gives you free counters for this.

### The Code

```go
package main

import (
	"fmt"
	"time"

	"github.com/alitto/pond/v2"
)

func main() {
	// A small pool to make metrics obvious
	pool := pond.NewPool(2, pond.WithQueueSize(5))

	// Fill the queue
	for i := 1; i <= 5; i++ {
		pool.Submit(func() {
			time.Sleep(2 * time.Second) // Make them wait in queue
		})
	}

	// Check stats immediately
	fmt.Printf("👷 Running Workers: %d\n", pool.RunningWorkers())
	fmt.Printf("📥 Waiting Tasks (Queue): %d\n", pool.WaitingTasks())
	fmt.Printf("📤 Submitted Tasks: %d\n", pool.SubmittedTasks())
	fmt.Printf("🗑️ Dropped Tasks: %d\n", pool.DroppedTasks())

	pool.StopAndWait()

	fmt.Println("\n--- After Completion ---")
	fmt.Printf("✅ Successful: %d\n", pool.SuccessfulTasks())
	fmt.Printf("❌ Failed: %d\n", pool.FailedTasks())
}
```

### Available Metrics

| Metric | What It Tells You |
|---|---|
| `RunningWorkers()` | How many workers are currently active. |
| `WaitingTasks()` | How many tasks are queued but not yet started. |
| `SubmittedTasks()` | Total tasks submitted since pool creation. |
| `DroppedTasks()` | Tasks rejected because the queue was full. |
| `SuccessfulTasks()` | Tasks that completed without error. |
| `FailedTasks()` | Tasks that errored or panicked. |
| `CompletedTasks()` | Successful + Failed. |
| `CanceledTasks()` | Tasks canceled by context before execution. |

### Why Your Team Lead Cares About This

In production, you'd connect these numbers to a dashboard (like Grafana or Datadog).

- If `WaitingTasks` is always high → your **workers are too slow** or **you don't have enough of them**.
- If `DroppedTasks` > 0 → your **queue is full** and you are losing work.
- If `FailedTasks` spikes → something is breaking in your tasks.

### Real-World Use Case

You deploy a new feature that processes user uploads. After 1 hour, your dashboard shows:
- `WaitingTasks` = 500 (always growing)
- `RunningWorkers` = 3 (maxed out)

**Action:** You need to either increase worker count (`pool.Resize(10)`) or optimize the task logic.

---

## Final Summary: When to Use What

| Feature | Use When... |
|---|---|
| `NewPool(N)` | You need to strictly limit concurrency (production default). |
| `Submit(func())` | You don't care about the result. Fire and forget. |
| `SubmitErr(func() error)` | You need to know if a task failed. |
| `ResultPool[T]` | You need the actual return value from tasks. |
| `NewGroup()` + `group.Wait()` | You have a batch of tasks and want to wait for all at once. |
| `NewGroupContext(ctx)` | You need timeout or cancellation for a batch. |
| `WithQueueSize(n)` | You want to limit how many tasks can wait in line. |
| `TrySubmit` | You want to fail fast instead of waiting when the queue is full. |
| `Resize(n)` | Workload changes and you need to scale up/down without restart. |
| `NewSubpool(n)` | Different types of work need different concurrency limits. |
| `pond.Submit()` | Quick scripts, no strict limits needed. |
| Metrics | Production monitoring and debugging. |

---

## One Real-World Production Example

**Scenario:** Upload 5,000 user images to AWS S3.

```go
// 1. Limit concurrency to 20 (protects memory & S3 rate limits)
pool := pond.NewPool(20)
group := pool.NewGroup()

// 2. Submit all 5,000 tasks
for i := 0; i < 5000; i++ {
    img := images[i]
    group.SubmitErr(func() error {
        return s3Client.Upload(img)
    })
}

// 3. Wait for ALL to finish and check if ANY failed
err := group.Wait()
if err != nil {
    fmt.Println("⚠️ At least one upload failed!")
} else {
    fmt.Println("✅ All 5,000 images uploaded!")
}

// 4. Check metrics for monitoring
fmt.Printf("Successful: %d, Failed: %d\n", 
    pool.SuccessfulTasks(), pool.FailedTasks())
```

**Features used:**
1. `NewPool(20)` — Limit workers to protect S3 and memory.
2. `NewGroup()` — Wait for the whole batch at once.
3. `SubmitErr` — Catch upload failures.
4. `group.Wait()` — Block until all 5,000 are done.
5. Metrics — Log how many succeeded/failed.

---

*This document was created as a learning resource for understanding worker pools in Go, from manual implementation to the `pond/v2` library.*
