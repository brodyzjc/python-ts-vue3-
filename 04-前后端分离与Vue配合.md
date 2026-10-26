# 前后端分离：FastAPI + Vue3 如何配合

## 本章导读

- 前后端分离架构的本质：职责切分 + HTTP API 契约，为什么现代项目几乎都这么玩
- 一次请求的完整生命周期：浏览器 → Nginx → 前端静态资源 / `/api` 反向代理到 FastAPI
- 跨域问题从原理到落地：同源策略、简单请求与预检请求（OPTIONS）、`Access-Control-Allow-*` 系列响应头、FastAPI `CORSMiddleware` 配置与高频坑
- 前端请求封装的标准姿势：axios 实例、baseURL、请求拦截器自动携带 Authorization、响应拦截器统一处理业务 code 与 401
- 开发环境联调：Vite dev server 的 proxy 配置与原理，为什么代理之后浏览器不再报跨域
- 登录鉴权全流程走通：登录接口 → 后端签发 JWT → 前端存储 → 请求携带 token → 401 无感刷新 token 的完整时序，前后端关键代码全给
- 生产环境部署配合：前端 build 产物交给 Nginx 托管、`/api` 前缀代理到 uvicorn、history 路由与 WebSocket 的代理注意点
- 联调排查清单：CORS 报错怎么读、404 / 405 / 422 各自代表什么、token 失效类问题怎么定位

## 1. 前后端分离架构原理

### 1.1 从"不分离"说起

要理解分离的价值，先看传统的不分离模式长什么样。早期 Web 应用（典型如服务端渲染的 Django/JSP/PHP 项目）里，**HTML 是由后端模板引擎拼出来的**：

```
浏览器 ──GET /user/list──> 后端 ──查数据库──> 模板引擎渲染 HTML ──> 返回完整页面
```

后端既要处理业务逻辑、数据库，又要负责页面长什么样。前端写好的 HTML 原型要交给后端"套模板"，改一个按钮文案可能要动后端代码、重启服务。前后端代码揉在一个仓库、一个进程里，互相等待、互相拖累。

### 1.2 前后端分离到底是什么

前后端分离的核心就一句话：**后端不再输出 HTML，只输出数据（JSON）；页面完全由浏览器里的前端代码渲染**。

- 前端（Vue3）：一个独立的静态站点——HTML + CSS + JS。浏览器加载它之后，由 JS 在浏览器里渲染页面、管理路由跳转、通过 HTTP 请求向后端要数据。
- 后端（FastAPI）：一个纯 API 服务——只接收 HTTP 请求、返回 JSON，完全不关心页面长什么样。

两端之间唯一的联系是一份**接口契约**：URL 是什么、方法是什么、请求参数长什么样、返回的 JSON 结构是什么。只要契约不变，两端可以独立开发、独立部署、独立扩缩容。

```
┌─────────┐   1. GET /            ┌───────────┐
│ 浏览器   │ ────────────────────> │   Nginx   │──> 返回 index.html + JS/CSS（前端静态资源）
│         │ <──────────────────── │           │
│         │                       │           │
│         │   2. GET /api/users   │           │──> 转发给 FastAPI（127.0.0.1:8000）
│         │ ────────────────────> │           │<── 返回 JSON
│         │ <──────────────────── │           │
└─────────┘   3. JS 拿到 JSON 渲染页面
```

注意时序：**浏览器先拿到的是前端应用本身（静态资源），再由前端应用去调 API 拿数据**。这是分离架构与模板渲染最本质的区别——数据和页面走了两条完全不同的链路。

### 1.3 为什么要分离

分离不是赶时髦，它解决了几个真实的工程问题：

1. **并行开发**：契约定好之后，前端可以用 mock 数据开发，后端专注业务，不用互相等待。联调只是最后一步的"对表"。
2. **一套 API 多端复用**：同一个 FastAPI 服务可以同时给 Web、App、小程序、第三方开放接口供数。模板渲染时代，每多一种终端就要重写一套输出层。
3. **独立部署与扩容**：前端是静态文件，扔到 Nginx 或 CDN 上，几乎不耗服务器算力；后端按接口压力独立扩容。更新前端页面不需要重启后端。
4. **技术栈自由演进**：前端从 Vue2 升到 Vue3、后端从 Flask 换到 FastAPI，只要契约不变，互不影响。

代价也要说清楚：跨域问题、鉴权要自己管（没有服务端 session 模板那种"天然登录态"）、首屏多一次请求往返、SEO 需要额外处理。本章后面几节就是把这几个代价逐个解决掉。

### 1.4 一次请求的完整生命周期

以生产环境的标准形态（Nginx 在最前面）为例，把两类请求各走一遍。

**第一类：静态资源请求**（打开页面、加载 JS/CSS/图片）

```
用户在地址栏输入 https://example.com/ 并回车
  │
  ├─ 1. DNS 解析 example.com → 服务器公网 IP
  ├─ 2. 浏览器与服务器 443 端口建立 TCP + TLS 连接
  ├─ 3. 请求 GET / 到达 Nginx
  ├─ 4. Nginx 匹配 location /，从磁盘 /var/www/myapp/dist/ 读出 index.html 返回
  ├─ 5. 浏览器解析 HTML，发现引用了 /assets/index-a8f3c2.js，再发请求
  │      Nginx 命中强缓存配置（hash 文件名），直接从磁盘返回并附带一年的缓存头
  └─ 6. JS 执行，Vue 应用挂载，页面骨架渲染出来
```

这条链路**完全不经过 FastAPI**。前端静态资源由 Nginx 直接服务，性能最好、后端零负担。

**第二类：API 请求**（页面加载后拉数据、用户操作触发）

```
JS 执行 axios.get('/api/users')
  │
  ├─ 1. 浏览器发出 GET https://example.com/api/users（与页面同源，无跨域）
  ├─ 2. 请求到达 Nginx，匹配 location /api/
  ├─ 3. Nginx 作为反向代理，把请求转发到 http://127.0.0.1:8000/api/users
  │      （同时补上 X-Real-IP / X-Forwarded-For 等头，告诉后端真实客户端是谁）
  ├─ 4. uvicorn 收到请求 → FastAPI 路由匹配 → 依赖注入（鉴权）→ 视图函数
  │      → 查数据库 → Pydantic 序列化 → 返回 JSON
  ├─ 5. Nginx 把响应原样带回给浏览器
  └─ 6. axios 拦截器处理响应 → 业务代码拿到数据 → Vue 响应式更新界面
```

关键点有三个：

- **对浏览器来说只有一个源头**（example.com），Nginx 把"静态资源"和"API"两个完全不同的后端藏在同一域名下，这直接消灭了生产环境的跨域问题（第 2 节会展开）。
- **Nginx 是唯一的入口**。uvicorn 只监听 `127.0.0.1:8000`，不直接暴露给公网，安全上少一层攻击面。
- **前端与后端的"配合"本质上是三个约定**：URL 前缀约定（`/api`）、数据格式约定（JSON 结构）、鉴权约定（`Authorization: Bearer <token>`）。本章剩下的内容就是把这三个约定逐个落地。

### 1.5 职责边界一览

| 关注点 | 前端（Vue3） | 后端（FastAPI） | 边界（Nginx） |
|--------|-------------|-----------------|---------------|
| 页面渲染 | 负责 | 不管 | 不管 |
| 路由 | 前端路由（Vue Router） | API 路由 | URL 前缀分发 |
| 数据 | 请求并展示 | 校验、查询、返回 | 不管 |
| 鉴权 | 存 token、带 token、处理 401 | 签发与校验 token | 不管 |
| 跨域 | 不需要处理（生产同源） | 开发期 CORSMiddleware | 生产用同源部署消灭跨域 |

