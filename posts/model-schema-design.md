# 📝 How Should the Relationship Between Models and Schemas Be Managed?

When building APIs, managing the relationship between models and schemas can get messy fast. This post breaks down the main approaches you can take. By the end, you'll know which pattern keeps things clean, consistent, and scalable.

## 📢 Attention! This post is co-authored by GPT-4o from OpenAI.

### 🔰 Definitions

* **Model**: Classes designed to interact directly with the database (e.g., ORM/ODM objects).
* **Schema**: Classes that define the structure of data for API communication (e.g., request/response bodies in FastAPI).

---

## ⚔️ The Dilemma

There’s a subtle but significant design question in API-based applications:

> **How should you relate your database models and your data schemas?**

There are three primary strategies to choose from:

---

### 👹 1. Defining Schemas and Models Independently

Some developers prefer to write models and schemas completely independently. While this provides maximum flexibility, it introduces **high maintenance costs**:

* Anytime a field is added, removed or changed in one, the other must be updated manually.
* It creates potential **unwanted inconsistencies** between internal representation (model) and external representation (schema).

---

### 🦕 2. Conventional Approach: Models Inherit → Schemas

In this approach, you consider the **model as the "source of truth"**, and build your schemas by inheriting from it (e.g., serializers depend on models in Django REST Framework).

**Pros:**

* Less duplication
* Schemas auto-inherit all model fields

**Cons:**

* You inherit **more than you want**:

  * Fields like `_id` (MongoDB/Beanie), `created_at`, `updated_at`, `password_hash`, etc., are included unless explicitly removed.
* Schemas may inherit **behavioral methods** or internal database config unrelated to API communication.
* You risk **exposing sensitive fields** unless you're meticulous with `exclude` and `include` flags.

**Examples of such fields**:

* `_id`: Beanie or MongoDB document identifier
* `created_at`, `updated_at`: Timestamps meant for internal use
* `user_id`, `password_hash`: Sensitive or relational internals

#### 🧪 Code Example

```python
# models.py
class Log(Document):
    uid: str = Field(default_factory=lambda: str(uuid4()))
    user: str
    tenant: str | None = None
    log: dict | str = Field(default_factory=dict)
    metadata: dict = Field(default_factory=dict)
    tag: str | None = None
    level: Level = Level.NOTSET
    created_at: datetime = Field(default_factory=datetime.now)

    class Settings:
        name = "logs"
```

```python
# schemas.py
class LogRetrieveSchema(Log):
    class Config:
        from_attributes = True
        fields = {
            "_id": {"exclude": True},
            "created_at": {"exclude": False},
        }
```

👎 **Issues**:

* You have to **exclude `_id` manually**
* Schema includes `insert`, `save`, and other `Document` methods — unnecessary for API payloads

---

### ⏪ 3. Reverse Approach: Schemas as the Base

This flips the direction: schemas are the "core" representations, and models inherit from them.

**Pros:**

* Schemas stay clean, lean, and API-focused
* Models extend schemas to add database logic

**Cons:**

1. Multiple schemas exist for different purposes (e.g., Create, Update, Retrieve) — **which one becomes the base?**
2. Models are tightly coupled to schema structure — **a schema change might force unexpected model changes**.
3. It conflicts with some ORM/ODM expectations, which want models to define structure top-down.

#### 🧪 Code Example

```python
# schemas.py
class BaseSchema(BaseModel):
    tenant: str | None = None
    log: dict | str = Field(default_factory=dict)
    metadata: dict = Field(default_factory=dict)
    tag: str | None = None
    level: Level = Level.NOTSET


class LogCreateSchema(BaseSchema):
    pass


class LogRetrieveSchema(BaseSchema):
    uid: str
    created_at: datetime
```

```python
# models.py
class Log(Document, LogCreateSchema):
    uid: str = Field(default_factory=lambda: str(uuid4()))
    user: str
    created_at: datetime = Field(default_factory=datetime.now)

    class Settings:
        name = "logs"
```

👎 **Problems**:

* `LogCreateSchema` was never meant to be reused like this
* Changes to schemas for API purposes may unintentionally affect your model’s DB logic

---

## 🧩 My Preferred Approach: Shared Abstract Base

### 🌟 Solution: Define a Shared Base Model

1. **Create an abstract `BaseLog`** with common fields (business logic).
2. Inherit from `BaseLog` in both your **Model** and **Schemas**.
3. Add model-specific fields (like `_id`, `created_at`) in your `Document` class.
4. Add schema-specific fields (e.g., computed fields, serialization tweaks) in schema subclasses.

#### 🧪 Code Example

```python
# models.py
from beanie import Document
from pydantic import BaseModel, Field
from enum import Enum
from datetime import datetime
from uuid import uuid4


class Level(Enum):
    INFO = "INFO"
    TRACE = "TRACE"
    DEBUG = "DEBUG"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"
    FATAL = "FATAL"
    NOTSET = "NOTSET"


class BaseLog(BaseModel):
    tenant: str | None = None
    log: dict | str = Field(default_factory=dict)
    metadata: dict = Field(default_factory=dict)
    tag: str | None = None
    level: Level = Level.NOTSET


class Log(Document, BaseLog):
    uid: str = Field(default_factory=lambda: str(uuid4()))
    user: str
    created_at: datetime = Field(default_factory=datetime.now)

    class Settings:
        name = "logs"
```

```python
# schemas.py
from datetime import datetime
from models import BaseLog


class LogCreateSchema(BaseLog):
    pass


class LogRetrieveSchema(BaseLog):
    uid: str
    created_at: datetime
```

✅ **Benefits**:

* Single source of shared field definitions
* No schema bloating from internal model fields
* Clear separation of concerns

---

## 🧠 Final Thoughts — **The Final Takeaway**

While each approach has pros and cons, **I strongly recommend the “Shared Abstract Base” approach as the cleanest and most maintainable solution**. This method balances:

* Reuse of shared field definitions without duplications
* Separation of concerns between internal database logic and external API schema
* Avoidance of unintended exposure of sensitive fields or unwanted methods

If you want a single, practical pattern that scales well in real projects — this is the one I use and advocate for.

> **In summary: define a common abstract base with shared fields and have both your models and schemas inherit from it, adding their own specifics separately.**