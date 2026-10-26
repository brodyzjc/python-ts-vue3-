# FastAPI 框架详解

在众多 Python Web 框架中，本书选择 FastAPI 作为可商用的现代后端框架，理由非常直接：它基于 ASGI 异步协议，性能在 Python 生态中属于第一梯队；它用 Python 类型注解驱动一切，实现了真正的类型安全——编辑器补全、运行时校验、自动文档全都来自同一份类型声明；它自带 Swagger UI 和 ReDoc 两套交互式 API 文档，前后端联调几乎不需要额外写接口文档；它的生态（Pydantic、Starlette、SQLAlchemy、uvicorn 等）成熟稳定，已被大量公司用于生产环境。学会 FastAPI，你就掌握了现代 Python 后端开发的主流范式。

## 本章导读

- FastAPI 的底层架构：Starlette、Pydantic、ASGI 与 uvicorn 各自扮演什么角色
- 从零跑起一个最小应用，认识自动生成的交互式文档
- 路由系统：路径参数、查询参数、请求体、Header 与 Cookie 的声明与校验
- Pydantic v2 模型：字段约束、嵌套模型、自定义校验器、模型配置
- 响应控制：用 response_model 过滤字段、状态码、自定义响应、统一异常处理
- 依赖注入系统：Depends 的执行原理，以及数据库会话注入这一经典范式
- 中间件机制与跨域（CORS）配置
- 后台任务、文件上传与静态文件服务
- 中大型项目的目录结构组织与配置管理
- 异步编程：事件循环原理，async def 与 def 路由的本质区别

---

## 一、FastAPI 是什么与底层架构

### 1.1 FastAPI 的本质：站在两个巨人的肩膀上

FastAPI 本身并不是一个"从零造轮子"的框架，它本质上是一层精心设计的"胶水"，把两个底层库粘合在一起：

- **Starlette**：负责 Web 层的所有事情——路由匹配、请求与响应对象、中间件、WebSocket、后台任务等。可以认为 FastAPI 的"Web 能力"几乎全部来自 Starlette。
- **Pydantic**：负责数据层的所有事情——用 Python 类型注解定义数据模型，自动完成数据校验、类型转换、序列化。FastAPI 的请求体解析、参数校验、`response_model` 过滤，底层都是 Pydantic 在工作。

理解了这一点，你就能解释 FastAPI 的工作流：一个 HTTP 请求进来，Starlette 负责把它解析成 Request 对象并匹配到路由；FastAPI 取出路由函数的类型注解，交给 Pydantic 去校验和转换参数；校验通过后调用你的业务函数；返回值再经过 Pydantic 序列化，由 Starlette 包装成 HTTP 响应发回客户端。

### 1.2 ASGI 与 WSGI 的区别

要理解 FastAPI 为什么快，必须先理解 ASGI。

**WSGI**（Web Server Gateway Interface）是 Python 传统的 Web 网关接口标准，Flask、Django 都基于它。WSGI 的模型非常简单：服务器每收到一个请求，就同步调用一次应用函数，函数返回响应，连接结束。这个模型有两个根本限制：

1. **同步阻塞**：一个请求处理过程中如果发生 I/O 等待（查数据库、调外部 API），这个线程/进程就只能干等，无法处理其他请求。并发能力完全依赖多线程或多进程，而线程是昂贵的操作系统资源。
2. **只支持请求-响应模式**：WSGI 协议本身没有为长连接设计，无法原生支持 WebSocket。

**ASGI**（Asynchronous Server Gateway Interface）是为解决这两个问题而生的新一代标准。ASGI 应用是一个可异步调用的对象，服务器与应用之间通过**事件（event）**通信：连接建立、收到数据、发送数据、连接断开，都是一个个事件。这带来两个核心能力：

1. **异步 I/O**：一个请求在等待数据库返回时，事件循环可以切去处理其他请求，单线程就能支撑成千上万的并发连接。
2. **全双工长连接**：原生支持 WebSocket、HTTP/2、Server-Sent Events 等协议。

FastAPI（经由 Starlette）是一个 ASGI 框架，这就是它高性能的协议基础。

### 1.3 uvicorn 的角色：ASGI 服务器

这里有一个初学者最容易混淆的概念：**FastAPI 是框架，不是服务器**。FastAPI 应用本身只是一个符合 ASGI 规范的 Python 对象，它不会自己监听端口、解析 HTTP 报文。你需要一个 **ASGI 服务器**来干这些底层工作，uvicorn 就是最常用的那一个。

uvicorn 基于 `uvloop`（用 Cython 实现的高性能事件循环，替代 asyncio 默认事件循环）和 `httptools`（C 实现的 HTTP 解析器），是整个链路里真正"接网线"的角色：

