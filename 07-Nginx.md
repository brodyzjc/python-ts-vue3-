# Nginx 实战

## 本章导读

- Nginx 是什么：master-worker 进程模型与事件驱动架构，为什么单机就能扛住数万并发
- 安装、启动、停止、优雅重载与信号机制，改完配置如何安全生效
- 配置文件的层级结构：main / events / http / server / location 各块职责
- 静态资源服务：root 与 alias 的区别、gzip 压缩、浏览器缓存控制
- location 匹配规则的优先级顺序，用完整例子走一遍匹配过程
- 反向代理：proxy_pass 结尾斜杠的坑、如何向后端传递真实客户端 IP
- 负载均衡：upstream 与轮询、weight、ip_hash、least_conn 策略
- HTTPS：证书申请、TLS 参数、HTTP 强制跳转 HTTPS
- 完整实战：一台 Nginx 同时托管 Vue3 前端 dist 和 FastAPI 后端，含 history 路由与 WebSocket
- 运维技巧：日志分析、限流、上传大小、超时参数、502/504 排查

## 1. Nginx 是什么与架构原理

### Nginx 是什么

Nginx（读作 "engine-x"）是一款开源的高性能 HTTP 服务器和反向代理服务器，由俄罗斯的 Igor Sysoev 在 2002 年发起，专门为解决 C10K 问题（单机同时处理一万个并发连接）而设计。

它在你的系统里通常扮演三个角色：

1. **静态资源服务器**：直接返回 HTML、CSS、JS、图片等文件，性能远超应用服务器。
2. **反向代理服务器**：接收客户端请求，转发给后端的 FastAPI、Django、Node 等应用，再把响应带回给客户端。
3. **负载均衡器**：把请求按策略分发给多台后端服务器，实现水平扩容。

在互联网架构图中，Nginx 几乎永远站在最前面：

```
客户端 ──TLS/HTTP──> Nginx ──> 静态文件（直接返回）
                       │
                       ├──> FastAPI 实例 1 (127.0.0.1:8000)
                       ├──> FastAPI 实例 2 (127.0.0.1:8001)
                       └──> FastAPI 实例 3 (127.0.0.1:8002)
```

### master-worker 进程模型

Nginx 启动后，你会看到一个 **master 进程**和多个 **worker 进程**：

```bash
$ ps aux | grep nginx
root      1234  nginx: master process /usr/sbin/nginx
www-data  1235  nginx: worker process
www-data  1236  nginx: worker process
www-data  1237  nginx: worker process
www-data  1238  nginx: worker process
```

两个角色的分工非常明确：

- **master 进程**：以 root 身份运行，不处理任何用户请求。它只做三件事——读取并校验配置、创建/销毁 worker 进程、接收管理员信号（如 reload、stop）并转发给 worker。
- **worker 进程**：实际干活的进程，处理所有客户端连接。worker 数量通常配置为 CPU 核数（`worker_processes auto;`），每个 worker 都是**单线程**的，独立运行、互不影响。

这种模型带来几个关键好处：

- **进程间隔离**：某个 worker 崩溃，master 会立刻拉起一个新 worker，服务不中断。
- **充分利用多核**：每个 worker 绑定一个 CPU 核心，避免多线程的锁竞争和上下文切换开销。
- **热部署**：reload 时 master 先加载新配置、启动新 worker，再让旧 worker 处理完手头请求后退出，整个过程不丢任何一个连接。

### 事件驱动、异步非阻塞

为什么 Nginx 一个单线程的 worker 能同时处理上万个连接？传统服务器（如早期 Apache）是"一个连接一个进程/线程"的模型：每个连接占一个线程，线程在等待磁盘 IO 或网络 IO 时会被阻塞挂起。一万个连接就是一万个线程，内存和上下文切换开销直接压垮机器。

Nginx 的 worker 采用**事件驱动 + 异步非阻塞 IO**模型，底层依赖操作系统提供的 IO 多路复用机制——Linux 上是 **epoll**，FreeBSD 上是 kqueue：

```
worker 主循环（简化版）
┌─────────────────────────────────────────────┐
│ while (True):                               │
│     events = epoll_wait(...)   # 等待就绪事件│
│     for event in events:                    │
│         handle(event)          # 非阻塞处理  │
└─────────────────────────────────────────────┘
```

工作原理用一句话概括：**worker 不为连接"等待"，而是让内核"通知"**。worker 把上万个连接的文件描述符注册到 epoll，然后调用 `epoll_wait` 睡眠；当某些连接上有数据到达（可读）或发送缓冲区有空位（可写）时，内核唤醒 worker 并告知哪些连接就绪，worker 只处理这些就绪的连接。

这和你学过的 Python `asyncio` 是同一个思想（asyncio 底层也是 epoll）：**一个执行流通过事件循环复用大量 IO 连接**。区别在于 Nginx 用 C 实现，把这套机制做到了极致。

### 为什么能扛高并发

把上面的机制串起来，Nginx 扛高并发的原因有四点：

1. **连接成本低**：一个连接只占 worker 里一小段内存（几 KB），不开线程不开进程。数万并发连接的内存占用依然可控。
2. **没有阻塞**：所有网络 IO 都是非阻塞的，慢客户端（网络差、迟迟不读完响应）不会拖累 worker——worker 把它挂回 epoll 继续服务别人。
3. **没有锁**：单线程 worker 内部不存在共享状态竞争，不需要互斥锁，CPU 时间全部花在处理请求上。
4. **高效零拷贝**：发送静态文件时使用 `sendfile` 系统调用，数据直接在内核空间从磁盘文件拷贝到网卡，不经过用户态，省掉了两次数据拷贝和两次上下文切换。

理解了这些，后面很多配置项——`worker_connections`、`sendfile`、`keepalive_timeout`——你都能明白它们在调什么。

## 2. 安装与常用命令

### 安装

Ubuntu / Debian：

```bash
sudo apt update
sudo apt install nginx
```

CentOS / Rocky Linux：

```bash
sudo dnf install nginx
```

macOS（开发用）：

```bash
brew install nginx
```

