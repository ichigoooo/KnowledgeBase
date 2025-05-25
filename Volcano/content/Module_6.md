## 模块六：API 安全性：认证与授权

API 的安全性是后端开发中至关重要的一环。一个不安全的 API 可能会导致数据泄露、未授权访问、服务滥用等严重问题。本模块将重点介绍如何在 FastAPI 应用中实现用户认证 (Authentication) 和授权 (Authorization)，以保护你的 API 端点和数据。

我们将主要使用 OAuth2 密码模式 (Password Flow) 结合 JSON Web Tokens (JWT) 来进行用户认证，并探讨基于角色的访问控制 (RBAC) 的基本思路。

### 1. 安全性基础概念

*   **认证 (Authentication)**：验证用户的身份，即确认用户是谁 (Is the user who they claim to be?)。通常通过用户名密码、Token、API 密钥等方式进行。
*   **授权 (Authorization)**：在用户身份被成功认证后，判断该用户是否有权限执行特定操作或访问特定资源 (What is this authenticated user allowed to do?)。
*   **HTTPS (HTTP Secure)**：所有 API 通信都应该通过 HTTPS 进行加密，以防止数据在传输过程中被窃听或篡改。在模块七中，我们将讨论如何在 Nginx 中配置 SSL/TLS 证书以启用 HTTPS。
*   **密码安全**：
    *   **绝不存储明文密码**：密码必须经过哈希处理后才能存储在数据库中。
    *   使用强大的哈希算法 (如 bcrypt, Argon2)。
    *   为每个密码使用唯一的盐 (salt)。

### 2. 认证机制：OAuth2 与 JWT

FastAPI 官方文档推荐使用 OAuth2 结合 JWT 来处理认证。

*   **OAuth2**：一个开放的授权标准，允许第三方应用访问用户在某个服务提供商上的资源，而无需将用户的凭证（如用户名密码）直接暴露给第三方应用。它定义了多种授权流程 (grant types)。我们将使用其中的 "Password Flow" (资源所有者密码凭证授予)，因为我们的 FastAPI 应用本身就是用户直接交互的服务。
*   **JWT (JSON Web Token)**：一种开放标准 (RFC 7519)，它定义了一种紧凑且自包含的方式，用于在各方之间安全地传输信息作为 JSON 对象。这些信息可以被验证和信任，因为它是数字签名的。
    *   **结构**：JWT 由三部分组成，用点 (`.`) 分隔：
        1.  **Header (头部)**：包含类型 (通常是 "JWT") 和所使用的签名算法 (如 HMAC SHA256 或 RSA)。
        2.  **Payload (负载)**：包含声明 (claims)。声明是关于实体 (通常是用户) 和附加数据的语句。有三种类型的声明：注册声明、公共声明和私有声明。
            *   常见的注册声明：`iss` (issuer), `exp` (expiration time), `sub` (subject), `aud` (audience)。
        3.  **Signature (签名)**：用于验证消息在传递过程中没有被更改，并且对于使用私钥签名的令牌，它还可以验证 JWT 的发送者是谁。

**认证流程 (Password Flow + JWT):**

1.  用户通过客户端 (如移动 App) 发送用户名和密码到 FastAPI 的登录端点 (例如 `/login` 或 `/token`)。
2.  服务器验证用户名和密码是否正确 (与数据库中存储的哈希密码对比)。
3.  如果凭证有效，服务器会生成一个 JWT (Access Token)，其中包含用户信息 (如用户 ID 或用户名) 和过期时间。
4.  服务器将此 Access Token 返回给客户端。
5.  客户端在后续请求受保护的 API 端点时，需要在 HTTP 请求的 `Authorization` 头部中携带此 Access Token (通常使用 `Bearer` 方案，如 `Authorization: Bearer <your_token>`)。
6.  服务器在收到请求后，会从 `Authorization` 头部提取 Token，验证其签名和有效性 (如是否过期)，并解析出用户信息。
7.  如果 Token 有效，服务器则处理该请求。否则，返回认证错误 (如 HTTP 401 Unauthorized)。

#### 安装必要的库

```bash
pip install python-jose[cryptography] passlib[bcrypt] python-multipart
```
*   `python-jose[cryptography]`: 用于 JWT 的创建、编码、解码和签名。`[cryptography]` 额外安装了 `cryptography` 库，提供了更广泛的加密算法支持。
*   `passlib[bcrypt]`: 用于密码哈希和验证。`[bcrypt]` 安装了 `bcrypt` 哈希算法的支持。
*   `python-multipart`: FastAPI 需要它来解析表单数据 (例如，当使用 OAuth2 密码流时，客户端通常以表单形式发送用户名和密码)。