## 2. 跨域问题原理与解决

跨域是前后端分离联调时遇到的第一个拦路虎。大多数人只会"加个 CORS 中间件"，但报错一变就懵。这一节把机制讲透，之后看到任何 CORS 报错你都知道它在说什么。

### 2.1 同源策略：浏览器的安全底线

**同源策略（Same-Origin Policy）** 是浏览器的核心安全机制：一个页面的脚本只能自由访问与自身**同源**的资源。"源（origin）"由三部分组成，**三者全部相同才算同源**：

- 协议（http / https）
- 域名（example.com / api.example.com / localhost）
- 端口（:5173 / :8000，http 默认 80、https 默认 443）

对照表（页面源是 `http://localhost:5173`）：

| 请求目标 | 是否同源 | 原因 |
|----------|----------|------|
| `http://localhost:5173/assets/app.js` | 同源 | 三者全同 |
| `http://localhost:8000/api/users` | 跨域 | 端口不同（5173 vs 8000） |
| `http://127.0.0.1:5173/api` | 跨域 | 域名不同（localhost 与 127.0.0.1 算不同源！） |
| `https://localhost:5173/api` | 跨域 | 协议不同 |

为什么要这么严格？设想没有同源策略：你登录了网银（cookie 里带着会话），又随手点开一个恶意网站，恶意网站的 JS 就能以你的身份向网银发请求转账——因为浏览器会自动带上网银的 cookie。同源策略把"别的源的脚本读不到我这个源的响应"作为默认底线，跨站请求必须得到目标服务器**显式许可**才行。

### 2.2 一个关键事实：跨域时请求其实发出去了

很多人以为跨域是"浏览器把请求拦截了没发出去"。错。**大多数跨域请求是正常发出、服务器正常处理并返回了的**，只是浏览器在收到响应后检查"服务器有没有许可我读取"，没有许可就把响应**对 JS 隐藏**（响应其实已经到了浏览器），并在控制台报 CORS 错误。

这解释了三个常见困惑：

- 为什么后端日志里明明收到了请求、返回了 200，前端却报跨域错？——因为拦在"读取响应"这一步，不在"发送请求"这一步。
- 为什么跨域的 POST 接口居然真的把数据写进数据库了？——简单请求（见下节）直接发出并执行了。
- 为什么用 Postman / curl 调同一个接口完全正常？——**CORS 是浏览器独有的安全机制**，非浏览器客户端根本不检查。这条事实也是排查利器：curl 能通、浏览器报错，问题一定在 CORS 配置上。

### 2.3 简单请求 vs 预检请求（OPTIONS）

浏览器把跨域请求分成两类，待遇完全不同。

**简单请求（Simple Request）**：同时满足以下全部条件——

1. 方法是 `GET`、`HEAD`、`POST` 之一；
2. 请求头只含浏览器自动设置的头和少数安全头（`Accept`、`Accept-Language`、`Content-Language`、`Content-Type` 等）；
3. `Content-Type` 只能是 `text/plain`、`application/x-www-form-urlencoded`、`multipart/form-data` 三者之一。

简单请求的处理方式：**直接发出真实请求**，请求头里带一个 `Origin: http://localhost:5173` 标明来源。服务器响应里如果包含 `Access-Control-Allow-Origin` 且匹配该来源，浏览器放行；否则响应被隐藏、控制台报错。

**预检请求（Preflight）**：只要不满足简单请求的任意一条，浏览器会**先发一个 `OPTIONS` 方法的预检请求**，问服务器"我打算这么发，你允许吗？"，得到肯定答复后才发真实请求。

现代前后端分离项目几乎**每个请求都会触发预检**，因为两条都踩了：

- 发 JSON 数据：`Content-Type: application/json` 不在三个安全值里；
- 带 token：自定义头 `Authorization: Bearer xxx` 不在安全头列表里。

一次带 token 的 POST 请求的完整对话：

```
── 预检 ────────────────────────────────────────────
浏览器 → OPTIONS /api/users HTTP/1.1
         Origin: http://localhost:5173
         Access-Control-Request-Method: POST
         Access-Control-Request-Headers: authorization, content-type

服务器 ← HTTP/1.1 204 No Content
         Access-Control-Allow-Origin: http://localhost:5173
         Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
         Access-Control-Allow-Headers: authorization, content-type
         Access-Control-Max-Age: 600

── 预检通过，真实请求 ───────────────────────────────
浏览器 → POST /api/users HTTP/1.1
         Origin: http://localhost:5173
         Authorization: Bearer eyJhbGciOi...
         Content-Type: application/json
         {"name": "张三"}

服务器 ← HTTP/1.1 200 OK
         Access-Control-Allow-Origin: http://localhost:5173
         {"code": 0, "data": {...}}
```

注意预检请求**不携带业务数据、不携带 Authorization**（它只是个"询问"），所以后端不能在 OPTIONS 上要求登录。`Access-Control-Max-Age: 600` 表示预检结果缓存 600 秒，期间同样的请求不再重复预检——线上接口如果 OPTIONS 请求特别多，可以把 max_age 调大（如 86400）。

### 2.4 `Access-Control-Allow-*` 响应头逐个讲

| 响应头 | 作用 | 出现在 |
|--------|------|--------|
| `Access-Control-Allow-Origin` | 允许哪些源读取响应。可以是具体源 `http://localhost:5173`，或 `*`（任意源） | 预检 + 真实响应 |
| `Access-Control-Allow-Methods` | 允许的方法列表 | 预检响应 |
| `Access-Control-Allow-Headers` | 允许携带的自定义请求头 | 预检响应 |
| `Access-Control-Allow-Credentials` | 为 `true` 时允许浏览器携带 cookie 等凭据 | 预检 + 真实响应 |
| `Access-Control-Max-Age` | 预检结果缓存秒数 | 预检响应 |
| `Access-Control-Expose-Headers` | 允许 JS 读取的自定义响应头（默认 JS 只能读到少数安全响应头，想在前端读 `Content-Disposition` 下载文件名之类必须暴露） | 真实响应 |

### 2.5 FastAPI 配置 CORSMiddleware

FastAPI（底层 Starlette）内置了 CORS 中间件，开发环境的典型配置：

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    # 允许的源：开发期写前端的完整源（协议+域名+端口），一个都不能错
    allow_origins=[
        "http://localhost:5173",
        "http://127.0.0.1:5173",
    ],
    # 允许携带 cookie 等凭据（用 cookie 存 token 才需要；用 Authorization 头不需要）
    allow_credentials=True,
    # 允许的方法；["*"] 表示全部（含 OPTIONS）
    allow_methods=["*"],
    # 允许的请求头；["*"] 表示全部（含 authorization、content-type）
    allow_headers=["*"],
    # 预检结果缓存 10 分钟，减少 OPTIONS 请求数量
    max_age=600,
)
```

中间件的原理：它拦截进入应用的每个请求——如果是带 `Origin` 和 `Access-Control-Request-Method` 的 OPTIONS 请求（预检），直接构造 204 响应返回，**根本不会进入你的路由**；否则放行请求，并在响应上补上 `Access-Control-Allow-Origin` 等头。这也解释了为什么 CORS 配置漏了的时候，你会看到 OPTIONS 返回 400/405——预检到达路由层，而你的路由只注册了 GET/POST。

### 2.6 常见坑（每个都值一次加班）

**坑 1：`allow_origins=["*"]` 与 `allow_credentials=True` 不能同时用**

规范规定：当请求携带凭据（cookie）时，`Access-Control-Allow-Origin` **不能是通配符 `*`**，必须是具体源。浏览器会直接拒绝。写法上要二选一：

```python
# 用 Authorization 头传 token（推荐）：不需要 credentials，可以用 *
allow_origins=["*"], allow_credentials=False
# 用 cookie 传 token：必须枚举具体源
allow_origins=["http://localhost:5173"], allow_credentials=True
```

顺带说明：把 token 放 `Authorization` 头而不是 cookie，还有一个巨大好处——**cookie 会被浏览器自动携带从而受 CSRF 攻击威胁，Authorization 头不会自动携带，天然免疫 CSRF**。本章第 5 节的方案用的就是 Authorization 头。

**坑 2：源字符串末尾多了一个斜杠**

`allow_origins=["http://localhost:5173/"]`（结尾有 `/`）永远匹配不上，因为浏览器发的 `Origin` 头是 `http://localhost:5173`，**没有结尾斜杠**。同样，`localhost` 和 `127.0.0.1` 是两个不同的源，前端用哪个地址访问就配哪个，或者两个都配。