```text
客户端 ←→ uvicorn（监听端口、解析HTTP、驱动事件循环）
              ←→ FastAPI/Starlette（路由、中间件、依赖注入）
                  ←→ Pydantic（参数校验、序列化）
                      ←→ 你的业务代码
```

生产环境中常见的部署方式是多进程 worker + 反向代理（如 Nginx），以利用多核 CPU，这一点在部署章节会详细展开。这里只需记住：开发用 `uvicorn` 或 `fastapi dev`，生产用多进程加反向代理。

---

## 二、快速开始

### 2.1 安装

```bash
pip install fastapi
pip install "uvicorn[standard]"
```

`uvicorn[standard]` 中的 `standard` 额外依赖包含 `uvloop`、`httptools`、`websockets` 等性能组件，开发环境建议直接装上。

### 2.2 最小应用

创建 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello, FastAPI!"}


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

### 2.3 运行

```bash
uvicorn main:app --reload
```

命令各部分含义：`main` 指 `main.py` 模块，`app` 指模块中名为 `app` 的 FastAPI 实例；`--reload` 开启代码热重载，**只在开发环境使用**（它通过监控文件变化重启进程实现，生产环境会造成额外开销和不稳定）。

新版 FastAPI 也提供了官方 CLI（安装 `fastapi[standard]` 后可用）：

```bash
fastapi dev main.py    # 开发模式，自带热重载
```

启动后访问 `http://127.0.0.1:8000/items/42`，你会得到：

```json
{"item_id": 42}
```

注意：URL 里传入的 `"42"` 本来是字符串，因为函数签名里声明了 `item_id: int`，FastAPI 自动完成了类型转换。如果访问 `/items/abc`，会直接返回 422 错误并附带详细的校验失败信息——这就是类型注解驱动的威力。

### 2.4 自动文档：Swagger UI 与 ReDoc

服务启动后，直接打开浏览器访问：

- `http://127.0.0.1:8000/docs` —— **Swagger UI**，交互式文档，可以直接在页面上填写参数、点击 "Try it out" 发起真实请求，是前后端联调的利器。
- `http://127.0.0.1:8000/redoc` —— **ReDoc**，排版更适合阅读的纯文档形式。

这两套文档都自动从 `/openapi.json` 生成，而 `openapi.json` 是 FastAPI 根据你的路由和类型注解自动生成的 **OpenAPI 规范**文件。也就是说：你写的类型注解同时承担了"运行校验"和"接口文档"两份职责，文档永远和代码保持一致，不会过时。

创建应用时可以传入元数据定制文档：

```python
app = FastAPI(
    title="订单系统 API",
    description="订单系统的后端接口文档",
    version="1.0.0",
    docs_url="/docs",       # 设为 None 可关闭 Swagger UI（生产环境常关闭）
    redoc_url=None,
)
```

---

## 三、路由与请求

FastAPI 用一个核心规则区分参数来源：**参数出现在路径里就是路径参数；参数类型是 Pydantic 模型就是请求体；其余带默认值的函数参数都是查询参数**。框架通过函数签名自动推断，不需要手动注册。

### 3.1 路径参数（Path Parameters）

路径参数是 URL 路径的一部分，用花括号在路径中声明，并在函数签名中声明类型：

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```

路径参数是**必填**的。访问 `/users/abc` 会得到 422 校验错误，而不是匹配不到路由——因为路由能匹配，只是类型转换失败了。

### 3.2 查询参数（Query Parameters）

不在路径中的普通函数参数自动成为查询参数，有默认值即为可选：

```python
@app.get("/items/")
async def list_items(skip: int = 0, limit: int = 10, keyword: str | None = None):
    return {"skip": skip, "limit": limit, "keyword": keyword}
```

请求 `GET /items/?skip=20&limit=5` 会得到 `skip=20, limit=5, keyword=None`。查询字符串里的一切本来都是字符串，FastAPI 根据类型注解自动转换：`?flag=true`、`?flag=1` 都能正确转成 `bool`。

一个实用细节：声明为 `bool` 类型的参数，FastAPI 接受 `true/false/1/0/yes/no/on/off` 等多种写法。

### 3.3 请求体（Request Body）

POST/PUT 等请求的数据通常放在请求体中。把参数声明为 Pydantic 模型，FastAPI 就会从请求体（JSON）解析它：

```python
from pydantic import BaseModel


class Item(BaseModel):
    name: str
    price: float
    description: str | None = None


@app.post("/items/")
async def create_item(item: Item):
    return {"created": item.name, "price_with_tax": item.price * 1.13}
```

请求体、路径参数、查询参数可以**混用**，FastAPI 自动按规则区分：

```python
@app.put("/items/{item_id}")          # item_id 来自路径
async def update_item(item_id: int, item: Item, importance: int = 1):
    # item 来自请求体，importance 来自查询字符串
    return {"item_id": item_id, "item": item, "importance": importance}