#### 密码哈希 (`auth.py` 或 `security.py`)

创建一个 `auth.py` (或 `security.py`) 文件来处理与认证相关的逻辑。

```python
# auth.py
from passlib.context import CryptContext
from datetime import datetime, timedelta, timezone # Ensure timezone is imported
from typing import Optional, Union, Any # Union and Any might be needed for jwt payload
from jose import JWTError, jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

from .config import get_settings # 导入 settings
from . import schemas, crud # 导入 Pydantic schemas 和 CRUD 操作
from .database import get_db # 导入数据库会话依赖
from sqlalchemy.ext.asyncio import AsyncSession

settings = get_settings()

# --- 密码哈希 --- #
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

# --- JWT 配置 --- #
SECRET_KEY = settings.SECRET_KEY # 从配置加载
ALGORITHM = settings.ALGORITHM   # 从配置加载
ACCESS_TOKEN_EXPIRE_MINUTES = settings.ACCESS_TOKEN_EXPIRE_MINUTES # 从配置加载

# OAuth2 密码模式的 Token URL (客户端将向此 URL 发送用户名密码)
# 通常这是 /token 或 /login 等端点
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Pydantic model for the token response
class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None
    # scopes: List[str] = [] # 如果使用 scopes

# --- JWT 创建与解码 --- #
def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(
    db: AsyncSession = Depends(get_db),
    token: str = Depends(oauth2_scheme)
) -> schemas.UserWithSensitiveInfo: # 返回包含哈希密码的内部用户模型
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: Optional[str] = payload.get("sub") # "sub" 通常用于存储用户名或用户ID
        if username is None:
            raise credentials_exception
        token_data = schemas.TokenData(username=username) # 使用Pydantic模型校验payload结构
    except JWTError:
        raise credentials_exception
    except Exception: #捕获 Pydantic ValidationError 等
        raise credentials_exception
    
    user = await crud.get_user_by_username(db, username=token_data.username)
    if user is None:
        raise credentials_exception
    # 为了后续的权限检查，可能需要返回完整的用户信息，包括密码哈希（但不直接暴露给外部）
    # 或者只返回 User Pydantic 模型（不含密码）
    # return user # 如果 crud.get_user_by_username 返回的是 models.User
    # 将 SQLAlchemy 模型转换为 Pydantic 模型
    user_pydantic = schemas.UserWithSensitiveInfo.from_orm(user) 
    return user_pydantic


async def get_current_active_user(
    current_user: schemas.UserWithSensitiveInfo = Depends(get_current_user)
) -> schemas.UserWithSensitiveInfo:
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

# (你可能还需要添加一个 get_current_admin_user 等依赖项，用于基于角色的授权)
```
*   **`pwd_context`**: 配置 `passlib` 使用 `bcrypt` 算法进行密码哈希。
*   **`verify_password`**: 验证明文密码是否与存储的哈希密码匹配。
*   **`get_password_hash`**: 生成密码的哈希值。
*   **`SECRET_KEY`, `ALGORITHM`, `ACCESS_TOKEN_EXPIRE_MINUTES`**: JWT 相关的配置，**务必在 `config.py` 中设置这些值**，特别是 `SECRET_KEY`，它应该是随机生成的长字符串，并严格保密。
    在 `config.py` 的 `Settings` 类中添加：
    ```python
    # config.py
    # ...
    class Settings(BaseSettings):
        DATABASE_URL: str = "postgresql+asyncpg://myuser:your_secure_password@localhost:5432/mydb"
        SECRET_KEY: str = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7" # 生成一个随机密钥
        ALGORITHM: str = "HS256"
        ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
        # ...
    ```
    你可以使用 `openssl rand -hex 32` 在终端生成一个随机密钥。
*   **`oauth2_scheme`**: FastAPI 的安全工具，用于从请求中提取 `Authorization: Bearer <token>` 头部。
*   **`Token`, `TokenData`**: Pydantic 模型，用于定义 Token 响应和 Token 内部数据的结构。
*   **`create_access_token`**: 创建 JWT。`data` 参数通常是一个包含 `{"sub": username}` 的字典。`exp` (过期时间) 声明是 JWT 的标准部分。
*   **`get_current_user`**: 这是一个 FastAPI 依赖项。它会：
    1.  通过 `oauth2_scheme` 从请求头获取 Token。
    2.  解码并验证 Token (签名、过期时间)。
    3.  从 Token 的 `sub` 声明中获取用户名。
    4.  从数据库中查询该用户。
    5.  如果任何步骤失败，则抛出 `HTTPException` (通常是 401 未授权)。
    6.  成功则返回用户信息 (这里返回了包含哈希密码的 `schemas.UserWithSensitiveInfo`，是为了后续可能的权限校验或内部使用，但绝不能直接返回给最终客户端)。