**坑 3：中间件顺序导致异常响应缺 CORS 头**

Starlette 的中间件是"后添加的先执行"（洋葱模型，后包的在外层）。如果某个更早执行（更外层）的中间件或异常处理器直接返回了响应、没有经过 CORSMiddleware，这个响应就没有 CORS 头，浏览器报跨域错——而真实原因（比如 500）被掩盖。**CORSMiddleware 应该尽量晚添加（贴近应用），让鉴权失败、500 等所有响应都能经过它**。诊断技巧：打开 DevTools Network 面板看那个"跨域失败"的响应到底返回了什么状态码和 body，多半另有隐情。

**坑 4：生产环境在 Nginx 又加了一遍 CORS 头**

后端加了 CORS，Nginx 又 `add_header Access-Control-Allow-Origin *;`，响应里出现**两个** `Access-Control-Allow-Origin` 头，浏览器同样拒绝。同源部署（第 6 节）时根本不需要任何 CORS 头，两端都别加。

**坑 5：前端 fetch/axios 的 baseURL 里协议域名端口写错**

前端用 `http://localhost:5173` 访问页面，代码里 baseURL 写成 `http://127.0.0.1:5173`——端口和页面不同源都要走 CORS；写错端口直接连不上。统一用环境变量管理（第 4 节）。

## 3. 前端调接口的封装

### 3.1 为什么不能裸用 axios

在组件里随手写 `axios.get('http://localhost:8000/api/users')`，三五个接口还行，接口一多就会失控：

- 后端地址散落在几十个文件里，换环境全靠全局替换；
- 每个请求都要手动塞 `Authorization` 头，漏一个就 401；
- 后端约定 `{code, data, message}` 结构，每个调用点都重复写 `if (res.data.code !== 0)`；
- token 过期后所有页面一起炸，没有统一处理。

标准做法是**封装一个请求模块**：一个全局 axios 实例 + 请求/响应拦截器，把所有横切逻辑收拢到一处。这也是面试官问"你们项目的 axios 是怎么封装的"时想听到的答案。

### 3.2 创建 axios 实例与 baseURL

```ts
// src/utils/request.ts
import axios from 'axios'

// 创建实例：与全局 axios 隔离，可以配置自己的 baseURL 和拦截器
const request = axios.create({
  // 开发环境走 Vite 代理（第 4 节），生产环境由 Nginx 分发，统一用相对路径 /api
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api',
  timeout: 15000, // 15 秒超时，防止请求永远挂起
})

export default request
```

几个设计要点：

- **`baseURL` 用相对路径 `/api` 而不是写死 `http://localhost:8000`**：开发环境由 Vite dev server 代理到后端（第 4 节），生产环境由 Nginx 代理（第 6 节），前端代码一份配置两处通用，并且天然同源无跨域。
- **`import.meta.env.VITE_API_BASE_URL`** 是 Vite 的环境变量读取方式（`.env` 文件里以 `VITE_` 开头的变量才会被注入到前端代码）。
- 超时必须设置。HTTP 请求没有默认超时，后端或网络卡死时 Promise 永远 pending，页面转圈到天荒地老。

### 3.3 请求拦截器：自动携带 Authorization

```ts
// src/utils/request.ts（续）
request.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token')
  if (token) {
    // config.headers 必然存在；用 set 方法避免类型问题
    config.headers.set('Authorization', `Bearer ${token}`)
  }
  return config
})
```

原理：拦截器是在请求真正发出前经过的"管道"，在这里统一注入 token，业务代码就完全不需要关心鉴权头。token 的存储用 `localStorage`（刷新页面不丢失），登出时清除。需要说明的安全权衡：`localStorage` 里的 token 可被页面内任意 JS（包括 XSS 注入的脚本）读取，所以必须严防 XSS（Vue 默认对插值做转义，危险的是 `v-html` 渲染不可信内容）。相比之下 cookie 方案防读但不防 CSRF——两者取舍不同，团队内统一即可。

### 3.4 响应拦截器：统一处理业务 code 与网络错误

先约定后端的统一响应格式（契约的一部分）：

```json
{ "code": 0, "data": { ... }, "message": "ok" }
```

`code` 为 0 表示成功，非 0 是业务错误（如 1001 = 余额不足）；HTTP 状态码表达传输层语义（401 未登录、404 不存在、422 参数错误），`code` 表达业务语义，两者分工。

```ts
// src/utils/request.ts（续）
import { ElMessage } from 'element-plus' // 以 Element Plus 为例，换成你的 UI 库

request.interceptors.response.use(
  // HTTP 2xx：检查业务 code
  (response) => {
    const body = response.data
    if (body.code === 0) {
      return body.data // 关键：把 data 解包出来，业务代码直接拿到数据本体
    }
    // 业务错误：统一提示，并 reject 让调用方能 catch 到
    ElMessage.error(body.message || '请求失败')
    return Promise.reject(new Error(body.message))
  },
  // HTTP 非 2xx 或网络错误
  async (error) => {
    const status = error.response?.status
    if (status === 401) {
      // token 过期/无效：尝试无感刷新，详见第 5 节
      return handle401(error)
    }
    if (status === 403) {
      ElMessage.error('没有权限执行此操作')
    } else if (status === 422) {
      // FastAPI 参数校验失败的细节在 detail 数组里，见第 7 节
      ElMessage.error(error.response.data?.detail?.[0]?.msg || '参数错误')
    } else if (error.code === 'ECONNABORTED') {
      ElMessage.error('请求超时，请重试')
    } else if (!error.response) {
      ElMessage.error('网络异常，请检查网络连接')
    } else {
      ElMessage.error(`请求失败（${status}）`)
    }
    return Promise.reject(error)
  },
)
```

拦截器里做了三件事，业务层因此非常干净：

1. **解包**：`return body.data` 让调用处直接拿到数据，不用每个地方都写 `.data.data`；
2. **统一提示**：业务错误和 HTTP 错误的提示集中在拦截器，页面不用重复写 message 弹窗；
3. **错误分类**：401 走刷新流程、422 提取 FastAPI 的校验信息、超时与断网单独提示。

### 3.5 业务代码里的使用方式

封装好之后，再包一层"按资源组织的 API 函数"，组件里就是一行的事：

```ts
// src/api/user.ts
import request from '@/utils/request'

export interface User {
  id: number
  name: string
  email: string
}

export const getUsers = () => request.get<unknown, User[]>('/users')
export const createUser = (data: { name: string; email: string }) =>
  request.post<unknown, User>('/users', data)
```

