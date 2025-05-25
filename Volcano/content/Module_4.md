## 模块四：FastAPI 进阶与 API 设计

在模块三中，我们成功搭建了 FastAPI 的基础运行环境。本模块将带你深入了解 FastAPI 的核心功能，并探讨如何设计出健壮、易用且易于维护的 API。

### 1. 回顾 FastAPI 核心功能

FastAPI 以其现代化的特性和高性能而备受青睐。以下是一些核心概念的回顾与深入：

#### 路径参数 (Path Parameters)

路径参数是 URL 路径的一部分，用于唯一标识资源。

```python
# main.py (示例)
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int): # user_id 会被自动转换为 int
    return {"user_id": user_id, "name": f"User {user_id}"}

@app.get("/files/{file_path:path}") # :path 表示参数可以包含 /
async def read_file(file_path: str):
    return {"file_path": file_path}
```
*   **类型提示**：FastAPI 利用 Python 的类型提示 (如 `user_id: int`) 来进行数据校验和转换。如果传入的 `user_id` 不能转换为整数，FastAPI 会自动返回一个包含清晰错误信息的 HTTP 422 Unprocessable Entity 响应。
*   **路径转换器 `:path`**：允许路径参数包含斜杠 `/`，常用于文件路径等场景。

#### 查询参数 (Query Parameters)

查询参数是 URL 中 `?` 之后的部分，用于过滤或指定资源的特定方面。

```python
# main.py (示例)
from fastapi import FastAPI
from typing import List, Optional

app = FastAPI()

@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10, q: Optional[str] = None, tags: List[str] = []):
    # skip: 默认值为 0
    # limit: 默认值为 10
    # q: 可选参数，默认为 None
    # tags: 列表参数，可以接收多个同名查询参数，如 /items/?tags=A&tags=B
    results = {"skip": skip, "limit": limit}
    if q:
        results.update({"q": q})
    if tags:
        results.update({"tags": tags})
    return results
```
*   **默认值**：可以为查询参数提供默认值 (如 `skip: int = 0`)。
*   **可选参数 `Optional[str] = None`** (或 Python 3.10+ 的 `str | None = None`)：表示参数是可选的。
*   **类型转换与校验**：同样会自动进行。
*   **列表/多个值**：通过类型提示 `List[str]`，FastAPI 可以处理形如 `?tags=foo&tags=bar` 的查询参数，将其收集到一个 Python 列表中。

#### 请求体 (Request Body)

当需要客户端发送复杂数据（如创建或更新资源）时，通常使用请求体。FastAPI 使用 Pydantic 模型来定义和校验请求体。

1.  **安装 Pydantic (如果尚未作为 FastAPI 依赖安装)**：
    FastAPI 通常会自动安装 Pydantic。如果你的虚拟环境中没有，可以手动安装：
    ```bash
    pip install pydantic
    ```

2.  **定义 Pydantic 模型：**
    ```python
    # main.py
    from fastapi import FastAPI
    from pydantic import BaseModel, EmailStr
    from typing import Optional

    class Item(BaseModel):
        name: str
        description: Optional[str] = None # 可选字段，默认值为 None
        price: float
        tax: Optional[float] = None

    class UserCreate(BaseModel):
        username: str
        email: EmailStr # Pydantic 提供的 EmailStr 会校验邮件格式
        password: str
        full_name: Optional[str] = None

    app = FastAPI()

    @app.post("/items/")
    async def create_item(item: Item): # 请求体将按照 Item 模型进行校验和转换
        return {"item_name": item.name, "item_price_with_tax": item.price + (item.tax or 0)}

    @app.post("/users/", status_code=201) # 指定成功响应的状态码为 201 Created
    async def create_user(user: UserCreate):
        # 实际应用中，这里会将 user 数据存入数据库
        return {"username": user.username, "email": user.email, "message": "User created successfully."}
    ```
    *   **`BaseModel`**：所有请求体模型都应继承自 `pydantic.BaseModel`。
    *   **字段类型**：定义模型字段及其类型。Pydantic 支持多种标准 Python 类型以及它自己提供的一些特殊类型 (如 `EmailStr`, `HttpUrl`)。
    *   **数据校验**：当请求到达时，FastAPI 会自动使用 Pydantic 模型校验请求体。如果数据无效 (如类型不匹配、缺少必填字段)，将返回 HTTP 422 错误。
    *   **数据转换**：如果数据有效，它会被转换为 Pydantic 模型实例，你可以在路径操作函数中直接使用。
    *   **编辑器支持**：由于有 Pydantic 模型，你在编辑器中可以获得很好的自动补全和类型检查。

