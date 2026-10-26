# Python 基础语法

> 目标读者：已经会至少一门编程语言（如 Java、C、JavaScript），但没有系统学过 Python 的自学者。
> 因此本章不解释"什么是变量""什么是循环"这类通用概念，而是把篇幅花在 **Python 特有的机制、设计哲学和坑** 上：为什么字符串不可变、可变对象作为默认参数为什么会出事、生成器为什么省内存、`__name__ == '__main__'` 到底在判断什么……
> 学完本章，你将具备阅读与编写中型 Python 项目的全部语法基础，并为后续学习 FastAPI、SQLAlchemy 等现代框架（它们大量依赖类型注解、装饰器、上下文管理器）扫清障碍。

## 本章导读

- 环境搭建与虚拟环境：`venv`、`pip`、国内镜像源、`requirements.txt`，以及 `pipx` / `poetry` / `uv` 的定位
- 基本数据类型：`int` / `float` / `str` / `bool` / `None`，字符串不可变性与常用方法，`f-string` 格式化
- 容器类型：`list` / `tuple` / `dict` / `set`、四种推导式、可变与不可变、深浅拷贝的底层原理
- 控制流：`if` / `while` / `for`、少有人讲透的 `else` 子句、`match-case` 结构模式匹配
- 函数：参数体系（位置/关键字/默认/`*args`/`**kwargs`）、`lambda`、LEGB 作用域规则、闭包的形成与用途
- 装饰器：函数是一等公民意味着什么、`@` 语法糖的本质、带参数装饰器、`functools.wraps` 解决什么问题
- 类与面向对象：`__init__`、类变量与实例变量的区别、继承与 MRO 查找顺序、`@property`、`dataclass`、`__str__` 与 `__repr__`
- 异常处理：`try/except/else/finally` 完整结构、`raise` 与异常链、自定义异常、`with` 上下文管理器的 `__enter__` / `__exit__` 原理
- 迭代器与生成器：`__iter__` / `__next__` 协议、`yield` 的执行机制、生成器为什么省内存
- 模块与包：`import` 时到底发生了什么、`__name__ == '__main__'` 的意义、`__init__.py` 的作用、相对导入
- 文件 IO 与 `json` / `pathlib`：现代 Python 的路径与文件处理方式
- 类型注解：`typing` 模块的常用类型，为 FastAPI 打基础
- 常用标准库速览：`os` / `sys` / `datetime` / `collections` / `itertools` / `re` / `logging` / `json`

---

## 1. 环境搭建与虚拟环境

### 1.1 为什么必须搞懂虚拟环境

如果你写过 Node.js，会习惯"每个项目一个 `node_modules`"的隔离方式。Python 默认不是这样：`pip install` 默认把包装到 **解释器级别的全局 site-packages 目录** 里。这会带来两个经典灾难：

1. **版本冲突**：项目 A 依赖 `requests==2.25`，项目 B 需要 `requests==2.31`，全局环境只能装一个。
2. **环境不可复现**：你机器上全局装了 50 个包，代码扔到服务器上，根本说不清到底依赖哪些。

虚拟环境（virtual environment）就是 Python 世界的"每个项目一个 `node_modules`"：它在项目目录下创建一套 **独立的 Python 目录结构**（独立的解释器入口 + 独立的 site-packages），激活后 `pip install` 只装进这套目录，互不干扰。

### 1.2 使用内置的 `venv` 模块

`venv` 是 Python 3.3 起内置的标准库模块，无需安装任何东西。推荐始终用 `python -m venv` 这种 `-m` 形式调用——`-m` 表示"把模块当脚本运行"，能确保你用的是 `python` 命令当前指向的那个解释器，避免系统里装了多个 Python 版本时张冠李戴。

```bash
# 在项目根目录下创建名为 .venv 的虚拟环境（.venv 是社区惯例命名）
python -m venv .venv

# 激活虚拟环境
# Windows (Git Bash):
source .venv/Scripts/activate
# Windows (CMD):
.venv\Scripts\activate.bat
# Windows (PowerShell):
.venv\Scripts\Activate.ps1
# macOS / Linux:
source .venv/bin/activate

# 激活后命令行提示符前会出现 (.venv) 前缀，此后：
python --version      # 指向虚拟环境里的解释器
pip install requests  # 只装进 .venv，不污染全局

# 退出虚拟环境
deactivate
```

**底层原理**：`venv` 并没有复制一套完整的 Python。它创建了一个目录，里面有指向系统解释器的符号链接（Windows 上是小的启动器副本），以及一个 `pyvenv.cfg` 配置文件。激活脚本做的核心事情只是 **修改 `PATH` 环境变量**，把 `.venv/Scripts`（或 `bin`）放到 `PATH` 最前面，于是 `python`、`pip` 命令优先命中虚拟环境里的可执行文件。理解这一点后，"激活"就不再神秘——你甚至可以不激活，直接调用 `.venv/Scripts/python.exe` 运行脚本，效果一样。

**注意**：`.venv` 目录体积大、与机器强相关，**绝不能提交到 Git**。记得在 `.gitignore` 中加入 `.venv/`。

### 1.3 pip 与国内镜像源

`pip` 是 Python 的包管理器，从 PyPI（Python Package Index）下载并安装第三方库。常用命令：

```bash
pip install requests             # 安装最新版
pip install "requests==2.32.3"   # 安装指定版本（生产环境务必锁定版本）
pip install "requests>=2.28"     # 版本范围
pip install --upgrade requests   # 升级
pip uninstall requests           # 卸载
pip list                         # 查看已安装的包
pip show requests                # 查看某个包的详细信息
```

由于 PyPI 服务器在国外，国内直连经常超时。解决方案是配置 **国内镜像源**。临时使用：

```bash
pip install requests -i https://pypi.tuna.tsinghua.edu.cn/simple
```

永久配置（推荐）：

```bash
# 清华源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

这条命令会写入 pip 的配置文件（Windows 在 `%APPDATA%\pip\pip.ini`，macOS/Linux 在 `~/.pip/pip.conf` 或 `~/.config/pip/pip.conf`），之后所有 `pip install` 都自动走镜像。常用镜像还有阿里云 `https://mirrors.aliyun.com/pypi/simple/`、中科大 `https://pypi.mirrors.ustc.edu.cn/simple/`。

### 1.4 requirements.txt：锁定依赖

团队协作和部署时需要一份"依赖清单"，让任何机器都能一键还原相同环境：

```bash
# 导出当前环境所有包及其精确版本
pip freeze > requirements.txt

# 在新环境一键安装
pip install -r requirements.txt
```

`requirements.txt` 长这样：

```text
requests==2.32.3
fastapi==0.115.0
uvicorn==0.30.6
```

`pip freeze` 会把 **间接依赖**（你的依赖的依赖）也一并列出并锁定精确版本，这对解决"在我机器上能跑、服务器上跑不了"问题是最好的解药。缺点是它不分直接依赖和间接依赖，想精细管理时可以手工维护这个文件，只写直接依赖。

### 1.5 进阶工具速览：pipx / poetry / uv

`venv + pip + requirements.txt` 是官方内置方案，足够用很多年。但社区针对它的痛点（不区分开发依赖与生产依赖、版本解析慢、没有锁文件保证完全一致的安装结果）发展出了更好的工具：

- **pipx**：解决的不是项目依赖，而是 **安装命令行工具** 的问题。比如 `black`、`httpie` 这类工具，你想全局使用，但又不想污染全局 site-packages。`pipx install black` 会为每个工具自动创建独立虚拟环境并把命令链接到全局 PATH。可以理解为"Python 版的全局 `npm install -g`，但带隔离"。
- **poetry**：对标 Node.js 的 `npm` 全家桶体验。用 `pyproject.toml` 声明依赖（区分主依赖与 dev 依赖），自动生成精确的 `poetry.lock` 锁文件，内置打包发布功能。常用命令：`poetry add requests`、`poetry install`。
- **uv**：Astral 公司（`ruff` 的作者）用 Rust 写的包管理器，近年迅速成为社区新宠。它把 `pip`、`venv`、`pipx`、`pyenv` 的功能整合到一个极快的二进制工具里，安装包的速度比 pip 快一到两个数量级。用法上对 pip 用户几乎零学习成本：

```bash
uv venv                  # 创建虚拟环境（替代 python -m venv）
uv pip install requests  # 安装包（替代 pip install，快很多）
uv add requests          # 项目模式：自动管理 pyproject.toml 与锁文件
```

**选型建议**：初学阶段用内置的 `venv + pip` 把原理搞明白；团队协作或新项目可以直接上 `uv`，它是目前的社区主流方向。

---

## 2. 基本数据类型

### 2.1 一切皆对象，变量是"标签"

理解 Python 数据类型前，先要建立一个与其他语言不同的底层认知：**Python 中一切皆对象，变量不是装值的盒子，而是贴在对象上的标签（名字绑定）**。

```python
a = 1000
b = a
```

在 C 语言里，`b = a` 会把值复制一份放进 `b` 的内存。在 Python 里，它只是让 `b` 这个标签也指向 `a` 指向的那个 `int` 对象——**没有发生任何拷贝**。可以用 `id()` 查看对象的身份（内存地址）来验证：

```python
a = 1000
b = a
print(id(a) == id(b))  # True，两个名字指向同一个对象
```

这个模型是理解后面"可变性""深浅拷贝""函数传参"所有问题的钥匙。

### 2.2 int 与 float

Python 的 `int` 有一个杀手级特性：**任意精度，不会溢出**。

```python
big = 2 ** 100
print(big)  # 1267650600228229401496703205376 —— 在 C/Java 里早就溢出了
```

底层上 CPython 把超出机器字长的整数自动转成"长整型"对象存储（一个变长的数字数组），代价是大整数运算比固定字长的整数慢，但对业务代码完全无感。

`float` 是 IEEE 754 双精度浮点数，和其他语言一样存在精度问题——这不是 Python 的 bug，而是浮点数的本质：

```python
print(0.1 + 0.2 == 0.3)  # False！
print(0.1 + 0.2)         # 0.30000000000000004
```

财务计算请用标准库 `decimal.Decimal`，科学计算注意误差累积。

常用运算：

```python
print(7 / 2)       # 3.5    —— / 永远是"真除法"，结果是 float
print(7 // 2)      # 3      —— // 整除（向下取整）
print(7 % 2)       # 1      —— 取余
print(2 ** 10)     # 1024   —— ** 幂运算
print(divmod(7, 2))  # (3, 1)，同时拿到商和余数
```

### 2.3 bool：它是 int 的子类