Windows 上 Nginx 一般只用于本地体验，生产环境几乎都是 Linux。验证安装：

```bash
nginx -v        # 查看版本
nginx -V        # 查看版本和编译参数（大写 V）
```

安装完成后，Linux 发行版通常会自动把 Nginx 注册为 systemd 服务。

### 启动、停止、重载

通过 systemd 管理（生产环境推荐）：

```bash
sudo systemctl start nginx      # 启动
sudo systemctl stop nginx       # 停止
sudo systemctl restart nginx    # 重启（会中断连接）
sudo systemctl reload nginx     # 优雅重载配置（不中断连接）
sudo systemctl enable nginx     # 开机自启
sudo systemctl status nginx     # 查看运行状态
```

也可以直接操作 nginx 命令：

```bash
sudo nginx                      # 启动
sudo nginx -s stop              # 快速停止（强制断开连接）
sudo nginx -s quit              # 优雅停止（处理完请求再退出）
sudo nginx -s reload            # 优雅重载配置
sudo nginx -s reopen            # 重新打开日志文件（日志切割时用）
```

### nginx -t：改配置后的保命动作

**每次修改配置后、reload 之前，必须先测试配置语法**：

```bash
sudo nginx -t
```

输出示例：

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

如果配置有语法错误，`nginx -t` 会指出文件和行号。skip 这一步直接 reload，轻则 reload 失败（Nginx 会继续用旧配置运行，你以为生效了其实没有），重则 restart 时整个服务起不来造成停机。`nginx -t && nginx -s reload` 应该成为你的肌肉记忆。

### 信号机制

`nginx -s reload` 本质上是向 master 进程发送 Unix 信号。理解信号机制，就理解了 Nginx 运维的底层：

| 信号 | 作用 | 等价命令 |
|------|------|----------|
| TERM / INT | 快速停止，立即断开所有连接 | `nginx -s stop` |
| QUIT | 优雅停止，worker 处理完当前请求后退出 | `nginx -s quit` |
| HUP | 重载配置 | `nginx -s reload` |
| USR1 | 重新打开日志文件 | `nginx -s reopen` |
| USR2 | 升级可执行文件（热升级二进制用） | — |
| WINCH | 优雅关闭 worker 进程 | — |

手动发信号的方式：

```bash
sudo kill -HUP $(cat /run/nginx.pid)   # 等价于 nginx -s reload
```

**reload 的内部过程**（这是面试高频考点）：

1. master 收到 HUP 信号，重新读取并用新配置做语法校验；校验失败则放弃 reload，master 和旧 worker 继续用旧配置工作，服务不受任何影响。
2. 校验通过后，master 用新配置启动一批**新 worker**。
3. master 通知**旧 worker 优雅退出**：旧 worker 停止接收新连接，但继续把已经建立连接的请求处理完。
4. 旧 worker 全部退出后，reload 完成。

这个"新旧 worker 平滑交接"的设计，保证了改配置期间**零停机、零连接丢失**。

## 3. 配置文件结构

### 整体层级

Nginx 的主配置文件是 `/etc/nginx/nginx.conf`。它由若干**块（block / context）**嵌套组成，层级是固定的：

```
main（全局块，文件最外层，不写名字）
├── events { }            # 事件模型配置
└── http { }              # HTTP 服务总配置
    ├── server { }        # 一个虚拟主机（一个站点）
    │   ├── location { }  # 匹配某类 URL 的处理规则
    │   └── location { }
    ├── server { }
    └── upstream { }      # 后端服务器组定义
```

各块职责：

- **main 全局块**：作用于整个 Nginx 实例的配置——运行用户、worker 数量、pid 文件、错误日志位置。
- **events 块**：配置事件驱动模型——使用哪种 IO 多路复用机制、每个 worker 的最大连接数。
- **http 块**：所有 HTTP 流量的公共配置——MIME 类型、日志格式、gzip、超时、sendfile 等。里面可以包含多个 server。
- **server 块**：定义一个**虚拟主机**（virtual host）。Nginx 根据请求的 `Host` 头（域名）和端口决定由哪个 server 处理。一台机器可以用多个 server 块托管多个网站。
- **location 块**：在 server 内部按 URL 路径细分处理规则——`/static/` 走静态文件，`/api/` 转发给后端。
- **upstream 块**：定义一组后端服务器，供反向代理和负载均衡引用。

一个麻雀虽小五脏俱全的骨架：

```nginx
# ========== main 全局块 ==========
user  www-data;                  # worker 进程的运行用户
worker_processes  auto;          # worker 数量，auto = CPU 核数
pid  /run/nginx.pid;             # master 进程 pid 文件
error_log  /var/log/nginx/error.log warn;   # 全局错误日志

# ========== events 块 ==========
events {
    worker_connections  1024;    # 每个 worker 的最大并发连接数
    # use epoll;                 # Linux 默认自动选择最优机制，一般不用显式写
    multi_accept  on;            # 一次 accept 尽可能多的新连接
}

# ========== http 块 ==========
http {
    include       /etc/nginx/mime.types;   # 文件扩展名 → Content-Type 映射
    default_type  application/octet-stream;

    sendfile  on;                # 零拷贝发送静态文件
    tcp_nopush  on;              # 配合 sendfile，攒够数据包再发，减少网络包数量
    keepalive_timeout  65;       # 长连接空闲多久后关闭（秒）

    # 日志格式（命名为 main，供 access_log 引用）
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" $request_time';

    access_log  /var/log/nginx/access.log  main;

    gzip  on;                    # 全局开启 gzip（详见第 4 节）

    include  /etc/nginx/conf.d/*.conf;       # 加载所有站点配置
    include  /etc/nginx/sites-enabled/*;     # Debian/Ubuntu 风格的站点目录
}
```

### include 组织方式：别把鸡蛋放在一个文件里

主配置里的两行 `include` 是 Nginx 工程化组织配置的关键。**不要把所有站点的配置都堆进 nginx.conf**，而是每个站点一个独立文件：