#### 数据校验与序列化 (Pydantic 模型)

Pydantic 不仅用于请求体验证，也广泛用于：

*   **响应模型 (Response Model)**：你可以指定路径操作函数返回的数据结构，FastAPI 会确保返回的数据符合该模型，并过滤掉模型之外的字段。这对于 API 的一致性和文档化非常有帮助。

    ```python
    # main.py
    from fastapi import FastAPI
    from pydantic import BaseModel, EmailStr
    from typing import List, Optional

    # ... (假设 Item 和 UserCreate 模型已定义) ...

    class UserResponse(BaseModel): # 定义响应模型，不包含密码等敏感信息
        username: str
        email: EmailStr
        full_name: Optional[str] = None

    app = FastAPI()
    # ... (create_item 可能不变) ...

    # 模拟数据库
    fake_users_db = {}

    @app.post("/users_with_response_model/", response_model=UserResponse, status_code=201)
    async def create_user_with_response(user: UserCreate):
        user_data = user.model_dump() # Pydantic v2 用 model_dump() 代替 .dict()
        # 实际应用中，user_data['password'] 会被哈希后存储
        fake_users_db[user.username] = user_data
        return user_data # FastAPI 会根据 UserResponse 模型过滤和序列化
    ```
    在 `create_user_with_response` 中，即使我们返回了包含密码的 `user_data`，由于指定了 `response_model=UserResponse`，FastAPI 会确保最终响应中只包含 `UserResponse` 模型中定义的字段。

*   **更复杂的校验规则**：Pydantic 允许你通过 `Field` 函数或自定义验证器来实现更复杂的校验逻辑 (如最小/最大长度、正则表达式匹配等)。

    ```python
    from fastapi import FastAPI, Body # 导入 Body
    from pydantic import BaseModel, Field # 导入 Field
    from typing import Optional

    class Product(BaseModel):
        name: str = Field(..., min_length=3, max_length=50, title="Product Name", description="Name of the product")
        price: float = Field(..., gt=0, description="Price must be greater than zero") # gt = greater than
        description: Optional[str] = Field(None, max_length=300)

    app = FastAPI()

    @app.post("/products/")
    async def create_product(product: Product):
        return product

    # 使用 Body 嵌入单个请求体参数
    @app.post("/offers/")
    async def create_offer(name: str = Body(..., embed=True), price: float = Body(..., embed=True)):
        return {"name": name, "price": price}
    ```
    *   `Field(...)`：`...` 表示该字段是必需的。
    *   `Body(..., embed=True)`：如果你的路径操作函数希望接收一个 JSON 对象，其键是参数名，而不是直接将参数作为 JSON 对象的顶级键，可以使用 `embed=True`。

#### 依赖注入系统 (Depends)

FastAPI 的依赖注入系统是一个非常强大且优雅的特性，用于处理共享逻辑、认证、数据库连接等。

*   **概念**：你可以定义一些可重用的函数 (称为"依赖项")，FastAPI 会在调用路径操作函数之前自动执行这些依赖项，并将它们返回的结果作为参数注入到路径操作函数中。
*   **用途**：
    *   **共享数据库会话**：创建一个依赖项来获取数据库连接/会话，并确保在使用后关闭它。
    *   **认证与授权**：验证用户身份 (如检查 Token) 并获取当前用户信息。
    *   **共享参数或配置**：从配置文件或环境变量中加载配置。
    *   **可重用的参数校验逻辑**。

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException, status, Header # Header needs to be imported
from typing import Optional

# 假设 oauth2_scheme (用于演示，实际需要 OAuth2PasswordBearer)
from fastapi.security import OAuth2PasswordBearer
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token") # "token" 是获取 token 的路径

app = FastAPI()