一个冷知识：`bool` 继承自 `int`，`True == 1`、`False == 0` 成立，甚至可以 `True + True == 2`。这个设计有历史原因（Python 早期没有 bool 类型），实用价值在于：

```python
results = [True, False, True, True]
passed = sum(results)  # 3 —— 利用 True 就是 1 来计数
```

更重要的是 **真值测试（truthiness）**：Python 中任何对象都有真假值。以下值在布尔上下文中为 `False`：`False`、`None`、`0`、`0.0`、空字符串 `""`、空容器 `[]`、`()`、`{}`、`set()`；其余一切为 `True`。这让判空写法极其简洁：

```python
items = []
if not items:          # 而不是 if len(items) == 0:
    print("列表为空")
```

### 2.4 None：空值的唯一表示

`None` 是 `NoneType` 类型的唯一实例（单例），表示"没有值"。它是 Python 版的 `null`，但有两点必须注意：

1. 判断是否为 `None` **必须用 `is` 而不是 `==`**。因为 `is` 比较的是对象身份（内存地址），而 `==` 调用的是对象自定义的 `__eq__` 方法——某个类的 `__eq__` 可能被写得很奇怪，导致 `x == None` 意外为真。`is None` 则永远可靠。
2. 函数不显式 `return` 时，默认返回 `None`。

```python
def f():
    pass

result = f()
print(result is None)  # True
```

### 2.5 str：不可变的文本序列

**字符串不可变（immutable）** 是 Python 的核心设计之一：字符串对象一旦创建，它指向的那段字符序列就永远无法修改。所有看似"修改"字符串的方法，实际上都是 **返回一个新字符串对象**。

为什么要这样设计？原因有三：

1. **安全共享**：不可变对象可以被任意多个变量、多个数据结构引用而不用担心被悄悄改掉，因此字符串可以安全地作为 `dict` 的键、`set` 的元素。
2. **哈希可缓存**：字符串的哈希值只需计算一次然后缓存在对象里，`dict` / `set` 查找极快。
3. **实现优化**：解释器可以安全地复用相同的字符串字面量对象（驻留机制），节省内存。

由此推出一个新手最常犯的性能错误——在循环里用 `+` 拼接字符串：

```python
# 差：每次循环都创建一个全新的字符串对象并拷贝之前的全部内容，整体 O(n²)
s = ""
for i in range(10000):
    s += str(i)

# 好：join 预先算出总长度，一次分配内存，O(n)
s = "".join(str(i) for i in range(10000))
```

常用方法（全部返回新对象，原字符串不变）：

```python
s = "  Hello, Python World  "

print(s.strip())                  # 去掉两端空白 → "Hello, Python World"
print(s.lower())                  # 全小写
print(s.upper())                  # 全大写
print(s.title())                  # 每个单词首字母大写
print(s.replace("Python", "Go"))  # 替换
print(s.split(", "))              # 按分隔符切分 → ['  Hello', 'Python World  ']
print(s.startswith("  He"))       # True
print(s.endswith("d  "))          # True
print("Python" in s)              # True —— in 运算符做子串判断
print(s.find("Python"))           # 9，找不到返回 -1
print(s.index("Python"))          # 9，找不到抛 ValueError

parts = ["2024", "01", "15"]
print("-".join(parts))            # "2024-01-15"，join 是字符串的方法，参数是可迭代对象
```

字符串切片是序列通用操作（`list`、`tuple` 同样适用），语法 `s[start:stop:step]`，**含头不含尾**：

```python
s = "Python"
print(s[0])     # 'P'，正向索引从 0 开始
print(s[-1])    # 'n'，负索引 -1 是最后一个字符
print(s[0:3])   # 'Pyt'
print(s[:3])    # 'Pyt'，start 省略表示从头
print(s[3:])    # 'hon'，stop 省略表示到尾
print(s[::-1])  # 'nohtyP'，步长为 -1 即反转，面试高频
print(s[::2])   # 'Pto'，隔一个取一个
```

### 2.6 f-string：字符串格式化

Python 历史上有三代格式化写法：`%` 操作符（C 风格，已过时）、`str.format()` 方法、`f-string`（Python 3.6+，**当前唯一推荐**）。f-string 在字符串前加 `f`，花括号里可以直接写 **任意表达式**，在运行时求值：

```python
name = "Alice"
age = 25
price = 3.14159

print(f"我叫 {name}，明年 {age + 1} 岁")   # 表达式直接在 {} 内求值
print(f"价格：{price:.2f}")                # 保留两位小数 → 3.14
print(f"大数字：{123456789:,}")            # 千分位 → 123,456,789
print(f"百分比：{0.856:.1%}")              # → 85.6%
print(f"右对齐：'{name:>10}'")             # 宽度 10，右对齐
print(f"居中：'{name:^10}'")               # 居中
print(f"补零：{42:05d}")                   # → 00042

# 调试神器：Python 3.8+ 支持 = 说明符，同时打印表达式和值
x, y = 10, 20
print(f"{x=} {y=} {x + y=}")  # x=10 y=20 x + y=30
```

格式说明符的语法是 `{表达式:格式规范}`，格式规范依次为 `[[填充]对齐][符号][0][宽度][,][.精度][类型]`。记不住全部没关系，掌握 `.2f`、`,`、`:>10`、`:05d`、`=` 这几个就覆盖了 90% 的场景。

---

## 3. 容器类型

### 3.1 可变 vs 不可变：本节最重要的概念

Python 容器分为两类，这个划分影响深远：

- **可变（mutable）**：`list`、`dict`、`set` —— 创建后可以原地增删改，对象身份（`id`）不变。
- **不可变（immutable）**：`tuple`、`str`、`int`、`frozenset` —— 创建后内容不可改，任何"修改"都产生新对象。

为什么这个划分如此重要？

1. **只有不可变（可哈希）对象才能做 `dict` 的键和 `set` 的元素**，因为哈希表靠键的哈希值定位；若键的内容能变，哈希值就会变，字典就再也找不到它了。所以 `{(1, 2): "ok"}` 合法，`{[1, 2]: "no"}` 直接抛 `TypeError: unhashable type: 'list'`。
2. **函数传参的行为差异**：函数接收的是对象的引用（标签的复制）。传可变对象进函数，函数内部的修改会 **影响到外部**；传不可变对象则不会。这是无数 bug 的来源：

```python
def add_item(item, bag):
    bag.append(item)   # 原地修改，调用方的列表也被改了！
    return bag

my_bag = [1, 2]
add_item(3, my_bag)
print(my_bag)  # [1, 2, 3] —— 函数外的数据被污染
```

### 3.2 list：动态数组

`list` 底层是 **动态数组**（一块连续的指针数组，存的是对象的引用）。因此按下标随机访问是 O(1)，`append` 均摊 O(1)（容量不够时整体扩容搬迁），但 `insert(0, x)`、`pop(0)` 需要移动后面所有元素，是 O(n)。如果需要频繁在头部操作，应该用 `collections.deque`（双端队列，两端操作都是 O(1)，见第 13 节）。

```python
nums = [3, 1, 4, 1, 5, 9, 2, 6]

nums.append(7)        # 尾部追加
nums.insert(0, 0)     # 指定位置插入
nums.remove(1)        # 删除第一个值为 1 的元素（注意：按值删，不是按下标）
popped = nums.pop()   # 弹出并返回末尾元素；pop(0) 弹头部
nums.sort()           # 原地排序，返回 None（Python 惯例：原地操作的方法返回 None）
sorted_nums = sorted(nums)   # sorted() 是内置函数，返回新列表，原列表不变
nums.reverse()        # 原地反转

print(len(nums))      # 长度
print(5 in nums)      # 成员判断，O(n)
print(nums.count(1))  # 统计出现次数

# 排序进阶：key 参数接收一个函数，按函数返回值排序
words = ["banana", "pie", "apple", "cherry"]
words.sort(key=len)                        # 按长度排
print(words)                               # ['pie', 'apple', 'banana', 'cherry']
words.sort(key=lambda w: w[-1], reverse=True)  # 按最后一个字母倒序
```

### 3.3 tuple：不可变序列

元组常被误解为"不可变的列表"，更准确的定位是：**用于表示一条结构固定的记录**。比如 `(x, y)` 坐标、`(ip, port)` 地址。不可变性带来哈希能力（可作字典键）和语义上的安全性（别人不能改你这条记录）。

```python
point = (3, 4)
rgb = 255, 128, 0      # 括号可以省略，逗号才是元组的本质

# 解包（unpacking）是元组最常用的操作
x, y = point
print(x, y)  # 3 4

# 交换变量，底层就是元组打包再解包
a, b = 1, 2
a, b = b, a

# 函数返回多个值，本质是返回一个元组
def get_user():
    return "Alice", 25          # 返回 ("Alice", 25)

name, age = get_user()          # 解包接收

# 用 * 收集剩余部分
first, *rest = [1, 2, 3, 4, 5]
print(first, rest)  # 1 [2, 3, 4, 5]
```

注意元组的"不可变"是浅层的——元组自己不能换槽位里的引用，但如果元素本身是可变对象，那个对象内部可以改：

```python
t = ([1, 2], 3)
t[0].append(99)  # 合法！([1, 2, 99], 3)
# t[0] = []      # TypeError，不能替换元组里的槽位
```

### 3.4 dict：哈希表

字典是 Python 的灵魂数据结构（Python 自身的命名空间、对象属性全是用 dict 实现的），底层是 **哈希表**：对键做哈希得到索引，直接定位到存储位置，因此查找/插入/删除平均都是 O(1)。

需要知道的现代特性：**自 Python 3.7 起，dict 保证按插入顺序遍历**（这现在是语言规范，不是实现细节）。

```python
user = {"name": "Alice", "age": 25}

print(user["name"])              # Alice，键不存在会抛 KeyError
print(user.get("email"))         # None，安全取值
print(user.get("email", "无"))   # 提供默认值

user["city"] = "Beijing"         # 增/改
del user["age"]                  # 删
print("name" in user)            # 键的成员判断，O(1)

# 遍历的三种方式
for key in user:                  # 遍历键（等价于 user.keys()）
    print(key)
for value in user.values():       # 遍历值
    print(value)
for key, value in user.items():   # 键值一起遍历，最常用
    print(f"{key} = {value}")

# 合并字典（Python 3.9+ 的 | 运算符；旧版写法是 {**d1, **d2}）
defaults = {"theme": "dark", "lang": "zh"}
custom = {"lang": "en"}
config = defaults | custom   # 后者覆盖前者 → {'theme': 'dark', 'lang': 'en'}

# setdefault 解决"键不存在时先初始化"的问题
groups = {}
for word in ["apple", "avocado", "banana"]:
    groups.setdefault(word[0], []).append(word)   # 键不存在则先设为 []
print(groups)  # {'a': ['apple', 'avocado'], 'b': ['banana']}
```