```
/etc/nginx/
├── nginx.conf              # 主配置：全局 + http 公共配置 + include
├── conf.d/
│   ├── myapp.conf          # 站点 A（一个 server 块）
│   └── blog.conf           # 站点 B
└── sites-enabled/          # Debian/Ubuntu 惯例，软链接到 sites-available
    └── myapp -> ../sites-available/myapp
```

这样的好处：新增站点就是新增一个文件、删掉文件就是下线站点，互不影响；出错时 `nginx -t` 会直接报出是哪个文件哪一行。

**配置继承规则**（重要）：很多指令（如 `gzip`、`access_log`、超时参数）可以写在多个层级，**内层块的配置会覆盖外层**。比如 http 块里开了 gzip，某个 server 里写 `gzip off;` 就只关闭这个站点的压缩。但要注意 `access_log` 特殊：只要内层写了自己的 access_log，外层的就不再生效（不是合并）。

## 4. 静态资源服务

Nginx 最基础也最高频的用途：托管前端构建产物（Vue3 / React 的 dist 目录）。

### root 与 alias 的区别

两者都用来把 URL 映射到磁盘路径，但拼接规则不同，写错一个字母就是 404。

`root`：**磁盘路径 = root 值 + 完整 URI**。

```nginx
location /static/ {
    root /var/www/myapp;
    # 请求 /static/css/app.css
    # 实际文件 /var/www/myapp/static/css/app.css
}
```

`alias`：**磁盘路径 = alias 值 + URI 去掉 location 匹配部分后的剩余部分**。

```nginx
location /static/ {
    alias /var/www/myapp/assets/;
    # 请求 /static/css/app.css
    # 实际文件 /var/www/myapp/assets/css/app.css
    # 注意 URI 里的 /static/ 前缀被"替换"掉了
}
```

记忆口诀：**root 是"拼接"，alias 是"替换"**。实践经验：

- URL 前缀和目录名一致时（`/static/` → 目录里有 static 这层），用 root 更直观。
- URL 前缀只是对外伪装、真实目录名不同的时候，用 alias。
- 用 alias 时结尾斜杠要么都写要么都不写（location 和 alias 保持一致），否则路径拼接会出错。
- 前端 dist 整站托管直接用 root 指到 dist 目录即可。

### gzip 压缩

文本资源（HTML/CSS/JS/JSON）压缩后体积通常能缩小 70% 以上，对首屏加载速度影响巨大：

```nginx
http {
    gzip  on;                        # 开启 gzip
    gzip_min_length  1k;             # 小于 1KB 的响应不压缩（压缩有 CPU 成本，小文件不划算）
    gzip_comp_level  5;              # 压缩级别 1-9，越大压得越狠但越费 CPU，4-6 是甜点区
    gzip_vary  on;                   # 响应头加 Vary: Accept-Encoding，让中间缓存区分版本
    gzip_proxied  any;               # 即使请求经过代理也压缩（默认不压）
    gzip_types
        text/plain
        text/css
        application/json
        application/javascript
        text/xml
        application/xml
        image/svg+xml;               # 只压缩这些类型；图片/视频本身已压缩，压了白费 CPU
}
```

两个原理性要点：

1. **gzip 只对文本有效**。jpg/png/mp4 这些格式内部已经压缩过，再 gzip 几乎不变小，纯属浪费 CPU，所以 `gzip_types` 里不要加图片。
2. **压缩是在线实时进行的**，每次响应都要耗 CPU。更高级的做法是构建时预生成 `.gz` 文件，配合 `gzip_static on;` 让 Nginx 优先发送现成压缩包，零 CPU 开销：

```nginx
location / {
    gzip_static  on;   # 存在 app.js.gz 就直接发它，不再现场压缩
}
```

Vite 构建时可用 `vite-plugin-compression` 插件预生成 `.gz` 文件。

### expires 缓存头与 Cache-Control

