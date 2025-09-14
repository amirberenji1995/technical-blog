# ⚖️ Entities are Essential but Don't Get Trapped by Them

In this post, we’ll explore the **entity trap** in software design and why it undermines true domain-driven design (DDD). We’ll start with a *trapped version* of an online bookstore, then show how to fix it by embedding behavior into entities. Along the way, we’ll reflect on the role of services in MVC-style applications.

This discussion offers:

1. ✅ A clear definition of the entity trap
2. 🚫 A realistic Python case study showing the trapped version first
3. 🧬 Guidance on when to put logic inside entities vs. services
4. 📦 Practical takeaways for API developers using MVC + services

## 📢 Attention! This post is co-authored by GPT-5 from OpenAI.

---

## 🧱 The Problem

In many projects, developers structure their applications around **entities** (like `User`, `Order`, `Product`) but treat them as *data-only classes*. All the business logic is pushed into separate services or utility modules.

This leads to:

* Anemic domain models — objects with data but no real behavior
* Procedural code hidden behind OOP wrappers
* Difficulty scaling and evolving business rules
* Tight coupling between layers

This anti-pattern is known as the **entity trap**.

---

## 🎯 The Goal

We want entities to be **first-class citizens** in the domain model. That means:

* ✨ Entities carry both data *and* the rules that govern it
* 🔒 Business invariants (rules that must always hold true) are enforced in the model itself
* 🧩 Behavior is located where it belongs, reducing the need for “god services”
* ♻️ Services exist, but they orchestrate — they don’t own core rules

---

## 🏗️ Case Study: Online Bookstore

Let’s model a simple bookstore where customers can place orders for books. We’ll start with the **entity trap version**, then show the corrected one.

---

### ❌ Entity Trap Version

Here, entities are dumb data holders. All rules are pushed to a service.

```python
from datetime import datetime

class Book:
    def __init__(self, title: str, price: float, stock: int):
        self.title = title
        self.price = price
        self.stock = stock


class Order:
    def __init__(self, customer_name: str):
        self.customer_name = customer_name
        self.items = []
        self.total = 0
        self.placed_at = None


class OrderService:
    def add_item(self, order: Order, book: Book, quantity: int):
        if quantity > book.stock:
            raise ValueError("Not enough stock available.")
        book.stock -= quantity
        cost = book.price * quantity
        order.items.append((book.title, quantity, cost))
        order.total += cost

    def place_order(self, order: Order):
        if not order.items:
            raise ValueError("Cannot place an empty order.")
        order.placed_at = datetime.now()
        return f"Order placed by {order.customer_name} at {order.placed_at}, total = {order.total}"
```

👉 `Book` and `Order` are just data structures. `OrderService` becomes bloated with rules that should belong to entities.

This is the **entity trap**.

---

### ✅ Correct DDD-Inspired Version

Here, entities contain both state and behavior.

```python
from datetime import datetime

class Book:
    def __init__(self, title: str, price: float, stock: int):
        self.title = title
        self.price = price
        self.stock = stock

    def reserve(self, quantity: int):
        if quantity > self.stock:
            raise ValueError("Not enough stock available.")
        self.stock -= quantity
        return self.price * quantity


class Order:
    def __init__(self, customer_name: str):
        self.customer_name = customer_name
        self.items = []
        self.total = 0
        self.placed_at = None

    def add_item(self, book: Book, quantity: int):
        cost = book.reserve(quantity)
        self.items.append((book.title, quantity, cost))
        self.total += cost

    def place(self):
        if not self.items:
            raise ValueError("Cannot place an empty order.")
        self.placed_at = datetime.now()
        return f"Order placed by {self.customer_name} at {self.placed_at}, total = {self.total}"
```

👉 The business rules live in the right places:

* `Book` knows how to reserve stock.
* `Order` knows it can’t be empty.

The model is **rich** and self-validating.

---

## 🧠 Where Should Logic Live?

A good rule of thumb:

* Put logic **inside the entity** if it directly enforces business rules for that entity (e.g., stock limits, minimum order size).
* Use **domain services** only for behavior that spans multiple entities or is not naturally the responsibility of a single entity (e.g., generating reports, sending notifications).

---

## 🧩 But What About MVC + Services?

If you’re building APIs with an MVC-style structure (models, views/controllers, services):

* Having a **services module** is *not inherently* an example of the entity trap.
* The trap happens when the **services module becomes the dumping ground** for *all* business logic, leaving entities hollow.
* A healthy design:

  * Entities own their rules.
  * Services orchestrate across multiple entities or handle logic that doesn’t belong to any single one.
  * Controllers focus on I/O (HTTP requests, responses).

### Example: A Service That Belongs in the Services Layer

```python
class ReportingService:
    def generate_sales_report(self, orders: list):
        total_revenue = sum(order.total for order in orders if order.placed_at)
        total_orders = len([o for o in orders if o.placed_at])
        return {
            "total_orders": total_orders,
            "total_revenue": total_revenue,
            "average_order_value": total_revenue / total_orders if total_orders else 0
        }
```

👉 This logic spans multiple `Order` entities and does not naturally belong to any single one. It’s a perfect fit for the **services module**.

---

## 🔚 Wrap-up

Entities are not just buckets of data. They are the **heart of your domain model**. Avoid the entity trap by:

* Embedding rules and operations directly into entities
* Keeping services lean and focused on orchestration
* Ensuring your model reflects both *what the system is* and *what it does*

This shift turns your code from a procedural structure disguised as OOP into a **truly domain-driven design**.

🚀 Next time you model an entity, ask: *“What rules does this object own?”* — and put them right there.