### 3.5 set：去重与集合运算

集合底层同样是哈希表，只存键不存值。两大用途：**去重** 和 **成员快速判断**（O(1)，而 list 的 `in` 是 O(n)，数据量大时差距巨大）。

```python
tags = {"python", "web", "python"}  # 字面量创建，自动去重
print(tags)  # {'web', 'python'}

nums = [1, 2, 2, 3, 3, 3]
unique = list(set(nums))   # 经典去重写法（不保证原顺序）

# 集合运算
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
print(a | b)   # 并集 {1, 2, 3, 4, 5, 6}
print(a & b)   # 交集 {3, 4}
print(a - b)   # 差集 {1, 2}
print(a ^ b)   # 对称差集 {1, 2, 5, 6}（只在一边出现的）

# 实战：判断大列表成员时先转 set
valid_ids = set(range(1_000_000))
print(999_999 in valid_ids)   # 瞬间完成
```

### 3.6 推导式：Python 的招牌语法

推导式（comprehension）用一行声明式代码完成"遍历 + 过滤 + 变换 + 收集"，比等价的 for 循环更简洁，而且通常 **更快**（推导式的循环逻辑在字节码层面有专门优化）。

```python
nums = range(10)

# 列表推导式：[表达式 for 变量 in 可迭代对象 if 条件]
squares = [x ** 2 for x in nums]
evens = [x for x in nums if x % 2 == 0]
matrix_flat = [cell for row in [[1, 2], [3, 4]] for cell in row]  # 嵌套：先 row 后 cell

# 字典推导式
square_map = {x: x ** 2 for x in nums}
inverted = {v: k for k, v in {"a": 1, "b": 2}.items()}   # 键值互换

# 集合推导式
lengths = {len(w) for w in ["apple", "pie", "banana"]}   # {3, 5, 6}

# 生成器表达式：把 [] 换成 ()，不立即求值，惰性生成（详见第 9 节）
total = sum(x ** 2 for x in range(1_000_000))   # 省内存的经典用法
```

**可读性警告**：推导式超过一层嵌套或带复杂条件时，请退回普通 for 循环。《Python 之禅》说 "Readability counts"，推导式是为简洁而生的，不是为炫技。

### 3.7 浅拷贝与深拷贝

赋值 `b = a` 只是贴标签，不拷贝。真正的拷贝有两档，区别在于 **是否递归复制嵌套的可变对象**：

```python
import copy

original = [[1, 2], [3, 4]]

shallow = original.copy()          # 或 copy.copy(original)、original[:]、list(original)
deep = copy.deepcopy(original)

original[0].append(99)   # 修改嵌套的内层列表
print(shallow)  # [[1, 2, 99], [3, 4]] —— 浅拷贝只复制了外层壳，内层列表还是同一个对象！
print(deep)     # [[1, 2], [3, 4]]     —— 深拷贝递归复制了所有层级，完全独立

original[1] = [9, 9]     # 替换外层元素
print(shallow)  # [[1, 2, 99], [3, 4]] —— 外层是独立的，不受影响
```

**原理**：浅拷贝创建一个新的外层容器，但容器里的每个槽位仍然指向 **原来的子对象**（只拷贝了引用）；深拷贝则沿着对象图递归往下走，遇到可变对象就复制一份，直到所有层级都是全新对象。深拷贝的代价是慢且耗内存；遇到循环引用（A 引用 B，B 又引用 A）时，`deepcopy` 内部用一个 memo 字典记录已拷贝过的对象，避免无限递归。

经验法则：数据结构只含标量（数字、字符串）时浅拷贝足够；内含可变子对象且需要完全独立时才用 `deepcopy`。

---

## 4. 控制流

### 4.1 if 与缩进：代码块即结构

Python 用 **缩进**（推荐 4 个空格）而非花括号界定代码块。这不是风格选择，而是语法本身——缩进错误直接报 `IndentationError`。好处是强制所有代码风格统一、所见即结构；代价是复制粘贴代码时要小心缩进层级。

```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:    # 注意是 elif，不是 else if
    grade = "B"
else:
    grade = "C"
```

Python 没有传统的 `switch`（3.10 之前的替代方案是 if-elif 链或字典分发），3.10 起引入了更强大的 `match`（见 4.4）。也没有 C 风格的三目运算符 `?:`，用的是表达式形式的条件：

```python
age = 20
status = "成年" if age >= 18 else "未成年"   # 值1 if 条件 else 值2
```

### 4.2 while 循环

```python
count = 0
while count < 5:
    print(count)
    count += 1     # Python 没有 ++/-- 运算符，只有 += 1
```

`break` 跳出整个循环，`continue` 跳过本次迭代进入下一次，`pass` 是什么都不做的占位语句（语法上需要一个语句但逻辑上无事可做时使用）。

### 4.3 for 循环：迭代的是"可迭代对象"，不是下标

C/Java 的 `for (i = 0; i < n; i++)` 是"计数循环"；Python 的 `for` 是 **foreach 式的迭代循环**，直接遍历可迭代对象里的元素。想拿下标就用 `enumerate`，想并行遍历多个序列就用 `zip`：

```python
names = ["Alice", "Bob", "Carol"]

for name in names:                       # 直接拿元素，最常用
    print(name)

for i, name in enumerate(names):         # 同时拿下标和元素
    print(i, name)

for i, name in enumerate(names, start=1):  # 下标从 1 开始
    print(i, name)

scores = [90, 85, 88]
for name, score in zip(names, scores):   # 并行遍历，长度不一致时以短的为准
    print(f"{name}: {score}")

for i in range(10, 0, -1):               # range(起点, 终点不含, 步长)
    print(i)
```

`range` 是惰性序列：它不在内存里真的存一百万个数字，只记录起止和步长，用到时才算出来。

### 4.4 循环的 else 子句：最容易被误解的语法

`for` 和 `while` 后面可以跟 `else`，语义是：**循环正常跑完（没有被 break 打断）时，执行 else 块**。理解它的关键：把 `else` 读成 `no_break`。

典型场景是"查找"：找到了就 break，没找到走 else。

```python
def find_first_even(nums):
    for n in nums:
        if n % 2 == 0:
            print(f"找到偶数: {n}")
            break
    else:
        print("没有找到偶数")   # 只有完整遍历完都没 break 才执行

find_first_even([1, 3, 4])   # 找到偶数: 4
find_first_even([1, 3, 5])   # 没有找到偶数
```

没有这个语法，你就得引入一个 `found = False` 标志位，代码明显更啰嗦。

### 4.5 match-case：结构模式匹配（Python 3.10+）

`match` 不只是其他语言的 switch——它能做 **结构解构**：按数据的形状匹配并同时提取内容。

```python
def handle_command(command):
    match command.split():
        case ["quit"]:                        # 匹配单元素列表
            return "退出"
        case ["go", direction]:               # 匹配两元素列表，并把第二个绑定到 direction
            return f"向 {direction} 移动"
        case ["drop", *items]:                # * 收集剩余元素
            return f"丢弃了 {len(items)} 个物品"
        case [x, y] if x == y:                # 带守卫条件
            return "在对角线上"
        case _:                               # _ 是通配符，兜底分支
            return "未知指令"

# 匹配映射：按 dict 的键匹配并提取值
def classify(resp):
    match resp:
        case {"code": 200, "data": data}:
            return f"成功: {data}"
        case {"code": code} if 400 <= code < 500:
            return f"客户端错误 {code}"
        case _:
            return "其他"
```

底层上 `match` 是 **按顺序** 逐一尝试各 case 的模式，第一个匹配成功的分支执行后即结束——不会像 C 的 switch 那样 fall-through，因此也不需要 `break`。匹配映射时只要求包含列出的键即可（多余键不影响匹配）。它还能匹配对象属性（如 `case Point(x=0, y=0)`），在解析 API 响应、AST、消息协议时极其顺手。

---

## 5. 函数

### 5.1 定义与调用：参数传递的本质

```python
def greet(name, greeting="你好"):
    """文档字符串（docstring）：说明函数的用途，help() 会显示它。"""
    return f"{greeting}，{name}！"
```

先厘清一个面试必考的底层问题：**Python 的参数传递既不是值传递也不是引用传递，而是"对象引用的传递"（call by object reference / 共享传递）**。形参得到的是实参指向的那个对象的一个新标签。由此推出两条规则：

- 函数内对形参 **重新赋值**（贴新标签），不影响外部实参；
- 函数内对形参指向的可变对象 **原地修改**，会影响外部。

```python
def demo(a_list, a_num):
    a_list.append(99)   # 原地修改可变对象 → 外部可见
    a_num = 100         # 重新绑定 → 外部无感

lst = [1, 2]
n = 5
demo(lst, n)
print(lst)  # [1, 2, 99]
print(n)    # 5
```

### 5.2 参数体系：位置、关键字、默认值

```python
def create_user(name, age, city="北京", active=True):
    return {"name": name, "age": age, "city": city, "active": active}

# 位置传参：按顺序对应
create_user("Alice", 25)

# 关键字传参：按名字对应，顺序随意，可读性更好
create_user(age=30, name="Bob", city="上海")

# 混合：位置参数在前，关键字参数在后
create_user("Carol", age=28)
```

**默认参数的惊天大坑（面试必考）**：默认值在 **函数定义时** 只求值一次并存在函数对象上，而不是每次调用时重新创建。如果默认值是可变对象，所有调用会共享同一个对象：

```python
# 错误写法：所有调用共享同一个列表
def add_to(item, bag=[]):
    bag.append(item)
    return bag

print(add_to(1))  # [1]
print(add_to(2))  # [1, 2] —— 预期 [2]！上次的列表被共享了

# 正确写法：默认 None，函数体内再创建
def add_to(item, bag=None):
    if bag is None:
        bag = []
    bag.append(item)
    return bag
```

### 5.3 *args 与 **kwargs：可变参数

```python
def func(a, b, *args, **kwargs):
    print(a, b)        # 前两个正常接收
    print(args)        # 多余的位置参数被收进一个 tuple
    print(kwargs)      # 多余的关键字参数被收进一个 dict

func(1, 2, 3, 4, x=10, y=20)
# 1 2
# (3, 4)
# {'x': 10, 'y': 20}
```

`*` 和 `**` 在 **调用处** 还有反向操作——解包：

