# Part 103: Python FastAPI กับ Database Cluster

## บทนำ

FastAPI เป็น modern Python web framework ที่ออกแบบมาสำหรับ high-performance APIs โดยใช้ Python 3.7+ type hints ในบทนี้เราจะเรียนรู้การใช้ FastAPI ร่วมกับ PostgreSQL (ผ่าน SQLAlchemy 2.0 async), Redis และ S3/MinIO เพื่อสร้าง production-ready API

---

## สารบัญ

1. Setup และ Project Structure
2. Async PostgreSQL ด้วย SQLAlchemy 2.0
3. Database Models
4. Alembic Migrations
5. Redis Async Client
6. S3/MinIO Integration
7. FastAPI Application
8. Authentication ด้วย JWT
9. Complete CRUD API Example
10. Testing

---

## 1. Setup และ Project Structure

### 1.1 Installation

```bash
# สร้าง virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# หรือ venv\Scripts\activate  # Windows

# ติดตั้ง dependencies
pip install fastapi
pip install "uvicorn[standard]"
pip install sqlalchemy[asyncio]
pip install asyncpg
pip install redis
pip install aioredis
pip install python-jose[cryptography]
pip install passlib[bcrypt]
pip install python-multipart
pip install aiofiles
pip install boto3
pip install aioboto3
pip install alembic
pip install pydantic-settings
pip install pytest
pip install httpx
pip install anyio[trio]
pip install pytest-asyncio
pip install python-dotenv
```

### 1.2 pyproject.toml

```toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "myapp-api"
version = "1.0.0"
description = "MyApp REST API with Database Cluster"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.104.1",
    "uvicorn[standard]>=0.24.0",
    "sqlalchemy[asyncio]>=2.0.23",
    "asyncpg>=0.29.0",
    "redis>=5.0.1",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "python-multipart>=0.0.6",
    "aiofiles>=23.2.1",
    "boto3>=1.34.0",
    "aioboto3>=12.0.0",
    "alembic>=1.12.1",
    "pydantic-settings>=2.1.0",
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.3",
    "httpx>=0.25.2",
    "anyio[trio]>=4.1.0",
    "pytest-asyncio>=0.21.1",
    "factory-boy>=3.3.0",
    "faker>=20.1.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
select = ["E", "F", "I", "N", "W"]
```

### 1.3 Project Structure

```
myapp-api/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── main.py              # FastAPI app entry point
│       ├── config.py            # Configuration settings
│       ├── database.py          # Database engine setup
│       ├── dependencies.py      # Dependency injection
│       ├── models/
│       │   ├── __init__.py
│       │   ├── user.py          # User ORM model
│       │   ├── product.py       # Product ORM model
│       │   └── order.py         # Order ORM model
│       ├── schemas/
│       │   ├── __init__.py
│       │   ├── user.py          # User Pydantic schemas
│       │   ├── product.py       # Product Pydantic schemas
│       │   └── order.py         # Order Pydantic schemas
│       ├── routers/
│       │   ├── __init__.py
│       │   ├── auth.py          # Authentication endpoints
│       │   ├── users.py         # User CRUD endpoints
│       │   ├── products.py      # Product CRUD endpoints
│       │   └── orders.py        # Order CRUD endpoints
│       ├── services/
│       │   ├── __init__.py
│       │   ├── auth.py          # Auth business logic
│       │   ├── cache.py         # Redis cache service
│       │   ├── storage.py       # S3/MinIO service
│       │   └── email.py         # Email service
│       └── middleware/
│           ├── __init__.py
│           ├── logging.py       # Request logging
│           └── timing.py        # Request timing
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── tests/
│   ├── conftest.py
│   ├── test_users.py
│   └── test_products.py
├── .env
├── .env.example
├── alembic.ini
└── pyproject.toml
```

### 1.4 Configuration

```python
# src/myapp/config.py

from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache
from typing import Optional


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )
    
    # App
    app_name: str = "MyApp API"
    app_version: str = "1.0.0"
    debug: bool = False
    environment: str = "production"
    
    # Database
    database_url: str
    database_pool_size: int = 20
    database_max_overflow: int = 10
    database_pool_timeout: int = 30
    database_pool_recycle: int = 1800
    database_echo: bool = False
    
    # Read replicas (comma-separated)
    database_replica_urls: Optional[str] = None
    
    # Redis
    redis_url: str = "redis://localhost:6379/0"
    redis_password: Optional[str] = None
    redis_max_connections: int = 50
    redis_decode_responses: bool = True
    
    # JWT
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 30
    jwt_refresh_token_expire_days: int = 7
    
    # S3/MinIO
    s3_endpoint_url: Optional[str] = None  # None ใช้ AWS S3
    s3_access_key_id: str = ""
    s3_secret_access_key: str = ""
    s3_bucket_name: str = "myapp-assets"
    s3_region: str = "ap-southeast-1"
    s3_presigned_url_expiry: int = 3600  # 1 hour
    
    # CORS
    cors_origins: list[str] = ["http://localhost:3000"]
    
    @property
    def replica_urls(self) -> list[str]:
        if self.database_replica_urls:
            return [url.strip() for url in self.database_replica_urls.split(",")]
        return []
    
    @property
    def async_database_url(self) -> str:
        """Convert postgres:// to postgresql+asyncpg://"""
        url = self.database_url
        if url.startswith("postgres://"):
            url = url.replace("postgres://", "postgresql+asyncpg://", 1)
        elif url.startswith("postgresql://"):
            url = url.replace("postgresql://", "postgresql+asyncpg://", 1)
        return url
    
    @property
    def async_replica_urls(self) -> list[str]:
        urls = []
        for url in self.replica_urls:
            if url.startswith("postgres://"):
                url = url.replace("postgres://", "postgresql+asyncpg://", 1)
            elif url.startswith("postgresql://"):
                url = url.replace("postgresql://", "postgresql+asyncpg://", 1)
            urls.append(url)
        return urls


@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

---

## 2. Async PostgreSQL ด้วย SQLAlchemy 2.0

### 2.1 Database Engine Setup

```python
# src/myapp/database.py