```

如果有多个请求体模型，全部声明即可，FastAPI 会合并成一个 JSON body 来解析：

```python
@app.post("/orders/")
async def create_order(item: Item, user: User):
    ...
```

### 3.4 Header 与 Cookie 参数

用 `Header` 和 `Cookie` 显式声明来自请求头/Cookie 的参数：

```python
from fastapi import Cookie, Header


@app.get("/profile/")
async def read_profile(
    authorization: str = Header(),
    user_agent: str | None = Header(default=None),
    session_id: str | None = Cookie(default=None),
):
    return {"authorization": authorization, "user_agent": user_agent, "session_id": session_id}
```

两个细节：

1. `Header()` 不传默认值时表示该参数**必填**（旧写法用 `Header(...)`，Ellipsis 表示必填，两种写法等价），缺失时返回 422。
2. HTTP 头部名习惯用连字符（`user-agent`），Python 变量名用下划线。`Header` 默认自动把下划线转成连字符去匹配，所以 `user_agent` 能正确读取 `User-Agent` 头。

### 3.5 Path / Query 参数校验

类型注解只能约束"是什么类型"，业务上还经常需要范围、长度等约束，这时用 `Path` 和 `Query` 附加校验规则：

```python
from fastapi import Path, Query


@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(ge=1, le=10000, description="商品ID，1到10000之间"),
    q: str | None = Query(default=None, min_length=3, max_length=50),
    page: int = Query(default=1, ge=1),
):
    return {"item_id": item_id, "q": q, "page": page}
```

常用约束参数：

- 数值：`gt`（大于）、`ge`（大于等于）、`lt`、`le`
- 字符串：`min_length`、`max_length`、`pattern`（正则）
- 通用：`description`、`examples`、`deprecated=True`（在文档中标记为已废弃）

这些约束不仅在运行时生效，还会完整呈现在 OpenAPI 文档里。

---

## 四、Pydantic 模型详解

Pydantic 是 FastAPI 的数据基石。Pydantic v2 用 Rust 重写了核心校验逻辑（`pydantic-core`），性能比 v1 提升数倍，API 也有较大变化——本书全部使用 v2 写法。

### 4.1 BaseModel 与字段类型

继承 `BaseModel` 即得到一个数据模型，字段用类型注解声明：

```python
from datetime import datetime
from pydantic import BaseModel


class User(BaseModel):
    id: int
    name: str
    email: str
    age: int | None = None              # 可选字段，默认 None
    is_active: bool = True              # 可选字段，有默认值
    created_at: datetime                # 自动解析 ISO 8601 字符串


user = User(id=1, name="张三", email="zhang@example.com", created_at="2024-01-01T10:00:00")
print(user.created_at)    # 2024-01-01 10:00:00（已是 datetime 对象）
print(user.model_dump())  # v2 的序列化方法，返回 dict
```

Pydantic 做的是**解析（parsing）而非严格类型检查**：传入 `"123"` 给 `int` 字段会被智能转换为 `123`，但传入 `"abc"` 会抛出 `ValidationError`。FastAPI 捕获这个异常并转成 422 响应。

### 4.2 Field：为字段附加约束与元数据

模型字段的校验用 `Field`，而不是上一节的 `Query`/`Path`（后两者用于函数参数）：

```python
from pydantic import BaseModel, Field


class Product(BaseModel):
    name: str = Field(min_length=1, max_length=100, description="商品名称")
    price: float = Field(gt=0, description="价格，必须大于0")
    stock: int = Field(default=0, ge=0)
    tags: list[str] = Field(default_factory=list)   # 可变默认值用 default_factory
```

一个经典陷阱：`tags: list[str] = []` 这样直接写可变默认值是不规范的写法（虽然 Pydantic 内部做了处理不会真正共享，但规范做法永远是用 `default_factory`）。

### 4.3 嵌套模型

模型可以任意嵌套，也可以嵌套标准库类型，Pydantic 会递归校验：

```python
from pydantic import BaseModel, HttpUrl


class Image(BaseModel):
    url: HttpUrl          # 自动校验是否为合法 URL
    name: str


class Item(BaseModel):
    name: str
    price: float
    image: Image | None = None              # 嵌套模型
    images: list[Image] = []                # 嵌套模型列表
    attributes: dict[str, str] = {}         # 键值都是字符串的字典
```

对应请求体 JSON 必须是同样的嵌套结构，任何一层不合法都会被精确定位并在 422 响应中指出路径，例如 `body -> images -> 0 -> url`。

### 4.4 field_validator：自定义校验

内置约束不够用时，用 `@field_validator` 写自定义校验逻辑。Pydantic v2 中它取代了 v1 的 `@validator`：

```python
from pydantic import BaseModel, field_validator