```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))                  # 把列表打散成三个位置参数 → 6

params = {"a": 1, "b": 2, "c": 3}
print(add(**params))               # 把字典打散成关键字参数 → 6
```

两个实用的语法细节：

```python
# 单独的 * 表示它后面的参数必须用关键字传递（keyword-only），防止调用者靠位置瞎传
def fetch(url, *, timeout=10, retries=3):
    pass
fetch("http://x.com", timeout=5)   # 合法
# fetch("http://x.com", 5)         # TypeError

# 单独的 / 表示它前面的参数只能用位置传递（positional-only，Python 3.8+）
def div(a, b, /):
    return a / b
```

### 5.4 lambda：匿名函数

`lambda` 是表达式级的函数定义，只能写一条表达式，适合作为"一次性小函数"传给高阶函数：

```python
add = lambda a, b: a + b        # 等价于 def add(a, b): return a + b（但不推荐这样赋值）

# 真正的用途：作为参数
users = [{"name": "Alice", "age": 25}, {"name": "Bob", "age": 20}]
users.sort(key=lambda u: u["age"])        # 按年龄排序

nums = [1, 2, 3, 4, 5]
print(list(map(lambda x: x * 2, nums)))       # [2, 4, 6, 8, 10]
print(list(filter(lambda x: x % 2 == 0, nums)))  # [2, 4]
```

注意：`map`/`filter` 的场景用推导式（`[x * 2 for x in nums]`）通常更具 Python 风格；lambda 的主战场是 `sort(key=...)`、`sorted`、`max/min(key=...)` 这类需要"提取排序依据"的地方。逻辑一旦复杂，请换回具名的 `def`。

### 5.5 LEGB 作用域规则

Python 查找一个名字的顺序是 **LEGB** 四层，由内到外：

1. **L (Local)**：当前函数内部；
2. **E (Enclosing)**：外层嵌套函数的局部作用域（闭包相关）；
3. **G (Global)**：当前模块的全局；
4. **B (Built-in)**：内置名字（`len`、`print`、`list` 等）。

找到即停，找不到抛 `NameError`。

```python
x = "global"   # G

def outer():
    x = "enclosing"   # E
    def inner():
        x = "local"   # L
        print(x)      # local：先在 L 找到就停了
    inner()

outer()
```

两个关键规则：

- 函数内 **读取** 全局变量没问题；要 **修改** 全局变量必须用 `global` 声明，否则赋值会被当作创建一个新的局部变量（经典报错 `UnboundLocalError` 就是由此而来：函数体内任何地方有对某名字的赋值，Python 就把这个名字在整个函数体内都视为局部变量）。
- 修改外层嵌套函数的变量用 `nonlocal` 声明（配合闭包使用）。

```python
count = 0

def increment():
    global count      # 声明：这里的 count 指全局那个
    count += 1

def make_counter():
    n = 0
    def add():
        nonlocal n    # 声明：这里的 n 指外层函数的 n
        n += 1
        return n
    return add
```

实际工程中应尽量避免 `global`（全局可变状态是万恶之源），`nonlocal` 配合闭包则是常见且优雅的用法。

### 5.6 闭包：函数记住了它的出生环境

**闭包（closure）**：内层函数引用了外层函数的局部变量，外层函数返回内层函数后，这些局部变量不会随外层函数执行结束而销毁，而是被内层函数"打包带走"。

```python
def make_multiplier(factor):
    def multiply(x):          # 内层函数引用了外层的 factor
        return x * factor
    return multiply           # 返回内层函数本身（注意：没有调用它）

double = make_multiplier(2)   # double 现在是一个闭包，带着 factor=2 的记忆
triple = make_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15
print(double.__closure__[0].cell_contents)  # 2 —— 可以看到被捕获的变量
```

**底层原理**：正常情况下函数的局部变量存在栈帧里，函数返回即销毁。但当 Python 编译器发现内层函数引用了外层变量时，会把这个变量升级为 **cell 对象**（一个盒子），外层和内层通过同一个 cell 共享它，生命周期由引用计数管理——这就是 `nonlocal` 能修改外层变量的机制。

闭包的价值：**让函数携带私有状态**，而不需要定义一个类。它是下一节装饰器的地基——装饰器本质上就是闭包的应用。

一个经典陷阱：循环中创建闭包时，所有闭包共享同一个变量，要用默认参数"固定"当前值：

```python
# 错误：三个函数都引用同一个 i，最终 i 都是 2
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2]

# 正确：利用默认参数在定义时求值的特性，把当前 i 存进各自的默认值
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2]
```

---

## 6. 装饰器

### 6.1 前提：函数是一等公民

"函数是一等公民"意味着函数和整数、字符串一样是对象，可以：

- 赋值给变量；
- 作为参数传给另一个函数；
- 作为返回值从另一个函数返回；
- 存在列表、字典里。

```python
def hello():
    return "你好"

f = hello          # 函数赋给变量（注意没有括号，不是调用）
print(f())         # 你好

def call_twice(func):     # 函数作为参数
    func()
    func()

call_twice(lambda: print("hi"))   # hi\nhi
```

这是装饰器存在的前提。顺带一提：`def` 语句的本质是"创建一个函数对象，并把它绑定到一个名字上"，和 `x = 1` 的绑定没有本质区别。

### 6.2 装饰器的本质：@ 只是语法糖

装饰器的完整定义：**一个接收函数、返回函数（通常是包装后的函数）的可调用对象**。下面不使用 `@`，纯手工实现一次"记录日志"的装饰器：

```python
def log_calls(func):                  # 接收一个函数
    def wrapper(*args, **kwargs):     # 定义包装函数，透传任意参数
        print(f"调用 {func.__name__}，参数: {args} {kwargs}")
        result = func(*args, **kwargs)   # 调用原函数
        print(f"{func.__name__} 返回: {result}")
        return result
    return wrapper                    # 返回包装函数

def add(a, b):
    return a + b

add = log_calls(add)   # 手动装饰：add 这个名字现在指向 wrapper
print(add(1, 2))
# 调用 add，参数: (1, 2) {}
# add 返回: 3
# 3
```

看懂了 `add = log_calls(add)` 这一行，就懂了装饰器的全部本质。`@` 语法糖做的事一模一样，只是写在函数定义上方：

```python
@log_calls
def add(a, b):        # 等价于：def add(...): ... 然后 add = log_calls(add)
    return a + b

print(add(1, 2))      # 效果完全相同
```

所以装饰器没有任何魔法——**`@decorator` 放在函数头上，等价于函数定义后立刻执行 `函数名 = decorator(函数名)`**。常见的内置装饰器有 `@property`、`@staticmethod`、`@classmethod`，第三方框架里 `@app.get("/")`（FastAPI）、`@pytest.fixture` 都是这个原理。

### 6.3 functools.wraps：修复"身份丢失"

上面的 `wrapper` 有一个问题：原函数的元信息被覆盖了：

```python
print(add.__name__)   # wrapper —— 原函数的 __name__、__doc__ 全丢了！
```

这会让调试器、文档工具、以及依赖 `__name__` 的框架（Flask 的路由注册就用函数名做默认 endpoint）出问题。解法是标准库 `functools.wraps`，它把原函数的元信息复制到包装函数上：

```python
import functools

def log_calls(func):
    @functools.wraps(func)          # 关键：把 func 的元信息拷贝到 wrapper
    def wrapper(*args, **kwargs):
        print(f"调用 {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def add(a, b):
    """两个数相加"""
    return a + b

print(add.__name__)  # add
print(add.__doc__)   # 两个数相加
```

**写装饰器时永远带上 `@functools.wraps`**，这是行业惯例，面试也常考。

### 6.4 带参数的装饰器：三层嵌套

如果装饰器本身需要参数（比如 `@retry(times=3)`），结构就变成三层：**最外层接收装饰器参数，中间层接收被装饰的函数，最内层是包装函数**。

```python
import functools
import time

def retry(times, delay=0.1):
    """带参数的装饰器：失败时重试 times 次"""
    def decorator(func):                  # 这一层才是真正的装饰器
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"第 {attempt} 次失败: {e}")
                    if attempt == times:
                        raise             # 最后一次仍失败，抛出
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(times=3)
def unstable_api():
    import random
    if random.random() < 0.7:
        raise ConnectionError("网络抖动")
    return "成功"

print(unstable_api())
```

理解它的调用链：`@retry(times=3)` 先执行 `retry(times=3)` 得到 `decorator`，再执行 `unstable_api = decorator(unstable_api)`。即 **带参数的装饰器 = 先调用一次拿到真正的装饰器，再做普通装饰**。

### 6.5 实战：计时装饰器

```python
import functools
import time

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} 耗时 {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_task():
    time.sleep(0.1)

slow_task()   # slow_task 耗时 0.1015s
```

装饰器是 Python 框架的骨架技术：FastAPI 的路由、SQLAlchemy 的事件、pytest 的 fixture、缓存 `functools.lru_cache`（它本身就是一个超好用的内置装饰器，一行 `@lru_cache` 就能给函数加记忆化缓存）全都建立在它之上。

---

## 7. 类与面向对象

### 7.1 定义类：`__init__` 与 self

```python
class User:
    def __init__(self, name, age):   # 构造器：创建实例后自动调用
        self.name = name             # self 指向"当前这个实例"，类似 this
        self.age = age

    def introduce(self):             # 实例方法的第一个参数永远是 self
        return f"我是 {self.name}，{self.age} 岁"

u = User("Alice", 25)      # 调用类 = 创建实例 + 自动执行 __init__
print(u.introduce())       # 我是 Alice，25 岁
```

几个与其他语言不同的要点：

- `self` **必须显式声明** 为方法的第一个参数，但调用时不用传（Python 自动把实例塞进去）。`u.introduce()` 等价于 `User.introduce(u)`——方法调用的本质就是"把实例作为第一个参数传给函数"。理解了这一点，`self` 就毫无神秘感。
- `__init__` 不是构造函数，真正的"创建对象"由 `__new__` 完成；`__init__` 负责在已创建的对象上做 **初始化**。日常开发几乎只需要写 `__init__`。
- 属性不需要提前声明，`self.xxx = ...` 赋值即创建（这也让 Python 对象的属性可以在运行时动态增删——对象内部就是一个 `__dict__` 字典）。
- 双下划线开头结尾的方法（`__init__`、`__str__` 等）叫 **魔术方法/dunder 方法**，由 Python 在特定时机自动调用，你不直接调用它们。

### 7.2 类变量 vs 实例变量

