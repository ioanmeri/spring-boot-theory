Yes. This is the **next major block** I would study for your interview.

Since you've already covered:

```text
JPA fundamentals
→ Relationships
→ Persistence Context
→ Transactions
→ Advanced Spring Data JPA
```

we can now focus on **what Hibernate is actually doing underneath JPA**, especially the things that cause **performance problems in production**.

# Advanced Hibernate + JPA Performance

I recommend mastering these topics in this order:

```text
1. Hibernate Session / EntityManager internals
2. Hibernate proxies
3. Lazy loading internals
4. Fetch strategies
5. JOIN FETCH
6. EntityGraph
7. N+1 queries
8. Batch fetching
9. JDBC batching
10. Cascades and orphanRemoval
11. Persistence context memory management
12. Flush modes
13. Query performance
14. Pagination pitfalls
15. Optimistic & pessimistic locking
16. Database connection pools
17. Hibernate SQL/logging/statistics
18. Second-level cache
19. Common Hibernate performance anti-patterns
```

Let's go through them.

---

# 1. Hibernate Session vs JPA EntityManager ⭐⭐⭐⭐⭐

You already know:

```java
@PersistenceContext
EntityManager entityManager;
```

`EntityManager` is the **JPA abstraction**.

Hibernate has its own API:

```java
Session session;
```

Conceptually:

```text
JPA
 │
 └── EntityManager
          │
          ↓
       Hibernate
          │
          └── Session
```

Hibernate's `Session` provides the actual Hibernate implementation of the persistence context.

So you will often hear:

> "Hibernate Session"

and:

> "JPA Persistence Context"

in discussions about essentially the same underlying unit of entity management, although the APIs/specification concepts aren't literally identical.

---

## Important interview point

Don't say:

> "EntityManager and Session are exactly the same thing."

Better:

> `EntityManager` is the standard JPA API, while Hibernate `Session` is Hibernate's native API. A Hibernate `Session` implements the persistence-context behavior required by JPA.

---

# 2. Hibernate Proxies ⭐⭐⭐⭐⭐

This is important for understanding lazy loading.

Suppose:

```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

You load an Order:

```java
Order order = repository.findById(id).orElseThrow();
```

Hibernate may not immediately load the Customer.

Instead, it can provide a **proxy/lazy representation**.

Conceptually:

```text
Order
 │
 └── customer
       ↓
   Hibernate Proxy
       ↓
   Database only when needed
```

When you do:

```java
order.getCustomer().getName();
```

Hibernate may initialize the proxy and execute SQL.

---

# 2.1 Why proxies exist

Imagine:

```text
Order
Customer
Address
Country
Invoices
Payments
...
```

If Hibernate loaded the entire object graph immediately, one simple query could cause massive amounts of data to be loaded.

Lazy proxies allow Hibernate to defer loading until necessary.

---

# 3. Lazy Loading Internals ⭐⭐⭐⭐⭐

Let's make the lifecycle explicit.

You execute:

```java
Order order = repository.findById(1L).orElseThrow();
```

Hibernate might execute:

```sql
SELECT *
FROM orders
WHERE id = 1;
```

But not:

```sql
SELECT *
FROM customer
WHERE id = ...;
```

The association is still uninitialized.

Then:

```java
order.getCustomer().getName();
```

causes Hibernate to initialize the association:

```text
getCustomer()
    ↓
proxy initialization
    ↓
SELECT customer...
    ↓
Customer loaded
```

This is the mechanism behind many **N+1 problems**.

---

# 3.1 The LazyInitializationException

If the persistence context is already closed:

```java
order.getCustomer().getName();
```

may fail with:

```text
LazyInitializationException
```

because Hibernate needs its session/persistence context to initialize the association.

This is why you should understand:

```text
Transaction
    ↓
Persistence Context
    ↓
Hibernate Session
    ↓
Lazy association
```

---

# 4. Fetch Strategies ⭐⭐⭐⭐⭐

This is one of the most important Hibernate performance topics.

There are two concepts people often mix together:

### Fetch type

```java
FetchType.LAZY
FetchType.EAGER
```

### Fetch strategy

How Hibernate actually retrieves the related data.

For example:

```text
SELECT
JOIN
SUBSELECT
BATCH
```

These are not exactly the same concept.

---

# 4.1 LAZY

```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