```vue
<!-- src/views/UserList.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { getUsers, type User } from '@/api/user'

const users = ref<User[]>([])
const loading = ref(false)

onMounted(async () => {
  loading.value = true
  try {
    users.value = await getUsers() // 拿到的就是 User[]，code/loading/token 全被封装层消化
  } catch {
    // 错误提示拦截器已经弹了，这里按需做兜底（比如清空数据）
  } finally {
    loading.value = false
  }
})
</script>
```

类型注解 `request.get<unknown, User[]>` 的第二个泛型参数是"解包后"的类型，对应拦截器里 `return body.data` 的行为——这是 axios 类型设计里配合响应拦截器的标准写法。

## 4. 开发环境联调

### 4.1 问题场景

开发时的典型状态：前端 Vite dev server 跑在 `http://localhost:5173`，后端 uvicorn 跑在 `http://127.0.0.1:8000`。两个端口，浏览器一调接口就是跨域。

前面第 2 节给出的解法是后端配 `CORSMiddleware`——能用，但有几个不舒服的地方：后端要为开发环境维护一份源白名单；团队成员的端口不一致时要互相迁就；如果想用 cookie 方案，`SameSite` 在跨站场景下还有一堆额外限制。更优雅的解法是**让开发环境也像生产环境一样同源**——Vite 的 dev server proxy 就是干这个的。

### 4.2 Vite proxy 配置

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      // 凡是以 /api 开头的请求，由 dev server 转发给后端
      '/api': {
        target: 'http://127.0.0.1:8000', // 后端地址
        changeOrigin: true,              // 把 Host 头改成 target 的域名
        // 注意：这里不需要 rewrite，因为后端路由本身就带 /api 前缀
      },
    },
  },
})
```

改完 `vite.config.ts` 需要**重启 dev server** 才生效。之后前端的 `request.get('/users')`（baseURL 是 `/api`）实际发向 `http://localhost:5173/api/users`，由 dev server 转发到 `http://127.0.0.1:8000/api/users`。

### 4.3 原理：为什么代理之后就没有跨域了

这是理解整个机制的关键。回顾第 2 节的结论：**CORS 是浏览器强加的限制，服务器与服务器之间的 HTTP 通信根本不受同源策略约束**。

代理后的请求链路：

```
浏览器 ──http://localhost:5173/api/users──> Vite dev server ──http://127.0.0.1:8000/api/users──> FastAPI
        （与页面同源，浏览器不检查）                                  （服务器之间通信，无 CORS 概念）
```

对浏览器来说，请求的目标就是 `localhost:5173`——和页面**完全同源**，同源策略压根不触发。跨不跨域这件事在浏览器眼里不存在了。dev server 收到请求后，在**服务器侧**（Node.js 进程）向后端发起一个全新的 HTTP 请求，拿到响应后再回给浏览器。这一步是后端到后端，没有任何限制。

所以"代理解决跨域"的本质是：**用服务器转发把跨域请求伪装成同源请求**。生产环境的 Nginx `/api` 代理（第 6 节）是同一件事，只是换了个执行者。这也是为什么生产环境通常不需要配 CORS——不是跨域被"解决"了，而是跨域被"避免"了。

### 4.4 rewrite：什么时候需要改写路径

proxy 配置里经常能看到 `rewrite`：

```ts
server: {
  proxy: {
    '/api': {
      target: 'http://127.0.0.1:8000',
      changeOrigin: true,
      // 把 /api/users 改写为 /users 再转发
      rewrite: (path) => path.replace(/^\/api/, ''),
    },
  },
},
```

**用不用 rewrite 取决于后端路由怎么注册**：

- 后端路由是 `@app.get("/api/users")`（带前缀，或 `APIRouter(prefix="/api")`）→ **不要 rewrite**，原样转发；
- 后端路由是 `@app.get("/users")`（不带前缀，前缀只是代理层的约定）→ **必须 rewrite**，否则后端收到 `/api/users` 匹配不到路由，返回 404。

建议和 FastAPI 侧统一：`APIRouter(prefix="/api")` 带上前缀，代理层原样转发。这样后端路由表自描述（一眼看出哪些是 API），且开发和生产环境的转发规则完全一致，少一个出错的维度。

`changeOrigin: true` 的作用是把转发请求的 `Host` 头改成 target 的域名。多数后端不看 Host 头，配不配无所谓；但如果后端做了基于 Host 的校验或虚拟主机分发，不配就会 404/403，养成配上的习惯。

### 4.5 用环境变量区分开发与生产

Vite 按模式加载不同的 `.env` 文件：

```bash
# .env.development（npm run dev 时加载）
VITE_API_BASE_URL=/api
```

```bash
# .env.production（npm run build 时加载）
VITE_API_BASE_URL=/api
```

两边都是 `/api`（开发走 Vite proxy、生产走 Nginx），前端代码完全不用感知环境差异。如果某天后端 API 部署到了独立域名（如 `https://api.example.com`），只需要改生产环境变量：

```bash
# .env.production
VITE_API_BASE_URL=https://api.example.com/api
```

注意这种"前后端不同域名"的部署方式会重新引入跨域，届时需要在 FastAPI 的 `CORSMiddleware` 里把 `https://example.com` 加进 `allow_origins`——第 2 节的知识在这里闭环。

## 5. 登录鉴权完整流程走通

登录鉴权是前后端配合里链路最长的一环，涉及两端各自写代码、还要考虑 token 过期后的体验。这一节给一个可以直接落地的完整方案：**access token（短期）+ refresh token（长期）+ 401 无感刷新**。

### 5.1 整体方案与时序

为什么用两个 token？单个长期 token 泄露后危害窗口太长；单个短期 token 又会让用户频繁掉线。双 token 折中：

- **access token**：有效期短（如 2 小时），每次请求携带，验它即可访问资源；
- **refresh token**：有效期长（如 7 天），只在 access token 过期后拿去换新，平时不发给资源接口。

登录成功时后端同时签发两个。前端每次请求带 access token；收到 401 后用 refresh token 调刷新接口拿新 access token，**然后自动重发刚才失败的请求**——用户完全无感知，这叫"无感刷新"。

完整时序：

```
┌─ 登录 ────────────────────────────────────────────────────────────┐
│ 前端 POST /api/auth/login {username, password}                    │
│ 后端 校验密码 → 签发 access_token(2h) + refresh_token(7d)         │
│ 前端 两个 token 存入 localStorage                                 │
└───────────────────────────────────────────────────────────────────┘
┌─ 正常访问 ─────────────────────────────────────────────────────────┐
│ 前端 请求头携带 Authorization: Bearer <access_token>              │
│ 后端 校验通过 → 返回数据                                          │
└───────────────────────────────────────────────────────────────────┘
┌─ access token 过期后 ──────────────────────────────────────────────┐
│ 1. 前端 请求 A 带旧 access_token                                  │
│ 2. 后端 返回 401                                                  │
│ 3. 前端 拦截器发现 401 → 暂停 A，用 refresh_token 调刷新接口      │
│ 4. 后端 校验 refresh_token → 签发新 access_token（+新 refresh）   │
│ 5. 前端 存新 token，带着新 token 自动重发请求 A → 拿到数据        │
│    （用户无感知）                                                  │
└───────────────────────────────────────────────────────────────────┘
┌─ refresh token 也过期/失效 ────────────────────────────────────────┐
│ 刷新接口返回 401 → 清空本地 token → 跳转登录页                    │
└───────────────────────────────────────────────────────────────────┘
```

### 5.2 后端实现：FastAPI 签发与校验 JWT

安装依赖（当前主流用 PyJWT，不依赖已停止维护的 python-jose）：

```bash
pip install "pyjwt[crypto]" "passlib[bcrypt]"
```

配置与 token 工具：

