# SQL Query Execution Order

Understanding the **logical execution order** of a SQL query is important for writing correct queries and answering SQL interview questions.

## Logical Execution Order

```text
FROM
  ↓
WHERE
  ↓
GROUP BY
  ↓
HAVING
  ↓
SELECT
  ↓
DISTINCT
  ↓
ORDER BY
  ↓
LIMIT
```

### In short

```text
FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

> **Note:** This is the **logical order in which SQL processes a query**, not the order in which we write the query.

---

## Example

```sql
SELECT department, COUNT(*) AS employee_count
FROM employees
WHERE salary > 50000
GROUP BY department
HAVING COUNT(*) > 5
ORDER BY employee_count DESC
LIMIT 10;
```

The database logically processes this query as follows:

### 1. FROM

First, SQL identifies the table from which the data will be retrieved.

```sql
FROM employees
```

At this point, SQL works with the rows from the `employees` table.

---

### 2. WHERE

Next, individual rows are filtered.

```sql
WHERE salary > 50000
```

Only employees whose salary is greater than `50000` remain.

**Important:** `WHERE` filters **rows**, not groups.

---

### 3. GROUP BY

The remaining rows are grouped.

```sql
GROUP BY department
```

Employees are divided into groups based on their department.

For example:

```text
Engineering → 12 employees
HR          → 7 employees
Finance     → 4 employees
```

---

### 4. HAVING

Now the groups themselves are filtered.

```sql
HAVING COUNT(*) > 5
```

Only departments containing more than 5 employees remain.

**Remember:**

```text
WHERE  → filters rows
HAVING → filters groups
```

---

### 5. SELECT

SQL determines which columns or expressions should appear in the final result.

```sql
SELECT department, COUNT(*) AS employee_count
```

The result now contains:

```text
department | employee_count
```

---

### 6. DISTINCT

Duplicate rows are removed if `DISTINCT` is specified.

```sql
SELECT DISTINCT department
```

For example:

```text
Engineering
Engineering
HR
HR
Finance
```

becomes:

```text
Engineering
HR
Finance
```

---

### 7. ORDER BY

The resulting rows are sorted.

```sql
ORDER BY employee_count DESC
```

`DESC` means descending order.

---

### 8. LIMIT

Finally, SQL restricts the number of rows returned.

```sql
LIMIT 10
```

Only the first 10 rows are returned.

---

## Syntax Order vs Logical Execution Order

### How we write SQL

```sql
SELECT ...
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
LIMIT ...
```

### How SQL logically processes it

```text
FROM
 ↓
WHERE
 ↓
GROUP BY
 ↓
HAVING
 ↓
SELECT
 ↓
DISTINCT
 ↓
ORDER BY
 ↓
LIMIT
```

This distinction is especially useful for understanding **SQL aliases and aggregate functions**.

---

## Interview Example: Why can't I normally use a SELECT alias in WHERE?

Consider:

```sql
SELECT salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 1000000;
```

This generally doesn't work because:

```text
WHERE → happens before → SELECT
```

The alias `annual_salary` is created during the `SELECT` phase, but `WHERE` has already been evaluated logically.

A common solution is to use a subquery:

```sql
SELECT annual_salary
FROM (
    SELECT salary * 12 AS annual_salary
    FROM employees
) t
WHERE annual_salary > 1000000;
```

---

## Easy Way to Remember

Think of it as:

```text
FROM       → Where does the data come from?
WHERE      → Which rows do I want?
GROUP BY   → How should I group them?
HAVING     → Which groups do I want?
SELECT     → What should I return?
DISTINCT   → Remove duplicates
ORDER BY   → How should I sort it?
LIMIT      → How many should I return?
```

### One-line interview answer

> **SQL's logical query processing order is FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT.**
