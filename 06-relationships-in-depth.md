Great. Relationships are probably the **most important JPA area after the basic mental model**. They are also where interview questions become tricky because JPA is simultaneously modeling:

1. Java object relationships
2. Database foreign keys
3. Which side owns the relationship
4. When related data is loaded
5. What happens when entities are saved/deleted

We'll build this from the database upward, since you already know PostgreSQL.

# 1. The fundamental idea

Suppose we have:

```text
users
----------------
id   PK
name


orders
----------------
id       PK
user_id  FK
amount
```

Database relationship:

```text
users
  1
  │
  │
  │ *
orders
```

One `User` has many `Order`s.

In Java we want:

```java
class User {
    List<Order> orders;
}

class Order {
    User user;
}
```

JPA provides annotations to tell Hibernate about this relationship:

```java
@OneToMany
@ManyToOne
```

The important thing is:

> **JPA relationships describe the relationship between Java objects, while `@JoinColumn`/foreign keys connect that relationship to the database.**

---

# 2. The four relationship types

You need to know these four:

| Java/JPA      | Meaning                       |
| ------------- | ----------------------------- |
| `@OneToOne`   | One entity ↔ one entity       |
| `@OneToMany`  | One entity ↔ many entities    |
| `@ManyToOne`  | Many entities ↔ one entity    |
| `@ManyToMany` | Many entities ↔ many entities |

The most common in real applications is:

```text
@ManyToOne
@OneToMany
```

---

# 3. `@ManyToOne` — start here

Consider:

```text
User
  1
  │
  │
  │ *
Order
```

Many orders belong to one user.

The database naturally represents this with:

```text
orders
----------------
id
user_id
amount
```

The foreign key is on `orders`.

Therefore:

```java
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    private BigDecimal amount;
}
```

This is saying:

> An `Order` has a reference to one `User`, and the database column `user_id` represents that relationship.

---

# 4. Understanding `@JoinColumn`

This:

```java
@JoinColumn(name = "user_id")
```

means:

```text
Order.user
     ↓
orders.user_id
     ↓
users.id
```

So:

```java
order.getUser()
```

corresponds to the foreign key:

```text
orders.user_id
```

You can think of it as telling Hibernate:

> "This is the database column that joins these two entities."

---

# 5. Adding the other side: `@OneToMany`

Now suppose we also want:

```java
user.getOrders();
```

We can add:

```java
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "user")
    private List<Order> orders = new ArrayList<>();
}
```

And:

```java
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
}
```

Now Java sees:

```text
User
 │
 │ orders
 ▼
Order
 │
 │ user
 ▼
User
```

The relationship is **bidirectional**.

---

# 6. The most important concept: owning side

This is probably the #1 thing you need to understand about JPA relationships.

Look at:

```java
@OneToMany(mappedBy = "user")
private List<Order> orders;
```

and:

```java
@ManyToOne
@JoinColumn(name = "user_id")
private User user;
```

Which side controls the database relationship?

**Order.**

Why?

Because:

```java
Order.user
```

contains:

```java
@JoinColumn(name = "user_id")
```

and `orders.user_id` is the actual foreign key.

Therefore:

```text
Order
  ↓
user
  ↓
user_id
  ↓
FOREIGN KEY
```

is the **owning side**.

---

# 7. What does `mappedBy` actually mean?

This:

```java
@OneToMany(mappedBy = "user")
```

means:

> "The relationship is mapped by the `user` field on the other entity."

Specifically:

```java
mappedBy = "user"
```

refers to:

```java
private User user;
```

inside `Order`.

It is **not** the database column name.

This is wrong:

```java
@OneToMany(mappedBy = "user_id")
```

because `user_id` is a database column.

This is correct:

```java
@OneToMany(mappedBy = "user")
```

because `user` is the Java field.

### Remember:

```text
mappedBy = Java field name
```

not:

```text
mappedBy = database column
```

---

# 8. Why does the owning side matter?

This is a very common interview question.

Imagine:

```java
User user = ...;
Order order = ...;

user.getOrders().add(order);
```

You changed the Java collection.

But if the owning side is:

```java
Order.user
```

you haven't actually told Hibernate:

```java
order.setUser(user);
```

So you should normally maintain **both sides**:

```java
user.getOrders().add(order);
order.setUser(user);
```

A nice pattern is to encapsulate this:

```java
public void addOrder(Order order) {
    orders.add(order);
    order.setUser(this);
}
```

Then:

```java
user.addOrder(order);
```

keeps the object graph consistent.

---

# 9. Very important interview trap

Suppose:

```java
@OneToMany(mappedBy = "user")
private List<Order> orders;
```

