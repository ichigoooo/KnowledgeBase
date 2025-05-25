## 模块五：数据库集成 (FastAPI 与 PostgreSQL)

在前面的模块中，我们已经准备好了 FastAPI 应用的骨架和 PostgreSQL 数据库服务器。本模块的核心任务是将两者连接起来，让我们的应用能够持久化存储和检索数据。我们将重点学习如何使用 SQLAlchemy (一个强大的 Python SQL 工具包和对象关系映射器) 与 FastAPI 异步集成，以及如何通过 Alembic 管理数据库模式的演进。

### 1. 数据库交互方式选择

与数据库交互主要有两种方式：

*   **直接使用数据库驱动程序**：
    *   对于 PostgreSQL，Python 中最著名的同步驱动是 `psycopg2`，而异步驱动则是 `asyncpg`。
    *   在 FastAPI 这样的异步框架中，为了充分发挥其性能优势，强烈推荐使用异步驱动如 `asyncpg`。
    *   直接使用驱动程序可以让你完全控制 SQL 查询，但通常需要编写更多的模板代码。

*   **使用 ORM (对象关系映射器)**：
    *   ORM 允许你使用 Python 对象来操作数据库记录，而无需直接编写 SQL 语句（尽管也可以执行原始 SQL）。
    *   **优点**：
        *   简化数据库操作，代码更 Pythonic，更易于理解和维护。
        *   通常能处理不同数据库后端的差异。
        *   与 Pydantic 等数据类库结合良好。
    *   **缺点**：
        *   可能会隐藏 SQL 的复杂性，导致性能问题如果使用不当。
        *   对于非常复杂的查询，ORM 的抽象可能不够灵活。
    *   流行的 Python ORM 包括 SQLAlchemy 和 Tortoise ORM。本教程将重点介绍 SQLAlchemy，因为它功能全面且社区庞大。

我们将主要采用 SQLAlchemy ORM 结合 `asyncpg` 驱动的方式。

### 2. SQLAlchemy 与 FastAPI 集成 (异步)

SQLAlchemy 2.0+ 版本对异步操作提供了优秀的原生支持。

#### 安装必要的库

```bash
pip install sqlalchemy "asyncpg" pydantic psycopg2-binary # psycopg2-binary 某些情况下SQLAlchemy或Alembic可能需要
# 或者更精确地安装异步支持
pip install sqlalchemy[asyncio] asyncpg pydantic
```
*   `sqlalchemy`: 核心库。
*   `asyncpg`: 用于 PostgreSQL 的异步驱动程序。
*   `pydantic`: FastAPI 用来进行数据校验和序列化，我们也会用它来定义 API 层的数据模型。
*   `psycopg2-binary`: 同步驱动，Alembic 默认配置可能需要它，或者作为 SQLAlchemy 的备用。

#### 配置文件 (`config.py` - 推荐)

为了方便管理数据库连接信息和其他配置，通常会创建一个配置文件。

在你的项目根目录下 (例如 `~/my_fastapi_app`) 创建 `config.py`:

```python
# config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    DATABASE_URL: str = "postgresql+asyncpg://myuser:your_secure_password@localhost:5432/mydb"
    # 其他配置...
    # 例如 JWT 密钥等 (将在模块六使用)
    # SECRET_KEY: str = "your_secret_key"
    # ALGORITHM: str = "HS256"
    # ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    class Config:
        env_file = ".env" # 可以通过 .env 文件覆盖默认值

@lru_cache() # 确保settings只加载一次
def get_settings():
    return Settings()

# 你可能需要安装 pydantic-settings: pip install pydantic-settings
```
*   **`DATABASE_URL`**: 遵循 SQLAlchemy 的格式 `dialect+driver://username:password@host:port/database`。
    *   `postgresql+asyncpg` 指定使用 PostgreSQL 方言和 `asyncpg` 驱动。
    *   请将 `myuser`, `your_secure_password`, `localhost`, `5432`, `mydb` 替换为你在模块三中配置的 PostgreSQL 用户、密码、主机、端口和数据库名。