前端构建产物里，带 hash 指纹的文件（如 `assets/index-a8f3c2.js`）内容变了文件名就会变，**可以放心让浏览器缓存一年**，这是前端性能优化里收益最大的一步：

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/myapp/dist;

    # 带 hash 的构建产物：强缓存一年
    location /assets/ {
        expires  1y;                              # 同时生成 Expires 和 Cache-Control: max-age=31536000
        add_header  Cache-Control  "public, immutable";   # immutable 告诉浏览器：别问，直接用
    }

    # 图片字体：缓存 30 天
    location ~* \.(jpg|jpeg|png|gif|ico|webp|woff2?)$ {
        expires  30d;
        add_header  Cache-Control  "public";
    }

    # index.html：不缓存，每次都要向服务器确认
    location = /index.html {
        add_header  Cache-Control  "no-cache, must-revalidate";
        expires  -1;
    }
}
```

为什么这样设计？核心是理解两种缓存验证方式：

- **强缓存**（`max-age`）：浏览器在有效期内直接用本地副本，**根本不发请求**，最快。适合内容不变（或文件名随内容变）的资源。
- **协商缓存**（`no-cache`）：浏览器每次发请求，服务器比较 ETag/Last-Modified，没变就回 304 Not Modified（只有头没有体）。适合 index.html——它引用了哪个 hash 的 JS 必须每次都确认，否则发新版后用户拿不到新入口。

`index.html` 不缓存 + 静态资源强缓存一年，这两个策略配合，就是"发版立即生效 + 加载极致快"的标准答案。

## 5. location 匹配规则

一个 server 里通常有多个 location，Nginx 收到请求后按**严格的优先级**选出唯一一个来处理。顺序不对是 404 和配置不生效的头号原因。

### 四种匹配方式

```nginx
location = /login { }        # 1. 精确匹配：URI 必须完全等于 /login
location ^~ /static/ { }     # 2. 优先前缀匹配：以 /static/ 开头，命中后不再检查正则
location ~ \.php$ { }        # 3. 正则匹配（区分大小写）
location ~* \.(jpg|png)$ { } # 4. 正则匹配（不区分大小写）
location / { }               # 5. 普通前缀匹配：最长前缀优先
```

### 匹配顺序（背下来）

Nginx 的决策流程：

1. **先查精确匹配 `=`**：有一个完全相等，立即使用，结束匹配。
2. **查普通前缀匹配**（包括 `^~` 和无修饰符的前缀）：找出**最长**的那个前缀匹配。
   - 如果这个最长前缀是 `^~` 修饰的 → 直接使用，**不再看正则**。
3. **按配置文件中的书写顺序**逐条检查正则 `~` / `~*`，**第一条命中的正则胜出**，结束匹配。
4. 没有任何正则命中 → 回退使用第 2 步找到的最长前缀匹配。

关键结论：**正则的优先级高于普通前缀，但低于 `^~` 和 `=`**；正则之间是"先到先得"（看书写顺序），前缀之间是"最长者胜"（与书写顺序无关）。

### 完整例子：跟着匹配过程走一遍

```nginx
server {
    listen 80;
    root /var/www/site;

    location = / {                 # A：精确匹配首页
        # ...
    }

    location / {                   # B：普通前缀（兜底）
        # ...
    }

    location /images/ {            # C：普通前缀
        # ...
    }

    location ^~ /static/ {         # D：优先前缀
        # ...
    }

    location ~* \.(gif|jpg|png)$ { # E：正则，图片
        # ...
    }

    location ~ \.php$ {            # F：正则
        # ...
    }
}
```

逐个 URI 分析：

| 请求 URI | 匹配过程 | 命中 |
|----------|----------|------|
| `/` | 第 1 步精确匹配直接命中 A | A |
| `/index.html` | 无精确匹配；最长前缀是 B；无 `^~`；检查正则：E、F 都不中 → 回退 B | B |
| `/images/logo.png` | 最长前缀是 C；C 不是 `^~`；正则 E 命中（.png）→ 用 E | E |
| `/static/logo.png` | 最长前缀是 D，D 是 `^~` → 直接用 D，**不看正则 E** | D |
| `/api/users.php` | 最长前缀是 B；正则 F 命中（.php，E 不命中因为 .php 不在列表）→ 用 F | F |

`/static/logo.png` 这条最能说明 `^~` 的价值：虽然它是图片，但因为我们想让它走静态目录规则而不是图片专用规则，`^~` 帮我们拦住了正则。

### 常见误区

- **location 的书写顺序只影响正则之间**，前缀匹配永远看长度不看顺序。
- `location = /api` 和 `location /api/` 是两个东西：前者只匹配 `/api` 这一个 URI，后者匹配 `/api/xxx`。
- 别名冲突：同层级的 location 中，正则从上到下第一条命中的生效，所以**范围窄的正则要写在范围宽的正则前面**。

## 6. 反向代理

反向代理是 Nginx 最核心的职能：客户端只跟 Nginx 说话，Nginx 再转发给内部的应用服务器。对客户端来说，后端是透明的——这就是"反向"（代理的是服务器）和"正向代理"（代理的是客户端，如翻墙工具）的区别。

### proxy_pass 基础

```nginx
server {
    listen 80;
    server_name example.com;

    location /api/ {
        proxy_pass  http://127.0.0.1:8000;
    }
}
```

请求 `GET /api/users` 到达 Nginx 后，被转发为 `http://127.0.0.1:8000/api/users`，FastAPI 收到的路径还是 `/api/users`。

### 结尾斜杠的坑

`proxy_pass` 后面**带不带 URI（哪怕只是一个 `/`）**，转发路径的规则完全不同——这是 Nginx 配置里最有名的坑：

```nginx
# 写法一：proxy_pass 后面没有 URI（只有主机和端口）
location /api/ {
    proxy_pass http://127.0.0.1:8000;
    # 请求 /api/users → 转发为 /api/users（原样转发，URI 不变）
}

# 写法二：proxy_pass 后面带了 URI（一个 / 也算！）
location /api/ {
    proxy_pass http://127.0.0.1:8000/;
    # 请求 /api/users → 转发为 /users（location 匹配到的 /api/ 被替换成 /）
}
```

规则一句话：**proxy_pass 带 URI 时做"替换"（类似 alias），不带时做"透传"（类似 root）**。

选哪种取决于后端路由怎么注册：

- FastAPI 路由注册的是 `/api/users`（带着前缀）→ 用写法一。
- FastAPI 路由注册的是 `/users`（不带前缀，API 前缀只存在于 Nginx 层）→ 用写法二。

转发路径和后端路由对不上，就会得到大片的 404，而且前端、后端日志看起来都"没错"，非常难查。遇到 404 先检查这里。

### proxy_set_header：传递 Host 与真实 IP

默认情况下，Nginx 转发请求时会**改写请求头**，后端看到的请求严重失真：

- `Host` 头变成 upstream 的名字（后端没法基于域名做判断）；
- 后端看到的客户端 IP 是 `127.0.0.1`（Nginx 自己），拿不到真实用户 IP——限流、日志、风控全部失效。

标准配置模板：

```nginx
location /api/ {
    proxy_pass  http://127.0.0.1:8000;

    # 把客户端请求的原始域名传给后端
    proxy_set_header  Host  $host;

    # 把真实客户端 IP 传给后端
    proxy_set_header  X-Real-IP  $remote_addr;

    # 追加到 X-Forwarded-For 链：客户端 → 经过的每个代理依次追加自己的来源 IP
    proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;

    # 告诉后端原始协议是 http 还是 https（后端生成重定向链接时需要）
    proxy_set_header  X-Forwarded-Proto  $scheme;
}
```

几个变量和头的原理：

- `$host`：请求 Host 头里的域名（不含端口）。
- `$remote_addr`：与 Nginx 建立 TCP 连接的对端 IP，即真实客户端 IP。
- `X-Forwarded-For`（XFF）：事实标准的代理链记录头。格式是 `客户端IP, 代理1IP, 代理2IP`。每经过一跳代理就追加一次。`$proxy_add_x_forwarded_for` 的含义就是"已有的 XFF 值 + 当前客户端 IP"，所以用追加而不是覆盖。
- **安全提醒**：XFF 是客户端可以伪造的普通请求头。只有最外层可信代理追加的最右侧之外的第一个 IP 才可信；后端做鉴权时不能无脑信任 XFF 的第一个值。