# 示例：一个简单的共享参数依赖
async def common_parameters(q: Optional[str] = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items_dep/")
async def read_items_dep(commons: dict = Depends(common_parameters)):
    # commons 将是 common_parameters 函数返回的字典
    # 在这里处理 items 的获取，使用 commons['q'], commons['skip'], commons['limit']
    return {"message": "Items fetched with common parameters", "params": commons}

# 示例：一个简单的认证依赖 (概念性)
async def get_current_user(token: str = Depends(oauth2_scheme)): # token type changed to str as it's expected by OAuth2PasswordBearer
    if not token: # 实际中会验证 token 的有效性、解析内容等
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Not authenticated",
            headers={"WWW-Authenticate": "Bearer"},
        )
    # 假设 token 有效，并能从中获取用户信息，这里返回一个假的用户
    # 在实际应用中，你会从token中解码用户信息或查询数据库
    return {"username": "fakeuser", "email": "user@example.com", "token": token} 

@app.get("/users/me/")
async def read_users_me(current_user: dict = Depends(get_current_user)):
    return current_user
```
*   `Depends(dependency_function)`：告诉 FastAPI 在调用路径操作函数之前需要先执行 `dependency_function`。
*   依赖项可以是 `async def` 或普通 `def` 函数。
*   依赖项也可以有自己的依赖项，形成依赖链。
*   如果依赖项中发生 `HTTPException`，FastAPI 会立即停止处理并返回该 HTTP 错误。

#### 中间件 (Middleware)

中间件允许你在请求被路径操作函数处理之前，以及响应被发送回客户端之前，执行一些代码。

*   **用途**：
    *   记录请求日志。
    *   添加自定义 HTTP 头部 (如 CORS 头部)。
    *   处理全局异常。
    *   Gzip 压缩响应。
    *   请求处理计时。

```python
# main.py
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request) # 执行后续的中间件和路径操作
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time) # 添加自定义头部
    return response

@app.get("/")
async def root():
    # Ensure this path is different from other root paths if combining files
    # or ensure this is the intended root for this specific app instance.
    return {"message": "Hello World from Middleware Example"}