```python
class Dog:
    species = "犬科"          # 类变量：属于类本身，所有实例共享同一份

    def __init__(self, name):
        self.name = name      # 实例变量：每个实例各有一份

d1 = Dog("旺财")
d2 = Dog("来福")

print(d1.species, d2.species)  # 犬科 犬科 —— 共享
Dog.species = "Canidae"        # 通过类修改
print(d1.species)              # Canidae —— 所有实例都看到新值

d1.species = "狼"              # 通过实例赋值：不会改类变量，而是给 d1 创建同名实例变量！
print(d2.species)              # Canidae —— d2 不受影响
print(Dog.species)             # Canidae —— 类变量也没变
```

**原理**：属性查找的顺序是"实例的 `__dict__` → 类的 `__dict__` → 父类"。读取 `d1.species` 时实例里没有，就去类里找；而 `d1.species = "狼"` 是赋值，直接在实例的 `__dict__` 里新建条目，从此遮蔽（shadow）了类变量。

**经典坑**：用可变对象做类变量，等于所有实例共享同一个可变对象：

```python
class Classroom:
    students = []              # 危险！所有实例共享这一个列表

    def add(self, name):
        self.students.append(name)   # 改的是类变量

c1, c2 = Classroom(), Classroom()
c1.add("Alice")
print(c2.students)   # ['Alice'] —— c2 也被影响了，通常不是想要的行为

# 正确：可变属性放 __init__ 里做实例变量
class Classroom:
    def __init__(self):
        self.students = []
```

### 7.3 继承与 MRO

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "..."

class Dog(Animal):                      # 继承 Animal
    def speak(self):                    # 方法重写（override）
        return "汪汪"

    def fetch(self):
        return f"{self.name} 捡回了球"

class Cat(Animal):
    def speak(self):
        return "喵喵"

animals = [Dog("旺财"), Cat("咪咪")]
for a in animals:
    print(a.speak())   # 汪汪 / 喵喵 —— 多态：同一接口，不同实现
```

`super()` 用于在子类中调用父类的实现，最常见的场景是子类 `__init__` 扩展父类初始化：

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

class Manager(Employee):
    def __init__(self, name, salary, department):
        super().__init__(name, salary)   # 复用父类初始化逻辑，不要复制粘贴
        self.department = department
```

**MRO（Method Resolution Order，方法解析顺序）**：多继承时，Python 按什么顺序查找方法？答案是 **C3 线性化算法**——一句话概括：子类优先于父类，继承列表中靠左的父类优先于靠右的，同时保证整个继承图的拓扑序一致。不需要手算，直接看：

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
# 查找顺序：D → B → C → A → object（菱形继承中 A 只被访问一次）
```

所有类都隐式继承自 `object`。多继承在 Python 中是合法且有成熟规则的，但工程上更推荐用 **mixin** 模式（小粒度、只做一件事的辅助类，如 `JsonSerializableMixin`）而不是深层继承树。

### 7.4 @property：把方法伪装成属性

问题场景：你想对属性的读写加校验或做计算，但又不想破坏"属性访问"的简洁语法（不像 Java 那样逼用户写 `getXxx()`/`setXxx()`）。`@property` 让方法以属性的形式被访问：

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius      # 单下划线开头是"内部使用"的约定（不强制）

    @property
    def radius(self):              # 读：c.radius 实际调用这个方法
        return self._radius

    @radius.setter
    def radius(self, value):       # 写：c.radius = x 实际调用这个方法
        if value <= 0:
            raise ValueError("半径必须为正数")
        self._radius = value

    @property
    def area(self):                # 只读的计算属性
        return 3.14159 * self._radius ** 2

c = Circle(5)
print(c.radius)   # 5 —— 像访问属性，实际走了方法
print(c.area)     # 78.539...
c.radius = 10     # 像赋值属性，实际走了 setter 校验
# c.radius = -1   # ValueError: 半径必须为正数
```

`@property` 的伟大之处在于 **重构自由**：先暴露朴素属性，之后需要加校验时改成 property，调用方的代码一行都不用改。这是 Python 不流行 getter/setter 的原因。

### 7.5 dataclass：消灭样板代码

写一个"主要用来装数据"的类，传统写法要手写 `__init__`、`__repr__`、`__eq__` 一堆样板。`@dataclass` 一行装饰器全帮你生成：

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    email: str = "未填写"              # 有默认值的字段必须放在无默认值字段之后
    tags: list[str] = field(default_factory=list)   # 可变默认值必须用 default_factory！

u1 = User("Alice", 25)
print(u1)                    # User(name='Alice', age=25, email='未填写', tags=[]) —— __repr__ 自动生成
print(u1 == User("Alice", 25))  # True —— __eq__ 自动生成（按字段逐个比较）
```

两个进阶参数：

```python
@dataclass(frozen=True)    # 实例创建后不可修改（同时变得可哈希，能当 dict 键）
class Point:
    x: int
    y: int

@dataclass(order=True)     # 自动生成 <、<=、>、>= 比较方法（按字段顺序比较）
class Version:
    major: int
    minor: int
```

注意 `tags: list[str] = field(default_factory=list)` 这个细节：默认值在类定义时只求值一次（和函数默认参数同一个坑），所以可变默认值不能直接写 `= []`，必须用 `default_factory` 指定一个工厂函数，让每次创建实例时调用 `list()` 生成新列表。

### 7.6 `__str__` 与 `__repr__`：两种字符串表示

- `__repr__`：**给开发者看** 的，目标是"明确无歧义"，理想情况下返回值能直接复制进解释器重建对象。`repr()` 函数、交互式解释器直接敲变量名、以及对象出现在列表/字典里时都用它。
- `__str__`：**给用户看** 的，追求可读性。`print()`、`str()`、f-string 用它。

如果只定义一个，定义 `__repr__`——因为 `__str__` 缺失时会退而使用 `__repr__`，反之不行。

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return f"User(name={self.name!r}, age={self.age!r})"   # 像构造调用

    def __str__(self):
        return f"{self.name}（{self.age} 岁）"

u = User("Alice", 25)
print(u)          # Alice（25 岁）          —— print 走 __str__
print(repr(u))    # User(name='Alice', age=25)
print([u])        # [User(name='Alice', age=25)] —— 容器里的元素走 __repr__
```

顺带一提 f-string 里的 `!r` 表示"对这个值用 `repr()`"，上面 `__repr__` 里用 `{self.name!r}` 是为了让字符串自动带上引号。

常用魔术方法还有：`__len__`（`len()`）、`__eq__`（`==`）、`__getitem__`（`obj[i]`，让对象支持下标）、`__enter__/__exit__`（`with`，见下一节）、`__call__`（让实例可以像函数一样被调用）。这套"数据模型"机制是 Python 鸭子类型的基石：**你的类只要实现了对应协议，就能无缝接入 Python 的语法**。

---

## 8. 异常处理

### 8.1 为什么用异常而不是返回错误码

C 语言风格是函数返回错误码，调用方逐级判断。Python 的哲学是 **EAFP**（Easier to Ask Forgiveness than Permission，"做了再说，出错再处理"）——先假设操作会成功，失败时让异常自然传播。好处：

1. 正常路径的代码干净，不被一堆 `if err != 0` 淹没；
2. 异常会自动沿调用栈向上传播，不可能被"忘记检查"（错误码可以）；
3. 异常对象携带完整的错误类型、信息和堆栈，远比一个数字信息量足。

### 8.2 try/except/else/finally 完整结构

```python
def divide(a, b):
    try:
        result = a / b                  # 可能抛异常的代码
    except ZeroDivisionError as e:      # 捕获特定异常，e 是异常对象
        print(f"除零错误: {e}")
        return None
    except (TypeError, ValueError):     # 一个 except 捕获多种异常
        print("类型不对")
        return None
    except Exception as e:              # 捕获剩余所有异常（要放在最后）
        print(f"未预料的错误: {e}")
        raise                           # 处理不了就重新抛出，别吞掉
    else:
        # 只有当 try 块【没有抛异常】时才执行
        print("计算成功")
        return result
    finally:
        # 无论是否异常、无论是否 return，都【必定执行】
        print("清理工作（关闭连接、释放锁等）")

print(divide(10, 2))   # 计算成功 → 清理工作 → 5.0
print(divide(10, 0))   # 除零错误: division by zero → 清理工作 → None
```

执行规则的精确语义：

- `except` 块 **按顺序匹配**，命中第一个匹配的分支后其余跳过，所以要把具体的异常类型放前面、宽泛的 `Exception` 放后面；
- `else` 只在无异常时执行——它的意义是把"可能抛异常的代码"和"成功后的后续逻辑"分开，缩小 try 的范围；
- `finally` **永远执行**，即使 try 块里有 `return`——`return` 之前会先执行完 `finally`。也因此：永远不要在 `finally` 里写 `return`，它会吞掉正在传播的异常和原来的返回值。

**最佳实践**：except 尽量捕获 **具体** 的异常类型。裸 `except:`（连 `Exception` 都不写）会连 `KeyboardInterrupt`（Ctrl+C）都吞掉，是公认的最差实践。

### 8.3 raise 与异常链

主动抛出异常用 `raise`，配合异常类型和消息：

```python
def set_age(age):
    if not isinstance(age, int):
        raise TypeError(f"年龄必须是整数，收到 {type(age).__name__}")
    if age < 0:
        raise ValueError(f"年龄不能为负: {age}")
    return age
```

在 except 块中处理时如果发现处理不了，可以 **转换异常** 并保留原始上下文——`raise ... from ...` 形成异常链，排错时能看到完整的因果：

```python
import json

def load_config(text):
    try:
        return json.loads(text)
    except json.JSONDecodeError as e:
        raise ValueError("配置文件格式错误") from e   # e 会被记录为 __cause__

# load_config("{bad json")
# ValueError: 配置文件格式错误
# 上方会同时打印 The above exception was the direct cause... 及原始 JSONDecodeError
```

内置常用异常类型：`ValueError`（值不对）、`TypeError`（类型不对）、`KeyError` / `IndexError`（键/下标不存在）、`FileNotFoundError`、`AttributeError`（属性不存在）、`RuntimeError`（通用运行时错误）、`NotImplementedError`（抽象方法占位）。

### 8.4 自定义异常

业务代码应该定义自己的异常体系，让调用方能精确捕获"你的错误"而不是笼统的 `Exception`：