class RegisterForm(BaseModel):
    username: str
    password: str
    password_confirm: str

    @field_validator("username")
    @classmethod
    def username_must_be_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("用户名只能包含字母和数字")
        return v

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("密码长度不能少于8位")
        return v
```

校验器里 `raise ValueError(...)` 会被 Pydantic 收集为校验错误，FastAPI 最终把它转成 422 响应。

跨字段校验（比如比较两次密码）在 v2 中用 `@model_validator`，它在整个模型校验完成后运行：

```python
from pydantic import BaseModel, model_validator


class RegisterForm(BaseModel):
    password: str
    password_confirm: str

    @model_validator(mode="after")
    def passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError("两次输入的密码不一致")
        return self
```

### 4.5 model_config：模型级配置

v2 中用 `model_config`（一个 `ConfigDict`）替代 v1 的内部 `Config` 类：

```python
from pydantic import BaseModel, ConfigDict


class UserORM(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # 支持从 ORM 对象创建模型

    id: int
    name: str


# 从 SQLAlchemy 等 ORM 对象直接构造（读取属性而非字典键）
user_model = UserORM.model_validate(db_user_object)
```

`from_attributes=True`（v1 中叫 `orm_mode = True`）是配合 `response_model` 返回 ORM 对象的关键配置。其他常用配置：

- `str_strip_whitespace=True`：自动去除字符串首尾空白
- `extra="forbid"`：禁止传入模型中未声明的多余字段（默认是忽略）
- `populate_by_name=True`：允许用字段名或别名（alias）两种方式赋值

---

## 五、响应

### 5.1 response_model：过滤与塑形输出

路由函数返回什么就直接序列化什么，是有风险的——比如不小心把 `hashed_password` 字段也返回给前端。`response_model` 的作用就是：**无论函数返回什么，最终输出都先经过这个模型过滤和校验**。

```python
from pydantic import BaseModel


class UserIn(BaseModel):          # 输入模型：包含密码
    username: str
    password: str
    email: str


class UserOut(BaseModel):         # 输出模型：不含密码
    username: str
    email: str


@app.post("/users/", response_model=UserOut)
async def create_user(user: UserIn):
    # 即使这里直接返回了 user（含 password），
    # 响应中也只会保留 UserOut 声明的字段
    return user
```

这套"输入模型 / 输出模型分离"的模式（常命名为 `XxxCreate` / `XxxOut` 或 `XxxRead`）是 FastAPI 项目中最常见的安全实践。`response_model` 还有两个实用参数：`response_model_exclude_unset=True` 只返回请求中显式设置过的字段（适合部分更新的 PATCH 接口）；`response_model_exclude={"field"}` 排除指定字段。

### 5.2 状态码

```python
from fastapi import status


@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(item: Item):
    return item


@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    return None        # 204 表示成功但无响应体
```

建议用 `fastapi.status` 里的常量而不是裸数字，语义清晰且不易写错。常见约定：创建成功 `201`，删除成功 `204`，参数错误 `400/422`，未认证 `401`，无权限 `403`，资源不存在 `404`，服务器错误 `500`。

### 5.3 自定义响应：JSONResponse

直接 `return` 字典时，FastAPI 默认用 `JSONResponse` 序列化。需要定制响应头、状态码或返回非 JSON 内容时，直接返回一个 Response 对象：

```python
from fastapi.responses import HTMLResponse, JSONResponse, PlainTextResponse


@app.get("/legacy/")
async def legacy():
    content = {"code": 0, "data": [], "msg": "ok"}   # 一些老系统约定的包装格式
    return JSONResponse(content=content, status_code=200, headers={"X-Custom": "value"})


@app.get("/health", response_class=PlainTextResponse)
async def health():
    return "ok"
```

注意：一旦直接返回 Response 对象，`response_model` 的过滤就**不会生效**，需要自行保证返回内容安全。

### 5.4 异常处理：HTTPException 与 exception_handler

业务出错时抛出 `HTTPException`，FastAPI 会转成对应状态码的 JSON 响应：

```python
from fastapi import HTTPException


@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id not in fake_db:
        raise HTTPException(status_code=404, detail="商品不存在")
    return fake_db[item_id]
```

`HTTPException` 不是 Python 普通异常，它会被 FastAPI 的内置处理器接住并转换为响应。生产项目通常需要**统一错误格式**，这时用 `@app.exception_handler` 注册自定义处理器：

```python
from fastapi import Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse


@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    # 把 422 校验错误包装成公司统一的错误格式
    return JSONResponse(
        status_code=422,
        content={"code": 10001, "message": "参数校验失败", "errors": exc.errors()},
    )


class BusinessError(Exception):
    """自定义业务异常"""

    def __init__(self, code: int, message: str):
        self.code = code
        self.message = message


@app.exception_handler(BusinessError)
async def business_exception_handler(request: Request, exc: BusinessError):
    return JSONResponse(
        status_code=400,
        content={"code": exc.code, "message": exc.message},
    )
```

这样业务代码里只需 `raise BusinessError(20001, "库存不足")`，错误响应格式全局统一。

---

## 六、依赖注入系统

依赖注入（Dependency Injection）是 FastAPI 最具威力的特性。它的思想是：路由函数不自己创建所需的资源（数据库连接、当前用户、分页参数），而是**声明**需要什么，由框架负责创建并注入。

### 6.1 Depends 的基本用法与执行原理

```python
from fastapi import Depends


async def common_params(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}


@app.get("/items/")
async def read_items(commons: dict = Depends(common_params)):
    return commons
```

当请求到达 `/items/` 时，FastAPI 看到 `Depends(common_params)`，执行流程是：

1. 分析 `common_params` 的签名，发现它需要 `skip` 和 `limit` 两个查询参数；
2. 从请求中解析、校验这些参数；
3. 调用 `common_params(skip, limit)` 得到返回值；
4. 把返回值注入到路由函数的 `commons` 参数；
5. 调用路由函数。

如果依赖自身还声明了 `Depends(...)`，FastAPI 会递归解析，构成一棵**依赖树**，自底向上依次执行。同一个请求中，同一个依赖只会被执行一次，结果会被缓存复用——比如三个依赖都依赖 `get_db`，它们拿到的是同一个数据库会话。

### 6.2 带参数的依赖与类依赖

依赖函数本身可以接收来自请求的参数；更进一步，FastAPI 支持把**类**作为依赖——它会把类当成可调用对象，分析其 `__init__` 的签名来解析参数，然后实例化注入：

```python
from fastapi import Depends, HTTPException


class Pagination:
    """可复用的分页依赖类"""

    def __init__(self, page: int = 1, size: int = 20, max_size: int = 100):
        if size > max_size:
            raise HTTPException(status_code=400, detail=f"size 不能超过 {max_size}")
        self.page = page
        self.size = size
        self.offset = (page - 1) * size


@app.get("/articles/")
async def list_articles(p: Pagination = Depends(Pagination)):
    return {"page": p.page, "size": p.size, "offset": p.offset}
```

请求 `/articles/?page=3&size=15` 时，`page` 和 `size` 自动从查询参数解析并校验，路由里拿到的是组装好的 `Pagination` 对象。类依赖特别适合把一组相关参数和派生逻辑封装成可复用单元。

### 6.3 yield 依赖：资源获取与清理

很多资源需要"使用前获取、用完释放"，比如数据库会话、文件句柄。FastAPI 支持在依赖函数中使用 `yield`：`yield` 之前的代码在请求处理前执行，`yield` 的值注入给路由，`yield` 之后的代码在响应处理完成后执行（类似上下文管理器）：

```python
from sqlalchemy.orm import Session

from database import SessionLocal   # SQLAlchemy 的 session 工厂


def get_db():
    db: Session = SessionLocal()
    try:
        yield db          # 注入给路由使用
    finally:
        db.close()        # 必定执行，释放连接


@app.get("/users/{user_id}")
async def read_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == user_id).first()
```

这就是**数据库会话注入的标准范式**：每个请求拿到独立的 session，请求结束（无论成功还是异常）都保证关闭，连接不会泄漏。清理代码在后台任务执行完之后运行，这是设计上保证后台任务仍能使用依赖资源。如果需要记录路由中抛出的异常，可以在 `yield` 外包 `try/except` 做日志记录后再向上抛出。

依赖注入的实际威力体现在**鉴权**场景：

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    user = decode_token(token)   # 伪代码：解析 JWT
    if user is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="无效的身份凭证",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return user


@app.get("/users/me")
async def read_me(current_user: User = Depends(get_current_user)):
    return current_user
```