```
*   中间件是一个 `async` 函数，接收 `request: Request` 和一个 `call_next` 函数作为参数。
*   `await call_next(request)` 调用链中的下一个处理程序 (可能是另一个中间件或实际的路径操作函数)。
*   你可以在 `await call_next(request)` 之前修改请求，之后修改响应。

### 2. 构建健壮的 API

设计良好的 API 对于开发者体验和应用的可维护性至关重要。

#### RESTful API 设计原则

REST (Representational State Transfer) 是一种流行的 API 设计风格。虽然 FastAPI 不强制你遵循 REST，但理解其原则有助于设计出更清晰、更一致的 API。

*   **使用名词表示资源**：URL 应该表示资源，而不是动作。
    *   好：`/users`, `/users/123`, `/users/123/orders`
    *   差：`/getUsers`, `/createUser`, `/getUserOrders?userId=123`
*   **使用 HTTP 方法 (Verbs) 表示操作**：
    *   `GET`：检索资源 (安全且幂等)。
    *   `POST`：创建新资源 (不幂等)。
    *   `PUT`：完整替换/更新现有资源 (幂等)。
    *   `PATCH`：部分更新现有资源 (不一定幂等)。
    *   `DELETE`：删除资源 (幂等)。
*   **使用 HTTP 状态码表示结果**：
    *   `2xx` (成功)：
        *   `200 OK`：通用成功。
        *   `201 Created`：资源创建成功 (通常在 POST 后返回，响应中可能包含新资源的 URL)。
        *   `204 No Content`：操作成功，但响应体中没有内容 (例如 DELETE 成功)。
    *   `3xx` (重定向)：
        *   `301 Moved Permanently`
        *   `302 Found`
    *   `4xx` (客户端错误)：
        *   `400 Bad Request`：请求无效 (例如，参数格式错误)。
        *   `401 Unauthorized`：需要认证。
        *   `403 Forbidden`：已认证，但无权访问。
        *   `404 Not Found`：资源不存在。
        *   `422 Unprocessable Entity`：请求体格式正确，但语义错误 (FastAPI 常用于 Pydantic 校验失败)。
    *   `5xx` (服务器错误)：
        *   `500 Internal Server Error`：服务器内部发生未知错误。
        *   `502 Bad Gateway`：上游服务器返回无效响应。
        *   `503 Service Unavailable`：服务暂时不可用 (例如，过载或维护)。
*   **支持过滤、排序和分页 (对于集合资源)**：
    *   过滤：`GET /items?status=published&author_id=1`
    *   排序：`GET /items?sort_by=created_at&order=desc`
    *   分页：`GET /items?skip=0&limit=20` (或基于游标的分页)
*   **版本控制**：当 API 发生不兼容的更改时，需要进行版本控制。
*   **无状态 (Stateless)**：每个来自客户端的请求都应包含服务器处理该请求所需的所有信息。服务器不应依赖于先前请求的会话状态。

#### 版本控制策略

当你的 API 演进并引入破坏性更改时，版本控制变得必要。

*   **URL 版本控制 (最常见)**：
    *   `/v1/users`
    *   `/v2/users`
    *   **优点**：简单直观，易于通过 URL 区分。
    *   **缺点**：URL 可能会变长，不同版本的代码可能需要不同的路由配置。
    *   **FastAPI 实现**：可以使用 `APIRouter` 来组织不同版本的路由。
        ```python
        from fastapi import APIRouter, FastAPI

        app_versioned = FastAPI() # Use a different app name to avoid conflicts if running in same context
        router_v1 = APIRouter(prefix="/v1")
        router_v2 = APIRouter(prefix="/v2")

        @router_v1.get("/items")
        async def get_items_v1():
            return {"version": "v1", "items": ["item1", "item2"]}

        @router_v2.get("/items")
        async def get_items_v2():
            return {"version": "v2", "items": ["new_item1", "new_item2"], "metadata": "some_new_data"}

        app_versioned.include_router(router_v1)
        app_versioned.include_router(router_v2)
        ```

*   **请求头版本控制 (Header Versioning)**：
    *   客户端通过自定义请求头指定 API 版本，如 `Accept-Version: v1` 或 `X-API-Version: 1`。
    *   **优点**：URL 保持干净。
    *   **缺点**：不如 URL 版本控制直观，客户端需要正确设置请求头。
    *   **FastAPI 实现**：可以通过依赖项或中间件读取请求头并分发到相应的处理逻辑。

*   **查询参数版本控制 (Query Parameter Versioning)**：
    *   `/users?version=1`
    *   **优点**：URL 相对干净。
    *   **缺点**：不如此常见，版本信息容易被忽略。

**建议**：对于大多数应用，URL 版本控制 (如 `/api/v1/...`) 是一个清晰且广泛接受的方法。

#### 统一的错误处理与响应格式

*   **全局异常处理器**：使用 FastAPI 的 `@app.exception_handler(CustomException)` 或 `@app.exception_handler(RequestValidationError)` (用于处理 Pydantic 校验错误) 来自定义错误响应的格式。

    ```python
    from fastapi import FastAPI, HTTPException, Request, status
    from fastapi.responses import JSONResponse
    from fastapi.exceptions import RequestValidationError # 用于 Pydantic 校验错误
    from pydantic import BaseModel

    class UnicornException(Exception):
        def __init__(self, name: str, message: str):
            self.name = name
            self.message = message

    app_errors = FastAPI() # Use a different app name

    # 自定义异常处理器
    @app_errors.exception_handler(UnicornException)
    async def unicorn_exception_handler(request: Request, exc: UnicornException):
        return JSONResponse(
            status_code=418, # I'm a teapot ;)
            content={"error_type": "UnicornError", "name": exc.name, "message": f"Oops! {exc.message} a unicorn did something."},
        )

    # 自定义 Pydantic 校验错误的处理器 (覆盖默认的 422 响应)
    @app_errors.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        # exc.errors() 包含详细的校验错误信息
        return JSONResponse(
            status_code=status.HTTP_400_BAD_REQUEST, # 或者保持 422
            content={"detail": "Validation Error", "errors": exc.errors()},
        )
    
    # 也可以处理通用的 HTTPException
    @app_errors.exception_handler(HTTPException)
    async def http_exception_handler(request: Request, exc: HTTPException):
        return JSONResponse(
            status_code=exc.status_code,
            content={"error_code": exc.status_code, "message": exc.detail},
        )


    class ItemForErrorHandling(BaseModel):
        name: str
        price: float

    @app_errors.get("/unicorns/{name}")
    async def read_unicorn(name: str):
        if name == "yolo":
            raise UnicornException(name=name, message="YOLO is not a real unicorn.")
        return {"unicorn_name": name}

    @app_errors.post("/items_validation/")
    async def create_item_validation(item: ItemForErrorHandling): # 如果请求体不符合 Item 模型，会触发 RequestValidationError
        return item
    ```

*   **统一响应结构**：考虑为所有成功的 API 响应（尤其是返回数据的 GET 请求）定义一个通用的包装结构，例如：
    ```json
    {
        "status": "success", 
        "data": { /* 实际数据 */ },
        "message": null 
    }
    ```
    或者对于列表：
    ```json
    {
        "status": "success",
        "data": [ /* 列表数据 */ ],
        "pagination": {
            "total_items": 100,
            "per_page": 10,
            "current_page": 1,
            "total_pages": 10
        },
        "message": null
    }
    ```
    这样做的好处是客户端可以以一致的方式处理 API 响应。但这并非强制，FastAPI 默认直接返回路径操作函数的结果。

#### API 文档自动生成与交互 (Swagger UI, ReDoc)

FastAPI 的一大亮点是基于 OpenAPI 标准自动生成交互式 API 文档。

*   **OpenAPI**：一个 API 描述格式规范，用于定义 RESTful API。
*   **Swagger UI**: 访问 `/docs` (例如 `http://127.0.0.1:8000/docs`) 可以看到 Swagger UI 界面，你可以在这里浏览所有 API 端点、查看其参数、请求体、响应模型，并直接在浏览器中发送请求进行测试。
*   **ReDoc**: 访问 `/redoc` (例如 `http://127.0.0.1:8000/redoc`) 可以看到 ReDoc 生成的替代文档界面，它通常更侧重于可读性。