```python
# app/core/security.py
from datetime import datetime, timedelta, timezone

import jwt
from passlib.context import CryptContext

SECRET_KEY = "change-me-in-production-use-env"  # 生产环境必须从环境变量读，且足够随机
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 2        # access token 2 小时
REFRESH_TOKEN_EXPIRE_DAYS = 7               # refresh token 7 天

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def create_token(user_id: int, token_type: str, expires_delta: timedelta) -> str:
    """签发 JWT。token_type 区分 access / refresh，防止拿 access token 来刷新。"""
    now = datetime.now(timezone.utc)
    payload = {
        "sub": str(user_id),       # JWT 规范要求 sub 是字符串
        "type": token_type,        # "access" 或 "refresh"
        "iat": now,                # 签发时间
        "exp": now + expires_delta,  # 过期时间，PyJWT 校验时自动检查
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def create_access_token(user_id: int) -> str:
    return create_token(user_id, "access", timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))


def create_refresh_token(user_id: int) -> str:
    return create_token(user_id, "refresh", timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS))
```

认证相关的依赖与路由：

```python
# app/api/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from pydantic import BaseModel
import jwt

from app.core.security import (
    ALGORITHM, SECRET_KEY,
    create_access_token, create_refresh_token, verify_password,
)

router = APIRouter(prefix="/api/auth", tags=["auth"])
bearer_scheme = HTTPBearer(auto_error=False)

# 演示用的用户存储；真实项目里换成数据库查询（SQLAlchemy 章节的内容）
FAKE_USERS = {
    "admin": {
        "id": 1,
        "username": "admin",
        # 明文是 "secret123"，用 hash_password("secret123") 预生成
        "hashed_password": "$2b$12$examplehash...",  # 省略：启动时用 hash_password 生成
    }
}


class LoginIn(BaseModel):
    username: str
    password: str


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


def decode_token(token: str, expected_type: str) -> int:
    """解码并校验 token；类型不符或过期一律 401。"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except jwt.PyJWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="token 无效或已过期",
            headers={"WWW-Authenticate": "Bearer"},  # 401 响应的规范头
        )
    if payload.get("type") != expected_type:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="token 类型错误")
    return int(payload["sub"])


@router.post("/login", response_model=TokenPair)
def login(data: LoginIn):
    user = FAKE_USERS.get(data.username)
    if not user or not verify_password(data.password, user["hashed_password"]):
        # 统一模糊提示，不区分"用户不存在"和"密码错误"，防止用户名枚举攻击
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="用户名或密码错误")
    return TokenPair(
        access_token=create_access_token(user["id"]),
        refresh_token=create_refresh_token(user["id"]),
    )


class RefreshIn(BaseModel):
    refresh_token: str


@router.post("/refresh", response_model=TokenPair)
def refresh(data: RefreshIn):
    # 校验 refresh token，通过则签发新的一对 token（refresh 也轮换，更安全）
    user_id = decode_token(data.refresh_token, expected_type="refresh")
    return TokenPair(
        access_token=create_access_token(user_id),
        refresh_token=create_refresh_token(user_id),
    )


def get_current_user_id(
    credentials: HTTPAuthorizationCredentials | None = Depends(bearer_scheme),
) -> int:
    """业务接口的鉴权依赖：校验 Authorization 头里的 access token。"""
    if credentials is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="未提供认证凭据",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return decode_token(credentials.credentials, expected_type="access")
```

业务接口只需要一个依赖就完成鉴权：

```python
# app/api/user.py
from fastapi import APIRouter, Depends

from app.api.auth import get_current_user_id

router = APIRouter(prefix="/api/users", tags=["users"])


@router.get("/me")
def read_me(user_id: int = Depends(get_current_user_id)):
    # 能走到这里，token 一定有效；user_id 就是当前登录用户
    return {"code": 0, "data": {"id": user_id, "name": "admin"}, "message": "ok"}
```

要点解读：

- **token 里放 `type` 声明并强制校验**：防止攻击者拿 access token 去调刷新接口（反之亦然），这是双 token 方案里最容易漏的一步。
- **JWT 是无状态的**：后端不存任何东西，验签（用 SECRET_KEY 验证签名未被篡改）+ 验过期时间即可。这也意味着**签发后无法主动吊销**——如果需要"踢人下线"，要引入 Redis 黑名单或版本号机制（进阶话题，知道有这个坑即可）。
- **`WWW-Authenticate: Bearer` 头**：HTTP 规范要求 401 响应必须携带，前端和抓包工具靠它确认认证方案。
- **密钥管理**：`SECRET_KEY` 泄露等于任何人都能伪造任意用户的 token。生产环境用 `os.environ["SECRET_KEY"]` 读取，且用足够长的随机串（`openssl rand -hex 32`）。

### 5.3 前端：登录与 token 存储

```ts
// src/api/auth.ts
import request from '@/utils/request'

export interface TokenPair {
  access_token: string
  refresh_token: string
}

export const login = (data: { username: string; password: string }) =>
  request.post<unknown, TokenPair>('/auth/login', data)
```

```ts
// src/utils/token.ts：token 存储的收口，便于以后换存储方案
const ACCESS_KEY = 'access_token'
const REFRESH_KEY = 'refresh_token'

export const getAccessToken = () => localStorage.getItem(ACCESS_KEY)
export const getRefreshToken = () => localStorage.getItem(REFRESH_KEY)
export function setTokens(pair: { access_token: string; refresh_token: string }) {
  localStorage.setItem(ACCESS_KEY, pair.access_token)
  localStorage.setItem(REFRESH_KEY, pair.refresh_token)
}
export function clearTokens() {
  localStorage.removeItem(ACCESS_KEY)
  localStorage.removeItem(REFRESH_KEY)
}
```

登录页提交逻辑：

```ts
// src/views/Login.vue 的 <script setup lang="ts"> 片段
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import { login } from '@/api/auth'
import { setTokens } from '@/utils/token'

const router = useRouter()
const form = ref({ username: '', password: '' })
const loading = ref(false)

async function onSubmit() {
  loading.value = true
  try {
    const tokens = await login(form.value) // 拦截器已解包，直接拿到 TokenPair
    setTokens(tokens)
    ElMessage.success('登录成功')
    // 跳回登录前想去的页面（路由守卫里存的 query），默认去首页
    router.replace((router.currentRoute.value.query.redirect as string) || '/')
  } catch {
    // 错误提示拦截器已统一弹出
  } finally {
    loading.value = false
  }
}
```

### 5.4 401 无感刷新：拦截器完整实现

这是整个鉴权链路里技术含量最高的一段。难点在于**并发控制**：页面加载瞬间常有三四个请求同时发出，access token 过期时它们会**同时收到 401**。如果每个 401 都去调一次刷新接口，轻则浪费，重则后端做了 refresh token 轮换后第二次刷新直接把第一次换来的 token 作废，造成"偶发掉线"的灵异 bug。

正确做法：**只允许一个刷新请求在飞（single-flight），其余 401 请求挂起等待刷新结果**。