*   **`get_current_active_user`**: 另一个依赖项，它依赖于 `get_current_user`，并额外检查用户是否处于活动状态。这是在 `get_current_user` 基础上添加更多检查的常见模式。

#### 登录端点 (`/token`)

在 `main.py` 或你的认证路由文件中 (例如 `routers/auth_router.py`) 创建登录端点。

```python
# main.py 或 routers/auth_router.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm # FastAPI 提供的用于密码流的表单数据模型
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import timedelta

from . import auth, crud, schemas # 导入相关模块
from .database import get_db

# app = FastAPI() # 如果在 main.py
router = APIRouter() # 如果使用路由器

@router.post("/token", response_model=auth.Token, tags=["Authentication"])
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends(), 
    db: AsyncSession = Depends(get_db)
):
    user = await crud.get_user_by_username(db, username=form_data.username)
    if not user or not auth.verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    if not user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")

    access_token_expires = timedelta(minutes=auth.ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = auth.create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return {"access_token": access_token, "token_type": "bearer"}

# 在你的主 app 文件中包含这个路由 (如果使用了 APIRouter)
# from .routers import auth_router
# app.include_router(auth_router.router) 
```
*   `OAuth2PasswordRequestForm = Depends()`: FastAPI 会自动从请求体中读取表单数据 (包含 `username` 和 `password` 字段)。客户端应以 `application/x-www-form-urlencoded` 格式发送这些数据。
*   端点会验证用户凭证，如果有效，则创建并返回 JWT。
*   `tags=["Authentication"]` 用于在 API 文档中分组。

#### 保护 API 端点

现在你可以使用 `Depends(get_current_active_user)` (或 `Depends(get_current_user)`) 来保护你的 API 端点。

```python
# routers/users.py (或 main.py)
# ... 其他导入 ...
from .. import auth # 假设 auth.py 在上一级目录
from ..schemas import User # 导入不含敏感信息的User Pydantic模型

# ... (router 定义)

@router.get("/users/me/", response_model=User, tags=["Users"])
async def read_users_me(current_user: User = Depends(auth.get_current_active_user)):
    # current_user 将是 get_current_active_user 返回的用户对象 (不含密码)
    # 注意：get_current_active_user 返回的是 schemas.UserWithSensitiveInfo
    # 但 FastAPI 会根据 response_model=schemas.User 进行序列化，只返回 User 中定义的字段
    return current_user

@router.post("/users/{user_id}/items/", response_model=schemas.Item, status_code=201, tags=["Items"])
async def create_item_for_user_protected(
    user_id: int, 
    item: schemas.ItemCreate, 
    db: AsyncSession = Depends(get_db),
    current_user: schemas.User = Depends(auth.get_current_active_user) # 认证
):
    # 简单的授权检查：确保当前登录用户只能为自己创建物品
    if current_user.id != user_id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Not allowed to create items for another user")

    db_user = await crud.get_user(db, user_id=user_id) # 确保目标用户存在
    if not db_user:
        raise HTTPException(status_code=404, detail="Owner user not found")
    return await crud.create_user_item(db=db, item=item, user_id=user_id)

# ... 其他需要保护的端点类似修改 ...
```
*   如果请求未提供有效 Token，或者 Token 无效/过期，`get_current_active_user` 依赖项会抛出 `HTTPException`，FastAPI 会自动返回 401 未授权响应。
*   如果 Token 有效，`current_user` 参数会包含已认证用户的信息，你可以在路径操作函数中使用它。

### 3. 授权机制：基于角色的访问控制 (RBAC) 示例

授权决定了已认证用户可以做什么。一个常见的授权模式是基于角色的访问控制 (RBAC)。

1.  **定义角色**：在你的 `User` 模型中添加一个 `role` 字段 (例如 `role: str`，可以是 "user", "admin", "editor" 等)。
    ```python
    # models.py
    class User(Base):
        # ... 其他字段 ...
        role = Column(String, default="user", nullable=False) # 添加角色字段
    ```
    别忘了使用 Alembic 生成并应用迁移来更新数据库表结构。
    更新 `schemas.py` 中的 `User` 和 `UserCreate` (如果允许创建时指定角色) 以及 `UserInDBBase` 模型以包含 `role`。