and:

```java
@ManyToOne
@JoinColumn(name = "user_id")
private User user;
```

If you only do:

```java
user.getOrders().add(order);
```

the database relationship may **not** be updated.

Why?

Because `User.orders` is the inverse side.

The owning side is:

```java
Order.user
```

So:

```java
order.setUser(user);
```

is the important operation for the relationship.

### Interview answer

> In a bidirectional relationship, the owning side is responsible for persisting the relationship. `mappedBy` marks the inverse side. Therefore, changes should be made on the owning side, while application code should usually keep both sides synchronized.

---

# 10. Unidirectional vs bidirectional

You don't necessarily need both sides.

You could have:

```java
@Entity
public class Order {

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
}
```

and no `orders` collection in `User`.

That's **unidirectional**:

```text
Order → User
```

You can navigate:

```java
order.getUser();
```

but not:

```java
user.getOrders();
```

Alternatively, you can have bidirectional:

```text
User → Orders
Order → User
```

### Which should you use?

Don't automatically make every relationship bidirectional.

Bidirectional relationships increase complexity:

- object graph management
- serialization problems
- lazy loading concerns
- potential circular references
- cascading behavior
- memory usage

A senior engineer should ask:

> "Do I actually need navigation in both directions?"

---

# 11. `@OneToOne`

Now consider:

```text
User
  1
  │
  │ 1
  ▼
UserProfile
```

Database:

```text
users
----------------
id
name


user_profiles
----------------
id
user_id UNIQUE
bio
```

The `UNIQUE` constraint ensures one profile per user.

Java:

```java
@Entity
public class User {

    @Id
    private Long id;

    @OneToOne
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}
```

Or the foreign key could live on `user_profiles`.

The key point:

> `@OneToOne` means one entity is associated with at most one instance of the other entity.

---

# 12. `@ManyToMany`

Consider:

```text
Student        Course

Student A ──── Java
     │
     └───────── Spring

Student B ──── Java
```

A student can take many courses.

A course can have many students.

That's:

```text
Many ↔ Many
```

Relational databases don't normally represent this with one foreign key.

You need a **join table**:

```text
students
---------
id
name

courses
---------
id
name

student_course
-------------
student_id
course_id
```

In JPA:

```java
@ManyToMany
@JoinTable(
    name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
private Set<Course> courses;
```

This tells Hibernate:

```text
Student
   ↓
student_course
   ↓
Course
```

---

# 13. Why `Set` is often preferred for `ManyToMany`

You frequently see:

```java
Set<Course> courses;
```

rather than:

```java
List<Course> courses;
```

because a many-to-many relationship usually represents a set of associations.

For example:

```text
Student → Java
Student → Java
```

shouldn't normally create duplicate associations.

But don't memorize "always use Set." The collection type should reflect the domain requirements.

---

# 14. `Cascade`

Now we reach another major interview topic.

Suppose:

```text
User
 |
 +--- Order
 +--- Order
 +--- Order
```

What happens when you save the User?

Should Hibernate automatically save the Orders?

That's what **cascade** controls.

Example:

```java
@OneToMany(
    mappedBy = "user",
    cascade = CascadeType.ALL
)
private List<Order> orders;
```

Now operations on the parent can propagate to children.

For example:

```java
userRepository.save(user);
```

can cascade persistence to the associated orders.

---

# 15. Cascade types

Know these:

```java
CascadeType.PERSIST
CascadeType.MERGE
CascadeType.REMOVE
CascadeType.REFRESH
CascadeType.DETACH
CascadeType.ALL
```

The most important:

```text
PERSIST
MERGE
REMOVE
ALL
```

### `PERSIST`

Persist parent → persist children.

### `MERGE`

Merge parent → merge children.

### `REMOVE`

Remove parent → remove children.

### `ALL`

All cascade operations.

---

# 16. Cascade ≠ database foreign key cascade

This is an important distinction.

JPA:

```java
cascade = CascadeType.REMOVE
```

is an **ORM behavior**.

Database:

```sql
ON DELETE CASCADE
```

is a **database behavior**.

They're not the same thing.

For example:

```text
JPA cascade
    ↓
Hibernate propagates operation

DB cascade
    ↓
PostgreSQL propagates operation
```

An interviewer may deliberately ask this.

---

# 17. `orphanRemoval`

Suppose:

```java
User
 |
 +--- Order A
 +--- Order B
```

and the relationship is:

```java
@OneToMany(
    mappedBy = "user",
    orphanRemoval = true
)
private List<Order> orders;
```

Now:

```java
user.getOrders().remove(orderA);
```