Means:

> Don't necessarily load the association immediately.

This is generally a good default for many associations.

---

# 4.2 EAGER

```java
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

Means the association is required to be eagerly available according to JPA semantics.

But **don't interpret this as "Hibernate will always use one JOIN."**

It may still execute additional queries depending on the situation.

This distinction is important.

---

# 5. JOIN FETCH ⭐⭐⭐⭐⭐

Suppose:

```java
@Entity
class Order {

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;
}
```

You want orders **and their customers**.

You can write:

```java
@Query("""
    SELECT o
    FROM Order o
    JOIN FETCH o.customer
""")
List<Order> findOrdersWithCustomers();
```

Now Hibernate can retrieve the relationship as part of the query.

Conceptually:

```sql
SELECT ...
FROM orders o
JOIN customer c
    ON ...
```

The important part is:

```text
JOIN FETCH
```

not simply:

```text
JOIN
```

---

# 5.1 JOIN vs JOIN FETCH

This is a very common interview question.

### JOIN

Used primarily to participate in query logic:

```java
SELECT o
FROM Order o
JOIN o.customer c
WHERE c.name = :name
```

You're using the relationship in the query.

### JOIN FETCH

Also tells Hibernate:

> Initialize this association as part of loading the entity.

```java
SELECT o
FROM Order o
JOIN FETCH o.customer
```

---

# 5.2 The collection fetch join problem

Suppose:

```java
Customer
   |
   └── orders
```

You do:

```java
SELECT c
FROM Customer c
JOIN FETCH c.orders
```

SQL produces one row for every customer/order combination.

For example:

```text
Customer 1 → Order 1
Customer 1 → Order 2
Customer 1 → Order 3
Customer 2 → Order 4
```

The SQL result has four rows.

But your Java result should contain:

```text
Customer 1
Customer 2
```

This is why you'll often see:

```java
SELECT DISTINCT c
FROM Customer c
JOIN FETCH c.orders
```

---

# 5.3 Collection fetch joins + pagination ⚠️

This is an important senior interview issue.

Don't casually combine:

```text
JOIN FETCH collection
+
Pageable
```

because pagination is fundamentally applied to rows, while a collection fetch join multiplies rows.

Hibernate may have to handle pagination in memory or otherwise produce inefficient behavior depending on the exact query/version/configuration.

For large datasets, this can become a serious performance issue.

A common solution is a **two-step query**:

```text
1. Fetch page of IDs
        ↓
2. Fetch entities + associations
        ↓
3. Reassemble results
```

This is a very useful architectural pattern.

---

# 6. EntityGraph ⭐⭐⭐⭐⭐

An alternative to writing fetch joins everywhere is:

```java
@EntityGraph(attributePaths = {"customer"})
List<Order> findByStatus(OrderStatus status);
```

This tells Spring Data/Hibernate:

> For this query, fetch the specified relationship.

The important architectural idea is:

```text
Default mapping:
customer = LAZY

Use case A:
customer required → fetch it

Use case B:
customer not required → don't fetch it
```

This is generally better than making the association globally EAGER.

---

# JOIN FETCH vs EntityGraph

|                  | JOIN FETCH          | EntityGraph    |
| ---------------- | ------------------- | -------------- |
| Defined in query | Yes                 | No             |
| JPQL needed      | Usually             | No             |
| Explicit         | Very                | Very           |
| Reusable         | Query-specific      | Often reusable |
| Great for        | Complex fetch logic | Fetch plans    |

Both are important tools.

---

# 7. N+1 Queries ⭐⭐⭐⭐⭐

You've already seen this, but at this level you need to diagnose it.

Suppose:

```java
List<Order> orders = repository.findAll();
```

One query:

```sql
SELECT * FROM orders;
```

Then:

```java
for (Order order : orders) {
    System.out.println(order.getCustomer().getName());
}
```

Hibernate might produce:

```text
1 query for orders
+
N queries for customers
```

For 100 orders:

```text
101 SQL queries
```

That's N+1.

---

# 7.1 Why N+1 is so bad

It's not necessarily the number of rows that's the main issue.

It's the **number of database round trips**.

For example:

```text
Application
    ↓
DB query
    ↓
network
    ↓
DB
    ↓
network
    ↓