*   **`pydantic-settings`**: 用于从环境变量或 `.env` 文件加载配置，非常方便。
*   你可以在项目根目录创建一个 `.env` 文件来覆盖这些值，例如：
    ```env
    # .env
    DATABASE_URL="postgresql+asyncpg://prod_user:prod_pass@prod_host/prod_db"
    ```

#### 数据库初始化 (`database.py`)

创建一个 `database.py` 文件来管理数据库引擎和会话。

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base
from .config import get_settings

settings = get_settings()

DATABASE_URL = settings.DATABASE_URL

engine = create_async_engine(DATABASE_URL, echo=True) # echo=True 会打印SQL语句，生产环境可关闭

# 创建异步会话工厂
AsyncSessionLocal = sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False # FastAPI中推荐设置为False
)

Base = declarative_base() # 所有模型类将继承自这个 Base

# 依赖项：获取数据库会话
async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit() # 如果操作成功，则提交
        except Exception:
            await session.rollback() # 如果发生异常，则回滚
            raise
        finally:
            await session.close() # 确保会话关闭
```
*   `create_async_engine`: 创建 SQLAlchemy 的异步引擎。`echo=True` 在开发时很有用，它会打印所有执行的 SQL 语句。
*   `AsyncSessionLocal`: 一个异步会话工厂，用于创建数据库会话。
*   `Base`: Declarative Base 类，我们的数据库模型将继承它。
*   `get_db`: 这是一个 FastAPI 依赖项。它会在每个请求开始时创建一个新的 `AsyncSession`，在请求处理完成后（无论成功或失败）关闭它，并处理事务的提交或回滚。

#### 定义 SQLAlchemy 模型 (`models.py`)

创建 `models.py` 文件来定义数据库表结构对应的 Python 类。

```python
# models.py
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Boolean
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func # 用于默认时间戳
from .database import Base
import datetime

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True, nullable=False)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String, index=True, nullable=True)
    is_active = Column(Boolean, default=True)
    # created_at = Column(DateTime(timezone=True), server_default=func.now())
    # updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    items = relationship("Item", back_populates="owner") # 假设用户可以拥有物品

class Item(Base):
    __tablename__ = "items"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True, nullable=False)
    description = Column(String, index=True, nullable=True)
    price = Column(Float, nullable=False)
    owner_id = Column(Integer, ForeignKey("users.id"))

    owner = relationship("User", back_populates="items")
```
*   每个类代表一张数据库表，继承自 `Base`。
*   `__tablename__` 指定表名。
*   `Column` 定义表的列，包括数据类型 (如 `Integer`, `String`, `Boolean`, `Float`, `DateTime`)、约束 (如 `primary_key`, `unique`, `index`, `nullable`, `ForeignKey`)。
*   `relationship` 定义表之间的关系 (如一对多)。`back_populates` 用于双向关系。

#### 定义 Pydantic 模型 (Schema) (`schemas.py`)

创建 `schemas.py` 用于定义 API 请求和响应的数据结构。这有助于 FastAPI 进行数据校验、序列化和文档生成。

```python
# schemas.py
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List
import datetime

# Item Schemas
class ItemBase(BaseModel):
    title: str = Field(..., min_length=1, example="我的物品")
    description: Optional[str] = Field(None, example="这是一个很棒的物品描述")
    price: float = Field(..., gt=0, example=10.99)

class ItemCreate(ItemBase):
    pass

class ItemUpdate(BaseModel): # 允许部分更新
    title: Optional[str] = Field(None, min_length=1)
    description: Optional[str] = None
    price: Optional[float] = Field(None, gt=0)

