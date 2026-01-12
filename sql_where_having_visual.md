# SQL WHERE vs HAVING: Complete Guide with Visual Step-by-Step Examples

## 📌 Core Difference: Row Level vs Group Level

| WHERE Clause | HAVING Clause |
|-------------|--------------|
| **ROW level** - works on individual rows | **GROUP level** - works on aggregated groups |
| Filters **BEFORE** grouping | Filters **AFTER** grouping |
| Cannot use aggregate functions | **Must** use aggregate functions |
| Used with/without GROUP BY | **Requires** GROUP BY |

## 🔄 Visual Execution Order
```
INDIVIDUAL ROWS 
    ↓ (WHERE filters rows)
FILTERED ROWS
    ↓ (GROUP BY groups rows)
GROUPS
    ↓ (Aggregates calculate per group)
GROUPS WITH AGGREGATES
    ↓ (HAVING filters groups)
FILTERED GROUPS
    ↓ (SELECT returns results)
FINAL OUTPUT
```

## 📊 Step-by-Step Visualization

### **Example Table: employees**
```
id | name     | department  | salary
---|----------|-------------|--------
1  | Alice    | Sales       | 50000
2  | Bob      | Sales       | 55000
3  | Charlie  | IT          | 60000
4  | David    | IT          | 65000
5  | Eve      | HR          | 45000
6  | Frank    | HR          | 48000
```

---

## 🎯 **Example 1: WHERE Only (Row-Level Filtering)**

### **Query:**
```sql
SELECT * FROM employees WHERE salary > 50000;
```

### **Step-by-Step Process:**

**Step 1: FROM employees** - Start with all 6 rows

**Step 2: WHERE salary > 50000** - Check each row individually:
```
ROW: [Alice, Sales, 50000] → 50000 > 50000? FALSE → discard
ROW: [Bob, Sales, 55000] → 55000 > 50000? TRUE → keep
ROW: [Charlie, IT, 60000] → 60000 > 50000? TRUE → keep
ROW: [David, IT, 65000] → 65000 > 50000? TRUE → keep
ROW: [Eve, HR, 45000] → 45000 > 50000? FALSE → discard
ROW: [Frank, HR, 48000] → 48000 > 50000? FALSE → discard
```

**Step 3: SELECT *** - Return remaining rows:
```
id | name     | department  | salary
---|----------|-------------|--------
2  | Bob      | Sales       | 55000
3  | Charlie  | IT          | 60000
4  | David    | IT          | 65000
```

---

## 🎯 **Example 2: HAVING Only (Group-Level Filtering)**

### **Query:**
```sql
SELECT department, COUNT(*) as employee_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 1;
```

### **Step-by-Step Process:**

**Step 1: Original Table** - All 6 rows

**Step 2: GROUP BY department** - Creates 3 groups:
```
Sales group: (2 rows)
id | name     | department  | salary
---|----------|-------------|--------
1  | Alice    | Sales       | 50000
2  | Bob      | Sales       | 55000

IT group: (2 rows)
id | name     | department  | salary
---|----------|-------------|--------
3  | Charlie  | IT          | 60000
4  | David    | IT          | 65000

HR group: (2 rows)
id | name     | department  | salary
---|----------|-------------|--------
5  | Eve      | HR          | 45000
6  | Frank    | HR          | 48000
```

**Step 3: SELECT department, COUNT(*)** - Calculate for each group:
```
department | employee_count
-----------|---------------
Sales      | 2    (because Sales group has 2 rows)
IT         | 2    (because IT group has 2 rows)
HR         | 2    (because HR group has 2 rows)
```

**Step 4: HAVING COUNT(*) > 1** - Filter groups:
```
Sales: 2 > 1 = TRUE → keep
IT: 2 > 1 = TRUE → keep
HR: 2 > 1 = TRUE → keep
```

**Final Result:**
```
department | employee_count
-----------|---------------
Sales      | 2
IT         | 2
HR         | 2
```

---

## 🎯 **Example 3: WHERE + HAVING Combined**

### **Query:**
```sql
SELECT department, AVG(salary) as dept_avg
FROM employees
WHERE salary >= 47000
GROUP BY department
HAVING AVG(salary) > 52000;
```

### **Step-by-Step Process:**

**Step 1: WHERE salary >= 47000** - Filter rows:
- Eve (45000) filtered out → 45000 >= 47000? FALSE
- All others kept
```
Remaining: 5 rows (Alice, Bob, Charlie, David, Frank)
```

**Step 2: GROUP BY department** - Create groups:
```
Sales group: Alice (50000), Bob (55000) → 2 rows
IT group: Charlie (60000), David (65000) → 2 rows
HR group: Frank (48000) → 1 row (Eve was filtered out)
```

**Step 3: SELECT department, AVG(salary)** - Calculate averages:
```
Sales: (50000 + 55000) / 2 = 52500
IT: (60000 + 65000) / 2 = 62500
HR: 48000 / 1 = 48000
```

**Step 4: HAVING AVG(salary) > 52000** - Filter groups:
```
Sales: 52500 > 52000 = TRUE → keep
IT: 62500 > 52000 = TRUE → keep
HR: 48000 > 52000 = FALSE → discard
```

**Final Result:**
```
department | dept_avg
-----------|---------
Sales      | 52500
IT         | 62500
```