```python
class AppError(Exception):
    """应用基础异常：本项目所有自定义异常的父类"""

class UserNotFoundError(AppError):
    """用户不存在"""

    def __init__(self, user_id):
        self.user_id = user_id
        super().__init__(f"用户 {user_id} 不存在")   # 把消息传给 Exception

class PermissionDeniedError(AppError):
    """权限不足"""

def get_user(user_id):
    if user_id <= 0:
        raise UserNotFoundError(user_id)
    return {"id": user_id, "name": "Alice"}

try:
    get_user(-1)
except UserNotFoundError as e:
    print(e)             # 用户 -1 不存在
    print(e.user_id)     # -1 —— 自定义属性也能拿到
except AppError as e:    # 父类可以捕获整个异常家族
    print(f"应用错误: {e}")
```

惯例：自定义异常继承 `Exception`（而不是 `BaseException`——后者还包括 `KeyboardInterrupt`、`SystemExit`，那些不该被业务代码捕获）；异常类名以 `Error` 结尾；建立层级（基类 + 具体子类）方便调用方按需捕获。

### 8.5 with 上下文管理器：`__enter__` / `__exit__` 原理

你一定见过：

```python
with open("data.txt", encoding="utf-8") as f:
    content = f.read()
# 出了 with 块，文件自动关闭——即使中间抛异常也一样
```

**原理**：`with` 语句要求对象实现两个魔术方法（即"上下文管理器协议"）：

- `__enter__(self)`：进入 with 块时调用，返回值绑定到 `as` 后面的变量；
- `__exit__(self, exc_type, exc_val, exc_tb)`：退出 with 块时 **必定调用**（相当于自动内嵌了一个 try/finally）。三个参数是异常信息；正常退出时全为 `None`。若 `__exit__` 返回 `True`，表示"异常已被处理"，异常不再向外传播。

手写一个上下文管理器感受机制：

```python
class Timer:
    def __enter__(self):
        import time
        self._start = time.perf_counter()
        return self                     # as 后面的变量拿到的是这个返回值

    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        print(f"耗时 {time.perf_counter() - self._start:.4f}s")
        return False                    # 不吞异常

with Timer():
    sum(range(10_000_000))   # 耗时 0.1xxx s
```

更常用的方式是用标准库 `contextlib` 把生成器函数变成上下文管理器（`yield` 之前是 enter 逻辑，之后是 exit 逻辑）：

```python
from contextlib import contextmanager

@contextmanager
def db_transaction(conn):
    cursor = conn.cursor()
    try:
        yield cursor        # with 块拿到的就是它
        conn.commit()       # 正常结束 → 提交
    except Exception:
        conn.rollback()     # 抛异常 → 回滚
        raise
    finally:
        cursor.close()      # 无论如何 → 释放资源
```

`with` 的本质价值：**把"获取资源"和"释放资源"这两件必须配对的事绑定在语法层面**，杜绝忘记释放（文件句柄、锁、数据库连接、网络会话）。FastAPI 的依赖注入系统（`Depends`）也大量依赖这个协议。

---

## 9. 迭代器与生成器

### 9.1 迭代协议：`__iter__` 与 `__next__`

`for` 循环为什么能遍历列表、字符串、字典、文件？因为它们都遵循 **迭代协议**：

- **可迭代对象（Iterable）**：实现了 `__iter__()`，调用它返回一个迭代器。列表、字符串、字典、集合都是可迭代对象。
- **迭代器（Iterator）**：实现了 `__next__()`（每次调用返回下一个元素，没有更多元素时抛 `StopIteration`），并且 `__iter__()` 返回自身。

`for x in obj` 的底层等价于：

```python
nums = [10, 20, 30]

it = iter(nums)        # for 第一步：调用 nums.__iter__() 拿到迭代器
while True:
    try:
        x = next(it)   # 反复调用 it.__next__()
    except StopIteration:
        break          # 迭代结束的信号就是这个异常
    print(x)
```

三个要点：

1. **迭代器是有状态的、一次性的**：`next` 会消耗元素，迭代完就空了，想再来一遍必须重新 `iter()` 拿新迭代器。这就是为什么迭代器可以表示"无限序列"。
2. **`StopIteration` 不是错误，是协议的一部分**：for 循环靠捕获它来判断结束。
3. 自定义类只要实现这两个方法，就能被 `for` 遍历、被 `list()` 转换、被 `sum()` 消费——鸭子类型在迭代上的体现。

### 9.2 生成器：用 yield 写的迭代器

手写迭代器类很啰嗦。生成器（generator）是 Python 给的捷径：**任何包含 `yield` 关键字的函数，调用后返回的不是执行结果，而是一个生成器对象（一种迭代器）**。

```python
def countdown(n):
    print("开始")
    while n > 0:
        yield n          # 每次产出一个值，函数在此"暂停"
        n -= 1
    print("结束")

gen = countdown(3)   # 注意：此刻函数体一行都没执行！
print(next(gen))     # 开始 → 3（执行到第一个 yield 暂停）
print(next(gen))     # 2（从上次暂停处继续）
print(next(gen))     # 1
# print(next(gen))   # 结束 → StopIteration

# 生成器是迭代器，所以可以直接 for
for x in countdown(3):
    print(x)
```

**`yield` 的执行机制**是理解生成器的关键：普通函数调用时在栈上创建帧，执行完销毁；生成器函数的帧被特殊对待——`yield` 时 **整个帧的状态（局部变量、指令指针）被冻结保存**，控制权交还调用者；下次 `next()` 时帧被解冻，从 `yield` 的下一行接着跑。函数因此可以"活多次"。`return`（或函数自然结束）则触发 `StopIteration`。

### 9.3 生成器为什么省内存

对比两种写法处理十亿行日志：

```python
# 列表：一次性把所有数据物化到内存
def read_all_lines(path):
    with open(path, encoding="utf-8") as f:
        return f.readlines()        # 10 亿行全在内存里，直接 OOM

# 生成器：数据"流"式地一行行产生，任意时刻内存里只有一行
def read_lines(path):
    with open(path, encoding="utf-8") as f:
        for line in f:              # 文件对象本身就是迭代器！
            yield line.strip()
```

省内存的本质：**生成器是"惰性（lazy）"的——它保存的是"如何计算下一个值"的指令，而不是"所有的值"**。内存占用是 O(1)（一个帧 + 当前状态），与数据总量无关。列表是"急切（eager）"的，内存 O(n)。

推论：生成器可以表示无限序列：

```python
def fibonacci():
    a, b = 0, 1
    while True:          # 无限循环也不怕，因为每次只在 next 时走一步
        yield a
        a, b = b, a + b

fib = fibonacci()
print([next(fib) for _ in range(10)])   # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### 9.4 生成器表达式与管道式处理

推导式把 `[]` 换成 `()` 就是生成器表达式：

```python
squares = (x ** 2 for x in range(1_000_000))   # 不立即计算，内存几乎为零
print(sum(squares))                             # 消费时才算

# 作为唯一参数时括号可省
total = sum(x ** 2 for x in range(1_000_000))
```

生成器最大的工程价值是搭 **处理管道**：每个环节都是惰性流，串起来依然只占常数内存：

```python
def read_lines(path):
    with open(path, encoding="utf-8") as f:
        yield from f          # yield from：把子可迭代对象的值逐个转发出去

def filter_errors(lines):
    for line in lines:
        if "ERROR" in line:
            yield line

def parse(lines):
    for line in lines:
        yield line.split(" ", 2)[-1]   # 提取消息部分

# 组装管道：此刻没有任何数据被读取
pipeline = parse(filter_errors(read_lines("app.log")))

for msg in pipeline:          # for 驱动整条管道逐行流动
    print(msg)
```

这是 Unix 管道 `cat app.log | grep ERROR | cut ...` 的 Python 版，处理 TB 级日志也不会撑爆内存。

---

## 10. 模块与包

### 10.1 import 机制：到底发生了什么

一个 `.py` 文件就是一个 **模块（module）**。`import` 时 Python 做了三件事：

1. **查找**：在 `sys.path`（一个路径列表，包含脚本所在目录、环境变量 `PYTHONPATH`、标准库目录、site-packages）中按顺序找到第一个匹配的模块；
2. **执行**：**从头到尾执行一遍模块的顶层代码**（顶层定义的函数、类、变量、print 全都会执行！）；
3. **绑定**：在当前命名空间创建一个名字，指向这个模块对象。

关键推论——**模块只被导入一次**：第一次 import 执行并缓存到 `sys.modules`，之后再 import 同名的模块直接取缓存，不会重复执行。这保证了模块级对象是天然的单例。

```python
import math                # 导入整个模块，用 math.sqrt 访问
from math import sqrt      # 只导入指定名字，直接用 sqrt
from math import pi as PI  # 起别名，避免名字冲突
import numpy as np         # 社区惯例别名

# 避免：from math import *   —— 污染命名空间，无法追踪名字来源
```

模块顶层代码会被执行这一点，直接引出下一小节。

### 10.2 `if __name__ == '__main__'`：每个 Python 文件都该有的"门卫"

每个模块都有一个内置变量 `__name__`：

- 当模块被 **直接运行**（`python app.py`）时，`__name__` 的值是字符串 `'__main__'`；
- 当模块被 **import** 时，`__name__` 是模块名（`'app'`）。

于是这个惯用法的作用是：**让同一个文件既能被导入（只提供函数/类，不执行演示代码），又能被直接运行（执行演示/测试/入口逻辑）**。

```python
# calculator.py
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

if __name__ == "__main__":
    # 只有 python calculator.py 直接运行时才会走到这里
    # import calculator 时不会执行
    print(add(1, 2))
```

没有这层保护，别人 `import calculator` 时你的测试 `print` 会全部执行一遍。写任何含可执行逻辑的脚本，都应把入口逻辑收进 `main()` 函数再套这个门卫：

```python
def main():
    ...

if __name__ == "__main__":
    main()
```

### 10.3 包：`__init__.py` 的作用

**包（package）** 是组织模块的目录。典型结构：

```text
myproject/
├── main.py
└── mypackage/
    ├── __init__.py
    ├── users.py
    └── orders.py
```

- `__init__.py` 标识这个目录是一个包（现代 Python 支持无 `__init__.py` 的"命名空间包"，但常规项目仍建议显式写上），它可以是空文件，也常用来 **做包级别的初始化和"出口整理"**：

```python
# mypackage/__init__.py
from .users import get_user        # 把深层模块的名字提到包的顶层
from .orders import create_order