class ItemInDBBase(ItemBase):
    id: int
    owner_id: int

    class Config:
        from_attributes = True # Pydantic V2 (以前是 orm_mode = True)
        # 这个配置告诉 Pydantic 模型从 ORM 对象的属性中读取数据

class Item(ItemInDBBase): # 用于读取/返回给客户端的 Item 模型
    pass

# User Schemas
class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=50, example="johndoe")
    email: EmailStr = Field(..., example="johndoe@example.com")
    full_name: Optional[str] = Field(None, example="John Doe")

class UserCreate(UserBase):
    password: str = Field(..., min_length=8, example="SecurePassword123")

class UserUpdate(BaseModel):
    email: Optional[EmailStr] = None
    full_name: Optional[str] = None
    is_active: Optional[bool] = None
    # 通常不直接通过此接口更新密码，会有专门的密码更新流程

class UserInDBBase(UserBase):
    id: int
    is_active: bool
    # hashed_password: str # 不应该返回给客户端

    class Config:
        from_attributes = True

class User(UserInDBBase): # 用于读取/返回给客户端的用户模型 (不含密码)
    items: List[Item] = [] # 返回用户时也返回其拥有的物品

class UserWithSensitiveInfo(UserInDBBase): # 用于内部或认证过程
    hashed_password: str
```
*   **`BaseModel`**：所有 Pydantic 模型继承自它。
*   **`ItemBase`, `UserBase`**：包含所有 Item/User 模型共享的字段。
*   **`ItemCreate`, `UserCreate`**：用于创建新记录时 API 接收的数据。通常包含所有必需字段，例如 `UserCreate` 包含 `password`。
*   **`ItemUpdate`, `UserUpdate`**：用于更新记录，字段通常是可选的。
*   **`Item`, `User`**：用于从 API 返回数据给客户端。通常不包含敏感信息 (如 `hashed_password`)。
*   **`Config.from_attributes = True`** (Pydantic V2) 或 `Config.orm_mode = True` (Pydantic V1)：允许 Pydantic 模型直接从 SQLAlchemy ORM 对象 (或其他具有属性的对象) 创建实例。这在从数据库读取数据并将其转换为 Pydantic 模型以供 API 响应时非常有用。
*   `Field` 用于添加额外的校验或元数据 (如 `example` 会显示在 API 文档中)。

#### CRUD 操作 (`crud.py`)

创建 `crud.py` 文件来封装所有数据库交互逻辑。这有助于保持 API 路径操作函数的简洁。

```python
# crud.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.future import select # select from sqlalchemy.sql for SQLAlchemy < 2.0
from sqlalchemy import update, delete # For SQLAlchemy 2.0 style updates/deletes
from . import models, schemas
from passlib.context import CryptContext # 用于密码哈希 (将在模块六中详细使用)

# 假设密码哈希上下文 (实际应在更全局的地方定义，如 auth.py 或 utils.py)
# pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# User CRUD
async def get_user(db: AsyncSession, user_id: int) -> Optional[models.User]:
    result = await db.execute(select(models.User).filter(models.User.id == user_id))
    return result.scalars().first()

async def get_user_by_email(db: AsyncSession, email: str) -> Optional[models.User]:
    result = await db.execute(select(models.User).filter(models.User.email == email))
    return result.scalars().first()

async def get_user_by_username(db: AsyncSession, username: str) -> Optional[models.User]:
    result = await db.execute(select(models.User).filter(models.User.username == username))
    return result.scalars().first()

async def get_users(db: AsyncSession, skip: int = 0, limit: int = 100) -> List[models.User]:
    result = await db.execute(select(models.User).offset(skip).limit(limit))
    return result.scalars().all()

async def create_user(db: AsyncSession, user: schemas.UserCreate, hashed_password: str) -> models.User:
    db_user = models.User(
        username=user.username,
        email=user.email,
        hashed_password=hashed_password, # 实际应传入已哈希的密码
        full_name=user.full_name
    )
    db.add(db_user)
    await db.flush() # flush 将更改发送到数据库，但不提交事务，以便获取ID
    await db.refresh(db_user) # refresh 从数据库重新加载对象属性 (如ID)
    return db_user