任何需要登录的接口只要加一个 `Depends(get_current_user)` 参数即可，鉴权逻辑零重复。

---

## 七、中间件与 CORS

### 7.1 中间件机制

中间件是包裹在所有路由外层的处理函数，请求到达路由**之前**和响应返回客户端**之前**都会经过它，适合处理全局性、横切性的逻辑：日志、计时、鉴权前置、压缩等。

```python
import time

from fastapi import Request


@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)     # 调用后续处理（可能还有别的中间件/路由）
    process_time = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{process_time:.4f}"
    return response
```

`call_next` 之前是"请求阶段"，之后是"响应阶段"。多个中间件按注册顺序**后进先出**（最后注册的最先看到请求），形成一个洋葱模型。

### 7.2 CORS 跨域配置

浏览器有同源策略：前端页面（如 `http://localhost:5173`）请求不同源的后端（`http://localhost:8000`）会被浏览器拦截。后端必须通过 CORS 响应头明确"允许谁跨域"。FastAPI 直接提供 Starlette 的 `CORSMiddleware`：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",       # 前端开发服务器
        "https://www.example.com",     # 生产域名
    ],
    allow_credentials=True,            # 允许携带 Cookie
    allow_methods=["*"],               # 允许所有 HTTP 方法
    allow_headers=["*"],               # 允许所有请求头
)
```

安全要点：**永远不要在生产环境用 `allow_origins=["*"]` 配合 `allow_credentials=True`**（规范上浏览器也会拒绝这种组合），应该显式列出可信域名。中间件要在所有路由注册之前通过 `add_middleware` 添加。

---

## 八、后台任务、文件上传与静态文件

### 8.1 BackgroundTasks：请求返回后再执行的轻量任务

有些操作不该阻塞响应：发邮件、写审计日志、推送通知。如果任务轻量且无需可靠投递保证，用内置的 `BackgroundTasks`：

```python
from fastapi import BackgroundTasks


