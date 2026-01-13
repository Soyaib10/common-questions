## **Complete Summary with Examples**

### **What is a Subquery?**
A SQL query nested inside another query.

---

### **1. NON-CORRELATED SUBQUERY**
**Definition:** Inner query runs independently, once.

**Example 1:** Products above average price
```sql
SELECT product_name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);  -- Runs ONCE
```

**Example 2:** Employees in departments that exist in NY office
```sql
SELECT name, department
FROM employees
WHERE department IN (
    SELECT DISTINCT department 
    FROM employees 
    WHERE office = 'NY'  -- Runs ONCE
);
```

**Characteristics:**
- Inner query executes first
- Result is fixed for all outer query rows
- Like a constant value

---

### **2. CORRELATED SUBQUERY**  
**Definition:** Inner query depends on outer query, runs multiple times.

**Example 1:** Employees earning above their department average
```sql
SELECT e1.name, e1.salary, e1.department
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e1.department  -- Changes per row!
);
```

**Example 2:** Products with the highest price in their category
```sql
SELECT p1.product_name, p1.category, p1.price
FROM products p1
WHERE price = (
    SELECT MAX(price)
    FROM products p2
    WHERE p2.category = p1.category  -- Changes per row!
);
```

**Characteristics:**
- Outer query row executes first
- Inner query runs for each outer query row (or unique value)
- Reference to outer query column (`e1.department`, `p1.category`)

---

### **3. VISUAL DIFFERENCE**

**Table: employees**
```
name    | department | salary
--------|------------|--------
Alice   | Sales      | 60,000
Bob     | Sales      | 45,000  
Charlie | IT         | 70,000
David   | IT         | 65,000
Eve     | HR         | 50,000
```

**Non-correlated query execution:**
```
Step 1: (SELECT AVG(salary) FROM employees) → 58,000
Step 2: Compare ALL rows to 58,000
```

**Correlated query execution:**
```
Row 1 (Alice, Sales): Compare to Sales avg = (60,000+45,000)/2 = 52,500
Row 2 (Bob, Sales): Compare to Sales avg = 52,500  
Row 3 (Charlie, IT): Compare to IT avg = (70,000+65,000)/2 = 67,500
Row 4 (David, IT): Compare to IT avg = 67,500
Row 5 (Eve, HR): Compare to HR avg = 50,000
```

---

### **4. WHEN TO USE WHICH**

**Use NON-CORRELATED when:**
- Comparing all rows to a single value
- Checking membership in a fixed list
- The inner query doesn't need data from outer query

**Use CORRELATED when:**
- Each row needs different comparison value
- Finding top/last items per group
- Row-by-row comparison needed

---

### **5. PERFORMANCE**
- **Non-correlated:** Generally faster (runs once)
- **Correlated:** Potentially slower (runs multiple times)
- **Alternative:** Correlated subqueries can often be rewritten as JOINs

**Example rewrite (correlated → JOIN):**
```sql
-- Correlated version
SELECT e1.name, e1.salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e1.department
);

-- JOIN version (often faster)
SELECT e1.name, e1.salary
FROM employees e1
JOIN (
    SELECT department, AVG(salary) as dept_avg
    FROM employees
    GROUP BY department
) dept_avgs ON e1.department = dept_avgs.department
WHERE e1.salary > dept_avgs.dept_avg;
```