async def update_user(db: AsyncSession, user_id: int, user_update: schemas.UserUpdate) -> Optional[models.User]:
    db_user = await get_user(db, user_id)
    if not db_user:
        return None
    
    update_data = user_update.model_dump(exclude_unset=True) # Pydantic V2
    # update_data = user_update.dict(exclude_unset=True) # Pydantic V1
    
    if not update_data: # 如果没有要更新的字段
        return db_user

    # SQLAlchemy 2.0 style update (更推荐)
    stmt = (
        update(models.User)
        .where(models.User.id == user_id)
        .values(**update_data)
        .execution_options(synchronize_session="fetch") # or False
    )
    await db.execute(stmt)
    # await db.commit() # commit is handled by get_db dependency
    await db.refresh(db_user)
    return db_user

async def delete_user(db: AsyncSession, user_id: int) -> Optional[models.User]:
    db_user = await get_user(db, user_id)
    if not db_user:
        return None
    
    # SQLAlchemy 2.0 style delete
    stmt = delete(models.User).where(models.User.id == user_id)
    await db.execute(stmt)
    # await db.commit() # commit is handled by get_db dependency
    return db_user


# Item CRUD
async def get_items(db: AsyncSession, skip: int = 0, limit: int = 100) -> List[models.Item]:
    result = await db.execute(select(models.Item).offset(skip).limit(limit))
    return result.scalars().all()

async def get_items_by_owner(db: AsyncSession, owner_id: int, skip: int = 0, limit: int = 100) -> List[models.Item]:
    result = await db.execute(
        select(models.Item)
        .filter(models.Item.owner_id == owner_id)
        .offset(skip)
        .limit(limit)
    )
    return result.scalars().all()


async def create_user_item(db: AsyncSession, item: schemas.ItemCreate, user_id: int) -> models.Item:
    db_item = models.Item(**item.model_dump(), owner_id=user_id) # Pydantic V2
    # db_item = models.Item(**item.dict(), owner_id=user_id) # Pydantic V1
    db.add(db_item)
    await db.flush()
    await db.refresh(db_item)
    return db_item

async def get_item(db: AsyncSession, item_id: int) -> Optional[models.Item]:
    result = await db.execute(select(models.Item).filter(models.Item.id == item_id))
    return result.scalars().first()

# (可以添加 update_item, delete_item 等)
```
*   `select`, `update`, `delete`: SQLAlchemy 的查询构建函数。
*   `db.execute(...)`: 执行查询。
*   `result.scalars().first()`: 获取单个结果。
*   `result.scalars().all()`: 获取所有结果。
*   `db.add(object)`: 将新对象添加到会话中以备插入。
*   `db.flush()`: 将会话中的更改同步到数据库，但不提交事务。这对于在提交前获取自动生成的 ID 很有用。
*   `db.refresh(object)`: 用数据库中的最新数据更新对象。
*   **密码哈希**: `create_user` 函数中提到了 `hashed_password`。实际的密码哈希逻辑将在模块六中实现。

#### API 路径操作 (`main.py` 或 routers)

现在可以将这些 CRUD 函数集成到你的 FastAPI 路径操作中。通常我们会为不同的资源创建不同的路由文件 (例如 `routers/users.py`, `routers/items.py`)。

假设在 `main.py` 中或 `routers/users.py`:

```python
# main.py (或 routers/users.py)
from fastapi import FastAPI, Depends, HTTPException, APIRouter
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List, Optional

from . import crud, models, schemas # 假设这些文件在同一目录下或正确导入
from .database import get_db, engine, Base # engine 和 Base 用于初始化
# from .auth import get_password_hash # 假设这个函数在 auth.py (模块六)