FastAPI 侧要相应配合——Starlette 会读取 `X-Forwarded-For` / `X-Forwarded-Proto`，但前提是 uvicorn 启动时声明信任代理头：

```bash
uvicorn main:app --proxy-headers --forwarded-allow-ips "127.0.0.1"
```

## 7. 负载均衡

一台后端扛不住时，开多个实例，让 Nginx 把请求分摊过去——这就是负载均衡，核心是 `upstream` 块。

### upstream 基础

```nginx
# 定义一组后端服务器（写在 http 块里，server 块外）
upstream  fastapi_backend {
    server  127.0.0.1:8000;
    server  127.0.0.1:8001;
    server  127.0.0.1:8002;
}

server {
    listen 80;

    location /api/ {
        proxy_pass  http://fastapi_backend;    # 用 upstream 名字代替具体地址
        proxy_set_header  Host  $host;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
}
```

upstream 里的 server 可以是本机不同端口，也可以是内网其他机器 `server 10.0.0.12:8000;`。对后端来说配置完全不用改，无感知扩容。

### 四种调度策略

**1. 轮询 round-robin（默认）**

不写任何策略就是轮询：请求按顺序轮流分发给每台服务器，适合后端性能均等、请求无状态的场景。

**2. 权重 weight**

机器配置不一样时，让强机器多扛请求：

```nginx
upstream fastapi_backend {
    server  127.0.0.1:8000  weight=3;   # 每 6 个请求分 3 个
    server  127.0.0.1:8001  weight=2;   # 分 2 个
    server  127.0.0.1:8002  weight=1;   # 分 1 个
}
```

weight 也常用于**灰度发布**：新版本先部署一台、给很小的 weight 观察，没问题再全量。

**3. ip_hash**

同一个客户端 IP 的哈希值固定映射到同一台后端：

```nginx
upstream fastapi_backend {
    ip_hash;
    server  127.0.0.1:8000;
    server  127.0.0.1:8001;
}
```

适用场景：后端把会话（session）存在本机内存里、没有做共享存储时，保证同一用户始终落到同一台机器，会话不丢失。**注意这只是过渡方案**：正确的做法是把 session 放 Redis 等共享存储里，后端无状态化，然后用轮询。ip_hash 的缺点很明显——后端重启或扩容会导致哈希重新映射，大量用户会话被打乱；且 NAT 下大量用户共享出口 IP，会导致分配不均。

**4. least_conn**

把新请求发给当前**活跃连接数最少**的后端：

```nginx
upstream fastapi_backend {
    least_conn;
    server  127.0.0.1:8000;
    server  127.0.0.1:8001;
}
```

适用场景：请求处理时长差异大（有的 10ms 有的 10s）。轮询是"不管后端忙不忙按顺序发"，least_conn 是"谁闲给谁发"，能避免慢请求在某台机器上堆积。

### 健康检查与故障转移

开源版 Nginx 提供**被动健康检查**（商业版 Nginx Plus 才有主动探活）：

```nginx
upstream fastapi_backend {
    server  127.0.0.1:8000  max_fails=3  fail_timeout=30s;
    server  127.0.0.1:8001  max_fails=3  fail_timeout=30s;
    server  127.0.0.1:8002  max_fails=3  fail_timeout=30s;
}
```

语义：某台后端在 30 秒内连续失败（连接拒绝、超时、5xx）3 次，Nginx 就把它标记为不可用，**30 秒内不再向它发请求**；30 秒后试探性地放一个请求过去，成功则恢复。这期间请求自动转发给其他健康节点——这就是故障转移。

另外两个常用参数：

```nginx
upstream fastapi_backend {
    server  127.0.0.1:8000  max_fails=3  fail_timeout=30s;
    server  127.0.0.1:8001  backup;       # 备用机：正常节点全挂时才启用
    server  127.0.0.1:8002  down;         # 临时摘除：不分配请求（下线维护用）
}
```

## 8. HTTPS 配置

### 证书申请概念

HTTPS = HTTP + TLS。要让浏览器信任你的站点，需要一个由**证书颁发机构（CA）**签发的数字证书，证书里绑定着你的域名和公钥。

主流获取方式：

- **Let's Encrypt（免费，最常用）**：通过 ACME 协议自动化申请，用 `certbot` 工具一键完成，证书有效期 90 天，certbot 会自动续期：

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
# certbot 会自动改你的 Nginx 配置、加好证书并 reload
```

- **云厂商免费证书**：阿里云、腾讯云等都提供免费 DV 证书，申请后下载 Nginx 格式（一个 `.pem`/`.crt` 证书文件 + 一个 `.key` 私钥文件）手动配置。
- **自签名证书**：仅限内网/测试，浏览器会警告不信任。

证书文件有两部分：**公钥证书**（可以公开，发给每个来访者）和**私钥**（必须保密，权限设为 600）。

### HTTPS server 配置

```nginx
server {
    listen       443  ssl;
    server_name  example.com;

    # 证书与私钥路径
    ssl_certificate      /etc/letsencrypt/live/example.com/fullchain.pem;   # 证书（含中间证书链）
    ssl_certificate_key  /etc/letsencrypt/live/example.com/privkey.pem;     # 私钥

    # TLS 安全参数
    ssl_protocols        TLSv1.2 TLSv1.3;    # 只允许 1.2 及以上，老版本 TLS1.0/1.1 有已知漏洞
    ssl_ciphers          HIGH:!aNULL:!MD5;   # 禁用空加密和 MD5 等弱算法
    ssl_prefer_server_ciphers  on;           # 服务端优先选择加密套件（TLS1.2 时生效）

    # 会话复用：减少 TLS 握手开销（一次完整握手要 2 个 RTT，复用只要 1 个）
    ssl_session_cache    shared:SSL:10m;     # 所有 worker 共享 10MB 会话缓存（约存 4 万个会话）
    ssl_session_timeout  10m;

    root  /var/www/myapp/dist;
    # ... 其余 location 配置
}
```

要点解释：

- `fullchain.pem` 而不是 `cert.pem`：fullchain 包含你的证书 + 中间 CA 证书的完整链条，缺了中间证书部分浏览器（尤其安卓老机型）会报不信任。
- TLS 握手是整个 HTTPS 里最贵的环节（非对称加密运算 + 额外网络往返），`ssl_session_cache` 让重复来访者跳过完整握手，收益明显。
- 私钥泄露 = 证书作废，必须 `chmod 600 privkey.pem`。

### HTTP 301 跳转 HTTPS

把所有 HTTP 流量永久重定向到 HTTPS：

```nginx
server {
    listen  80;
    server_name  example.com;

    # 301 永久重定向，浏览器会记住并缓存这个跳转
    return  301  https://$host$request_uri;
}
```

- 用 **301**（永久）而不是 302（临时）：301 会被浏览器缓存，下次用户输 HTTP 地址浏览器自己就跳了，连请求都不发；同时搜索引擎会把权重转移到 HTTPS 版本。
- `$request_uri` 保留完整路径和查询参数，`/docs?id=1` 会跳到 `https://.../docs?id=1`，不会丢参数。

