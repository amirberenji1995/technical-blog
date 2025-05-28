# 📝 How Should the Relationship Between Models and Schemas Be Managed?

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
* Schemas may inherit **behavioral methods** or internal database config that has nothing to do with external API communication.
* You risk **exposing sensitive fields** unless you're meticulous with `exclude` and `include` flags.

**Examples of such fields**:

* `_id`: Beanie or MongoDB document identifier
* `created_at`, `updated_at`: Timestamps meant for internal use
* `user_id`, `password_hash`: Sensitive or relational internals

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

---

## 🧩 My Preferred Approach: Shared Abstract Base

To strike a balance between reuse and separation of concerns:

### 🌟 Solution: Define a Shared Base Model

1. **Create an abstract `BaseLog`** with common fields (business logic).
2. Inherit from `BaseLog` in both your **Model** and **Schemas**.
3. Add model-specific fields (like `_id`, `created_at`) in your `Document` class.
4. Add schema-specific fields (e.g., computed fields, serialization tweaks) in schema subclasses.

---

## ✅ Code: Preferred Design (Shared Base Class)

### `models.py`

```python
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

### `schemas.py`

```python
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

## 🔄 Alternative #1: Models as Source of Truth

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

* You have to **exclude \_id manually**
* Schema includes `insert`, `save`, and other `Document` methods — unnecessary for API payloads

---

## 🔁 Alternative #2: Schema as Source of Truth

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
---

👎 **Problems**:

* `LogCreateSchema` was never meant to be reused like this
* Changes to schemas for API purposes may unintentionally affect your model’s DB logic

---

## 🧠 Final Thoughts

> Use inheritance **only when it provides clarity and eliminates duplication**. Otherwise, composition and explicit field definitions can be more maintainable.

By defining a **shared abstract base**, you get:

* **Explicit structure**
* **Reduced duplication**
* **Clear boundaries** between data layers

This hybrid pattern lets you scale both your **business logic** and your **data model** cleanly.