can cause Hibernate to delete `orderA` from the database.

That's because `orderA` became an **orphan**.

Conceptually:

```text
Before:

User → Order A
User → Order B

After:

User → Order B

Order A has no parent
       ↓
DELETE Order A
```

---

# 18. `CascadeType.REMOVE` vs `orphanRemoval`

This is a classic interview question.

### Cascade REMOVE

If the parent is deleted:

```java
entityManager.remove(user);
```

the removal can cascade to its children.

```text
DELETE User
    ↓
DELETE Orders
```

### orphanRemoval

If a child is removed from the parent's relationship:

```java
user.getOrders().remove(order);
```

the child can be deleted.

```text
remove from collection
        ↓
child becomes orphan
        ↓
DELETE child
```

### Easy way to remember

```text
Cascade REMOVE
    Parent deleted
        ↓
    Child deleted

orphanRemoval
    Child disconnected
        ↓
    Child deleted
```

---

# 19. Be careful with `CascadeType.ALL`

This is a senior-level consideration.

You shouldn't blindly write:

```java
cascade = CascadeType.ALL
```

on every relationship.

Imagine:

```text
Order → Customer
```

If you delete an Order, do you really want to delete the Customer?

Probably not.

So cascading should follow **ownership/domain semantics**, not simply convenience.

A typical aggregate relationship such as:

```text
Order
 ├── OrderItem
 ├── OrderItem
 └── OrderItem
```

may reasonably use:

```java
cascade = CascadeType.ALL,
orphanRemoval = true
```

because the order items belong to the order.

---

# 20. LAZY vs EAGER — now in relationship context

This deserves another look.

Example:

```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

When you load:

```java
User user = userRepository.findById(id).orElseThrow();
```

Hibernate doesn't necessarily immediately load all orders.

But:

```java
user.getOrders();
```

may trigger another SQL query.

This is good because imagine:

```text
User
 ├── 10,000 orders
 ├── 500 payments
 ├── 100 addresses
 └── 50,000 audit records
```

You probably don't want all of that loaded every time you retrieve a User.

---

# 21. Default fetch types

This is worth memorizing for the exam/interview:

| Relationship  | Default |
| ------------- | ------- |
| `@OneToMany`  | LAZY    |
| `@ManyToMany` | LAZY    |
| `@ManyToOne`  | EAGER   |
| `@OneToOne`   | EAGER   |

However, **default does not mean you should leave everything at the default**.

In practice, many teams explicitly prefer:

```java
@ManyToOne(fetch = FetchType.LAZY)
```

to avoid unexpected loading.

---

# 22. The N+1 problem with relationships

Let's make this concrete.

```java
List<Order> orders = orderRepository.findAll();

for (Order order : orders) {
    System.out.println(order.getUser().getName());
}
```

If `Order.user` is loaded lazily:

```text
Query 1:
SELECT * FROM orders;

Query 2:
SELECT * FROM users WHERE id = 1;

Query 3:
SELECT * FROM users WHERE id = 2;

Query 4:
SELECT * FROM users WHERE id = 3;

...
```

If there are 100 orders:

```text
1 + 100 = 101 queries
```

That's the **N+1 problem**.

---

# 23. Fetch join

One solution:

```java
@Query("""
    SELECT o
    FROM Order o
    JOIN FETCH o.user
""")
List<Order> findOrdersWithUsers();
```

Hibernate can generate SQL conceptually similar to:

```sql
SELECT ...
FROM orders o
JOIN users u
    ON o.user_id = u.id;
```

Instead of:

```text
1 query
+
N queries
```

you can fetch the necessary data efficiently.

---

# 24. But don't solve everything with JOIN FETCH

This is important for senior interviews.

Someone might say:

> "I'll just use JOIN FETCH everywhere."

That's not a good answer.

Large joins can cause:

- duplicate rows
- huge result sets
- memory pressure
- pagination complications
- Cartesian-product-like explosions when fetching multiple collections

For example:

```text
User
 ├── Orders
 ├── Payments
 └── Addresses
```

Trying to fetch all three collections in one giant query can produce a massive result set.

A better senior answer is:

> "I would design the query based on the use case, using fetch joins, entity graphs, DTO projections, batch fetching, or separate queries as appropriate, and verify the generated SQL and execution plan."

That's the type of answer an experienced interviewer likes.

---

# 25. The most important relationship diagram

You should be able to look at this:

```text
                    USER
                     │
                     │ 1
                     │
                     │ *
                     ▼
                   ORDER
                     │
                     │ 1
                     │
                     │ *
                     ▼
                 ORDER_ITEM
```

And model it as:

```java
class User {