# app = FastAPI() # 如果在 main.py
router = APIRouter() # 如果使用路由器

# --- 初始化数据库表 (仅用于开发/测试，生产环境使用Alembic) ---
# @app.on_event("startup") # 如果在 main.py
# async def startup_event():
#     async with engine.begin() as conn:
#         # await conn.run_sync(Base.metadata.drop_all) # 删除所有表 (小心!)
#         await conn.run_sync(Base.metadata.create_all) # 创建所有表
# --------------------------------------------------------------


# 假设这个函数在 auth.py (模块六)
def get_password_hash(password: str):
    # 这是一个占位符，实际哈希逻辑将在模块六实现
    return f"hashed_{password}"


@router.post("/users/", response_model=schemas.User, status_code=201, tags=["Users"])
async def create_new_user(user: schemas.UserCreate, db: AsyncSession = Depends(get_db)):
    db_user_by_email = await crud.get_user_by_email(db, email=user.email)
    if db_user_by_email:
        raise HTTPException(status_code=400, detail="Email already registered")
    db_user_by_username = await crud.get_user_by_username(db, username=user.username)
    if db_user_by_username:
        raise HTTPException(status_code=400, detail="Username already taken")
    
    hashed_password = get_password_hash(user.password) # 哈希密码
    return await crud.create_user(db=db, user=user, hashed_password=hashed_password)

@router.get("/users/", response_model=List[schemas.User], tags=["Users"])
async def read_all_users(skip: int = 0, limit: int = 100, db: AsyncSession = Depends(get_db)):
    users = await crud.get_users(db, skip=skip, limit=limit)
    return users

@router.get("/users/{user_id}", response_model=schemas.User, tags=["Users"])
async def read_single_user(user_id: int, db: AsyncSession = Depends(get_db)):
    db_user = await crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

@router.put("/users/{user_id}", response_model=schemas.User, tags=["Users"])
async def update_existing_user(user_id: int, user_update: schemas.UserUpdate, db: AsyncSession = Depends(get_db)):
    db_user = await crud.update_user(db, user_id=user_id, user_update=user_update)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found to update")
    return db_user

@router.delete("/users/{user_id}", response_model=schemas.User, tags=["Users"]) # 或者返回 204 No Content
async def delete_existing_user(user_id: int, db: AsyncSession = Depends(get_db)):
    db_user = await crud.delete_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found to delete")
    return db_user # 返回被删除的用户信息，或成功消息

# --- Item Endpoints ---
@router.post("/users/{user_id}/items/", response_model=schemas.Item, status_code=201, tags=["Items"])
async def create_item_for_user(
    user_id: int, item: schemas.ItemCreate, db: AsyncSession = Depends(get_db)
):
    db_user = await crud.get_user(db, user_id=user_id) # 确保用户存在
    if not db_user:
        raise HTTPException(status_code=404, detail="Owner user not found")
    return await crud.create_user_item(db=db, item=item, user_id=user_id)

@router.get("/items/", response_model=List[schemas.Item], tags=["Items"])
async def read_all_items(skip: int = 0, limit: int = 100, db: AsyncSession = Depends(get_db)):
    items = await crud.get_items(db, skip=skip, limit=limit)
    return items

@router.get("/items/{item_id}", response_model=schemas.Item, tags=["Items"])
async def read_single_item(item_id: int, db: AsyncSession = Depends(get_db)):
    db_item = await crud.get_item(db, item_id=item_id)
    if db_item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return db_item