Application
```

Repeat that 100 times and latency becomes significant.

---

# 7.2 N+1 solutions

You should know at least these:

### Solution 1 — JOIN FETCH

```java
JOIN FETCH o.customer
```

### Solution 2 — EntityGraph

```java
@EntityGraph(attributePaths = "customer")
```

### Solution 3 — DTO projection

Retrieve exactly what you need.

### Solution 4 — Batch fetching

Group lazy loads into batches.

---

# 8. Batch Fetching ⭐⭐⭐⭐

Suppose Hibernate sees:

```text
Order 1 → Customer 1
Order 2 → Customer 2
Order 3 → Customer 3
...
```

Instead of:

```sql
SELECT * FROM customer WHERE id = 1;
SELECT * FROM customer WHERE id = 2;
SELECT * FROM customer WHERE id = 3;
```

batch fetching can allow:

```sql
SELECT *
FROM customer
WHERE id IN (1,2,3,...);
```

For example:

```java
@BatchSize(size = 20)
```

Conceptually:

```text
N queries

        ↓

N / 20 queries
```

It's a useful optimization, but don't think of it as automatically better than a carefully designed fetch query.

---

# 9. JDBC Batching ⭐⭐⭐⭐⭐

This is different from batch fetching.

### Batch fetching

Optimizes **SELECTing related entities**.

### JDBC batching

Optimizes **INSERT/UPDATE statements**.

Suppose:

```java
for (...) {
    entityManager.persist(order);
}
```

Without batching:

```text
INSERT
INSERT
INSERT
INSERT
...
```

With JDBC batching, Hibernate can group statements for the JDBC driver.

Conceptually:

```text
1000 INSERTs
      ↓
batches of 50
      ↓
20 JDBC batches
```

This can dramatically improve bulk write performance.

---

# 9.1 Important distinction

Memorize:

```text
Batch fetching
    → loading related entities

JDBC batching
    → grouping INSERT/UPDATE/DELETE operations
```

Interviewers love this distinction.

---

# 10. Cascades ⭐⭐⭐⭐⭐

You should already know the basics, but now think about **performance and lifecycle consequences**.

Suppose:

```java
@OneToMany(
    mappedBy = "customer",
    cascade = CascadeType.ALL
)
private List<Order> orders;
```

Then operations on Customer can cascade to Orders.

For example:

```java
entityManager.persist(customer);
```

can persist associated orders.

---

## Cascade types

Know:

```text
PERSIST
MERGE
REMOVE
REFRESH
DETACH
ALL
```

Important:

> Cascade is about propagating entity lifecycle operations.

It is **not the same thing as database foreign-key cascading**.

---

# 10.1 Cascade REMOVE danger

Suppose:

```text
Customer
  └── Orders
       └── Payment
```

Using:

```java
cascade = CascadeType.ALL
```

carelessly can cause a remove operation to propagate through a large graph.

That's a potential production problem.

Don't automatically use:

```java
CascadeType.ALL
```

everywhere.

---

# 11. `orphanRemoval` ⭐⭐⭐⭐⭐

Suppose:

```java
@OneToMany(
    mappedBy = "customer",
    orphanRemoval = true
)
private List<Order> orders;
```

You remove:

```java
customer.getOrders().remove(order);
```

Hibernate can interpret that as:

> This Order is now an orphan and should be deleted.

So it may execute:

```sql
DELETE FROM orders
WHERE id = ?;
```

This is different from simply:

```text
cascade = REMOVE
```

### Key distinction

```text
cascade REMOVE
    → deleting parent propagates removal

orphanRemoval
    → removing child from relationship can delete child
```

Very important interview distinction.

---

# 12. Persistence Context Memory Management ⭐⭐⭐⭐

Consider processing:

```text
1,000,000 records
```

with:

```java
for (...) {
    entityManager.persist(entity);
}
```

If you never clear the persistence context, Hibernate may keep references to a huge number of managed entities.

You can end up with:

```text
Memory
  ↑
  │
  │        /
  │      /
  │    /
  │  /
  │ /
  └────────── records
