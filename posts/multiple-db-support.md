# 🧱 Single application with simulaneous multiple database support - Repository Design Pattern

If you’ve ever wondered how to develop a backend service that seamlessly supports **different databases** — say, MongoDB and PostgreSQL — this post is for you. In this tutorial, we use the [Repository Design Pattern](https://www.geeksforgeeks.org/repository-design-pattern/) to create a **simple yet flexible FastAPI application** that can switch between databases with minimal effort.

## 📢 Attention! This post is co-authored by GPT-4o from OpenAI.

---

## 📚 What is the Repository Pattern?

The **Repository Pattern** is a design principle that separates the business logic from the data access logic. It introduces an abstraction layer between the domain and data mapping layers using interfaces. This helps your application to:

- Support multiple databases.
- Stay testable and modular.
- Easily switch or mock data sources.

In short, it brings **decoupling** and **swappability** to your data layer.

---

## 📌 Case Study

To demonstrate the implementation, let’s consider a simple use case for managing a `User` entity with three basic operations:

- **Create a user**
- **Retrieve a user by ID**
- **List all users**

```bash
User:
    id
    name
    email
```

---

## 🏗️ Implementation Steps

### 0. Defining the Schemas

```python
# app/schemas.py

from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    name: str
    email: EmailStr

class UserRead(UserCreate):
    id: str
```

---

### 1. Defining the repositories

#### 1.1 Abstract repository

```python
# app/repositories/abstract.py

from abc import ABC, abstractmethod
from typing import List, Optional
from app.schemas import UserCreate, UserRead

class AbstractUserRepository(ABC):
    @abstractmethod
    async def create(self, user: UserCreate) -> UserRead: ...

    @abstractmethod
    async def list_all(self) -> List[UserRead]: ...

    @abstractmethod
    async def get_by_id(self, user_id: str) -> Optional[UserRead]: ...
```

---

#### 1.2 Mongo-based repository (Beanie)

```python
# app/repositories/mongo.py

from app.repositories.abstract import AbstractUserRepository
from app.models.mongo_models import UserDocument
from app.schemas import UserCreate, UserRead
from typing import List, Optional

class MongoUserRepository(AbstractUserRepository):
    async def create(self, user: UserCreate) -> UserRead:
        doc = UserDocument(**user.dict())
        await doc.insert()
        return UserRead(id=str(doc.id), **user.dict())

    async def list_all(self) -> List[UserRead]:
        users = await UserDocument.find_all().to_list()
        return [UserRead(id=str(u.id), name=u.name, email=u.email) for u in users]

    async def get_by_id(self, user_id: str) -> Optional[UserRead]:
        user = await UserDocument.get(user_id)
        if user:
            return UserRead(id=str(user.id), name=user.name, email=user.email)
        return None
```

---

#### 1.3 PostgreSQL Repository (SQLAlchemy)

```python
# app/repositories/postgre.py

from app.repositories.abstract import AbstractUserRepository
from app.schemas import UserCreate, UserRead
from app.models.pg_models import UserSQL
from sqlalchemy.future import select

class PostgresUserRepository(AbstractUserRepository):
    def __init__(self, session_dependency):
        self.get_session = session_dependency

    async def create(self, user: UserCreate) -> UserRead:
        async for session in self.get_session():
            db_user = UserSQL(**user.model_dump())
            session.add(db_user)
            await session.commit()
            await session.refresh(db_user)
            return UserRead(**db_user.__dict__)

    async def list_all(self) -> list[UserRead]:
        async for session in self.get_session():
            result = await session.execute(select(UserSQL))
            return [UserRead(**user.__dict__) for user in result.scalars().all()]

    async def get_by_id(self, user_id: str) -> UserRead | None:
        async for session in self.get_session():
            return await session.get(UserSQL, user_id)
```

---

### 2. Defining the Models

#### 2.1 MongoDB Model (Beanie)

```python
# app/models/mongo_models.py

from beanie import Document

class UserDocument(Document):
    name: str
    email: str

    class Settings:
        name = "users"
```

#### 2.2 PostgreSQL Model (SQLAlchemy)

```python
# app/models/pg_models.py

from sqlalchemy import Column, String
from uuid import uuid4
from app.databases.pg_database import Base

class UserSQL(Base):
    __tablename__ = "users"

    id = Column(String, primary_key=True, index=True, default=lambda: str(uuid4()))
    name = Column(String, nullable=False)
    email = Column(String, nullable=False, unique=True)
```

---

### 3. Database Initialization

#### 3.1 MongoDB

```python
# app/databases/mongo_database.py

from motor.motor_asyncio import AsyncIOMotorClient
from beanie import init_beanie
import os
from app.models.mongo_models import UserDocument

async def init_mongo_db():
    client = AsyncIOMotorClient(os.getenv("MONGODB_URI", "mongodb://mongo:27017"))
    db = client.get_database(os.getenv("MONGODB_NAME", "user_db"))
    await init_beanie(database=db, document_models=[UserDocument])
```

#### 3.2 PostgreSQL

```python
# app/databases/pg_database.py

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    create_async_engine,
    async_sessionmaker,
)
from sqlalchemy.orm import declarative_base
import os

Base = declarative_base()
engine = None
AsyncSessionLocal = None

async def init_pg_db():
    global engine, AsyncSessionLocal
    engine = create_async_engine(
        os.getenv("POSTGRES_URI", "postgresql+asyncpg://postgres:postgres@postgres:5432/user_db"),
        echo=True
    )
    AsyncSessionLocal = async_sessionmaker(bind=engine, class_=AsyncSession, expire_on_commit=False)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def get_async_session() -> AsyncSession:
    if AsyncSessionLocal is None:
        raise RuntimeError("Database not initialized. Call init_pg_db() first.")
    async with AsyncSessionLocal() as session:
        yield session
```

---

### 4. Settings and Repository Switch

```python
# app/settings.py

from enum import Enum
from pydantic_settings import BaseSettings
from app.repositories.abstract import AbstractUserRepository
from app.databases.mongo_database import init_mongo_db
from app.databases.pg_database import init_pg_db, get_async_session
from app.repositories.mongo import MongoUserRepository
from app.repositories.postgre import PostgresUserRepository

class DatabaseBackend(str, Enum):
    MONGO = "mongo"
    POSTGRES = "postgres"

class Settings(BaseSettings):
    db_backend: DatabaseBackend = DatabaseBackend.MONGO  # default

    class Config:
        env_file = ".env"

    async def get_user_repo(self) -> AbstractUserRepository:
        if self.db_backend == DatabaseBackend.MONGO:
            await init_mongo_db()
            return MongoUserRepository()
        elif self.db_backend == DatabaseBackend.POSTGRES:
            await init_pg_db()
            return PostgresUserRepository(get_async_session)
        raise ValueError(f"DB backend {self.db_backend} is not supported.")
```

---

### 5. API Endpoints

```python
# app/user_router.py

from fastapi import APIRouter, Depends, HTTPException
from app.schemas import UserCreate, UserRead
from app.repositories.abstract import AbstractUserRepository
from app.settings import Settings

settings = Settings()

async def get_user_repo() -> AbstractUserRepository:
    return await settings.get_user_repo()

router = APIRouter()

@router.post("/", response_model=UserRead)
async def create_user(user: UserCreate, repo: AbstractUserRepository = Depends(get_user_repo)):
    return await repo.create(user)

@router.get("/", response_model=list[UserRead])
async def list_users(repo: AbstractUserRepository = Depends(get_user_repo)):
    return await repo.list_all()

@router.get("/{user_id}/", response_model=UserRead | None)
async def get_user(user_id: str, repo: AbstractUserRepository = Depends(get_user_repo)):
    user = await repo.get_by_id(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

---

### 6. App Declaration

```python
# app/main.py

from fastapi import FastAPI
from app.user_router import router

app = FastAPI()
app.include_router(router, prefix="/users", tags=["users"])
```

---

## 🧪 Running the Setup

Once cloned, from the root directory, simply run:

```bash
docker compose up --build -d
```

This will launch the FastAPI app with **PostgreSQL** as the default backend.

### 📁 `docker-compose.yaml` (PostgreSQL)

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    depends_on:
      - postgres
    environment:
      - POSTGRES_URI=postgresql+asyncpg://postgres:postgres@postgres:5432/user_db

  postgres:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: user_db
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## 🔁 Switching to MongoDB

1. Change the `.env` to use:

```
DB_BACKEND=mongo
```

2. Update your `docker-compose.yaml`:

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    depends_on:
      - mongo
    environment:
      - MONGODB_URI=mongodb://mongo:27017

  mongo:
    image: mongo:latest
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

3. Rebuild and run:

```bash
docker compose up --build -d
```

---

## 🧠 Final Thoughts

This design pattern allows you to:

✅ Easily toggle between MongoDB and PostgreSQL
✅ Keep your codebase clean and extensible
✅ Write database-agnostic logic with a shared repository interface

> The Repository Pattern adds clarity and flexibility to your backend — especially when juggling multiple databases or planning for future migrations.