    @OneToMany(mappedBy = "user")
    List<Order> orders;
}
```

```java
class Order {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    User user;

    @OneToMany(
        mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    List<OrderItem> items;
}
```

```java
class OrderItem {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    Order order;
}
```

Database:

```text
users
----------------
id
name


orders
----------------
id
user_id FK
total


order_items
----------------
id
order_id FK
product_id
quantity
```

This is a **very realistic Spring/JPA model**.

---

# 26. One subtle but important point: object graph vs database

Don't think:

```text
Java relationship = database relationship
```

They are related, but they're not identical.

Java:

```java
user.orders
```

is an object reference/collection.

Database:

```text
orders.user_id
```

is a foreign key.

Hibernate is the layer translating between them:

```text
Java object graph
        ↕
     Hibernate
        ↕
Relational database
```

That's essentially what ORM is doing.

---

# 27. Interview questions you should now be able to answer

These are the questions I'd expect in a Spring Boot interview.

### Basic

**1. What is `@ManyToOne`?**

Many instances of one entity are associated with one instance of another entity.

---

**2. What is `mappedBy`?**

It marks the inverse side of a bidirectional relationship and points to the Java field that owns the relationship.

---

**3. What is the owning side?**

The side responsible for mapping/persisting the relationship, normally the side containing the foreign-key mapping.

---

**4. Why is `@ManyToOne` usually the owning side of a one-to-many relationship?**

Because the foreign key normally exists on the "many" table.

```text
orders.user_id
```

therefore:

```java
Order.user
```

owns the relationship.

---

### Intermediate

**5. What's the difference between unidirectional and bidirectional relationships?**

Unidirectional allows navigation in one direction:

```text
Order → User
```

Bidirectional allows both:

```text
Order ↔ User
```

---

**6. What does `cascade` do?**

It propagates persistence operations from one entity to associated entities.

---

**7. What is `orphanRemoval`?**

It allows an associated child entity to be removed when it is no longer associated with its parent.

---

**8. What's the difference between cascade remove and orphan removal?**

```text
cascade REMOVE
parent deleted → child deleted

orphanRemoval
child removed from relationship → child deleted
```

---

**9. What is LAZY loading?**

Associated entities are loaded when needed rather than immediately.

---

**10. What is EAGER loading?**

Associated data is requested as part of loading the entity rather than being deferred.

---

### Senior

**11. What is the N+1 query problem?**

Loading a collection with one query and then triggering an additional query for each associated entity, producing `1 + N` database queries.

---

**12. How would you solve N+1?**

Possible solutions:

- `JOIN FETCH`
- `@EntityGraph`
- DTO projections
- batch fetching
- carefully designed queries

and verify the generated SQL.

---

**13. Would you make every relationship bidirectional?**

No. Only when navigation in both directions is actually useful. Bidirectional mappings add complexity and can create serialization and performance problems.

---

**14. Would you use `CascadeType.ALL` everywhere?**

No. Cascading should reflect ownership/domain semantics. For example, an `Order` may own its `OrderItem`s, but an `Order` generally should not own/delete its `Customer`.

---

# 28. The mental model I want you to memorize

If you remember only one thing from this lesson, remember this:

```text
              Java
               │
        ┌──────┴──────┐
        │             │
      User          Order
        │             │
   orders │           │ user
        │             │
        └──────┬──────┘
               │
            Hibernate
               │
               ▼
         Database FK
```

And:

```text
@OneToMany
    ↓
one → many

@ManyToOne
    ↓
many → one

@OneToOne
    ↓
one → one

@ManyToMany
    ↓
many ↔ many
```

Then the critical rules:

```text
@JoinColumn
    ↓
database FK mapping

mappedBy
    ↓
points to Java field on owning side

Owning side
    ↓
controls relationship persistence

LAZY
    ↓
load association when needed

EAGER
    ↓
load association eagerly

cascade
    ↓
propagate entity operations

orphanRemoval
    ↓
delete child when disconnected

JOIN FETCH
    ↓
common tool for avoiding N+1
```

## Where we should go next

The next topic should be **Transactions + Persistence Context in depth**, because it connects almost everything we've just learned:

```text
@Transactional
       ↓
Transaction begins
       ↓
Persistence Context
       ↓
Entity becomes MANAGED
       ↓
Dirty Checking
       ↓
Flush
       ↓
SQL
       ↓
COMMIT / ROLLBACK
```

That topic will also let us tackle several **very common interview traps**: `save()` vs dirty checking, `flush()` vs `commit()`, detached entities, `LazyInitializationException`, transaction propagation, and what actually happens when a Spring service method executes.