```

For large batch jobs, you may periodically:

```java
entityManager.flush();
entityManager.clear();
```

For example:

```java
for (int i = 0; i < items.size(); i++) {

    entityManager.persist(items.get(i));

    if (i % 50 == 0) {
        entityManager.flush();
        entityManager.clear();
    }
}
```

This:

```text
flush()
```

synchronizes pending changes.

Then:

```text
clear()
```

detaches managed entities.

This is a very important practical Hibernate optimization.

---

# 13. Flush Modes ⭐⭐⭐⭐

Hibernate needs to decide:

> When should the persistence context synchronize with the database?

JPA defines flush modes such as:

```text
AUTO
COMMIT
```

Hibernate has additional capabilities/modes.

The common conceptual difference:

### AUTO

Hibernate may flush when necessary, including before transaction completion and potentially before queries when required for consistency.

### COMMIT

Try to delay flushing until transaction completion.

You don't need to memorize every Hibernate-specific detail initially.

Understand the principle:

```text
flush mode
    ↓
controls when pending changes are synchronized
```

---

# 14. Query Performance ⭐⭐⭐⭐⭐

This is where you move from:

> "I know JPA."

to:

> "I can diagnose a production Hibernate problem."

Suppose:

```java
List<Customer> customers =
    repository.findCustomers(...);
```

The application is slow.

Don't immediately blame Hibernate.

Investigate:

```text
Application
    ↓
Hibernate
    ↓
Generated SQL
    ↓
Database
    ↓
Execution plan
```

---

# 14.1 Always understand the SQL

For performance debugging, you need to be comfortable looking at:

```sql
SELECT ...
FROM customer c
JOIN ...
WHERE ...
ORDER BY ...
```

Then ask:

- Is there an index?
- Is the query using the index?
- Is there a full table scan?
- How many rows are returned?
- Is a join exploding the result set?
- Is pagination being used?
- Is a count query expensive?

---

# 14.2 Database `EXPLAIN`

For PostgreSQL, for example:

```sql
EXPLAIN ANALYZE
SELECT ...
```

You can inspect:

- sequential scans
- index scans
- joins
- estimated vs actual rows
- execution time
- sorting
- aggregation

Given your PostgreSQL background, this should be an important part of your Spring performance toolkit.

---

# 15. Pagination Pitfalls ⭐⭐⭐⭐⭐

You already know:

```java
Pageable
```

but now consider the database.

Traditional pagination:

```sql
LIMIT 20 OFFSET 1000000;
```

can become expensive.

The database may have to process/skip a large number of rows.

For very large datasets, consider **keyset/seek pagination**.

Instead of:

```text
page = 50000
```

use something like:

```sql
WHERE id > :lastSeenId
ORDER BY id
LIMIT 20
```

Conceptually:

```text
First request:
id > 0
LIMIT 20

Next:
id > 20
LIMIT 20

Next:
id > 40
LIMIT 20
```

This is often much more scalable for large datasets.

---

# 15.1 Fetch join + pagination

Remember this particularly important combination:

```text
Collection
+
JOIN FETCH
+
Pagination
```

can be problematic.

For example:

```java
Page<Customer> findCustomersWithOrders(Pageable pageable);
```

with:

```text
JOIN FETCH customer.orders
```

can produce a multiplied result set.

A safer architecture for large data can be:

```text
Query 1
    ↓
Get customer IDs for page
    ↓
Query 2
    ↓
Fetch customers + orders
    ↓
Reconstruct requested order
```

This is a very good senior-level discussion point.

---

# 16. Optimistic Locking ⭐⭐⭐⭐⭐

You already saw this with transactions, but now connect it to Hibernate.

```java
@Version
private Long version;
```

Suppose:

```text
Account
id = 10
version = 5
```

Hibernate updates using the version:

```sql
UPDATE account
SET balance = ?,
    version = 6
WHERE id = 10
AND version = 5;
```

If another transaction changed it first:

```text
version = 6
```

the update affects zero rows.

Hibernate detects the conflict.

This prevents silent lost updates.

---

# 17. Pessimistic Locking ⭐⭐⭐⭐⭐

You can request a database lock:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
```

Conceptually:

```sql
SELECT ...
FROM account
WHERE id = ?
FOR UPDATE;
```

The database locks the row.

Useful when:

- contention is high
- you absolutely need serialized access
- the business operation requires a lock

But excessive pessimistic locking can lead to:

- blocking
- deadlocks
- reduced concurrency
- connection pool pressure

So don't use it by default.

---

# 18. Connection Pooling ⭐⭐⭐⭐⭐