```ts
// src/utils/request.ts 完整版
import axios, { AxiosError, type AxiosRequestConfig } from 'axios'
import { ElMessage } from 'element-plus'
import { clearTokens, getAccessToken, getRefreshToken, setTokens } from '@/utils/token'

const request = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api',
  timeout: 15000,
})

// ===== 请求拦截器：自动带 token =====
request.interceptors.request.use((config) => {
  const token = getAccessToken()
  if (token) {
    config.headers.set('Authorization', `Bearer ${token}`)
  }
  return config
})

// ===== 401 无感刷新：single-flight 状态 =====
// 正在刷新时的 Promise；非空表示"有人在刷"，后来的 401 等它就行
let refreshingPromise: Promise<string> | null = null

async function refreshAccessToken(): Promise<string> {
  const refreshToken = getRefreshToken()
  if (!refreshToken) {
    throw new Error('no refresh token')
  }
  // 注意：用裸 axios 而不是 request 实例调刷新接口，
  // 避免刷新请求自己又走拦截器造成循环
  const { data } = await axios.post('/api/auth/refresh', {
    refresh_token: refreshToken,
  }, { baseURL: import.meta.env.VITE_API_BASE_URL || '/api' })
  // 后端返回统一结构 { code, data: { access_token, refresh_token }, message }
  setTokens(data.data)
  return data.data.access_token
}

function forceLogout() {
  clearTokens()
  // 带上当前路径，登录后跳回来
  const redirect = encodeURIComponent(window.location.pathname + window.location.search)
  window.location.href = `/login?redirect=${redirect}`
}

async function handle401(error: AxiosError) {
  const originalConfig = error.config as AxiosRequestConfig & { _retried?: boolean }

  // 防死循环：已经重试过一次的请求不再重试
  if (!originalConfig || originalConfig._retried) {
    forceLogout()
    return Promise.reject(error)
  }
  originalConfig._retried = true

  try {
    // single-flight：没人在刷就我来刷；有人在刷就等它刷完
    refreshingPromise ??= refreshAccessToken().finally(() => {
      refreshingPromise = null // 无论成功失败都清空，允许下一轮刷新
    })
    const newToken = await refreshingPromise

    // 用新 token 重发原请求
    originalConfig.headers = {
      ...originalConfig.headers,
      Authorization: `Bearer ${newToken}`,
    }
    return request(originalConfig)
  } catch {
    // 刷新也失败（refresh token 过期/被吊销）→ 只能重新登录
    forceLogout()
    return Promise.reject(error)
  }
}

// ===== 响应拦截器 =====
request.interceptors.response.use(
  (response) => {
    const body = response.data
    if (body.code === 0) {
      return body.data
    }
    ElMessage.error(body.message || '请求失败')
    return Promise.reject(new Error(body.message))
  },
  async (error: AxiosError) => {
    if (error.response?.status === 401) {
      return handle401(error)
    }
    // ... 其余错误处理同第 3 节，此处省略
    return Promise.reject(error)
  },
)

export default request
```

把并发场景代入走一遍：请求 A、B、C 同时 401 → A 最先进入 `handle401`，发现 `refreshingPromise` 为空，发起刷新；B、C 随后进入，发现有人在刷，直接 `await` 同一个 Promise → 刷新完成，A、B、C 拿着同一个新 token 各自重发。**整个页面只调了一次刷新接口**，且所有请求自动续上，用户完全无感。

三个细节再强调：

- `_retried` 标记是防死循环的保险丝。如果后端 bug 导致新 token 也 401，没有它前端会无限刷新。
- 刷新请求必须用**裸 axios**（或第二个不带拦截器的实例），否则刷新请求 401 时又触发刷新逻辑，递归爆炸。
- `refreshingPromise = null` 放在 `finally` 里：成功要清（下次过期再刷），失败也要清（否则所有后续 401 都等一个永远 reject 的 Promise）。

### 5.5 路由守卫：未登录不许进

axios 层解决"带 token、续 token"，但"没有 token 的页面根本不该打开"要在路由层拦：

```ts
// src/router/index.ts 片段
import { createRouter, createWebHistory } from 'vue-router'
import { getAccessToken } from '@/utils/token'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/login', component: () => import('@/views/Login.vue'), meta: { public: true } },
    { path: '/', component: () => import('@/views/Home.vue') },
    // ...
  ],
})

router.beforeEach((to) => {
  // 白名单页面直接放行
  if (to.meta.public) return true
  // 无 token 去登录页，并记住本来想去哪
  if (!getAccessToken()) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  return true
})
```

注意职责划分：**路由守卫只看"有没有"token，不管 token 是否还有效**——有效性由后端校验，401 时拦截器兜底刷新或踢去登录。两端各管一层，不要试图在前端解析 token 的 exp 来做精确判断（本地时钟与服务器可能不一致，且刷新逻辑反正都要写）。

### 5.6 登出

```ts
// src/api/auth.ts（前端登出）
import { clearTokens } from '@/utils/token'

export function logout() {
  // 可选：通知后端把 refresh token 加入黑名单（如果后端实现了吊销机制）
  // request.post('/auth/logout').catch(() => {})
  clearTokens()
  window.location.href = '/login'
}
```

JWT 无状态特性决定了"纯前端登出"就是**删掉本地 token**——后端无法让已签发的 token 立即失效。常规系统的接受度是：access token 有效期短（2 小时内自然失效），风险可控；安全要求高的系统再给 refresh token 加服务端吊销（存 Redis，刷新时检查是否被拉黑）。

## 6. 生产环境部署配合

开发环境靠 Vite proxy，生产环境的同构方案是 Nginx：**前端静态资源直接由 Nginx 托管，`/api` 前缀的请求反向代理到 uvicorn**。浏览器从头到尾只跟一个域名打交道，同源、无跨域、无需 CORS 配置。

### 6.1 前端构建产物

```bash
npm run build
```

Vite 把源码编译、打包、压缩成纯静态文件，输出到 `dist/`：

```
dist/
├── index.html                  # 入口 HTML
└── assets/
    ├── index-a8f3c2.js         # 业务 JS，文件名带内容 hash
    ├── index-b7d2e1.css        # 样式
    └── logo-c3f9a4.png
```

几个对部署重要的特性：

- **文件名带内容 hash**：内容变了 hash 就变，所以这些文件可以让浏览器**强缓存一年**而不用担心用户拿到旧版本——这是前端缓存策略的基础。
- **`index.html` 是引用这些 hash 文件的"地图"**：它必须**不缓存**，否则发新版后用户浏览器还用着旧 HTML，引用旧 hash 的 JS，新功能上不去。
- 产物是纯静态文件，**不需要 Node.js 环境**，任何静态服务器（Nginx、CDN）都能托管。

把产物传到服务器（CI/CD 或手动均可）：

```bash
scp -r dist/ user@server:/var/www/myapp/
```

### 6.2 Nginx 完整配置

```nginx
# /etc/nginx/conf.d/myapp.conf
server {
    listen       80;
    server_name  example.com;

    root   /var/www/myapp/dist;
    index  index.html;

    # 上传文件大小上限，默认 1m，有上传功能必须调大
    client_max_body_size  20m;

    # ===== 带 hash 的静态资源：强缓存一年 =====
    location /assets/ {
        expires  1y;
        add_header  Cache-Control  "public, immutable";
    }

    # ===== API 反向代理：/api 转发给 FastAPI =====
    location /api/ {
        # proxy_pass 末尾不带 URI：/api/users 原样转发为 /api/users
        # （后端路由带 /api 前缀，与第 4 节 Vite proxy 不 rewrite 的约定一致）
        proxy_pass  http://127.0.0.1:8000;

        # 传递真实客户端信息，否则后端看到的 IP 全是 127.0.0.1
        proxy_set_header  Host              $host;
        proxy_set_header  X-Real-IP         $remote_addr;
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;

        # ===== WebSocket 支持（有实时功能才需要，详见 6.3）=====
        proxy_http_version  1.1;
        proxy_set_header    Upgrade    $http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_read_timeout  3600s;   # WebSocket 长连接空闲时不要被掐断
    }

    # ===== history 路由支持：找不到的文件回退到 index.html =====
    location / {
        try_files  $uri  $uri/  /index.html;
    }

    # index.html 不缓存，保证发版立即生效
    location = /index.html {
        add_header  Cache-Control  "no-cache, must-revalidate";
    }
}
```

后端 uvicorn 启动命令：