## 9. 完整实战：Vue3 前端 + FastAPI 后端同机部署

这是把前面所有知识点串起来的真实场景：

- Vue3 项目 `npm run build` 产物在 `/var/www/myapp/dist`
- FastAPI 由 uvicorn 跑在 `127.0.0.1:8000`（路由带 `/api` 前缀）
- 前端用了 Vue Router 的 history 模式
- 有 WebSocket 实时功能（`/api/ws`）

```nginx
# /etc/nginx/conf.d/myapp.conf

# ---- HTTP：301 跳转到 HTTPS ----
server {
    listen  80;
    server_name  example.com;
    return  301  https://$host$request_uri;
}

# ---- HTTPS 主站 ----
server {
    listen       443  ssl;
    server_name  example.com;

    # ===== HTTPS 证书与 TLS 参数 =====
    ssl_certificate      /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols        TLSv1.2 TLSv1.3;
    ssl_ciphers          HIGH:!aNULL:!MD5;
    ssl_session_cache    shared:SSL:10m;

    # ===== 安全响应头（可选但推荐）=====
    add_header  X-Content-Type-Options  nosniff always;          # 禁止 MIME 嗅探
    add_header  X-Frame-Options         DENY always;             # 禁止被 iframe 嵌套（防点击劫持）
    add_header  Referrer-Policy         strict-origin-when-cross-origin always;

    # 上传文件大小上限（详见第 10 节）
    client_max_body_size  20m;

    # ===== 前端静态资源（Vue3 dist 目录）=====
    root   /var/www/myapp/dist;
    index  index.html;

    # 带 hash 指纹的构建产物：强缓存一年
    location /assets/ {
        expires  1y;
        add_header  Cache-Control  "public, immutable";
    }

    # ===== 反向代理：/api 转发给 FastAPI =====
    location /api/ {
        # proxy_pass 不带 URI：/api/users 原样转发为 /api/users
        proxy_pass  http://127.0.0.1:8000;

        # 传递真实客户端信息
        proxy_set_header  Host              $host;
        proxy_set_header  X-Real-IP         $remote_addr;
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;

        # ===== WebSocket 支持（FastAPI 的 /api/ws 走这里）=====
        # WebSocket 握手是一个带 Upgrade 头的 HTTP 请求，
        # 必须显式把 Upgrade/Connection 头传给后端，否则握手失败
        proxy_http_version  1.1;                        # WebSocket 要求 HTTP/1.1
        proxy_set_header    Upgrade    $http_upgrade;   # 透传 Upgrade: websocket
        proxy_set_header    Connection "upgrade";       # 把 Connection 头固定设为 upgrade

        # WebSocket 连接通常长时间没有数据，必须调大读超时，
        # 否则 Nginx 默认 60s 无数据就断开连接
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }

    # ===== Vue Router history 模式支持 =====
    location / {
        # 依次尝试：请求的文件本身 → 同名目录 → 回退到 index.html
        # history 模式下 /user/123 这样的路径在磁盘上不存在，
        # 必须交给 index.html 让前端路由接管，否则刷新就是 404
        try_files  $uri  $uri/  /index.html;
    }

    # index.html 不缓存，保证发版后立即生效
    location = /index.html {
        add_header  Cache-Control  "no-cache, must-revalidate";
        expires  -1;
    }
}
```

### 关键设计点讲解

**1. 为什么 history 路由需要 try_files？**

Vue Router 的 history 模式下，URL 是干净的 `/user/123`（没有 `#`）。用户从这个路径刷新浏览器时，浏览器会直接向服务器请求 `/user/123`——但磁盘上只有 `index.html`，没有 `user/123` 这个文件，于是 404。

`try_files $uri $uri/ /index.html;` 的语义：**先找磁盘上真实存在的文件（JS/CSS/图片都走这里），找不到就把请求内部重写到 `/index.html`**，由 Vue Router 在前端解析路径、渲染对应页面。

如果路由用 hash 模式（URL 里带 `#`），`#` 后面的部分根本不会发给服务器，就不需要这行——但 hash 模式对 SEO 不友好、URL 丑，生产上基本都用 history 模式。

**2. 为什么 WebSocket 需要特殊配置？**

WebSocket 的生命周期是：先用 HTTP 发起一个带 `Upgrade: websocket` 和 `Connection: Upgrade` 头的握手请求，服务器回复 `101 Switching Protocols` 后，这条 TCP 连接就"升级"为全双工的 WebSocket。

问题在于：Nginx 做反向代理时默认会"规整"请求头——`Connection` 头会被改掉，后端收不到升级意图，握手失败。所以要显式：

- `proxy_http_version 1.1`：proxy_pass 默认用 HTTP/1.0 连后端，而 Upgrade 机制是 1.1 的特性。
- `proxy_set_header Upgrade $http_upgrade;`：把客户端的 Upgrade 头原样透传。
- `proxy_set_header Connection "upgrade";`：把 Connection 头设置为 upgrade。
- `proxy_read_timeout 3600s`：WebSocket 连接空闲时没有任何数据流动，Nginx 默认 60 秒没读到后端数据就判定超时断连。要么调大超时，要么在应用层做心跳保活（生产上两者都做）。