This is often overlooked in Hibernate interviews.

Your application isn't normally opening a brand-new database connection for every query.

Spring Boot commonly uses **HikariCP** as the connection pool.

Conceptually:

```text
Spring Boot
     ↓
HikariCP
     ↓
Connection Pool
 ┌───┬───┬───┬───┐
 │ C │ C │ C │ C │
 └───┴───┴───┴───┘
     ↓
 Database
```

A connection is borrowed:

```text
request
  ↓
get connection
  ↓
execute SQL
  ↓
release connection
```

---

## Connection pool exhaustion

Suppose:

```text
Pool size = 10
```

but you have 100 concurrent requests all holding connections for a long time.

Requests start waiting.

Potential causes:

- long transactions
- slow queries
- deadlocks
- connection leaks
- excessive DB work

This is why:

> **Long transactions are a performance problem.**

---

# 19. Hibernate SQL Logging / Statistics ⭐⭐⭐⭐⭐

For diagnosing Hibernate issues, you need visibility.

You should know concepts such as:

```text
SQL logging
bind parameter logging
Hibernate statistics
```

You want to discover things like:

```text
Why did this endpoint execute 127 queries?
```

or:

```text
Why did Hibernate execute this SELECT?
```

or:

```text
Why did loading 20 objects cause 100 queries?
```

---

# 20. Second-Level Cache ⭐⭐⭐

Hibernate can have a second-level cache.

Remember:

```text
Persistence Context
     ↓
First-level cache
```

versus:

```text
EntityManagerFactory / SessionFactory
     ↓
Second-level cache
```

The second-level cache can be shared across persistence contexts.

Conceptually:

```text
Transaction A
   ↓
Persistence Context
   ↓
Second-level cache
   ↑
Persistence Context
   ↑
Transaction B
```

Potentially useful for relatively stable, frequently-read data.

But caching isn't automatically beneficial.

You need to consider:

- invalidation
- consistency
- memory
- hit rate
- distributed deployments

---

# 21. Common Hibernate Performance Anti-Patterns ⭐⭐⭐⭐⭐

These are particularly valuable for interviews.

---

## Anti-pattern 1

### Everything EAGER

```java
@ManyToMany(fetch = FetchType.EAGER)
```

Result:

```text
huge object graphs
+
unexpected queries
+
memory usage
```

Better:

```text
LAZY defaults
+
explicit fetch plans
```

---

## Anti-pattern 2

### Fix N+1 with EAGER

Wrong:

```text
N+1
 ↓
EAGER
```

Better:

```text
N+1
 ↓
JOIN FETCH / EntityGraph / DTO / batching
```

---

## Anti-pattern 3

### Returning entities directly from REST

For example:

```java
@GetMapping
public List<Customer> getCustomers() {
    return repository.findAll();
}
```

Potential problems:

- lazy loading during serialization
- huge object graphs
- N+1 queries
- API tightly coupled to persistence model

Prefer DTOs for many API use cases.

---

## Anti-pattern 4

### Large persistence context

```java
for (1_000_000 records) {
    entityManager.persist(...);
}
```

without:

```java
flush();
clear();
```

Potential memory/performance issues.

---

## Anti-pattern 5

### Long transactions

Don't do:

```text
BEGIN
 ↓
query
 ↓
remote HTTP request
 ↓
wait 2 seconds
 ↓
another query
 ↓
process data
 ↓
COMMIT
```

You're holding transactional/database resources while waiting on an external system.

Better architecture often separates the external call from the database transaction or minimizes the transaction boundary.

---

# The Most Important Architecture Diagram

At this point, you should understand the complete pipeline:

```text
             HTTP Request
                  │
                  ▼
             Controller
                  │
                  ▼
              Service
           @Transactional
                  │
                  ▼
          Spring Data JPA
                  │
                  ▼
             EntityManager
                  │
                  ▼
         Hibernate Session
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
 Persistence Context    Query Engine
        │                   │
        │                   ▼
        │              Generated SQL
        │                   │
        ▼                   ▼
 Dirty Checking       JDBC Connection
        │                   │
        └─────────┬─────────┘
                  ▼
             HikariCP
                  │
                  ▼
              PostgreSQL
```

And performance problems can occur at **every layer**.

---

# ⭐ Interview Priority

