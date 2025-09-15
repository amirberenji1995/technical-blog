# 🕳️ Rules Lost in the Sinkhole: How to Avoid Centralizing All Logic in One Layer

In this post, we explore the **sinkhole problem** in software development. This anti-pattern occurs when most of the important business logic sinks into a single layer (often services or controllers), leaving entities and schemas hollow. We’ll examine this in a FastAPI app, then demonstrate a cleaner design that distributes responsibility properly.

#### 📢 Attention! This post is co-authored by GPT-5 from OpenAI.

---

## 🧱 The Problem

Many software applications utilize the layered architecture style, at some level. A frequent caveat of the layered style is when the business logics and rules are concentrated heavily in particular layers, instead of being distributed logically, according to the nature of the layers (e.g. handling the data validation in the controller or services, instead of the schemas).

In the presence of the sinkhole problem, most of the layers within the software hierarchy are just pass-throughs that do not contribute effectively to the processing of the data; this not only violates the separation of concerns, but also makes the logic-enriched layer the bottleneck of the system.

Consider a Model-View-Controller architecture with an additional services module; each component of such architecture is responsible for:
* **Models**: represent domain objects and enforce their own invariants.
* **Schemas** (Views): provide request/response validation and serialization.
* **Controllers**: translate between HTTP and the domain, orchestrating calls without embedding rules.
* **Services**: encapsulate cross-entity workflows and business processes.

When the sinkhole problem appears, entities and schemas become hollow, while controllers and services are overloaded with logic, breaking this intended balance.

---

## 🎯 The Goal

We want to restore balance to the layered architecture by distributing responsibilities where they belong:

* 🧩 **Entities** encapsulate their own rules and behaviors (using Pydantic for validation and invariants).
* 🧹 **Services** orchestrate workflows across multiple entities without becoming bloated god-classes.
* 📦 **Schemas** focus on validating request payloads and structuring response data.
* 🎛️ **Controllers** remain slim, focusing on HTTP <-> domain translation rather than housing business logic.


---

## 🏗️ Implementation

Let’s build a **simple order management app** with `models.py`, `schemas.py`, `services.py`, and `controllers.py`. We’ll first show the sinkhole version, then the corrected version.

---

### 🚨 Case Study: Sinkhole Version (Bad)

#### `models.py`

```python
from pydantic import BaseModel
from typing import List

class OrderItem(BaseModel):
    product_id: str
    quantity: int

class Order(BaseModel):
    id: str
    customer_id: str
    items: List[OrderItem]
    total: float = 0.0
```

👉 Entities are hollow, just Pydantic data holders.

#### `schemas.py`

```python
from .models import Order, OrderItem

class OrderItemSchema(OrderItem):
    pass

class OrderCreateSchema(Order):
    class Config:
        exclude = {"id", "total"}

class OrderResponseSchema(Order):
    pass
```

👉 Schemas inherit from entities to avoid repeating the **models** module, but still no business rules.

#### `services.py`

```python
import uuid
from .models import Order, OrderItem
from .schemas import OrderCreateSchema, OrderResponseSchema

def calculate_total(items: list[OrderItem]) -> float:
    prices = {"p1": 10.0, "p2": 20.0}
    return sum(prices[item.product_id] * item.quantity for item in items)

def create_order(order_data: OrderCreateSchema) -> OrderResponseSchema:
    items = [OrderItem(**item.dict()) for item in order_data.items]
    total = calculate_total(items)

    order = Order(
        id=str(uuid.uuid4()),
        customer_id=order_data.customer_id,
        items=items,
        total=total,
    )
    return OrderResponseSchema(**order.dict())
```

👉 All the business rules are here.

#### `controllers.py`

```python
from fastapi import APIRouter
from .schemas import OrderCreateSchema, OrderResponseSchema
from .services import create_order

router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("/", response_model=OrderResponseSchema)
def create_order_endpoint(order: OrderCreateSchema):
    return create_order(order)
```

👉 Controllers are slim, but services are bloated.

---

### ✅ Case Study: Avoiding the Sinkhole

#### `models.py`

```python
from pydantic import BaseModel, field_validator
from typing import List

class OrderItem(BaseModel):
    product_id: str
    quantity: int

    @field_validator("quantity")
    def validate_quantity(cls, v):
        if v <= 0:
            raise ValueError("Quantity must be positive")
        return v

class Order(BaseModel):
    id: str
    customer_id: str
    items: List[OrderItem]
    total: float = 0.0

    def calculate_total(self, price_lookup: dict[str, float]) -> None:
        self.total = sum(
            price_lookup[item.product_id] * item.quantity for item in self.items
        )

    def validate(self) -> None:
        if not self.items:
            raise ValueError("Order must contain at least one item")
        if self.total <= 0:
            raise ValueError("Order total must be positive")
```

👉 Entities are now smart: they enforce rules and update their own state.

#### `schemas.py`

```python
from .models import Order, OrderItem

class OrderItemSchema(OrderItem):
    pass

class OrderCreateSchema(Order):
    class Config:
        exclude = {"id", "total"}

class OrderResponseSchema(Order):
    pass
```

👉 Schemas still inherit from entities, keeping your DRY style.

#### `services.py`

```python
import uuid
from .models import Order, OrderItem
from .schemas import OrderCreateSchema, OrderResponseSchema

PRODUCT_PRICES = {"p1": 10.0, "p2": 20.0}

def create_order(order_data: OrderCreateSchema) -> OrderResponseSchema:
    items = [OrderItem(**item.dict()) for item in order_data.items]
    order = Order(
        id=str(uuid.uuid4()),
        customer_id=order_data.customer_id,
        items=items,
    )

    order.calculate_total(PRODUCT_PRICES)
    order.validate()

    return OrderResponseSchema(**order.dict())
```

👉 Services orchestrate without swallowing all the business rules.

#### `controllers.py`

```python
from fastapi import APIRouter
from .schemas import OrderCreateSchema, OrderResponseSchema
from .services import create_order

router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("/", response_model=OrderResponseSchema)
def create_order_endpoint(order: OrderCreateSchema):
    return create_order(order)
```

👉 Controllers remain slim.

---

## 🧠 Why This Works

✅ **Cohesive Entities**: Pydantic-based models enforce invariants and state transitions.
✅ **Schema Inheritance**: You avoid duplication by reusing entity definitions.
✅ **Slim Services**: Services orchestrate workflows without bloating.
✅ **Balanced Layers**: Logic is spread to the right places, preventing sinkholes.

---

## 🔚 Wrap-up

The sinkhole problem happens when **all the interesting code sinks into one layer**, leaving the rest hollow. By combining Pydantic-powered entities with schema inheritance and pushing business rules into the domain, you can keep FastAPI applications clean, balanced, and easy to evolve.