# 如果在 main.py 且使用了 router
# app.include_router(router, prefix="/api/v1") # 举例，添加统一前缀
```
*   **数据库表创建**: `Base.metadata.create_all(bind=engine)` 可以用来创建在模型中定义的表。**注意**：这在开发初期很方便，但在生产环境中，强烈建议使用数据库迁移工具 (如 Alembic) 来管理数据库模式的变更。`drop_all`会删除所有表，使用时要格外小心。
*   `Depends(get_db)`: 将数据库会话注入到路径操作函数中。
*   `response_model`: 指定响应应该遵循的 Pydantic 模型，FastAPI 会自动进行数据过滤和序列化。
*   错误处理：如果资源未找到，返回 `HTTP 404 Not Found`；如果请求无效（如邮箱已注册），返回 `HTTP 400 Bad Request`。

### 3. 数据库迁移 (Alembic)

当你的数据模型发生变化时 (例如添加新表、新列、修改列类型)，你需要一种方法来更新数据库模式以匹配你的代码，同时保留现有数据。Alembic 是 SQLAlchemy 的官方迁移工具。

#### 安装 Alembic

```bash
pip install alembic
```

#### 初始化 Alembic 环境

在你的项目根目录 (例如 `~/my_fastapi_app`) 运行：
```bash
alembic init alembic
```
这会创建一个 `alembic` 目录和一个 `alembic.ini` 配置文件。

#### 配置 Alembic

1.  **`alembic.ini`**:
    *   找到 `sqlalchemy.url` 行，将其设置为你的数据库连接 URL (与 `config.py` 中的 `DATABASE_URL` 一致，但这里用的是同步驱动格式，因为 Alembic 的某些部分可能默认使用同步操作，或者你需要确保其可以连接)。
        ```ini
        sqlalchemy.url = postgresql://myuser:your_secure_password@localhost:5432/mydb
        ```
        **注意**：尽管我们的应用使用 `asyncpg`，Alembic 的某些核心操作（如生成脚本）可能仍会尝试使用同步连接，或者其默认配置可能基于同步。如果遇到问题，确保 `psycopg2-binary` 已安装。Alembic 也支持配置为异步，但这会更复杂一些。对于基本的迁移生成和应用，同步URL通常是可行的。

2.  **`alembic/env.py`**:
    *   **指向你的模型**: Alembic 需要知道你的 SQLAlchemy 模型定义在哪里，以便自动生成迁移。
        找到 `target_metadata = None` 这一行。你需要将其修改为指向你的 `Base.metadata`。
        在 `env.py` 的顶部添加导入：
        ```python
        from my_fastapi_app.models import Base # 假设你的应用叫 my_fastapi_app
        # 或者根据你的项目结构调整导入路径，例如：
        # import sys
        # from os.path import abspath, dirname
        # sys.path.insert(0, dirname(dirname(abspath(__file__)))) # 将项目根目录添加到sys.path
        # from models import Base # 如果 models.py 在根目录
        ```
        然后设置 `target_metadata`:
        ```python
        # target_metadata = None
        target_metadata = Base.metadata
        ```
    *   **(可选) 配置 Alembic 使用异步引擎运行迁移 (更高级)**：
        默认情况下，Alembic 的 `env.py` 中的 `run_migrations_online` 函数使用同步引擎。如果你希望迁移本身也通过异步引擎执行（例如，某些数据库特性可能需要），你需要修改 `env.py`。这通常涉及到修改 `run_migrations_online` 函数，使其能够处理异步连接。
        一个简化的异步配置思路是在 `run_migrations_online` 中：
        ```python
        # ...
        # from my_fastapi_app.database import engine as async_engine # 导入你的异步引擎

        # def do_run_migrations(connection):
        #     context.configure(connection=connection, target_metadata=target_metadata)
        #     with context.begin_transaction():
        #         context.run_migrations()

        # async def run_migrations_online():
        #     """Run migrations in 'online' mode.
        #     In this scenario we need to create an Engine
        #     and associate a connection with the context.
        #     """
        #     connectable = async_engine # 使用你的异步引擎
        #     async with connectable.connect() as connection:
        #         await connection.run_sync(do_run_migrations)
        #     await connectable.dispose()
        # ...
        # if context.is_offline_mode():
        #    run_migrations_offline()
        # else:
        #    import asyncio
        #    asyncio.run(run_migrations_online()) # 运行异步函数
        ```
        **注意**：Alembic 的异步支持仍在发展，上述是一个简化示例。确保查阅最新的 Alembic 和 SQLAlchemy 文档。对于多数标准迁移，默认的同步方式通常足够。

#### 生成迁移脚本

当你修改了 `models.py` 中的 SQLAlchemy 模型后 (例如添加了一个新列)，你可以让 Alembic 自动生成一个迁移脚本：
```bash
alembic revision -m "add_new_column_to_users_table" # 描述你的更改
```
这会在 `alembic/versions/` 目录下创建一个新的 Python 文件，例如 `xxxx_add_new_column_to_users_table.py`。

#### 编辑迁移脚本

打开新生成的迁移文件。它会包含 `upgrade()` 和 `downgrade()` 函数。
*   `upgrade()`: 定义了如何应用这个版本的更改 (例如 `op.add_column(...)`)。
*   `downgrade()`: 定义了如何撤销这个版本的更改 (例如 `op.drop_column(...)`)。

Alembic 会尝试根据你的模型和当前数据库状态的差异来填充这些函数，但有时你需要手动调整它们，特别是对于复杂的操作。Alembic 提供了丰富的 `op` 对象来进行各种 DDL (数据定义语言) 操作。

例如，如果我们在 `User` 模型中添加了 `last_login = Column(DateTime, nullable=True)`:
```python
# xxxx_add_last_login_to_users.py
"""add last_login to users

Revision ID: xxxx
Revises: yyyy # 上一个迁移的版本号
Create Date: ...

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = 'xxxx'
down_revision = 'yyyy'
branch_labels = None
depends_on = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('users', sa.Column('last_login', sa.DateTime(), nullable=True))
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_column('users', 'last_login')
    # ### end Alembic commands ###
```

#### 应用迁移

要将所有未应用的迁移更新到数据库：
```bash
alembic upgrade head
```
`head` 指的是最新的迁移版本。你也可以指定一个特定的版本号来升级或降级。

第一次运行时，如果数据库是空的，它会创建所有表 (基于你的第一个迁移，该迁移通常是根据初始模型生成的)。

#### 其他常用 Alembic 命令

*   `alembic current`: 显示当前数据库的版本。
*   `alembic history`: 显示迁移历史。
*   `alembic downgrade -1`: 回滚到上一个版本。
*   `alembic stamp head`: 将数据库标记为最新版本，而不实际运行迁移 (当你手动同步了数据库时有用)。

### 4. 连接池

数据库连接的建立和销毁是相对耗时的操作。连接池通过维护一组预先建立的数据库连接来重用它们，从而提高性能并减少延迟。

SQLAlchemy 的引擎 (`create_engine` 或 `create_async_engine`) 默认会管理一个连接池。你可以通过引擎的参数来配置连接池的行为 (如池大小、超时等)，但对于大多数 FastAPI 应用，默认配置通常已经足够好。

### 小结

在本模块中，我们学习了如何将 FastAPI 应用与 PostgreSQL 数据库通过 SQLAlchemy ORM 进行强大的异步集成。我们涵盖了：
*   安装和配置 SQLAlchemy 与 `asyncpg`。
*   定义 SQLAlchemy 模型 (`models.py`) 和 Pydantic schema (`schemas.py`)。
*   创建数据库会话管理依赖项 (`database.py`)。
*   实现 CRUD (创建、读取、更新、删除) 操作 (`crud.py`)。
*   在 FastAPI 路径操作函数中使用这些 CRUD 操作。
*   使用 Alembic 进行数据库迁移，以管理数据库模式的演变。

通过这些知识，你现在可以为你的 FastAPI 应用构建功能完善的数据持久层了。在下一个模块中，我们将关注 API 的安全性，实现用户认证和授权。

--- 