from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    AsyncEngine,
    async_sessionmaker,
)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.pool import NullPool
import logging
from typing import AsyncGenerator

from myapp.config import get_settings

logger = logging.getLogger(__name__)
settings = get_settings()


class Base(DeclarativeBase):
    """Base class สำหรับ ORM models ทั้งหมด"""
    pass


# Primary database engine
engine: AsyncEngine = create_async_engine(
    settings.async_database_url,
    echo=settings.database_echo,
    pool_size=settings.database_pool_size,
    max_overflow=settings.database_max_overflow,
    pool_timeout=settings.database_pool_timeout,
    pool_recycle=settings.database_pool_recycle,
    pool_pre_ping=True,  # Test connections before use
    connect_args={
        "server_settings": {
            "application_name": "myapp-api",
            "jit": "off",  # Disable JIT for OLTP
        },
        "command_timeout": 60,
    },
)

# Read replica engines
replica_engines: list[AsyncEngine] = []
for replica_url in settings.async_replica_urls:
    replica_engine = create_async_engine(
        replica_url,
        echo=settings.database_echo,
        pool_size=10,
        max_overflow=5,
        pool_pre_ping=True,
        connect_args={
            "server_settings": {
                "application_name": "myapp-api-replica",
                "default_transaction_read_only": "on",
            }
        },
    )
    replica_engines.append(replica_engine)

# Session factories
AsyncSessionFactory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)

