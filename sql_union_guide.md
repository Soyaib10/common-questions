## **What is UNION ALL?**

`UNION ALL` simply combines the results of two or more SELECT statements **including all duplicate rows**.

Think of it like taking two piles of paper and putting them together into one big pile. All pages stay in the pile.

## **What is UNION?**

`UNION` combines the results of two or more SELECT statements **but removes duplicate rows**.

Think of it like taking two piles of paper, putting them together, then removing any identical pages.

---

**Visual Example:**

If Table A has: Apple, Banana, Apple

And Table B has: Banana, Cherry, Banana

- `UNION ALL` gives: Apple, Banana, Apple, Banana, Cherry, Banana (6 items)
- `UNION` gives: Apple, Banana, Cherry (3 unique items)

---
*The following section refers to hypothetical `employees_ny` and `employees_sf` tables derived from previous examples.*

**Task for you:**

Look back at our two employee tables. If we use:

1. `UNION ALL` to combine them, how many total rows will we get?
2. `UNION` to combine them, how many total rows will we get?

**Task :** You need to create a contact list of all employees from both offices. But you want to:

1. Include ALL employees (even if in both offices)
2. Add a column showing which office they're from
3. Sort by name alphabetically

Write a query using `UNION ALL` that shows:

- Employee name
- Department
- Office location ('NY' or 'SF')

**ANS:** 

```sql
SELECT id, name, department, 'NY' as office
FROM employees_ny
UNION ALL
SELECT id, name, department, 'SF' as office
FROM employees_sf
ORDER BY name;
```