For your interview, I would rank the topics:

| Topic                      |   Priority |
| -------------------------- | ---------: |
| N+1 queries                | ⭐⭐⭐⭐⭐ |
| Lazy loading               | ⭐⭐⭐⭐⭐ |
| JOIN FETCH                 | ⭐⭐⭐⭐⭐ |
| EntityGraph                | ⭐⭐⭐⭐⭐ |
| Fetch strategies           | ⭐⭐⭐⭐⭐ |
| Hibernate proxies          | ⭐⭐⭐⭐⭐ |
| JDBC batching              | ⭐⭐⭐⭐⭐ |
| Persistence context memory |   ⭐⭐⭐⭐ |
| Cascades                   | ⭐⭐⭐⭐⭐ |
| `orphanRemoval`            | ⭐⭐⭐⭐⭐ |
| Pagination                 | ⭐⭐⭐⭐⭐ |
| Optimistic locking         | ⭐⭐⭐⭐⭐ |
| Pessimistic locking        |   ⭐⭐⭐⭐ |
| SQL / query plans          | ⭐⭐⭐⭐⭐ |
| Connection pooling         | ⭐⭐⭐⭐⭐ |
| Flush modes                |   ⭐⭐⭐⭐ |
| Batch fetching             |   ⭐⭐⭐⭐ |
| Second-level cache         |     ⭐⭐⭐ |

---

# The Questions You Should Be Able to Answer

After this module, you should be able to answer these without memorized definitions:

### Hibernate internals

1. What is the relationship between `EntityManager`, persistence context, and Hibernate `Session`?
2. What is a Hibernate proxy?
3. How does lazy loading actually work?
4. Why does `LazyInitializationException` occur?

### Fetching

5. What's the difference between `LAZY` and `EAGER`?
6. What's the difference between `JOIN` and `JOIN FETCH`?
7. When would you use `EntityGraph`?
8. When would you use a DTO projection?
9. Why shouldn't you simply make everything EAGER?

### Performance

10. What is N+1?
11. How would you detect N+1?
12. How would you fix N+1?
13. What's batch fetching?
14. What's JDBC batching?
15. How do you optimize a large batch insert?
16. Why can a large persistence context consume significant memory?

### Query design

17. Why can `JOIN FETCH` + collection + pagination be problematic?
18. How would you implement efficient pagination for millions of rows?
19. When would you use keyset pagination?
20. How would you investigate a slow Hibernate query?

### Concurrency

21. How does `@Version` work?
22. Optimistic vs pessimistic locking?
23. When would you use `PESSIMISTIC_WRITE`?
24. How can locking cause performance problems?

### Production

25. What is HikariCP?
26. What causes connection pool exhaustion?
27. Why are long transactions dangerous?
28. How would you diagnose an endpoint that suddenly executes 500 SQL queries?

---

# ⭐ The most important scenario

Imagine the interviewer says:

> **"We have an endpoint that returns 100 customers. Each customer has orders. The endpoint currently takes 4 seconds and generates 101 SQL queries. How would you fix it?"**

You should think:

```text
101 queries
     ↓
N+1
     ↓
Why?
     ↓
Lazy association accessed repeatedly
     ↓
Possible solutions:
     ├── JOIN FETCH
     ├── EntityGraph
     ├── DTO projection
     └── batch fetching
```

Then you should **not** blindly say:

> "Make orders EAGER."

Instead:

> "I'd first inspect the generated SQL and identify the N+1. If the endpoint genuinely requires the orders, I'd use a query-specific fetch plan such as a fetch join, entity graph, or DTO projection. If pagination is involved with a collection fetch, I'd be careful about row multiplication and may use a two-step ID pagination strategy. If appropriate, batch fetching can reduce round trips as another option."

That is the level of answer I'd aim for in your interview.

---

## Where we should go next

At this point, we've covered the **theory** of Advanced Hibernate/JPA performance. The next thing I'd recommend is **not moving to another Spring topic yet**.

We should do a dedicated deep dive into the most heavily tested area:

> **LAZY vs EAGER → Hibernate proxies → JOIN vs JOIN FETCH → EntityGraph → N+1 → batch fetching → pagination + fetch joins**

with concrete entities, generated SQL, and **interview scenarios/trick questions**.

That will turn the theory above into something you can actually reason about during a senior interview.
