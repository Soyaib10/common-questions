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