**3. 部署命令串联**

```bash
# 前端：本地构建，产物上传到服务器
npm run build
scp -r dist/ user@server:/var/www/myapp/

# 后端：安装依赖、启动（生产建议用 systemd 守护）
pip install -r requirements.txt
uvicorn main:app --host 127.0.0.1 --port 8000 --workers 4 --proxy-headers

# Nginx：检查配置并优雅重载
sudo nginx -t && sudo systemctl reload nginx
```

注意 uvicorn 监听 `127.0.0.1` 而不是 `0.0.0.0`：后端只接受本机 Nginx 的转发，不直接对外暴露端口，少一层攻击面。

## 10. 运维技巧

### 日志分析

Nginx 的两类日志：

- **access_log**：每条请求一行，记录谁在什么时候请求了什么、结果如何。
- **error_log**：Nginx 自身运行和代理过程中的错误，排查故障第一现场。

自定义日志格式，把对排查最有价值的字段加进去：

```nginx
log_format  main  '$remote_addr - [$time_local] "$request" '
                  '$status $body_bytes_sent '
                  '"$http_referer" "$http_user_agent" '
                  'rt=$request_time uct=$upstream_connect_time '
                  'urt=$upstream_response_time';

access_log  /var/log/nginx/access.log  main;
```

关键变量：

- `$request_time`：从收到请求到响应发完的**总耗时**（含客户端网络慢的影响）。
- `$upstream_response_time`：Nginx 等**后端**响应花了多久。`request_time` 远大于它 → 瓶颈在客户端网络；它远大于预期 → 瓶颈在后端应用。
- `$status`：状态码，5xx 盯后端，4xx 盯客户端/前端。

常用分析命令（access.log 是一行一条记录，天然适合 awk）：

```bash
# 实时盯着日志
tail -f /var/log/nginx/access.log

# 统计访问最多的前 10 个 IP
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# 统计状态码分布（看 5xx 占比）
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 找出响应最慢的请求
awk '{print $NF, $7}' /var/log/nginx/access.log | sort -rn | head -20

# 查某个时间段内的错误
grep "2024/01/15:1[45]" /var/log/nginx/error.log
```

### limit_req 限流

防止接口被刷、扛住突发流量，Nginx 用**漏桶算法**限流：

```nginx
http {
    # 定义限流区：以客户端 IP 为 key，共享内存 10MB（约存 16 万个 IP 状态）
    # rate=10r/s 表示平均每秒放行 10 个请求
    limit_req_zone  $binary_remote_addr  zone=api_limit:10m  rate=10r/s;
}

server {
    location /api/ {
        proxy_pass  http://127.0.0.1:8000;

        # burst=20：允许突发 20 个请求排队等待（平滑峰值）
        # nodelay：排队的请求立即处理，配合 burst 使用体验更好
        limit_req  zone=api_limit  burst=20  nodelay;

        # 被限流时返回 429（默认是 503，429 Too Many Requests 语义更准确）
        limit_req_status  429;
    }
}
```

工作原理：`rate=10r/s` 意味着每 100ms 放行一个请求。超出速率的请求进"桶"（队列）排队，队列长度就是 `burst`；桶也满了就直接拒绝。**不加 nodelay 时排队的请求会被延迟**（严格按速率出水），加 nodelay 则立即放行、用后续的空闲额度偿还，实际响应更快。

### client_max_body_size：上传大小限制

Nginx 默认只接受 **1MB** 以内的请求体，用户上传稍大一点的图片就会收到 `413 Request Entity Too Large`——这个错发生在 Nginx 层，请求根本到不了后端，后端日志里什么都看不到，是个经典陷阱：

```nginx
server {
    client_max_body_size  20m;    # 允许最大 20MB 的请求体

    # 相关参数：请求体先放内存缓冲区，超出的部分写临时文件
    client_body_buffer_size  128k;
}
```

### 超时参数

生产环境建议显式配置这几组超时，避免连接被无限占用：

```nginx
http {
    # ===== 面向客户端（下游）=====
    client_header_timeout  10s;    # 客户端必须在 10s 内发完请求头（防 Slowloris 慢速攻击）
    client_body_timeout    30s;    # 客户端发送请求体的间隔超时
    keepalive_timeout      65s;    # 长连接空闲 65s 后关闭
    send_timeout           30s;    # 向客户端发送响应的间隔超时

    # ===== 面向后端（上游）=====
    proxy_connect_timeout  5s;     # 与后端建立 TCP 连接的超时（内网应该很快，5s 足够）
    proxy_send_timeout     30s;    # 向后端发送请求的超时
    proxy_read_timeout     60s;    # 等待后端响应的超时（有慢接口或 WebSocket 要单独调大）
}
```

`proxy_read_timeout` 是最常被问到的一个：后端有个要跑 5 分钟的报表导出接口，60 秒时 Nginx 就断开报 504 了。解法是把那个 location 的 `proxy_read_timeout` 单独调大，或者（更推荐）把长任务改成异步轮询模式。

### 502 / 504 排查思路

这两个是 Nginx 运维中出现率最高的错误，含义和排查路径完全不同，必须分清：

**502 Bad Gateway = Nginx 连不上后端，或后端"说了不该说的话"**

Nginx 作为网关去请求后端，得到了一个**无法理解的响应**或根本连不上。排查步骤：

1. **看 error_log**，错误原因写得很直白：
   - `connect() failed (111: Connection refused)` → 后端进程挂了或没监听这个端口。`systemctl status 后端服务`、先 `ss -tlnp | grep 8000` 确认端口在听。
   - `upstream prematurely closed connection` → 后端处理中崩溃（比如 FastAPI 未捕获异常导致 worker 被杀）。去查后端自己的日志。
   - `Permission denied` → 多半是 SELinux 拦了 Nginx 的出站连接（CentOS/Rocky 常见），`setsebool -P httpd_can_network_connect 1`。