这些文档是根据你的 Python 代码 (路径操作、类型提示、Pydantic 模型、`docstring` 等) 自动生成的。
你可以通过在路径操作函数、参数、Pydantic 模型字段中使用 `docstring` 或 `title`, `description` 等参数来丰富文档内容。

```python
from fastapi import FastAPI, Path, Query, Header # Header was missing
from pydantic import BaseModel, Field
from typing import Optional

app_docs = FastAPI(
    title="我的酷炫应用 API",
    description="这个 API 提供了管理用户和物品的功能。",
    version="1.0.0",
) # Use a different app name

class ItemForDocs(BaseModel):
    name: str = Field(..., description="物品的名称")
    description: Optional[str] = Field(None, description="物品的详细描述")
    price: float = Field(..., gt=0, description="物品价格，必须大于0")

@app_docs.post("/items_docs/", response_model=ItemForDocs, summary="创建一个新物品", tags=["Items"]) # Changed path to avoid conflict
async def create_item_docs(
    item: ItemForDocs,
    secret_header: Optional[str] = Header(None, description="一个秘密的头部信息") 
):
    """
    创建一个物品，包含名称、可选描述和价格。
    - **名称**: 每个物品必须有一个唯一的名称。
    - **价格**: 必须是正数。
    """
    return item

@app_docs.get(
    "/items_docs/{item_id}", # Changed path to avoid conflict
    response_model=ItemForDocs,
    summary="根据 ID 获取物品",
    description="通过路径中的 item_id 来检索单个物品的详细信息。",
    tags=["Items"] # 用于在文档中对端点进行分组
)
async def get_item_docs(
    item_id: int = Path(..., description="要检索的物品 ID", ge=1), # ge = greater than or equal to
    q: Optional[str] = Query(None, description="可选的搜索查询字符串")
):
    # 实际中会从数据库获取 item
    return {"name": f"Item {item_id}", "price": 10.0 + item_id, "description": "An item from docs example"}
```
*   `FastAPI(title="...", description="...", version="...")`: 配置 API 的全局元数据。
*   `tags=["Items"]`: 用于在 Swagger UI 和 ReDoc 中对相关的 API 端点进行分组。
*   `summary="..."`: 为端点提供简短摘要。
*   `description="..."`: 为端点或参数提供更详细的描述，可以使用 Markdown。
*   Pydantic `Field`, FastAPI `Path`, `Query`, `Header` 等函数中的 `description` 参数也会显示在文档中。

### 3. 异步编程与性能优化

FastAPI 的设计初衷之一就是支持高性能的异步操作。