def send_email(email: str, message: str):
    # 伪代码：发送邮件
    print(f"send to {email}: {message}")


@app.post("/register/")
async def register(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_email, email, "欢迎注册！")
    return {"message": "注册成功，欢迎邮件稍后送达"}
```

响应会先返回给客户端，之后框架才执行 `send_email`。

**边界要清楚**：BackgroundTasks 运行在同一个进程里，进程重启任务就丢了，也不支持重试、定时。订单超时关闭、视频转码这类重任务，应该用 Celery、RQ 等真正的任务队列。

### 8.2 文件上传：UploadFile

文件上传走 `multipart/form-data`，必须先安装 `python-multipart`：

```bash
pip install python-multipart
```

```python
from fastapi import File, UploadFile


@app.post("/upload/")
async def upload_file(file: UploadFile):
    content = await file.read()          # 异步读取全部内容（小文件）
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(content),
    }
```

为什么推荐 `UploadFile` 而不是 `bytes = File(...)`？`UploadFile` 使用**磁盘缓冲**：文件超过内存阈值后会写到临时文件而不是全部驻留内存，大文件上传不会撑爆内存。它提供 `await file.read()`、`await file.seek()`、`await file.write()` 等异步方法，以及同步的 `file.file` 底层文件对象。

大文件的正确处理方式是分块读取写入磁盘：

```python
import aiofiles   # pip install aiofiles


@app.post("/upload-large/")
async def upload_large(file: UploadFile):
    async with aiofiles.open(f"uploads/{file.filename}", "wb") as out:
        while chunk := await file.read(1024 * 1024):   # 每次读 1MB
            await out.write(chunk)
    return {"filename": file.filename}
```

多文件上传把参数声明为 `list[UploadFile]` 即可。

### 8.3 静态文件

挂载一个目录提供静态文件服务（上传后的文件回显、前端构建产物等）：

```python
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="static"), name="static")
```

之后 `static/avatar.png` 就可以通过 `http://localhost:8000/static/avatar.png` 访问。生产环境大流量场景静态文件通常交给 Nginx 或对象存储/CDN，应用内挂载适合中小项目。

---

## 九、项目结构组织

把所有接口写在一个 `main.py` 里，超过几百行就无法维护了。FastAPI 提供 `APIRouter` 做模块化拆分。

### 9.1 APIRouter 拆分路由

`APIRouter` 相当于一个"迷你 FastAPI"，只负责收集路由，最后挂载到主应用：

```python
# app/routers/users.py
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/users",                              # 该模块所有路由的统一前缀
    tags=["users"],                               # 文档中的分组标签
    dependencies=[Depends(get_current_user)],     # 模块级依赖：所有路由都需登录
)


@router.get("/")
async def list_users():
    return [{"name": "张三"}]


@router.get("/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}
```

```python
# app/main.py
from fastapi import FastAPI

from app.routers import items, users

app = FastAPI(title="我的应用")
app.include_router(users.router)
app.include_router(items.router)
```

这样 `/users/` 和 `/users/{user_id}` 就挂到了主应用上，文档里也会按 tag 分组展示。

### 9.2 推荐的多文件项目目录结构

一个可商用的中大型项目结构示范：

```text
myproject/
├── app/
│   ├── main.py                # 应用入口：创建 app、注册中间件和路由
│   ├── config.py              # 配置管理（pydantic-settings）
│   ├── database.py            # 数据库引擎与会话工厂
│   ├── dependencies.py        # 公共依赖（get_db、get_current_user）
│   ├── models/                # SQLAlchemy ORM 模型（数据库表结构）
│   │   ├── __init__.py
│   │   └── user.py
│   ├── schemas/               # Pydantic 模型（请求/响应的数据结构）
│   │   ├── __init__.py
│   │   └── user.py            # UserCreate / UserOut 等
│   ├── routers/               # 路由层：只负责参数解析和调用 service
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── items.py
│   └── services/              # 业务逻辑层：复杂业务从路由中剥离
│       ├── __init__.py
│       └── user_service.py
├── tests/                     # 测试
├── .env                       # 环境变量（不要提交到 git）
├── .env.example               # 环境变量模板（提交到 git）
└── requirements.txt
```