```bash
# 只监听本机回环地址：只接受 Nginx 的转发，不直接暴露公网
# --proxy-headers：信任 Nginx 传来的 X-Forwarded-* 头，后端才能拿到真实 IP 和协议
uvicorn main:app --host 127.0.0.1 --port 8000 --workers 4 --proxy-headers
```

配置生效的标准流程（Nginx 章节详细讲过）：

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 6.3 关键配合点逐个讲

**`try_files $uri $uri/ /index.html;` 是 history 路由的救命配置。** Vue Router 的 history 模式下 URL 是干净的 `/user/123`，但磁盘上没有这个文件——用户在该路径刷新浏览器时，请求直达 Nginx，没有这行就是 404。加上之后：Nginx 先找真实文件（JS/CSS/图片不受影响），找不到就把请求交给 `index.html`，由前端路由接管渲染对应页面。

**`proxy_pass` 末尾的斜杠决定转发路径。** `proxy_pass http://127.0.0.1:8000;`（无 URI）是原样透传：`/api/users` → `/api/users`。若写成 `http://127.0.0.1:8000/;`（哪怕只多一个 `/`），则变成替换语义：`/api/users` → `/users`。必须和后端路由注册的前缀保持一致——本项目后端路由带 `/api` 前缀，所以用透传写法。这个对不上就是大片 404，且两端日志看起来都"正常"，极其难查。

**`/api` 与静态资源的 location 互不干扰是 location 匹配规则保证的。** `location /api/` 和 `location /` 都是前缀匹配，Nginx 选**最长前缀**——`/api/users` 命中 `/api/`，`/assets/app.js` 命中 `/`。这也是为什么前缀约定（`/api`）是整个分离架构里最重要的那一个约定。

### 6.4 WebSocket 代理注意点

FastAPI 的 WebSocket 接口（如 `/api/ws`）经过 Nginx 代理时，握手经常失败，原因在协议机制上：WebSocket 先用 HTTP 发起一个带 `Upgrade: websocket` 和 `Connection: Upgrade` 头的握手请求，服务器回 `101 Switching Protocols` 后，连接升级为全双工通道。而 Nginx 反代默认用 HTTP/1.0 连后端、并会改写 `Connection` 头，后端收不到升级意图，握手直接失败。

必需的三行（上面配置已含）：

```nginx
proxy_http_version  1.1;                     # Upgrade 是 HTTP/1.1 特性
proxy_set_header    Upgrade    $http_upgrade;  # 透传客户端的 Upgrade 头
proxy_set_header    Connection "upgrade";    # 固定设置 Connection 头
```

外加一个超时问题：WebSocket 连接空闲时没有数据流动，Nginx 默认 60 秒读不到后端数据就主动断连——表现为"连接建立成功，过一分钟莫名其妙断开"。`proxy_read_timeout 3600s;` 把空闲容忍拉长到 1 小时；生产上还应同时在**应用层做心跳**（前端每 30 秒发个 ping，后端回 pong），双保险。前端连接时注意协议：页面是 HTTPS 时 WebSocket 必须用 `wss://`，用 `ws://` 会被浏览器按混合内容（Mixed Content）拦截。

### 6.5 部署 checklist

1. 前端 `npm run build`，确认 `dist/` 里 `index.html` 引用的资源路径正确（默认绝对路径 `/assets/`，与 Nginx `root` 对应）；
2. 后端环境变量齐备：`SECRET_KEY`、数据库连接串等，且**不是默认值**；
3. 后端 `CORSMiddleware`：同源部署下可以整段去掉；保留的话也不要再加 `add_header` 到 Nginx（两头都加会重复）；
4. uvicorn 监听 `127.0.0.1` 而非 `0.0.0.0`，加 `--proxy-headers`；
5. `nginx -t` 通过后 reload；
6. 冒烟验证：`curl https://example.com/`（200 + HTML）、`curl https://example.com/api/docs`（FastAPI 文档页）、浏览器完整走一遍登录 → 调接口 → 刷新页面。

## 7. 联调常见问题排查清单

联调出问题时不要乱猜，按错误类型对号入座。

### 7.1 CORS 报错怎么读

浏览器控制台的 CORS 报错其实写得很直白，关键是**逐词读**：

```
Access to XMLHttpRequest at 'http://localhost:8000/api/users' from origin
'http://localhost:5173' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

- `from origin '...'`：页面的源（前端跑在哪）；
- `blocked by CORS policy` 后面那一句是**具体原因**，常见的几种：
  - `No 'Access-Control-Allow-Origin' header` → 后端根本没配 CORS，或响应是异常路径返回的（中间件顺序问题，见 2.6 坑 3）；
  - `The value of the 'Access-Control-Allow-Origin' header ... must not be the wildcard '*'` → 携带了凭据但后端用了 `*`，见 2.6 坑 1；
  - `Request header field authorization is not allowed by Access-Control-Allow-Headers` → 预检时后端没放行 `authorization` 头，`allow_headers` 加上或设为 `["*"]`；
  - `Method PUT is not allowed by Access-Control-Allow-Methods` → `allow_methods` 没包含 PUT。

排查三板斧：① 用 curl 模拟预检确认后端行为：

```bash
curl -i -X OPTIONS http://127.0.0.1:8000/api/users \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: authorization,content-type"
```

返回里看 `Access-Control-Allow-*` 头是否符合预期；② DevTools Network 面板看预检的 OPTIONS 请求本身的状态码（200/204 正常，405 说明预检漏到了路由层）；③ 确认前端发的 Origin 和后端白名单**逐字符一致**（协议、域名、端口、无尾斜杠）。

### 7.2 404 / 405 / 422 分别代表什么

**404 Not Found——路由没匹配上。** 前后端分离项目里的 404 按概率排查：

1. 前端请求的 URL 和后端注册的**逐字符对比**（常见：`/api/users` vs `/api/user`、复数漏 s）；
2. baseURL 拼接问题：baseURL 是 `/api`，代码里又写 `/api/users`，实际请求变成 `/api/api/users`；
3. 代理路径转发错误：Vite `rewrite` 或 Nginx `proxy_pass` 斜杠问题导致后端收到的路径变了（第 4.4、6.3 节）；
4. history 路由刷新 404：Nginx 缺 `try_files`。

**405 Method Not Allowed——URL 对了，方法不对。** 后端注册的是 `@router.post("/login")`，前端发了 GET。FastAPI 的 405 响应里会带 `Allow: POST` 头告诉你正确的方法。另一个高频场景：预检 OPTIONS 穿透到了路由层（CORS 中间件没配或顺序不对），OPTIONS 没有对应路由处理，返回 405，浏览器随后报跨域错——**跨域报错的根因有时是 405**，要看 Network 面板里 OPTIONS 请求的状态码确认。

**422 Unprocessable Entity——FastAPI 的参数校验失败。** 这是 FastAPI 的特色状态码：请求到达了视图函数**之前**，Pydantic 校验请求参数失败。响应体里的 `detail` 数组精确到字段：

```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "username"],
      "msg": "Field required",
      "input": { "password": "123" }
    },
    {
      "type": "string_too_short",
      "loc": ["body", "password"],
      "msg": "String should have at least 8 characters",
      "input": "123"
    }
  ]
}
```

读法：`loc` 是出错位置（`body`/`query`/`path` + 字段名），`msg` 是原因，第一条说 username 没传，第二条说 password 太短。前端提交体与后端模型字段名不一致（大小写、下划线 vs 驼峰）是最高频原因。第 3 节的响应拦截器里已经把 `detail[0].msg` 提出来做提示。

### 7.3 token 失效问题排查

**"登录后调接口一直 401"**：① 请求头里有没有 `Authorization`？没有 → 检查请求拦截器与 token 存储的 key 是否一致；② 头格式是不是 `Bearer <token>`（注意 Bearer 后有一个空格）；③ 后端 `decode_token` 的 `expected_type` 是不是传成了 `"refresh"`；④ 前后端服务器时钟差太大，token 被判定未生效或已过期（JWT 的时间校验依赖服务器时钟，容器/虚拟机时间漂移是经典坑）。

**"token 明明没过期却被踢回登录页"**：① 无感刷新逻辑死循环保险丝（`_retried`）误触发——检查刷新接口本身是否正常；② 多标签页竞争：A 标签页刷新成功换了新 refresh token，B 标签页还拿旧的刷，被后端轮换机制拒绝（后端若做了 refresh token 轮换，前端多标签页要做存储同步或后端对旧 refresh token 给宽限期）；③ `localStorage` 被清——检查是否有代码在错误处理里无条件 `clearTokens()`。

**"每隔固定时间就掉线一次"**：几乎一定是 access token 过期而**无感刷新没生效**。在 Network 面板过滤 `/auth/refresh`：过期时刻有没有刷新请求发出？发出了返回什么？最常见的 bug 是刷新请求走了带拦截器的 axios 实例，刷新请求本身 401 又触发刷新，被保险丝拦下后直接 `forceLogout`。

**"刷新接口调了 N 次"**：并发 401 没做 single-flight（5.4 节），每个并发请求各刷一次。后端若做了 refresh token 轮换，这还会导致后续刷新全部失败、集体掉线。

### 7.4 其他高频问题

**`net::ERR_CONNECTION_REFUSED`**：请求根本没到达服务器。检查后端进程是否在跑、端口对不对、代理 target 写对没有。开发环境先 `curl http://127.0.0.1:8000/api/docs` 确认后端活着。