#### FastAPI 的异步支持 (`async/await`)

*   **ASGI (Asynchronous Server Gateway Interface)**：FastAPI 构建在 ASGI 之上，这是一个为异步 Python Web 框架设计的标准接口，与传统的 WSGI (Web Server Gateway Interface) 不同，ASGI 支持异步操作。
*   **`async def` 路径操作函数**：你可以将你的路径操作函数定义为 `async def`。这意味着在该函数内部，你可以使用 `await` 来调用其他异步函数 (例如，异步数据库查询、异步 HTTP 请求到其他服务) 而不会阻塞整个服务器进程。

    ```python
    import asyncio
    from fastapi import FastAPI

    app_async = FastAPI() # Use a different app name

    async def fake_db_call(duration: int):
        await asyncio.sleep(duration) # 模拟耗时的 I/O 操作
        return {"message": f"Data fetched after {duration} seconds"}

    @app_async.get("/async-data/")
    async def get_async_data():
        print("Request received for async data")
        result1 = await fake_db_call(2) # 调用异步函数
        result2 = await fake_db_call(1) # 再次调用
        print("Async calls completed")
        return {"result1": result1, "result2": result2}

    @app_async.get("/sync-data/")
    def get_sync_data(): # 普通的同步函数
        print("Request received for sync data")
        # 如果这里有长时间阻塞的 I/O (未使用 await)
        # import time
        # time.sleep(3) # 假设这是一个阻塞操作
        print("Sync operation completed")
        return {"message": "Sync data fetched"}
    ```
    当 `get_async_data` 中的 `await fake_db_call(...)` 执行时，事件循环可以将 CPU 交给其他任务 (例如处理另一个请求)，直到异步操作完成。这使得 FastAPI 能够用较少的进程/线程处理大量的并发连接，尤其是在 I/O 密集型应用中。

#### 利用异步处理 I/O 密集型任务

*   **I/O 密集型任务**：指那些大部分时间花在等待外部操作完成的任务，例如：
    *   数据库查询。
    *   调用外部 API (HTTP 请求)。
    *   文件读写。
*   **为什么异步有优势**：对于这类任务，当一个请求在等待 I/O 时，服务器的 CPU 是空闲的。同步服务器通常会为每个请求分配一个线程/进程，如果线程在等待 I/O，它就不能处理其他请求，导致资源浪费和并发能力下降。异步服务器 (如 Uvicorn + FastAPI) 可以在一个线程内通过事件循环高效地处理多个等待 I/O 的请求。

*   **与 PostgreSQL 的异步交互**：
    *   要实现真正的异步数据库操作，你需要使用支持 `async/await` 的数据库驱动程序。
    *   对于 PostgreSQL，流行的异步驱动是 `asyncpg`。
    *   ORM (对象关系映射器) 如 SQLAlchemy (配合其异步扩展) 或 Tortoise ORM 也提供了对异步操作的支持。我们将在模块五中详细讨论。

    **重要提示：**
    *   **不要在 `async def` 函数中调用阻塞的 I/O 操作**：如果你在 `async def` 路径操作函数中直接调用了一个传统的、阻塞的库函数 (例如标准库的 `time.sleep()` 或一个同步的数据库驱动的查询方法)，它仍然会阻塞整个事件循环，抵消异步带来的好处。
    *   **FastAPI 处理同步函数的方式**：如果你将路径操作函数定义为普通的 `def` (同步函数)，FastAPI 会在一个单独的线程池中运行它，这样它就不会阻塞主事件循环。这对于逐步迁移旧的同步代码或使用不支持异步的库很有用。但为了最大化性能，对于 I/O 密集型操作，首选 `async def` 并配合异步库。

### 小结

本模块我们深入探讨了 FastAPI 的核心功能，包括路径参数、查询参数、请求体、Pydantic 模型、依赖注入和中间件。我们还讨论了构建健壮 API 的原则，如 RESTful 设计、版本控制、统一错误处理和利用 FastAPI 强大的自动文档生成。最后，我们强调了异步编程 (`async/await`) 在处理 I/O 密集型任务 (如与 PostgreSQL 的异步交互) 时的重要性及其对性能的提升。

掌握这些进阶功能将使你能够构建出更复杂、更可靠、更高性能的 FastAPI 应用。

---

[返回大纲](#outline) 