---

## 🎯 **Level of Operation Explained**

### **1. WHERE → ROW Level**
- Works on **individual rows**
- Filters **BEFORE** grouping
- Example: `WHERE salary > 50000` checks each row:
  ```
  ROW: [Alice, Sales, 50000] → 50000 > 50000? FALSE → discard
  ROW: [Bob, Sales, 55000] → 55000 > 50000? TRUE → keep
  ```

### **2. GROUP BY → GROUP Level**
- Creates **groups** from rows
- Changes working level from rows to groups
- Example: `GROUP BY department` creates:
  ```
  BEFORE: 6 individual rows
  AFTER: 3 groups (Sales, IT, HR)
  ```

### **3. Aggregate Functions → GROUP Level**
- Work on **entire groups**, not individual rows
- `COUNT(*)` = count rows in current group
- `AVG(salary)` = average within current group
- Example for Sales group:
  ```
  AVG(salary) = (50000 + 55000) / 2 = 52500
  ```

### **4. HAVING → GROUP Level**
- Works on **groups** (after GROUP BY)
- Filters groups based on **aggregated values**
- Example: `HAVING AVG(salary) > 52000`:
  ```
  Sales group: 52500 > 52000? TRUE → keep
  HR group: 46500 > 52000? FALSE → discard
  ```

---

## 🧩 **Simple Analogy**

Imagine sorting marbles by color and counting them:

1. **WHERE** = "Only pick up red and blue marbles" *(row level)*
2. **GROUP BY** = "Put red marbles together, blue marbles together" *(create groups)*
3. **COUNT(*)** = "Count marbles in each color group" *(group level)*
4. **HAVING** = "Only show colors with more than 5 marbles" *(group level)*

---

## 📝 **Common Patterns & Examples**

### **Pattern 1: Basic WHERE**
```sql
SELECT * FROM table WHERE column = value;
-- Filters individual rows
```

### **Pattern 2: Basic HAVING**
```sql
SELECT column, COUNT(*)
FROM table
GROUP BY column
HAVING COUNT(*) > N;
-- Filters groups after counting
```

### **Pattern 3: WHERE + HAVING**
```sql
SELECT column1, AVG(column2)
FROM table
WHERE condition_on_rows      -- Filter rows first
GROUP BY column1
HAVING AVG(column2) > value; -- Filter groups after
```

---

## ⚠️ **Common Mistakes & Corrections**

### **Mistake 1: Aggregate in WHERE**
```sql
-- ❌ WRONG: Aggregate not allowed in WHERE
SELECT department, AVG(salary)
FROM employees
WHERE AVG(salary) > 50000  -- ERROR!
GROUP BY department;

-- ✅ CORRECT: Aggregate in HAVING
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;  -- CORRECT!
```

### **Mistake 2: HAVING without aggregate (inefficient)**
```sql
-- ❌ INEFFICIENT: Use WHERE instead
SELECT department, COUNT(*)
FROM employees
GROUP BY department
HAVING department = 'Sales';  -- Works but inefficient

-- ✅ EFFICIENT: Use WHERE
SELECT department, COUNT(*)
FROM employees
WHERE department = 'Sales'  -- Filter rows first
GROUP BY department;
```

---

## 💡 **Pro Tips**

1. **Always use WHERE for non-aggregate conditions** - It's faster (filters before grouping)
2. **Use HAVING only for aggregate conditions** - `COUNT()`, `SUM()`, `AVG()`, etc.
3. **Order matters**: WHERE → GROUP BY → HAVING → SELECT
4. **Test step-by-step**: Write WHERE query first, then add GROUP BY/HAVING
5. **Visualize**: Always think through the step-by-step process

---

## 🔗 **Practice Problems (LeetCode)**

### **Beginner:**
1. [Big Countries](https://leetcode.com/problems/big-countries/) - WHERE practice
2. [Duplicate Emails](https://leetcode.com/problems/duplicate-emails/) - HAVING COUNT
3. [Classes More Than 5 Students](https://leetcode.com/problems/classes-more-than-5-students/) - HAVING with GROUP BY

### **Intermediate:**
4. [Customer Placing the Largest Number of Orders](https://leetcode.com/problems/customer-placing-the-largest-number-of-orders/) - HAVING with COUNT and ORDER
5. [Article Views I](https://leetcode.com/problems/article-views-i/) - WHERE with DISTINCT

### **Advanced:**
6. [Customer Who Visited but Did Not Make Any Transactions](https://leetcode.com/problems/customer-who-visited-but_did_not_make_any_transactions/) - WHERE with NOT IN + GROUP BY
7. [Monthly Transactions I](https://leetcode.com/problems/monthly-transactions-i/) - Complex WHERE + GROUP BY + HAVING

---

## 📚 **Study Strategy**

1. **Start simple**: Master WHERE alone
2. **Add grouping**: Practice GROUP BY with aggregates
3. **Add HAVING**: Filter groups with aggregate conditions
4. **Combine**: Use WHERE and HAVING together
5. **Visualize**: Always think through the step-by-step process

**Save this guide as `sql_where_having_visual.md` for reference!**

*Remember: WHERE filters rows, HAVING filters groups. Visualize the groups, and you'll never mix them up again!*