# 🧰 Building Reusable Responses and Errors in FastAPI: A Scalable Pattern

In this post, we explore a powerful pattern for managing **responses** and **exceptions** in a FastAPI project. This approach offers:

1. ✅ Generic, documented response structures
2. 🚫 Reusable, domain-agnostic error classes
3. 🧬 Inheritance-based design for consistency and DRYness
4. 📄 Tight coupling of errors with example payloads for API docs

## 📢 Attention! This post is co-authored by GPT-4o from OpenAI.

---

## 🧱 The Problem

In most FastAPI applications, you’ll eventually encounter these challenges:

* Copy-pasting the same error-handling logic across services
* Returning vague or undocumented errors in Swagger/Redoc
* Repeating boilerplate response schemas
* Inconsistent formatting for errors and success responses

---

## 🎯 The Goal

We want to design a clean, consistent approach for both successful and error responses that is:

* ✨ Documented (via `responses={...}` and `response_model=...`)
* ♻️ Reusable (errors like "not found" can apply to many resources)
* 📦 Self-contained (errors include their doc examples)
* 🔧 Extensible (easy to add new response or error types)

---

## 🏗️ Implementation

Now, let’s walk through a **realistic case study**, define our base classes, and show exactly how and where to use them — from business logic to API controllers.

---

### 🧪 Case Study

We’ll simulate a **simple user management system**, where users can be retrieved by UID. Here’s the basic structure:

#### `models.py`

```python
from pydantic import BaseModel, Field
from beanie import Document
from datetime import datetime
from uuid6 import uuid7

class BaseUser(BaseModel):
    user_name: str
    phone: str
    email: str
    occupation: str

class User(Document, BaseUser):
    uid: str = Field(default_factory=lambda: str(uuid7()))
    created_at: datetime = Field(default_factory=datetime.now)

    class Settings:
        name = "users"
        indexes = ["uid", "user_name", [("created_at", -1)]]
```

#### `schemas.py`

```python
from pydantic import BaseModel, model_validator
from datetime import datetime
from typing import Optional

class UserCreateSchema(BaseUser):
    pass

class UserRetrieveSchema(BaseUser):
    uid: str
    created_at: datetime

class UserUpdateSchema(BaseModel):
    user_name: Optional[str]
    phone: Optional[str]
    email: Optional[str]
    occupation: Optional[str]

    @model_validator(mode="after")
    def check_fields(self):
        if not any(self.model_dump().values()):
            raise ValueError("At least one field must be provided for update.")
        return self
```

---

### 📦 Generic Responses

We want all success responses to follow the same structure, so we start with a generic base class:

#### `base_response.py`

```python
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel

T = TypeVar("T")

class BaseResponse(BaseModel, Generic[T]):
    message: str
    data: Optional[T]
```

💡 **Why use generics?**

* Let Swagger/OpenAPI auto-generate accurate response models
* Reuse the same format with any schema (e.g., user, product, etc.)
* Keep controllers clean and focused on logic, not formatting

---

Now define a **custom response** class per use case to keep your endpoints explicit.

#### `responses.py`

```python
from app.base_response import BaseResponse
from app.users.schemas import UserRetrieveSchema

class UserRetrieveResponse(BaseResponse[UserRetrieveSchema]):
    def __init__(self, data: UserRetrieveSchema, message: str = "User retrieved successfully."):
        super().__init__(message=message, data=data)
```

This makes our return statements as expressive and descriptive as possible.

---

### 🚫 Errors

To handle errors cleanly and consistently, we start with a **BaseError** that wraps `HTTPException`.

#### `base_error.py`

```python
from fastapi import HTTPException

class BaseError:
    error: HTTPException
    example: dict

    def __init__(self, status_code: int, detail: str):
        self.error = HTTPException(status_code=status_code, detail=detail)
        self.example = {"detail": detail}
```

Each error subclass will:

* Define its own status code and message
* Include an `example` for docs
* Expose `.error` to raise inside `services.py`

---

Now define reusable, descriptive errors like this:

#### `errors.py`

```python
from fastapi import status
from app.base_error import BaseError

class NotFoundError(BaseError):
    def __init__(self, resource: str):
        detail = f"{resource} not found"
        super().__init__(status_code=status.HTTP_404_NOT_FOUND, detail=detail)
        self.example = {"detail": detail}
```

We can now reuse this for users, articles, comments — anything!

---

### 🧠 How to Raise Errors

Your services (business logic layer) should raise the domain-specific errors like this:

#### `services.py`

```python
from app.users.models import User
from app.users.schemas import UserRetrieveSchema
from app.errors import NotFoundError

async def get_user_service(uid: str):
    user = await User.find_one(User.uid == uid)
    if not user:
        raise NotFoundError("User").error
    return UserRetrieveSchema(**user.model_dump())
```

📌 Using `.error` allows you to raise cleanly, and the class name keeps the context obvious.

---

### 📤 How to Return Responses

In your API layer (controllers), you can now declare:

* the custom response class for formatting
* the response model via `BaseResponse[Schema]`
* an `example` of the error in the `responses={...}` section

#### `controllers.py`

```python
from fastapi import APIRouter, status
from app.users.schemas import UserRetrieveSchema
from app.users.responses import UserRetrieveResponse
from app.users.services import get_user_service
from app.base_response import BaseResponse
from app.errors import NotFoundError

router = APIRouter(prefix="/my_app", tags=["my_app"])

@router.get(
    "/users/{uid}",
    response_model=BaseResponse[UserRetrieveSchema],
    status_code=status.HTTP_200_OK,
    responses={
        status.HTTP_404_NOT_FOUND: {
            "description": "User not found",
            "content": {"application/json": {"example": NotFoundError("User").example}},
        }
    },
)
async def get_user_by_uid(uid: str):
    user = await get_user_service(uid)
    return UserRetrieveResponse(user)
```

🔍 This results in:

* A **documented success response** via `BaseResponse[UserRetrieveSchema]`
* A **documented error** with proper HTTP code and payload
* Clean, readable logic with minimal formatting clutter

---

## 🧠 Why This Pattern Works

✅ **Clean Documentation**
FastAPI shows structured examples in Swagger thanks to `BaseResponse` and `.example` from errors.

🔁 **Reusable Errors**
Use the same `NotFoundError("User")` across any model: users, posts, products, etc.

🧱 **Inheritance & DRYness**
Avoid duplicating status codes, message formats, and error logic.

📦 **Coupled Examples**
Each error includes its own example payload — no need to manually define it in every route.

---

## 🔚 Wrap-up

This pattern makes your FastAPI app more:

* 🔍 Observable (clear docs for consumers)
* 📐 Consistent (every route responds in the same format)
* 🧹 Clean (no redundant schemas/errors)