# 之后使用方可以 from mypackage import get_user
# 而不必 from mypackage.users import get_user —— 隐藏了内部文件结构
```

- 使用：`from mypackage.users import get_user` 或 `from mypackage import users`。

### 10.4 相对导入与常见坑

包内部模块互相引用用 **相对导入**（点号表示层级）：

```python
# mypackage/orders.py
from .users import get_user        # . 表示当前包
from ..utils.helpers import log    # .. 表示上一级包
```

相对导入的好处是包整体改名字、挪位置时代码不用改。

**最常见的坑**：**直接运行包内的模块文件**（如 `python mypackage/orders.py`）会报 `ImportError: attempted relative import with no known parent package`。原因：直接运行时 Python 把这个文件当作顶层脚本，它不属于任何包，相对导入就失去了参照。正确做法是在项目根目录用 `-m` 运行：

```bash
# 在 myproject/ 目录下执行，把模块当作包内成员运行
python -m mypackage.orders
```

`-m` 让 Python 以包结构的方式定位并执行模块，相对导入因此能正常工作。这也是 `python -m venv`、`python -m pip`、`python -m pytest` 这些用法的原理——模块即程序。

---

## 11. 文件 IO 与 json / pathlib

### 11.1 文件读写：with + encoding 是铁律

```python
# 读整个文件
with open("notes.txt", "r", encoding="utf-8") as f:
    content = f.read()

# 按行遍历（文件对象是迭代器，大文件逐行读不占内存）
with open("notes.txt", encoding="utf-8") as f:
    for line in f:
        print(line.strip())

# 写文件："w" 覆盖写，"a" 追加写；写中文务必指定 encoding
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("第一行\n")
    f.writelines(["第二行\n", "第三行\n"])
```

两条铁律：

1. **永远用 `with`**：保证文件句柄被释放（异常时也一样），否则依赖垃圾回收时机，Windows 上常因此出现"文件被占用无法删除"；
2. **永远显式指定 `encoding="utf-8"`**：不指定时 Python 用系统默认编码（Windows 中文系统是 GBK），同一份代码换个系统就乱码——这是跨平台中文乱码的头号根源。

读写二进制文件（图片、压缩包）用 `"rb"` / `"wb"`，此时拿到的是 `bytes` 对象。

### 11.2 pathlib：面向对象的路径处理

字符串拼路径（`dir + "\\" + file`）又丑又容易踩 Windows/Linux 分隔符差异。标准库 `pathlib` 用 `Path` 对象统一处理路径，是现代 Python 的推荐做法：

```python
from pathlib import Path

data_dir = Path("data") / "2024" / "logs"    # / 运算符拼接路径，自动处理分隔符
print(data_dir)                # data/2024/logs（Windows 上显示反斜杠）

file = data_dir / "app.log"
print(file.name)               # app.log —— 文件名
print(file.stem)               # app —— 不含扩展名
print(file.suffix)             # .log —— 扩展名
print(file.parent)             # data/2024/logs —— 父目录
print(file.exists())           # 是否存在
print(file.is_file())          # 是文件还是目录

# 常用操作
data_dir.mkdir(parents=True, exist_ok=True)   # 递归创建目录，已存在不报错
text = file.read_text(encoding="utf-8")       # 一步读文本（内部自动 with）
file.write_text("你好", encoding="utf-8")     # 一步写文本

# 遍历目录
for p in Path(".").glob("**/*.py"):     # 递归匹配所有 .py 文件
    print(p)

home = Path.home()                      # 用户主目录，跨平台
abs_path = file.resolve()               # 转为绝对路径
```

`Path.read_text()` / `write_text()` 内部帮你处理 open/close，一行搞定；需要流式读写大文件时再退回 `with open()`（或者 `path.open()`）。

### 11.3 json：与外部世界交换数据

JSON 是 API 通信和配置文件的事实标准。`json` 模块做两件事：Python 对象 ↔ JSON 字符串（`dumps`/`loads`）、Python 对象 ↔ JSON 文件（`dump`/`load`）。记忆口诀：**带 s 的是字符串（string）操作**。

```python
import json

user = {"name": "张三", "age": 25, "skills": ["Python", "SQL"], "active": True}

# 序列化：Python 对象 → JSON 字符串
s = json.dumps(user, ensure_ascii=False, indent=2)
print(s)
# {
#   "name": "张三",
#   ...
# }
# ensure_ascii=False 让中文原样输出（默认转义成 \u5f20\u4e09）
# indent=2 美化缩进，给人看；机器传输时省略以省体积

# 反序列化：JSON 字符串 → Python 对象
obj = json.loads(s)
print(obj["name"])   # 张三

# 直接读写文件
with open("user.json", "w", encoding="utf-8") as f:
    json.dump(user, f, ensure_ascii=False, indent=2)

with open("user.json", encoding="utf-8") as f:
    data = json.load(f)
```

类型映射关系：Python 的 `dict`/`list`/`str`/`int`/`float`/`True`/`False`/`None` 对应 JSON 的 object/array/string/number/true/false/null。**注意 `tuple` 会被转成 JSON array，读回来变成 `list`**；`set`、自定义对象不能直接序列化（会抛 `TypeError: Object of type set is not JSON serializable`），需要先转换。

---

## 12. 类型注解

### 12.1 定位：注解是给"人"和"工具"看的，解释器不强制

Python 是动态类型语言，类型注解（type hints）**不改变运行时行为**——解释器完全忽略它们，注解错了代码照样跑。那为什么要写？

1. **静态检查**：`mypy`、`pyright` 等工具（以及 IDE）能在运行前发现类型错误，把 bug 从运行时提前到编码时；
2. **IDE 智能提示**：有了类型，编辑器才能准确补全和跳转；
3. **文档作用**：看签名就知道传什么，不用翻函数体；
4. **框架依赖**：**FastAPI 就是靠类型注解自动生成请求校验、文档和数据转换的**——这是本系列课程必须掌握注解的直接原因。

### 12.2 基本语法与 typing 常用类型

```python
def greet(name: str, age: int) -> str:
    return f"{name} is {age}"

# 变量注解
count: int = 0
names: list[str] = []            # Python 3.9+ 可直接用内置泛型（小写 list/dict/tuple/set）

# typing 模块的经典写法（3.9 之前必须用；理解它对读老代码很重要）
from typing import List, Dict, Optional, Union, Callable, Any, Tuple

def process(items: List[str],               # 等价于 list[str]
            mapping: Dict[str, int],        # 等价于 dict[str, int]
            point: Tuple[int, int] = (0, 0)  # 等价于 tuple[int, int]
            ) -> None:
    ...
```

最常用的几个泛型：

```python
from typing import Optional, Union, Callable, Any

# Optional[X] 等价于 Union[X, None]：参数可以是 X 或 None
def find_user(user_id: int) -> Optional[dict]:
    if user_id <= 0:
        return None
    return {"id": user_id}

# Python 3.10+ 更简洁的写法：X | None（推荐）
def find_user2(user_id: int) -> dict | None:
    ...

# Union：多种类型之一
def parse(value: Union[int, str]) -> int:      # 3.10+ 写作 int | str
    return int(value)

# Callable：注解"函数类型"，参数类型列表 + 返回类型
def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

print(apply(lambda x, y: x + y, 1, 2))   # 3

# Any：放弃检查，什么类型都行（万不得已再用，它会传染式地污染检查效果）
def legacy(data: Any) -> Any:
    ...
```

`Optional[X]` 的语义必须吃透：**它表示"可以是 None"，而默认参数给了值只是"可以省略"**，两者不是一回事。参数允许传 None 却不标 `Optional`，类型检查器会报错。

### 12.3 容器嵌套与自定义类

```python
# 嵌套结构：一个"用户 id → 该用户的订单列表"的字典
orders_by_user: dict[int, list[dict[str, Any]]] = {}

# 函数列表
handlers: list[Callable[[str], None]] = []

# 自定义类直接当类型用（这是最常见的注解）
class User:
    def __init__(self, name: str, age: int):
        self.name = name
        self.age = age

def create_admin(user: User) -> User:
    ...

# 类自己的方法里引用自己（前向引用），3.11+ 可直接写；更早版本用字符串
class Node:
    def link(self, other: "Node") -> None:
        ...
```

### 12.4 为 FastAPI 铺垫：注解即契约

预告一下后续课程：FastAPI 把类型注解用到了极致——

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):          # pydantic 模型：注解即校验规则
    name: str
    price: float
    tags: list[str] = []

@app.post("/items/")
def create_item(item: Item):    # 注解告诉 FastAPI：请求体要解析成 Item
    return item                 # 不符合注解的请求自动返回 422 错误
```

你写的每个注解都是一份机器可执行的契约：自动校验请求、自动生成 OpenAPI 文档、自动完成编辑器提示。这套"注解驱动开发"就是现代 Python Web 框架的核心范式，而它的语法基础就是本节内容。

---

## 13. 常用标准库速览

Python 的口号是"电池内置（batteries included）"。下面按场景过一遍最高频的标准库，每个给出最常用的 API。原则：**能import解决的，绝不自己造轮子**。

### 13.1 os 与 sys：和操作系统、解释器打交道

```python
import os
import sys

# os：环境变量、进程环境（路径操作用 pathlib 更现代）
print(os.environ.get("HOME") or os.environ.get("USERPROFILE"))  # 读环境变量
os.environ["DEBUG"] = "1"                                       # 设置环境变量
print(os.getcwd())          # 当前工作目录
print(os.listdir("."))      # 列出目录内容（返回文件名列表）

# sys：解释器相关
print(sys.argv)             # 命令行参数列表，sys.argv[0] 是脚本名
print(sys.version)          # Python 版本
print(sys.path)             # 模块搜索路径（import 机制就靠它）
print(sys.platform)         # 'win32' / 'linux' / 'darwin'
sys.exit(0)                 # 退出程序并指定退出码

# 命令行参数处理：简单场景直接 sys.argv，正经 CLI 工具用 argparse
import argparse
parser = argparse.ArgumentParser(description="示例工具")
parser.add_argument("filename")                      # 位置参数
parser.add_argument("-n", "--count", type=int, default=1)  # 可选参数
args = parser.parse_args()   # python tool.py a.txt -n 3 → args.count == 3
```

### 13.2 datetime：日期时间

```python
from datetime import datetime, date, timedelta, timezone

now = datetime.now()
print(now)                          # 2024-01-15 10:30:45.123456
print(now.strftime("%Y-%m-%d %H:%M:%S"))  # 对象 → 格式化字符串
print(now.date(), now.time())       # 拆出日期和时间

# 字符串 → 对象（解析日志、接口返回值常用）
dt = datetime.strptime("2024-01-15 10:30:00", "%Y-%m-%d %H:%M:%S")

# 时间运算：timedelta 表示"时间差"
tomorrow = now + timedelta(days=1)
last_week = now - timedelta(weeks=1)
delta = datetime(2024, 2, 1) - datetime(2024, 1, 1)
print(delta.days)                   # 31

# 时间戳互转
ts = now.timestamp()                        # → float 秒数
back = datetime.fromtimestamp(ts)

# 时区感知：生产环境务必使用带时区的时间（或统一存 UTC）
utc_now = datetime.now(timezone.utc)
```