2.  **创建角色相关的依赖项**：
    ```python
    # auth.py
    # ...
    class UserRole:
        ADMIN = "admin"
        USER = "user"
        EDITOR = "editor"
        # ... 其他角色

    def require_role(required_role: str):
        async def role_checker(current_user: schemas.UserWithSensitiveInfo = Depends(get_current_active_user)) -> schemas.UserWithSensitiveInfo:
            # 假设 current_user Pydantic 模型中包含了 role 字段
            if current_user.role != required_role:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail=f"User does not have the required role: {required_role}"
                )
            return current_user
        return role_checker

    # 示例：依赖项，要求用户是管理员
    # get_current_admin_user = require_role(UserRole.ADMIN)
    ```
    *   `require_role` 是一个依赖项工厂，它返回一个依赖项，该依赖项会检查当前用户的角色。
    *   确保你的 `schemas.UserWithSensitiveInfo` (以及可能被依赖项返回的 Pydantic User 模型) 中包含 `role` 字段。

3.  **在端点中使用角色依赖项**：
    ```python
    # routers/admin_router.py (示例)
    # from ..auth import require_role, UserRole

    # router = APIRouter(prefix="/admin", tags=["Admin"], dependencies=[Depends(require_role(UserRole.ADMIN))])

    # @router.get("/dashboard/")
    # async def admin_dashboard(current_admin: schemas.User = Depends(auth.get_current_active_user)):
    #     # 此处 current_admin 已经是经过角色验证的管理员了
    #     return {"message": f"Welcome to the admin dashboard, {current_admin.username}!"}
    ```
    *   你可以在单个路径操作上使用 `Depends(require_role(UserRole.ADMIN))`。
    *   或者，如果你有一个包含多个管理员端点的 `APIRouter`，可以在 `APIRouter` 初始化时通过 `dependencies` 参数应用该角色检查，这样该路由器下的所有端点都会自动进行此权限检查。

**更细粒度的权限 (Scopes/Permissions):**

对于更复杂的应用，你可能需要比角色更细粒度的权限控制 (例如，"edit_article", "delete_user")。OAuth2 提供了 "scopes" 的概念。
*   你可以为用户分配一组权限 (scopes)。
*   在创建 JWT 时，将用户的 scopes 包含在 Token 中。
*   在 `OAuth2PasswordBearer` 中定义你的应用支持的所有 scopes，并在依赖项中检查 Token 是否包含特定端点所需的 scope。
*   FastAPI 的 `SecurityScopes` 可以帮助处理这个。

### 4. 其他安全注意事项

*   **输入校验**：FastAPI 和 Pydantic 自动处理了大部分输入校验 (类型、格式)。但对于业务逻辑相关的校验 (例如，确保某个值在特定范围内)，你仍需在代码中实现。
*   **输出编码**：FastAPI 默认会正确处理 JSON 响应的编码，防止 XSS (跨站脚本攻击)。如果你手动构建 HTML 响应，确保对用户输入进行适当的转义。
*   **依赖项安全**：定期更新你的项目依赖项 (`pip list --outdated`, `pip install -U <package>`)，以修复已知的安全漏洞。
*   **速率限制 (Rate Limiting)**：防止暴力破解 (如登录尝试) 或滥用 API。可以使用第三方库 (如 `slowapi`) 或在 Nginx/API 网关层面实现。
*   **日志记录与监控**：记录重要的安全事件 (如登录失败、未授权访问尝试)，并监控系统异常。
*   **CSRF (Cross-Site Request Forgery) 保护**：如果你的 API 同时服务于 Web 前端 (特别是使用 cookie 进行会话管理时)，需要考虑 CSRF 保护。对于纯粹由移动 App 或其他非浏览器客户端使用的 Token 认证 API，CSRF 通常不是主要问题。
*   **安全头部**：在 Nginx 或 FastAPI 中间件中添加安全相关的 HTTP 头部，如 `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy` 等。

### 小结

本模块详细介绍了如何在 FastAPI 中实现用户认证和授权，这是构建安全后端服务的核心。我们学习了：
*   使用 `passlib` 进行安全的密码哈希与验证。
*   基于 OAuth2 密码模式和 JWT 生成和验证访问令牌。
*   创建依赖项 (`get_current_user`, `get_current_active_user`) 来保护 API 端点。
*   实现了一个 `/token` 端点用于用户登录并获取 JWT。
*   探讨了基于角色的访问控制 (RBAC) 的基本实现思路。
*   强调了其他重要的安全实践，如 HTTPS、输入校验和依赖项安全。

通过实施这些安全措施，你可以大大增强你的 FastAPI 应用抵御常见威胁的能力。记住，安全是一个持续的过程，需要不断学习和适应新的挑战。

--- 