AsyncReplicaSessionFactory = async_sessionmaker(
    replica_engines[0] if replica_engines else engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Dependency: Primary database session"""
    async with AsyncSessionFactory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()


async def get_read_db() -> AsyncGenerator[AsyncSession, None]:
    """Dependency: Read replica database session"""
    async with AsyncReplicaSessionFactory() as session:
        try:
            yield session
        except Exception:
            raise
        finally:
            await session.close()


async def init_db() -> None:
    """Initialize database - สร้าง tables"""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    logger.info("Database initialized")


async def close_db() -> None:
    """Close database connections"""
    await engine.dispose()
    for replica in replica_engines:
        await replica.dispose()
    logger.info("Database connections closed")
```

---

## 3. Database Models

```python
# src/myapp/models/user.py

from sqlalchemy import (
    String, Boolean, DateTime, Text, Integer,
    ForeignKey, Index, func, UniqueConstraint
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB
from datetime import datetime
from typing import Optional
import uuid

from myapp.database import Base


class User(Base):
    __tablename__ = "users"
    __table_args__ = (
        Index("ix_users_email", "email"),
        Index("ix_users_created_at", "created_at"),
        {"schema": "public"},
    )
    
    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
    )
    
    email: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        nullable=False,
        index=True,
    )
    
    username: Mapped[str] = mapped_column(
        String(100),
        unique=True,
        nullable=False,
    )
    
    hashed_password: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
    )
    
    full_name: Mapped[Optional[str]] = mapped_column(
        String(255),
        nullable=True,
    )
    
    avatar_url: Mapped[Optional[str]] = mapped_column(
        String(500),
        nullable=True,
    )
    
    is_active: Mapped[bool] = mapped_column(
        Boolean,
        default=True,
        nullable=False,
    )
    
    is_verified: Mapped[bool] = mapped_column(
        Boolean,
        default=False,
        nullable=False,
    )
    
    is_superuser: Mapped[bool] = mapped_column(
        Boolean,
        default=False,
        nullable=False,
    )
    
    metadata: Mapped[Optional[dict]] = mapped_column(
        JSONB,
        nullable=True,
        default=dict,
    )
    
    last_login_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True),
        nullable=True,
    )
    
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
        nullable=False,
    )
    
    # Relationships
    orders: Mapped[list["Order"]] = relationship(
        "Order",
        back_populates="user",
        lazy="noload",
    )
    
    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email}>"


# src/myapp/models/product.py

from sqlalchemy import (
    String, Text, Numeric, Integer, Boolean,
    DateTime, ForeignKey, Index, func, CheckConstraint
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB, ARRAY
from decimal import Decimal
from datetime import datetime
from typing import Optional
import uuid

from myapp.database import Base


class Category(Base):
    __tablename__ = "categories"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    slug: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    parent_id: Mapped[Optional[int]] = mapped_column(ForeignKey("categories.id"), nullable=True)
    
    parent: Mapped[Optional["Category"]] = relationship("Category", remote_side=[id])
    products: Mapped[list["Product"]] = relationship("Product", back_populates="category")


class Product(Base):
    __tablename__ = "products"
    __table_args__ = (
        CheckConstraint("price >= 0", name="ck_products_price_positive"),
        CheckConstraint("stock_quantity >= 0", name="ck_products_stock_non_negative"),
        Index("ix_products_sku", "sku"),
        Index("ix_products_category_id", "category_id"),
        Index("ix_products_created_at", "created_at"),
    )
    
    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
    )
    
    sku: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)
    compare_at_price: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 2), nullable=True)
    
    stock_quantity: Mapped[int] = mapped_column(Integer, default=0, nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    is_digital: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    
    images: Mapped[Optional[list[str]]] = mapped_column(ARRAY(String), nullable=True)
    tags: Mapped[Optional[list[str]]] = mapped_column(ARRAY(String), nullable=True)
    attributes: Mapped[Optional[dict]] = mapped_column(JSONB, nullable=True)
    
    category_id: Mapped[Optional[int]] = mapped_column(ForeignKey("categories.id"), nullable=True)
    
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
    
    category: Mapped[Optional[Category]] = relationship("Category", back_populates="products")
    order_items: Mapped[list["OrderItem"]] = relationship("OrderItem", back_populates="product")
```

---

## 4. Alembic Migrations

### 4.1 alembic.ini

```ini
[alembic]
script_location = alembic
file_template = %%(year)d%%(month).2d%%(day).2d_%%(hour).2d%%(minute).2d_%%(rev)s_%%(slug)s
prepend_sys_path = .
version_path_separator = os
sqlalchemy.url = driver://user:pass@localhost/dbname

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

### 4.2 alembic/env.py

```python
# alembic/env.py

import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context
import os
import sys

# Add project root to path
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from myapp.database import Base
from myapp.config import get_settings

# Import all models to register them
from myapp.models import user, product, order  # noqa

config = context.config
settings = get_settings()

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Use async database URL
config.set_main_option("sqlalchemy.url", settings.async_database_url)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
        compare_server_default=True,
    )
    
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection: Connection) -> None:
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_type=True,
        compare_server_default=True,
    )
    
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

```bash
# สร้าง migration
alembic revision --autogenerate -m "create_initial_tables"

# Upgrade to latest
alembic upgrade head

# Downgrade one step
alembic downgrade -1

# Show current revision
alembic current

# Show migration history
alembic history --verbose
```

---

## 5. Schemas (Pydantic)

```python
# src/myapp/schemas/user.py

from pydantic import BaseModel, EmailStr, Field, field_validator
from datetime import datetime
from typing import Optional
import uuid
import re


class UserBase(BaseModel):
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_-]+$")
    full_name: Optional[str] = Field(None, max_length=255)


class UserCreate(UserBase):
    password: str = Field(..., min_length=8, max_length=100)
    
    @field_validator("password")
    @classmethod
    def validate_password(cls, v: str) -> str:
        if not re.search(r"[A-Z]", v):
            raise ValueError("Password must contain at least one uppercase letter")
        if not re.search(r"[a-z]", v):
            raise ValueError("Password must contain at least one lowercase letter")
        if not re.search(r"\d", v):
            raise ValueError("Password must contain at least one digit")
        return v


class UserUpdate(BaseModel):
    full_name: Optional[str] = Field(None, max_length=255)
    avatar_url: Optional[str] = Field(None, max_length=500)
    metadata: Optional[dict] = None


class UserResponse(UserBase):
    id: uuid.UUID
    avatar_url: Optional[str] = None
    is_active: bool
    is_verified: bool
    created_at: datetime
    updated_at: datetime
    
    model_config = {"from_attributes": True}


class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    per_page: int
    pages: int


# src/myapp/schemas/product.py

from pydantic import BaseModel, Field, field_validator
from decimal import Decimal
from datetime import datetime
from typing import Optional
import uuid


class ProductBase(BaseModel):
    sku: str = Field(..., min_length=1, max_length=100)
    name: str = Field(..., min_length=1, max_length=255)
    description: Optional[str] = None
    price: Decimal = Field(..., ge=0, decimal_places=2)
    compare_at_price: Optional[Decimal] = Field(None, ge=0, decimal_places=2)
    stock_quantity: int = Field(0, ge=0)
    is_active: bool = True
    is_digital: bool = False
    tags: Optional[list[str]] = None
    attributes: Optional[dict] = None
    category_id: Optional[int] = None


class ProductCreate(ProductBase):
    pass


class ProductUpdate(BaseModel):
    name: Optional[str] = Field(None, min_length=1, max_length=255)
    description: Optional[str] = None
    price: Optional[Decimal] = Field(None, ge=0, decimal_places=2)
    stock_quantity: Optional[int] = Field(None, ge=0)
    is_active: Optional[bool] = None
    tags: Optional[list[str]] = None


class ProductResponse(ProductBase):
    id: uuid.UUID
    images: Optional[list[str]] = None
    created_at: datetime
    updated_at: datetime
    
    model_config = {"from_attributes": True}
```

---

## 6. Redis Async Service

```python
# src/myapp/services/cache.py

import json
import logging
from typing import Any, Optional, TypeVar, Callable
from functools import wraps
import hashlib
import redis.asyncio as redis

from myapp.config import get_settings

logger = logging.getLogger(__name__)
settings = get_settings()

T = TypeVar("T")


class CacheService:
    _client: Optional[redis.Redis] = None
    
    @classmethod
    def get_client(cls) -> redis.Redis:
        if cls._client is None:
            cls._client = redis.from_url(
                settings.redis_url,
                password=settings.redis_password,
                max_connections=settings.redis_max_connections,
                decode_responses=True,
                socket_connect_timeout=5,
                socket_timeout=5,
                retry_on_timeout=True,
                health_check_interval=30,
            )
        return cls._client
    
    @classmethod
    async def close(cls) -> None:
        if cls._client:
            await cls._client.aclose()
            cls._client = None
    
    @classmethod
    async def get(cls, key: str) -> Optional[Any]:
        try:
            client = cls.get_client()
            value = await client.get(key)
            if value is None:
                return None
            return json.loads(value)
        except Exception as e:
            logger.warning(f"Cache GET failed for key {key}: {e}")
            return None
    
    @classmethod
    async def set(
        cls,
        key: str,
        value: Any,
        ttl: int = 3600,
    ) -> bool:
        try:
            client = cls.get_client()
            serialized = json.dumps(value, default=str)
            await client.setex(key, ttl, serialized)
            return True
        except Exception as e:
            logger.warning(f"Cache SET failed for key {key}: {e}")
            return False
    
    @classmethod
    async def delete(cls, key: str) -> bool:
        try:
            client = cls.get_client()
            await client.delete(key)
            return True
        except Exception as e:
            logger.warning(f"Cache DELETE failed for key {key}: {e}")
            return False
    
    @classmethod
    async def delete_pattern(cls, pattern: str) -> int:
        """Delete all keys matching pattern"""
        try:
            client = cls.get_client()
            keys = await client.keys(pattern)
            if keys:
                return await client.delete(*keys)
            return 0
        except Exception as e:
            logger.warning(f"Cache DELETE PATTERN failed for {pattern}: {e}")
            return 0
    
    @classmethod
    async def exists(cls, key: str) -> bool:
        try:
            client = cls.get_client()
            return bool(await client.exists(key))
        except Exception as e:
            logger.warning(f"Cache EXISTS failed for key {key}: {e}")
            return False
    
    @classmethod
    async def incr(cls, key: str, ttl: Optional[int] = None) -> int:
        try:
            client = cls.get_client()
            value = await client.incr(key)
            if ttl and value == 1:
                await client.expire(key, ttl)
            return value
        except Exception as e:
            logger.warning(f"Cache INCR failed for key {key}: {e}")
            return 0
    
    @classmethod
    async def hset(cls, name: str, mapping: dict) -> int:
        try:
            client = cls.get_client()
            return await client.hset(name, mapping=mapping)
        except Exception as e:
            logger.warning(f"Cache HSET failed: {e}")
            return 0
    
    @classmethod
    async def hget(cls, name: str, key: str) -> Optional[str]:
        try:
            client = cls.get_client()
            return await client.hget(name, key)
        except Exception as e:
            logger.warning(f"Cache HGET failed: {e}")
            return None
    
    @classmethod
    async def hgetall(cls, name: str) -> dict:
        try:
            client = cls.get_client()
            return await client.hgetall(name) or {}
        except Exception as e:
            logger.warning(f"Cache HGETALL failed: {e}")
            return {}


def cache(
    key_prefix: str,
    ttl: int = 3600,
    key_builder: Optional[Callable] = None,
):
    """Cache decorator สำหรับ async functions"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            # Build cache key
            if key_builder:
                cache_key = key_builder(*args, **kwargs)
            else:
                key_parts = [key_prefix]
                for arg in args[1:]:  # Skip 'self'
                    key_parts.append(str(arg))
                for k, v in sorted(kwargs.items()):
                    key_parts.append(f"{k}:{v}")
                cache_key = ":".join(key_parts)
            
            # Try to get from cache
            cached = await CacheService.get(cache_key)
            if cached is not None:
                logger.debug(f"Cache HIT: {cache_key}")
                return cached
            
            # Execute function
            result = await func(*args, **kwargs)
            
            # Store in cache
            if result is not None:
                await CacheService.set(cache_key, result, ttl=ttl)
                logger.debug(f"Cache MISS -> SET: {cache_key}")
            
            return result
        
        return wrapper
    return decorator


# Rate limiter ด้วย Redis
class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
    
    async def is_allowed(self, identifier: str) -> tuple[bool, int, int]:
        """
        Returns: (allowed, current_count, remaining)
        """
        key = f"ratelimit:{identifier}"
        client = CacheService.get_client()
        
        try:
            async with client.pipeline(transaction=True) as pipe:
                current_time = int(asyncio.get_event_loop().time())
                window_start = current_time - self.window_seconds
                
                await pipe.zremrangebyscore(key, 0, window_start)
                await pipe.zadd(key, {str(current_time): current_time})
                await pipe.zcard(key)
                await pipe.expire(key, self.window_seconds)
                
                _, _, count, _ = await pipe.execute()
            
            allowed = count <= self.max_requests
            remaining = max(0, self.max_requests - count)
            
            return allowed, count, remaining
        
        except Exception as e:
            logger.warning(f"Rate limiter error: {e}")
            return True, 0, self.max_requests
```

---

## 7. S3/MinIO Storage Service

```python
# src/myapp/services/storage.py

import aioboto3
import boto3
from botocore.exceptions import ClientError
import logging
import uuid
import os
from typing import Optional, BinaryIO
from datetime import datetime

from myapp.config import get_settings

logger = logging.getLogger(__name__)
settings = get_settings()


class StorageService:
    
    def _get_s3_client(self):
        """สร้าง boto3 S3 client"""
        kwargs = {
            "region_name": settings.s3_region,
            "aws_access_key_id": settings.s3_access_key_id,
            "aws_secret_access_key": settings.s3_secret_access_key,
        }
        if settings.s3_endpoint_url:
            kwargs["endpoint_url"] = settings.s3_endpoint_url
        
        return boto3.client("s3", **kwargs)
    
    def _get_async_session(self):
        """สร้าง aioboto3 session"""
        return aioboto3.Session(
            aws_access_key_id=settings.s3_access_key_id,
            aws_secret_access_key=settings.s3_secret_access_key,
            region_name=settings.s3_region,
        )
    
    def _generate_key(self, original_filename: str, prefix: str = "") -> str:
        """สร้าง S3 key ที่ unique"""
        ext = os.path.splitext(original_filename)[1].lower()
        unique_id = uuid.uuid4().hex
        date_prefix = datetime.utcnow().strftime("%Y/%m/%d")
        
        if prefix:
            return f"{prefix}/{date_prefix}/{unique_id}{ext}"
        return f"{date_prefix}/{unique_id}{ext}"
    
    async def upload_file(
        self,
        file: BinaryIO,
        original_filename: str,
        content_type: str,
        prefix: str = "uploads",
        metadata: Optional[dict] = None,
    ) -> dict:
        """Upload file to S3"""
        key = self._generate_key(original_filename, prefix)
        
        kwargs = {
            "endpoint_url": settings.s3_endpoint_url,
        } if settings.s3_endpoint_url else {}
        
        session = self._get_async_session()
        async with session.client("s3", **kwargs) as s3:
            extra_args = {
                "ContentType": content_type,
                "ServerSideEncryption": "AES256",
            }
            if metadata:
                extra_args["Metadata"] = {k: str(v) for k, v in metadata.items()}
            
            await s3.upload_fileobj(
                file,
                settings.s3_bucket_name,
                key,
                ExtraArgs=extra_args,
            )
        
        url = f"https://{settings.s3_bucket_name}.s3.{settings.s3_region}.amazonaws.com/{key}"
        
        logger.info(f"Uploaded file to S3: {key}")
        
        return {
            "key": key,
            "url": url,
            "bucket": settings.s3_bucket_name,
            "filename": original_filename,
            "content_type": content_type,
        }
    
    def generate_presigned_upload_url(
        self,
        filename: str,
        content_type: str,
        prefix: str = "uploads",
        max_size: int = 10 * 1024 * 1024,  # 10MB
    ) -> dict:
        """สร้าง presigned URL สำหรับ direct upload"""
        key = self._generate_key(filename, prefix)
        client = self._get_s3_client()
        
        presigned_post = client.generate_presigned_post(
            Bucket=settings.s3_bucket_name,
            Key=key,
            Fields={
                "Content-Type": content_type,
            },
            Conditions=[
                {"Content-Type": content_type},
                ["content-length-range", 1, max_size],
            ],
            ExpiresIn=settings.s3_presigned_url_expiry,
        )
        
        return {
            "url": presigned_post["url"],
            "fields": presigned_post["fields"],
            "key": key,
            "expires_in": settings.s3_presigned_url_expiry,
        }
    
    def generate_presigned_download_url(self, key: str) -> str:
        """สร้าง presigned URL สำหรับ download"""
        client = self._get_s3_client()
        
        url = client.generate_presigned_url(
            "get_object",
            Params={
                "Bucket": settings.s3_bucket_name,
                "Key": key,
            },
            ExpiresIn=settings.s3_presigned_url_expiry,
        )
        
        return url
    
    async def delete_file(self, key: str) -> bool:
        """Delete file from S3"""
        kwargs = {"endpoint_url": settings.s3_endpoint_url} if settings.s3_endpoint_url else {}
        
        session = self._get_async_session()
        async with session.client("s3", **kwargs) as s3:
            try:
                await s3.delete_object(
                    Bucket=settings.s3_bucket_name,
                    Key=key,
                )
                logger.info(f"Deleted file from S3: {key}")
                return True
            except ClientError as e:
                logger.error(f"Failed to delete S3 file {key}: {e}")
                return False
    
    async def file_exists(self, key: str) -> bool:
        """Check if file exists in S3"""
        kwargs = {"endpoint_url": settings.s3_endpoint_url} if settings.s3_endpoint_url else {}
        
        session = self._get_async_session()
        async with session.client("s3", **kwargs) as s3:
            try:
                await s3.head_object(
                    Bucket=settings.s3_bucket_name,
                    Key=key,
                )
                return True
            except ClientError:
                return False


storage_service = StorageService()
```

---

## 8. FastAPI Main Application

```python
# src/myapp/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import logging
import time
import uuid

from myapp.config import get_settings
from myapp.database import init_db, close_db
from myapp.services.cache import CacheService
from myapp.routers import auth, users, products, orders

settings = get_settings()
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup และ Shutdown events"""
    logger.info("Starting up...")
    
    # Initialize database
    await init_db()
    logger.info("Database initialized")
    
    # Test Redis connection
    try:
        client = CacheService.get_client()
        await client.ping()
        logger.info("Redis connected")
    except Exception as e:
        logger.warning(f"Redis not available: {e}")
    
    yield
    
    # Shutdown
    logger.info("Shutting down...")
    await close_db()
    await CacheService.close()
    logger.info("Shutdown complete")


app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    description="MyApp REST API with Database Cluster",
    docs_url="/api/docs" if settings.debug else None,
    redoc_url="/api/redoc" if settings.debug else None,
    lifespan=lifespan,
)

# ─────────────────────────────────────────
# Middleware
# ─────────────────────────────────────────

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# GZip
app.add_middleware(GZipMiddleware, minimum_size=1000)


@app.middleware("http")
async def request_timing_middleware(request: Request, call_next):
    """Track request timing"""
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id
    
    start_time = time.perf_counter()
    
    response = await call_next(request)
    
    process_time = (time.perf_counter() - start_time) * 1000
    
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{process_time:.2f}ms"
    
    logger.info(
        f"Request: {request.method} {request.url.path} "
        f"Status: {response.status_code} "
        f"Time: {process_time:.2f}ms "
        f"RequestID: {request_id}"
    )
    
    return response


# ─────────────────────────────────────────
# Exception Handlers
# ─────────────────────────────────────────

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "error": "validation_error",
            "message": "Request validation failed",
            "details": exc.errors(),
        },
    )


@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    logger.exception(f"Unhandled exception: {exc}")
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "error": "internal_server_error",
            "message": "An internal error occurred",
        },
    )


# ─────────────────────────────────────────
# Routes
# ─────────────────────────────────────────

app.include_router(auth.router, prefix="/api/v1/auth", tags=["authentication"])
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])
app.include_router(products.router, prefix="/api/v1/products", tags=["products"])
app.include_router(orders.router, prefix="/api/v1/orders", tags=["orders"])


@app.get("/health", tags=["health"])
async def health_check():
    return {"status": "ok", "version": settings.app_version}


@app.get("/metrics", tags=["metrics"])
async def metrics():
    """Prometheus metrics endpoint"""
    # ใช้ prometheus_client library
    from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
    from fastapi.responses import Response
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

---

## 9. Complete CRUD Router

```python
# src/myapp/routers/products.py

from fastapi import APIRouter, Depends, HTTPException, Query, BackgroundTasks, UploadFile, File
from fastapi import status
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, or_, and_
from sqlalchemy.orm import selectinload
from typing import Optional, Annotated
import uuid
import logging

from myapp.database import get_db, get_read_db
from myapp.models.product import Product, Category
from myapp.schemas.product import (
    ProductCreate, ProductUpdate, ProductResponse, ProductListResponse
)
from myapp.services.cache import CacheService, cache
from myapp.services.storage import storage_service
from myapp.dependencies import get_current_user, get_current_superuser
from myapp.models.user import User

router = APIRouter()
logger = logging.getLogger(__name__)

CACHE_TTL = 300  # 5 minutes


@router.get("", response_model=ProductListResponse)
async def list_products(
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    search: Optional[str] = Query(None, max_length=100),
    category_id: Optional[int] = Query(None),
    min_price: Optional[float] = Query(None, ge=0),
    max_price: Optional[float] = Query(None, ge=0),
    is_active: bool = Query(True),
    sort_by: str = Query("created_at", regex="^(created_at|name|price|stock_quantity)$"),
    sort_order: str = Query("desc", regex="^(asc|desc)$"),
    db: AsyncSession = Depends(get_read_db),
):
    """List products with filtering, pagination and search"""
    
    # Build cache key
    cache_key = (
        f"products:list:{page}:{per_page}:{search}:{category_id}:"
        f"{min_price}:{max_price}:{is_active}:{sort_by}:{sort_order}"
    )
    
    # Try cache
    cached = await CacheService.get(cache_key)
    if cached:
        return cached
    
    # Build query
    stmt = select(Product)
    count_stmt = select(func.count(Product.id))
    
    # Filters
    conditions = [Product.is_active == is_active]
    
    if search:
        conditions.append(
            or_(
                Product.name.ilike(f"%{search}%"),
                Product.sku.ilike(f"%{search}%"),
                Product.description.ilike(f"%{search}%"),
            )
        )
    
    if category_id:
        conditions.append(Product.category_id == category_id)
    
    if min_price is not None:
        conditions.append(Product.price >= min_price)
    
    if max_price is not None:
        conditions.append(Product.price <= max_price)
    
    if conditions:
        stmt = stmt.where(and_(*conditions))
        count_stmt = count_stmt.where(and_(*conditions))
    
    # Count total
    total_result = await db.execute(count_stmt)
    total = total_result.scalar_one()
    
    # Sort
    sort_column = getattr(Product, sort_by)
    if sort_order == "desc":
        stmt = stmt.order_by(sort_column.desc())
    else:
        stmt = stmt.order_by(sort_column.asc())
    
    # Paginate
    offset = (page - 1) * per_page
    stmt = stmt.offset(offset).limit(per_page)
    
    # Load with relationships
    stmt = stmt.options(selectinload(Product.category))
    
    result = await db.execute(stmt)
    products = result.scalars().all()
    
    pages = (total + per_page - 1) // per_page
    
    response = ProductListResponse(
        items=[ProductResponse.model_validate(p) for p in products],
        total=total,
        page=page,
        per_page=per_page,
        pages=pages,
    )
    
    # Cache result
    await CacheService.set(cache_key, response.model_dump(), ttl=CACHE_TTL)
    
    return response


@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(
    product_id: uuid.UUID,
    db: AsyncSession = Depends(get_read_db),
):
    """Get single product by ID"""
    cache_key = f"products:{product_id}"
    
    cached = await CacheService.get(cache_key)
    if cached:
        return cached
    
    stmt = (
        select(Product)
        .where(Product.id == product_id)
        .options(selectinload(Product.category))
    )
    result = await db.execute(stmt)
    product = result.scalar_one_or_none()
    
    if not product:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Product {product_id} not found",
        )
    
    response = ProductResponse.model_validate(product)
    await CacheService.set(cache_key, response.model_dump(), ttl=CACHE_TTL)
    
    return response


@router.post("", response_model=ProductResponse, status_code=status.HTTP_201_CREATED)
async def create_product(
    product_in: ProductCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_superuser),
):
    """Create new product"""
    # Check SKU uniqueness
    stmt = select(Product).where(Product.sku == product_in.sku)
    result = await db.execute(stmt)
    if result.scalar_one_or_none():
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=f"Product with SKU '{product_in.sku}' already exists",
        )
    
    product = Product(**product_in.model_dump())
    db.add(product)
    await db.flush()
    await db.refresh(product)
    
    # Invalidate cache in background
    background_tasks.add_task(CacheService.delete_pattern, "products:list:*")
    
    return ProductResponse.model_validate(product)


@router.patch("/{product_id}", response_model=ProductResponse)
async def update_product(
    product_id: uuid.UUID,
    product_in: ProductUpdate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_superuser),
):
    """Update product"""
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    product = result.scalar_one_or_none()
    
    if not product:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Product {product_id} not found",
        )
    
    update_data = product_in.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(product, field, value)
    
    await db.flush()
    await db.refresh(product)
    
    # Invalidate caches
    background_tasks.add_task(CacheService.delete, f"products:{product_id}")
    background_tasks.add_task(CacheService.delete_pattern, "products:list:*")
    
    return ProductResponse.model_validate(product)


@router.delete("/{product_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_product(
    product_id: uuid.UUID,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_superuser),
):
    """Soft delete product"""
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    product = result.scalar_one_or_none()
    
    if not product:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Product {product_id} not found",
        )
    
    product.is_active = False
    await db.flush()
    
    background_tasks.add_task(CacheService.delete, f"products:{product_id}")
    background_tasks.add_task(CacheService.delete_pattern, "products:list:*")


@router.post("/{product_id}/images")
async def upload_product_image(
    product_id: uuid.UUID,
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_superuser),
):
    """Upload product image to S3"""
    # Validate product exists
    stmt = select(Product).where(Product.id == product_id)
    result = await db.execute(stmt)
    product = result.scalar_one_or_none()
    
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    
    # Validate file type
    allowed_types = {"image/jpeg", "image/png", "image/webp", "image/gif"}
    if file.content_type not in allowed_types:
        raise HTTPException(
            status_code=400,
            detail=f"File type {file.content_type} not allowed. Allowed: {allowed_types}"
        )
    
    # Upload to S3
    upload_result = await storage_service.upload_file(
        file=file.file,
        original_filename=file.filename,
        content_type=file.content_type,
        prefix=f"products/{product_id}",
        metadata={"product_id": str(product_id)},
    )
    
    # Update product images
    current_images = product.images or []
    current_images.append(upload_result["url"])
    product.images = current_images
    
    await db.flush()
    
    return {
        "message": "Image uploaded successfully",
        "url": upload_result["url"],
        "key": upload_result["key"],
    }
```

---

## 10. Authentication

```python
# src/myapp/services/auth.py

from datetime import datetime, timedelta, timezone
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
import logging

from myapp.config import get_settings

settings = get_settings()
logger = logging.getLogger(__name__)

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(
    data: dict,
    expires_delta: Optional[timedelta] = None,
) -> str:
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(
            minutes=settings.jwt_access_token_expire_minutes
        )
    
    to_encode.update({"exp": expire, "type": "access"})
    
    return jwt.encode(
        to_encode,
        settings.jwt_secret_key,
        algorithm=settings.jwt_algorithm,
    )


def create_refresh_token(user_id: str) -> str:
    expire = datetime.now(timezone.utc) + timedelta(
        days=settings.jwt_refresh_token_expire_days
    )
    
    data = {
        "sub": user_id,
        "exp": expire,
        "type": "refresh",
    }
    
    return jwt.encode(
        data,
        settings.jwt_secret_key,
        algorithm=settings.jwt_algorithm,
    )


def decode_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            settings.jwt_secret_key,
            algorithms=[settings.jwt_algorithm],
        )
        return payload
    except JWTError as e:
        logger.warning(f"JWT decode failed: {e}")
        raise


# src/myapp/dependencies.py

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from myapp.database import get_db
from myapp.models.user import User
from myapp.services.auth import decode_token
from myapp.services.cache import CacheService

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = decode_token(credentials.credentials)
        user_id: str = payload.get("sub")
        token_type: str = payload.get("type")
        
        if user_id is None or token_type != "access":
            raise credentials_exception
    except Exception:
        raise credentials_exception
    
    # Check if token is blacklisted
    blacklisted = await CacheService.exists(f"token:blacklist:{credentials.credentials}")
    if blacklisted:
        raise credentials_exception
    
    # Get user
    stmt = select(User).where(User.id == user_id, User.is_active == True)
    result = await db.execute(stmt)
    user = result.scalar_one_or_none()
    
    if user is None:
        raise credentials_exception
    
    return user


async def get_current_superuser(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough privileges",
        )
    return current_user
```

---

## 11. Testing

```python
# tests/conftest.py

import asyncio
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.pool import NullPool

from myapp.main import app
from myapp.database import Base, get_db
from myapp.models.user import User
from myapp.services.auth import hash_password, create_access_token

TEST_DATABASE_URL = "postgresql+asyncpg://test:test@localhost:5432/test_myapp"


@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()


@pytest_asyncio.fixture(scope="session")
async def test_engine():
    engine = create_async_engine(TEST_DATABASE_URL, poolclass=NullPool)
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    yield engine
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    
    await engine.dispose()


@pytest_asyncio.fixture
async def db_session(test_engine):
    session_factory = async_sessionmaker(test_engine, expire_on_commit=False)
    
    async with session_factory() as session:
        yield session
        await session.rollback()


@pytest_asyncio.fixture
async def client(db_session):
    async def override_get_db():
        yield db_session
    
    app.dependency_overrides[get_db] = override_get_db
    
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
    
    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def test_user(db_session: AsyncSession) -> User:
    user = User(
        email="test@example.com",
        username="testuser",
        hashed_password=hash_password("TestPass123"),
        full_name="Test User",
        is_active=True,
        is_verified=True,
    )
    db_session.add(user)
    await db_session.commit()
    await db_session.refresh(user)
    return user


@pytest_asyncio.fixture
def auth_headers(test_user: User) -> dict:
    token = create_access_token({"sub": str(test_user.id)})
    return {"Authorization": f"Bearer {token}"}


# tests/test_products.py

import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

from myapp.models.product import Product
from myapp.models.user import User


@pytest.mark.asyncio
async def test_list_products(client: AsyncClient):
    response = await client.get("/api/v1/products")
    assert response.status_code == 200
    data = response.json()
    assert "items" in data
    assert "total" in data
    assert "page" in data


@pytest.mark.asyncio
async def test_create_product_unauthorized(client: AsyncClient):
    product_data = {
        "sku": "TEST-001",
        "name": "Test Product",
        "price": "99.99",
        "stock_quantity": 100,
    }
    response = await client.post("/api/v1/products", json=product_data)
    assert response.status_code == 401


@pytest.mark.asyncio
async def test_create_product_as_admin(
    client: AsyncClient,
    db_session: AsyncSession,
    test_user: User,
):
    # Make user superuser
    test_user.is_superuser = True
    await db_session.commit()
    
    from myapp.services.auth import create_access_token
    token = create_access_token({"sub": str(test_user.id)})
    headers = {"Authorization": f"Bearer {token}"}
    
    product_data = {
        "sku": "TEST-001",
        "name": "Test Product",
        "price": "99.99",
        "stock_quantity": 100,
        "is_active": True,
    }
    
    response = await client.post(
        "/api/v1/products",
        json=product_data,
        headers=headers,
    )
    
    assert response.status_code == 201
    data = response.json()
    assert data["sku"] == "TEST-001"
    assert data["name"] == "Test Product"
    assert float(data["price"]) == 99.99
```

---

## การ Run Application

```bash
# Development
uvicorn myapp.main:app --reload --host 0.0.0.0 --port 8000

# Production
uvicorn myapp.main:app \
  --host 0.0.0.0 \
  --port 8000 \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --access-log \
  --log-level info

# ด้วย Gunicorn
gunicorn myapp.main:app \
  -w 4 \
  -k uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --access-logfile - \
  --error-logfile -
```

---

## สรุป

FastAPI ร่วมกับ SQLAlchemy 2.0 async และ asyncpg ทำให้สร้าง high-performance Python API ที่ทำงานร่วมกับ PostgreSQL ได้อย่างมีประสิทธิภาพ โดยมีจุดเด่น:

1. **Async ตลอดทาง**: ไม่มี blocking I/O ตั้งแต่ HTTP layer ถึง database
2. **Type safety**: Pydantic + Python type hints ช่วยจับ errors ตั้งแต่ development time
3. **Connection pooling**: SQLAlchemy async engine จัดการ connection pool อย่างมีประสิทธิภาพ
4. **Caching**: Redis cache ลด database load
5. **Testing**: pytest + httpx ทดสอบได้ง่ายโดยไม่ต้องการ running server

---

*เนื้อหาส่วนนี้เป็นส่วนหนึ่งของ Database Cluster Course - Advanced Content*