格式代码最常用的几个：`%Y` 四位年、`%m` 月、`%d` 日、`%H` 24 制小时、`%M` 分、`%S` 秒。

### 13.3 collections：容器增强包

```python
from collections import Counter, defaultdict, deque, namedtuple

# Counter：计数器，统计神器
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
c = Counter(words)
print(c)                        # Counter({'apple': 3, 'banana': 2, 'cherry': 1})
print(c.most_common(2))         # [('apple', 3), ('banana', 2)] —— Top N 问题一行解决
print(c["apple"])               # 3

# defaultdict：带默认工厂的 dict，免去了 setdefault 的啰嗦
groups = defaultdict(list)      # 访问不存在的键时自动调用 list() 创建 []
for w in words:
    groups[w[0]].append(w)
print(dict(groups))             # {'a': ['apple', ...], 'b': [...], 'c': ['cherry']}

# deque：双端队列，两端操作都是 O(1)（list 头部操作是 O(n)）
q = deque(maxlen=3)             # maxlen：固定长度的滑动窗口，满了自动挤出最旧的
q.append(1)
q.appendleft(0)
q.popleft()

# namedtuple：带名字的轻量"记录"，比 dict 省内存、有字段提示
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
print(p.x, p[0])                # 3 3 —— 既能按名字也能按下标访问
```

### 13.4 itertools：迭代器工具箱

```python
from itertools import chain, islice, groupby, combinations, product, count, cycle

# chain：把多个可迭代对象串成一个
print(list(chain([1, 2], [3, 4], "ab")))   # [1, 2, 3, 4, 'a', 'b']

# islice：对迭代器切片（普通切片不支持迭代器/无限序列）
print(list(islice(count(1), 5, 10)))       # [6, 7, 8, 9, 10]

# groupby：分组（注意：只合并相邻的相同键，数据需先按键排序）
data = [("A", 1), ("A", 2), ("B", 3)]
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))    # A [('A', 1), ('A', 2)]  /  B [('B', 3)]

# combinations / product：组合与笛卡尔积
print(list(combinations([1, 2, 3], 2)))    # [(1, 2), (1, 3), (2, 3)]
print(list(product("AB", [1, 2])))         # [('A', 1), ('A', 2), ('B', 1), ('B', 2)]
```

`itertools` 的所有函数都返回迭代器（惰性），与生成器管道配合使用威力最大。

### 13.5 re：正则表达式

```python
import re

text = "联系方式：138-1234-5678 或 139-9876-5432"

# search：找第一个匹配，返回 Match 对象或 None
m = re.search(r"1\d{2}-\d{4}-\d{4}", text)
print(m.group())        # 138-1234-5678
print(m.start(), m.end())  # 匹配的起止位置

# findall：找所有匹配，返回字符串列表
print(re.findall(r"1\d{2}-\d{4}-\d{4}", text))
# ['138-1234-5678', '139-9876-5432']

# 分组捕获：() 圈出的部分可以单独取出
m = re.search(r"(\d{3})-(\d{4})-(\d{4})", text)
print(m.groups())       # ('138', '1234', '5678')
print(m.group(1))       # 138

# sub：替换
print(re.sub(r"\d", "*", text))   # 联系方式：***-****-**** 或 ***-****-****

# split：按模式切分
print(re.split(r"[,，;；]", "a,b，c;d"))   # ['a', 'b', 'c', 'd']

# 标志位：IGNORECASE 忽略大小写，MULTILINE 多行模式
print(re.findall(r"error", "Error ERROR error", re.IGNORECASE))

# 性能提示：反复使用同一模式时先编译
pattern = re.compile(r"1\d{2}-\d{4}-\d{4}")
print(pattern.findall(text))
```

模式字符串前的 `r` 表示原始字符串（raw string），让 `\d` 中的反斜杠不被 Python 字符串层转义——写正则永远加 `r`。

### 13.6 logging：别再用 print 调试了

`print` 的问题：无法分级、无法关闭、没有时间和来源信息、不能写文件。`logging` 是标准答案：

```python
import logging

logging.basicConfig(
    level=logging.INFO,                # 低于此级别的日志被丢弃
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    # handlers=[logging.FileHandler("app.log", encoding="utf-8")]  # 输出到文件
)

logger = logging.getLogger(__name__)   # 惯例：每个模块用自己的 logger

logger.debug("调试细节，排查问题时才看")
logger.info("程序正常运行的关键节点")
logger.warning("还能跑，但值得注意")
logger.error("出错了，但程序继续")
logger.exception("在 except 块里用，自动附带异常堆栈")   # 等价于 error + traceback
```

级别从低到高：`DEBUG < INFO < WARNING < ERROR < CRITICAL`。设置 `level=INFO` 后 `debug` 日志自动静默——上线把级别调高，排障时调低，**代码一行不用改**，这是 print 永远做不到的。模块化项目中每个模块调用 `getLogger(__name__)`，日志里就能看清消息来自哪个文件。

### 13.7 json 回顾与更多宝藏

`json` 已在第 11 节讲过，不再重复。标准库中还有许多后续课程会用到的宝藏，这里留个印象，用时再查文档：

- `functools`：`lru_cache`（一行加缓存）、`wraps`（装饰器必备）、`partial`（部分应用，冻结函数的部分参数）
- `dataclasses`：见第 7 节
- `contextlib`：`contextmanager`、`suppress`（忽略指定异常）、`ExitStack`
- `threading` / `multiprocessing` / `asyncio`：并发三件套（asyncio 是 FastAPI 高并发的根基）
- `hashlib`：MD5/SHA 系列哈希（密码、文件校验）
- `secrets`：加密安全的随机数（生成 token 用它，不要用 `random`）
- `urllib.parse`：URL 解析与拼接
- `time` / `timeit`：计时与小段代码的性能测量
- `enum`：枚举类型，管理有限状态集合
- `queue`：线程安全的队列
- `unittest` / `doctest`：内置测试框架（第三方首选 pytest）

---

## 小结

本章覆盖了 Python 日常开发的全部核心语法，重点回顾这些"为什么"：

- **虚拟环境**解决依赖隔离与可复现性，本质是修改 `PATH`；新手先用 `venv + pip`，进阶选 `uv`。
- **一切皆对象、变量是标签**——字符串与容器的一切行为（不可变性、传参语义、深浅拷贝）都由这个模型推出。
- **可变 vs 不可变**是最重要的类型划分：它决定谁能当字典键、函数会不会污染外部数据、默认参数为什么踩坑。
- `for ... else` 读作 `no_break`；`match-case` 是带解构的模式匹配。
- **LEGB** 决定名字查找顺序；**闭包**让函数携带私有状态，是装饰器的地基。
- **装饰器**本质就是 `func = decorator(func)` 的语法糖；写装饰器必带 `@functools.wraps`；带参数装饰器是三层函数。
- 类变量所有实例共享（可变类变量是坑）；**MRO** 用 C3 算法解决多继承查找；`@property` 给你重构自由；`@dataclass` 消灭样板代码。
- 异常处理遵循 EAFP；`finally` 必定执行；**`with`** 把资源获取与释放绑定在语法层面，依赖 `__enter__` / `__exit__` 协议。
- **生成器**用 `yield` 冻结/恢复函数帧，以 O(1) 内存处理任意大的数据流。
- `import` 会执行模块顶层代码且只执行一次；`__name__ == '__main__'` 让文件兼具"可导入"与"可运行"双重身份；包内相对导入要用 `python -m` 运行。
- 文件操作铁律：`with` + `encoding="utf-8"`；路径用 `pathlib`；JSON 记住"带 s 操作字符串"。
- **类型注解**不影响运行时，但它是静态检查、IDE 提示和 FastAPI 自动校验的地基。
- 标准库即军火库：`Counter`、`defaultdict`、`deque`、`itertools`、`logging`、`datetime` 几乎天天用。

## 常见问题

**1. 面试题：Python 的默认参数陷阱是什么？如何避免？**

默认参数的值在 **函数定义时** 求值一次并保存在函数对象上，不是每次调用时新建。所以 `def f(items=[])` 会让所有调用共享同一个列表，前一次调用的修改会泄漏到后一次。正确写法是默认值用 `None`，函数体内再 `if items is None: items = []`。同理，`@dataclass` 中可变默认值要用 `field(default_factory=list)`。

**2. `is` 和 `==` 有什么区别？判断 None 为什么必须用 `is`？**

`is` 比较对象身份（内存地址是否相同），`==` 调用对象的 `__eq__` 方法比较"值"。`None` 是全解释器唯一的单例，`is None` 直接比较身份，快且绝对可靠；而 `== None` 可能被某个类自定义的 `__eq__` 欺骗。另外小整数、短字符串因解释器缓存可能 `is` 成立，但这是实现细节，**判值永远用 `==`，判单例（None、枚举成员）才用 `is`**。

**3. 深拷贝和浅拷贝的区别？什么场景必须深拷贝？**

浅拷贝（`copy()`、`[:]`、`copy.copy`）只复制外层容器，内层子对象仍是同一个引用；深拷贝（`copy.deepcopy`）递归复制整个对象图，完全独立。当数据结构中包含嵌套的可变对象、且你需要"改副本不影响原件"时必须深拷贝，例如：基于配置模板生成多份配置再各自修改、撤销/快照功能。纯标量结构浅拷贝即可，深拷贝慢且耗内存，滥用是性能浪费。

**4. 生成器和列表的本质区别是什么？为什么生成器省内存？**

列表是"急切"的：创建时把所有元素物化进内存，内存占用 O(n)。生成器是"惰性"的：它只保存函数帧的状态和"如何计算下一个值"的逻辑，每次 `next()` 才推进一步，任意时刻内存里只有一个元素加一帧，占用 O(1)。`yield` 的机制是冻结当前帧（局部变量、指令位置）并交还控制权，下次调用恢复执行。因此生成器能处理远超内存的文件、甚至表示无限序列，还能串成管道做流式处理。

**5. 为什么要在脚本里写 `if __name__ == '__main__'`？**

`import` 一个模块时 Python 会 **执行它的全部顶层代码**。如果没有这层判断，你的测试代码、示例调用会在别人 import 时全部跑一遍。`__name__` 在直接运行时是 `'__main__'`、被导入时是模块名，所以这个判断把"入口逻辑"保护起来：直接运行时执行，被导入时跳过。这让一个文件既能作为库被复用，又能作为脚本独立运行。