分层的核心原则：**routers 只做 HTTP 的事，services 做业务逻辑，models 管表结构，schemas 管数据传输格式**。ORM 模型和 Pydantic 模型分开存放是社区惯例——前者映射数据库，后者约束 API 边界，两者的演化节奏不同。

### 9.3 pydantic-settings 配置管理

配置（数据库地址、密钥、开关）不应该硬编码在代码里，而应该来自环境变量。`pydantic-settings` 把环境变量也变成类型安全的模型：

```bash
pip install pydantic-settings
```

```python
# app/config.py
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    app_name: str = "MyApp"
    debug: bool = False
    database_url: str
    secret_key: str
    allowed_origins: list[str] = []


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`.env` 文件：

```bash
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
SECRET_KEY=change-me-in-production
ALLOWED_ORIGINS='["http://localhost:5173"]'
```

启动时 `Settings()` 自动读取环境变量和 `.env` 文件并校验类型：缺了 `secret_key` 会在启动时直接报错，而不是运行到一半才暴露，这就是"配置即校验"。`@lru_cache` 保证整个进程只解析一次。配合依赖注入使用：

```python
from typing import Annotated

from fastapi import Depends

SettingsDep = Annotated[Settings, Depends(get_settings)]


@app.get("/info")
async def info(settings: SettingsDep):
    return {"app_name": settings.app_name, "debug": settings.debug}
```

`Annotated[类型, Depends(...)]` 是 FastAPI 推荐的写法：它把依赖声明和类型注解合并，函数签名更干净，也对类型检查器更友好。

---

## 十、异步编程

### 10.1 事件循环原理

理解 `async/await` 的关键是理解**事件循环（event loop）**。

传统同步代码是"一条道走到黑"：执行到 `db.query(...)`，线程就阻塞住，直到数据库返回结果。等待期间 CPU 完全空闲，但线程什么别的也干不了。高并发下只能开大量线程，而线程切换和内存开销都很大。

异步模型只有一个线程在跑一个事件循环。事件循环维护一个任务队列，周而复始地：

1. 从队列取一个就绪的任务执行；
2. 任务执行到 `await`（一个 I/O 操作，比如数据库查询）时，任务**主动挂起**，把控制权交还事件循环，并登记"这个 I/O 完成后叫醒我"；
3. 事件循环立刻去执行其他就绪任务；
4. 某个 I/O 完成了，操作系统通知事件循环，对应任务被标记为就绪，等待下一次调度继续执行。

所以异步的本质是：**用一个线程的"等待时间"去干别的活**。它不减少任何一次 I/O 的耗时，但极大地提高了单线程的吞吐。代价是所有 I/O 必须用支持异步的库（`asyncpg`、`aiomysql`、`redis.asyncio`、`httpx`），否则一个同步阻塞调用会卡住整个事件循环，所有请求一起卡死。

`async def` 定义一个协程函数，调用它只创建协程对象，并不执行；`await` 才是"执行并等待结果"的意思，且 `await` 只能出现在 `async def` 内部。

### 10.2 async def vs def 路由的区别

FastAPI 同时支持两种路由函数，但处理方式完全不同：

```python
import time

import httpx


@app.get("/async-io")
async def async_route():
    # 使用异步库做 I/O：事件循环在等待时可以处理其他请求
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://api.example.com/data")
    return resp.json()


@app.get("/sync-io")
def sync_route():
    # 使用同步库：FastAPI 会把它放到线程池执行，不阻塞事件循环
    time.sleep(1)     # 模拟同步阻塞操作
    return {"ok": True}