2. 手动验证后端是否健康：`curl -v http://127.0.0.1:8000/health`。
3. 负载均衡场景：可能是 upstream 里某一台挂了，间歇性 502。检查每台后端。

**504 Gateway Timeout = 连上了，但后端太慢没回**

Nginx 在 `proxy_read_timeout` 内没等到后端的响应。排查步骤：

1. 确认是"全都慢"还是"个别接口慢"：看 access_log 里的 `urt=$upstream_response_time`，按耗时排序。
2. 个别接口慢 → 这是后端性能问题，不是 Nginx 问题：查慢 SQL、N+1 查询、缺索引、同步 IO 阻塞事件循环。
3. 全站都慢 → 后端资源耗尽：CPU 100%？数据库连接池打满？worker 数量不够？
4. 确认接口本身就是长任务 → 单独调大该 location 的 `proxy_read_timeout`，或改异步架构。

一张排查决策图：

```
用户报障
   │
   ├─ 502 ──> 后端进程活着吗？端口在听吗？
   │            ├─ 没起来 → 拉服务、查后端崩溃日志
   │            └─ 活着 → SELinux / 防火墙 / 后端返回异常
   │
   ├─ 504 ──> upstream_response_time 普遍大吗？
   │            ├─ 是 → 后端资源（CPU/DB/连接池）瓶颈
   │            └─ 个别接口 → 慢查询、阻塞代码，或调大超时
   │
   └─ 413/429 ──> 不是故障：client_max_body_size / limit_req 配置生效了
```

## 小结

- Nginx 的高并发能力来自 **master-worker 进程模型 + epoll 事件驱动 + 非阻塞 IO + sendfile 零拷贝**，单线程 worker 就能hold住上万连接。
- 改配置的黄金流程：`nginx -t` 验证 → `reload` 优雅生效，新旧 worker 平滑交接，零停机。
- 配置层级是 main → events → http → server → location，内层覆盖外层；站点配置用 include 拆分成独立文件。
- 静态服务三件套：`root`/`alias`（拼接 vs 替换）、`gzip`（只压文本）、缓存策略（hash 文件强缓存一年 + index.html 不缓存）。
- location 匹配顺序：`=` > `^~` > 正则（按书写顺序）> 普通前缀（最长者胜）。
- `proxy_pass` 结尾带不带 `/` 决定转发路径是"替换"还是"透传"；必须配 `proxy_set_header` 传递 Host 和真实 IP。
- upstream 支持轮询、weight、ip_hash、least_conn 四种策略，被动健康检查实现故障摘除。
- 实战部署的核心配置：`/api` 代理到后端、`try_files` 支持 history 路由、Upgrade 头支持 WebSocket。
- 运维抓四件事：日志字段（尤其 `upstream_response_time`）、`limit_req` 限流、`client_max_body_size`、以及分清 502（连不上）和 504（太慢）的排查路径。

## 常见问题

**1. reload 和 restart 有什么区别？生产环境用哪个？**

restart 会先杀掉所有进程再启动，期间服务完全中断，已建立的连接全部断开。reload 是向 master 发 HUP 信号：master 校验新配置 → 启动新 worker → 旧 worker 处理完现有请求后优雅退出，全程零停机、零丢连接。生产环境改配置**只用 reload**，并且先 `nginx -t` 验证；配置有错误时 reload 会被安全地放弃，旧配置继续生效，而 restart 会导致服务起不来。

**2. root 和 alias 到底怎么选？**

记住映射规则：root 是 `root值 + 完整URI`，alias 是 `alias值 + URI去掉location前缀`。当 URL 前缀和磁盘目录结构一致时用 root（如 `location /static/` 对应 `/var/www/app/static/`）；当 URL 前缀只是对外伪装、真实目录名不同时用 alias（如 `location /download/` 指向 `/data/files/`）。用 alias 时注意 location 和 alias 的路径要么都以 `/` 结尾要么都不以，避免拼接出错误路径。

**3. 后端为什么拿不到用户的真实 IP？怎么修？**

因为请求经过 Nginx 反向代理后，与后端建立 TCP 连接的是 Nginx 自己，后端看到的对端 IP 是 127.0.0.1。修法分两步：Nginx 侧加 `proxy_set_header X-Real-IP $remote_addr;` 和 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`；后端侧读取这两个头（FastAPI/uvicorn 需要启动时加 `--proxy-headers`）。注意 X-Forwarded-For 可被客户端伪造，只有可信代理追加的部分才可信。

**4. Vue history 模式部署后，刷新页面 404，怎么解决？**

history 模式下 `/user/123` 这样的路径在磁盘上没有对应文件，刷新时浏览器直接请求服务器就会 404。在静态资源的 location 里加一行 `try_files $uri $uri/ /index.html;`：先找真实文件（JS/CSS 不受影响），找不到就回退到 index.html 交给前端路由处理。同时注意 index.html 要设 `Cache-Control: no-cache` 不缓存，否则发新版后用户拿不到新入口。

**5. 502 和 504 有什么区别？拿到报错先做什么？**

502 Bad Gateway 是 Nginx **连不上**后端或后端返回了非法响应——查后端进程是否存活、端口是否监听、SELinux 是否拦截。504 Gateway Timeout 是**连上了但后端在超时时间内没响应完**——查后端慢在哪里（慢 SQL、资源耗尽），或确认是否是需要调大 `proxy_read_timeout` 的长任务。两者的第一手资料都是 `/var/log/nginx/error.log`，里面明确写着 `Connection refused`、`upstream timed out` 等具体原因，先看日志再动手。

**6. WebSocket 经过 Nginx 代理就连不上，缺什么配置？**

WebSocket 握手依赖 HTTP/1.1 的 `Upgrade: websocket` 和 `Connection: Upgrade` 头，而 Nginx 反代默认用 HTTP/1.0 且会改写 Connection 头。需要在 location 里加三行：`proxy_http_version 1.1;`、`proxy_set_header Upgrade $http_upgrade;`、`proxy_set_header Connection "upgrade";`。另外 WebSocket 连接空闲时没有数据流，记得调大 `proxy_read_timeout`（如 3600s），否则默认 60 秒就被 Nginx 断开。