**`Mixed Content: The page was loaded over HTTPS, but requested an insecure resource`**：页面是 HTTPS，代码里却请求 HTTP 地址，浏览器直接拦截。生产环境一律用相对路径 `/api`（自动继承页面协议），不要在代码里写死 `http://`。

**接口慢但后端日志显示很快**：看是不是每个请求都带了一个 OPTIONS 预检且没有缓存（`Access-Control-Max-Age`），或者请求在 Nginx → 后端之间耗时大（access log 里的 `$upstream_response_time` 字段）。

**Response 里有数据但 JS 读不到某个自定义响应头**：跨域场景下 JS 默认只能读少数安全响应头，后端要在 CORS 配置里加 `expose_headers=["X-Total-Count"]` 之类的声明（分页总数、下载文件名常用）。同源部署无此问题。

## 小结

- 前后端分离的本质：后端只输出 JSON、前端负责渲染，两端靠**接口契约**（URL 前缀、数据结构、鉴权方式）协作，换来并行开发、多端复用、独立部署。
- 生产环境的标准形态是 Nginx 同源托管：静态资源直接返回、`/api` 反代到 uvicorn，浏览器只见一个源，**跨域被避免而不是被解决**。
- CORS 是浏览器机制：简单请求直接发、响应没许可就拦截；带 JSON 或 Authorization 的请求先走 OPTIONS 预检。FastAPI 用 `CORSMiddleware` 放行，注意 `*` 与 credentials 互斥、源字符串逐字符匹配、中间件顺序、别在 Nginx 重复加头四个坑。
- axios 封装四件套：实例 + baseURL（相对路径 `/api`，一处配置开发生产通用）、请求拦截器带 token、响应拦截器解包 code 并分流 401。
- 开发环境用 Vite proxy 把跨域请求伪装成同源；rewrite 与否取决于后端路由是否带 `/api` 前缀，建议后端带前缀、代理原样转发。
- 鉴权全链路：登录拿双 token → 请求带 access token → 401 时 single-flight 刷新 → 自动重发原请求；刷新失败才登出。`type` 声明区分两种 token，`_retried` 防死循环，刷新请求用裸 axios。
- 部署配合三要点：`try_files` 救 history 路由、`proxy_pass` 斜杠语义、WebSocket 的 Upgrade 三行头加读超时。
- 排查对号入座：CORS 报错逐词读并用 curl 模拟预检；404 查 URL 拼接与代理路径，405 查方法与预检穿透，422 看 `detail` 数组定位字段；token 问题看 Authorization 头、刷新请求、服务器时钟。

## 常见问题

**1. 已经配置了 Vite proxy / Nginx 同源部署，后端还要不要配 CORSMiddleware？**

同源部署下不需要——浏览器看到的是同一个源，CORS 机制根本不触发，多配无害但无意义（且 Nginx 再加 `add_header` 造成响应头重复反而会触发浏览器报错）。CORSMiddleware 真正的用武之地是：前后端**不同域名**的部署（如 `app.example.com` 调 `api.example.com`）、给第三方站点开放 API、或不使用代理的开发方式。原则是：能同源部署就同源，必须跨域才配 CORS，且只在一处配。

**2. token 到底存 localStorage 还是 cookie？**

各防一种攻击：localStorage 里的 token 可被 XSS 注入的脚本读走（防 XSS 靠框架转义与输入过滤）；cookie（HttpOnly）脚本读不到，但浏览器会自动携带从而面临 CSRF（防 CSRF 靠 SameSite=Strict/Lax、CSRF token）。用 cookie 还会把跨域配置复杂化（credentials + 具体源白名单）。对多数内部系统，存 localStorage + 严防 XSS（不信任任何 `v-html`）是更简单务实的选择；面向公网且安全要求高的系统，HttpOnly cookie + SameSite 更稳。无论哪种，access token 有效期都要短。

**3. 为什么刷新 token 要做 single-flight（只刷一次）？**

页面初始化常并发发出多个请求，token 过期时它们同时收到 401。若每个 401 都独立调刷新接口：轻则浪费请求；重则在后端做了 refresh token 轮换的情况下，第一个刷新成功后旧 refresh token 即作废，其余并发刷新全部失败，用户被误判为"登录失效"集体踢回登录页——表现为偶发、难复现的掉线 bug。single-flight 让第一个 401 负责刷新、其余挂起等待同一个 Promise，全页面只刷一次且行为确定。

**4. 后端返回 422 但参数"明明传了"，怎么快速定位？**

422 的响应体已经给出全部线索，不要猜：看 `detail` 数组里每条错误的 `loc`（出错位置：`body`/`query`/`path` + 字段名）和 `msg`（原因）。最高频的原因是字段名不匹配——前端发 `userName`、后端模型是 `user_name`（FastAPI/Pydantic 默认不做命名风格转换），或 Content-Type 不对导致 body 没被解析成 JSON（用 axios 的 `post(url, data)` 默认就是 JSON，但用 `FormData` 或手写 `application/x-www-form-urlencoded` 时后端要对应改用 `Form()` 接收）。把 Network 面板里的请求体原始内容和后端模型定义并排一比，问题立现。

**5. 生产环境部署后 WebSocket 连不上或每分钟断一次，怎么排查？**

连不上：先确认 Nginx 的 `/api` location 里有三行——`proxy_http_version 1.1;`、`proxy_set_header Upgrade $http_upgrade;`、`proxy_set_header Connection "upgrade";`，缺了握手请求到不了后端（表现是前端握手报 400/426 或直接失败）。每分钟断：这是 Nginx 默认 `proxy_read_timeout 60s` 在掐空闲连接，调大到 `3600s` 并在应用层加心跳（前端 30 秒一次 ping）。页面是 HTTPS 的话，前端连接地址必须用 `wss://` 且域名与页面一致（复用同一 Nginx server 块即可自动满足）。