```

关键机制：

- `async def` 路由：直接在当前事件循环中运行。如果里面出现任何**同步阻塞调用**（`time.sleep`、同步的 `requests`、同步的数据库驱动），整个事件循环被卡死，所有并发请求全部排队。
- 普通 `def` 路由：FastAPI 检测到你定义的是普通函数，会自动把它丢到**外部线程池**（默认 40 个线程）中运行，阻塞的只是池中的一个线程，事件循环不受影响。代价是线程切换开销和并发上限。

### 10.3 什么时候该用异步

决策规则其实很简单：

- **路由内全是异步 I/O 操作**（异步数据库驱动、`httpx.AsyncClient`、`redis.asyncio`）→ 用 `async def`，这是性能最优的姿势。
- **路由内有同步阻塞调用**（同步 ORM 如 SQLAlchemy 默认用法、`requests`、重型同步 SDK）→ 用普通 `def`，让线程池兜底。硬套 `async def` 反而把事件循环卡死，比同步框架还慢。
- **CPU 密集型操作**（图像处理、大数计算）→ 两者都救不了你，异步解决的是 I/O 等待，不是计算。应丢给进程池（`ProcessPoolExecutor`）或任务队列。

一句话总结：**async 不是"更快"，而是"等待时不占座"**。I/O 越密集、并发越高，异步收益越大。

---

## 小结

- FastAPI = Starlette（Web 能力）+ Pydantic（数据能力），运行在 uvicorn 等 ASGI 服务器之上，ASGI 的异步事件模型是它高性能的根基。
- 类型注解是 FastAPI 的核心：同一份注解同时驱动参数校验、类型转换和 OpenAPI 文档生成。
- Pydantic v2 用 `Field` 加约束、`@field_validator`/`@model_validator` 自定义校验、`model_config` 配置模型行为。
- 永远用 `response_model` 过滤输出字段，防止敏感数据泄漏；用 `HTTPException` 和 `exception_handler` 构建统一的错误响应。
- 依赖注入（`Depends`）是代码复用的核心机制，`yield` 依赖是管理数据库会话等资源的标准范式。
- 中间件处理横切逻辑，CORS 通过 `CORSMiddleware` 显式配置可信源。
- 轻量善后工作用 `BackgroundTasks`，文件上传用磁盘缓冲的 `UploadFile`，重任务交给 Celery 等任务队列。
- 项目变大后用 `APIRouter` 模块化，用 `pydantic-settings` 做类型安全的配置管理。
- 路由用 `async def` 还是 `def` 取决于内部 I/O 是否为异步库；同步阻塞代码放进 `async def` 会卡死整个事件循环。

---

## 常见问题

### 1. 接口返回 422 是什么意思？和 400 有什么区别？

422 是 FastAPI 对**请求参数校验失败**（Pydantic `ValidationError`）的默认响应码，响应体里会详细列出哪个字段、因为什么规则校验失败，例如 `body -> price: Input should be greater than 0`。400 是更通用的"请求本身有问题"。实践中：让 FastAPI 自动处理参数格式问题（422），业务规则错误（如"余额不足"）用 `HTTPException(status_code=400, detail=...)` 或自定义异常主动抛出。

### 2. 为什么我的 `async def` 接口并发性能还不如预期？（面试高频）

几乎可以肯定是在 `async def` 里写了同步阻塞调用，最常见的是用了同步的数据库驱动（如 `psycopg2`、`pymysql`）或 `requests`。这些调用不会 `await`，会阻塞整个事件循环，导致单进程退化为串行处理。解决方案二选一：换成异步驱动（`asyncpg`、`aiomysql`、`httpx`）；或者把路由改成普通 `def`，让 FastAPI 用线程池执行。可以用一个简单实验验证：在 `async def` 路由里写 `time.sleep(5)`，同时发两个请求，会发现第二个请求等了将近 10 秒。

### 3. 输入模型和输出模型为什么要分开？直接复用一个不行吗？

核心原因是安全。用户注册时提交的模型包含 `password`（或 `hashed_password`），如果直接把这个模型返回，密码字段就泄漏给了前端；如果返回的是数据库 ORM 对象，还可能带出内部字段（如 `role`、`is_deleted`）。`response_model=UserOut` 相当于一道强制过滤器，无论业务函数返回什么，输出都被裁剪到白名单字段。其次，输入和输出的演化节奏本就不同（创建时不需要 `id`，读取时必须有 `id`），分开定义更符合单一职责。

### 4. 依赖注入中的 `yield` 依赖，清理代码什么时候执行？如果路由抛异常了还会执行吗？

`yield` 之后的清理代码在请求处理流程结束时执行（后台任务也完成之后）。即使路由函数抛出异常，只要异常被 exception handler 接住并产生了响应，清理代码依然会执行——这正是它适合做数据库会话管理的原因：`finally: db.close()` 保证任何路径下连接都会归还。需要留意的是，FastAPI 较新版本调整了 `yield` 依赖的退出时机（改为响应发送前完成清理），使用时以当前版本文档为准，但"必然执行清理"这一保证是不变的。

### 5. 生产环境部署 FastAPI 有什么标准做法？

典型的生产部署链路是：Nginx（反向代理、HTTPS 终止、静态文件）→ 多进程 ASGI 服务器 → FastAPI 应用。多进程通常用 `uvicorn --workers N`（N 一般取 CPU 核数）或 `gunicorn -w N -k uvicorn.workers.UvicornWorker`。另外几个要点：关闭或保护 `/docs`（`docs_url=None` 或加鉴权）；`--reload` 只能用于开发；密钥走环境变量而非代码；配合 Docker 容器化时通常一个容器跑一个进程组，由编排系统管扩缩容。
