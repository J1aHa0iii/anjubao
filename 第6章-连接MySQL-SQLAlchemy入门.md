# 第 6 章 连接 MySQL：SQLAlchemy 入门

> **本章定位**：数据库落地篇（上）。五章以来的内存数据时代在本章开始谢幕：SQLAlchemy 把第 1 章定稿的 5 张表"翻译"成 Python 类、建进 MySQL 5.6；**房屋与公告两类资源完成数据源心脏移植**（6 个接口的查询与增删改全部走数据库）。工单与附件涉及外键关联与事务，按坡度设计留到第 7 章——本章专心单表 CRUD，把 ORM 基本功练扎实。
>
> **本章的升级纪律（"心脏移植"条款）**：每个接口只更换**数据访问行**——校验逻辑、错误码、docstring、注释一律保持原样；每个接口的改造都附"改动清单"。有两处**显式声明的附带升级**（时间字段类型、from_attributes 配置），见第 4 节开头的声明框。
>
> **开发约定**（延续）：`uvicorn main:app --reload` 运行，`http://127.0.0.1:8000/docs` 测试。本章新增两个文件 `db.py`、`models.py`、一个种子脚本 `seed_data.py`，`store.py` 迎来第一次瘦身。

---

## 学习目标

完成本章学习后，你应当能够：

1. **懂 ORM**：说出三层映射关系（类↔表、实例↔行、属性↔列），以及相比手写 SQL 的收益与代价。
2. **会连接**：完成 SQLAlchemy + PyMySQL 连接 MySQL 5.6 的全部准备（建库 utf8mb4、连接 URL、create_engine、SessionLocal、get_db）。
3. **会建模**：用声明式语法定义 5 张表的 ORM 模型（主键自增、可空、唯一、默认值、联合唯一、ForeignKey 基本用法），用 create_all 建表并用 echo 核对生成的 SQL。
4. **懂会话**：说出 Session 三铁律（查询不用 commit / 增删改必须 commit / commit 前未真正入库），用 `Depends(get_db)` 管理请求级会话。
5. **会移植**：把"内存字典版"接口逐行改写为"数据库版"（query/filter、add/commit/refresh、delete、setattr 更新），并解释 from_attributes 为什么必需。

---

## 1. 本章项目任务与热身：数据的"土拨鼠之日"

### 本章项目任务

| # | 交付物 | 说明 |
|---|--------|------|
| 1 | `db.py` | 引擎、会话工厂、get_db 依赖函数 |
| 2 | `models.py` | 5 张表的 ORM 模型（字段与第 1 章表设计逐字一致）+ create_all 建表 |
| 3 | 接口数据库化 × 6 | houses 模块 1 个 + notices 模块 5 个，全部切到 MySQL |
| 4 | `seed_data.py` | 种子脚本：插入与原内存版一致的初始数据，可重复执行 |
| 5 | `store.py` 瘦身 | HOUSES、NOTICES 两个数据集下岗 |

### 热身：内存数据的四宗罪

第 5 章结束时我们说过"重启服务，ATTACHMENTS 清零"。其实何止附件——五个章节以来，所有数据都活在 main.py 的进程内存里：

1. **重启即失忆**：每章测试造的数据，一重启全部归零，明天继续课得从头造；
2. **改数据要改代码**：想给小区添一套新房？打开 store.py 编辑 Python 列表——数据和行为混在一起，这是工程大忌；
3. **无法共享**：你的服务和我的服务各有一份内存，数据永远对不上（真实世界：多进程、多服务器，内存根本不互通）；
4. **没有并发与事务保障**：第 3 章 `max(id)+1` 生成工单号的并发隐患是实锤——两个请求同时到达就生成重复 id。

四宗罪的解药你早就认识：**数据库**。而你已学过 MySQL 5.6——本章不推翻任何 MySQL 知识，只是**换一种方式执行它们**。

### 问题驱动：直接用 pymysql 裸写 SQL 行不行？

最直接的方案：接口里直接用 pymysql 写 SQL。以"查公告列表"为例，裸写版长这样（**反面示范，无需跟敲**）：

```python
# 裸 SQL 版"查公告"——反面示范
import pymysql

conn = pymysql.connect(host="127.0.0.1", user="root", password="***", database="anjubao")
cursor = conn.cursor()
cursor.execute("SELECT id, title, content, category, created_at FROM notice WHERE category = %s", (category,))
rows = cursor.fetchall()
result = []
for row in rows:                      # 手动把"行的元组"拼成字典
    result.append({"id": row[0], "title": row[1], "content": row[2], ...})
cursor.close()
conn.close()
```

能用，但每个接口都要复制一遍这套样板，而且数出三个痛点：

| 痛点 | 具体表现 |
|------|---------|
| **SQL 字符串满天飞** | 表名、列名散落在字符串里，改表名要全局搜字符串；手滑拼接还有 SQL 注入风险 |
| **结果手动转对象** | fetchall 给你一堆元组，字段和下标对不上号，每查一次拼一次字典 |
| **连接管理全是样板** | connect/close/try/finally，写十遍错三遍 |

> 💡 **一句话引出主角**
> 如果表能变成类、行能变成对象、SQL 能自动生成、连接能自动管理呢？这就是 **ORM**（Object-Relational Mapping，对象关系映射）——Python 世界里最主流的 ORM 框架就是本章的 **SQLAlchemy**。

### 小结

- 内存数据四宗罪：失忆、改数据=改代码、不共享、无并发事务保障。
- 裸写 SQL 三痛点：字符串满天飞、结果手动转对象、连接样板代码。
- 本章用 SQLAlchemy 完成"表变类、行变对象"的翻译，MySQL 知识全部有效。

---

## 2. SQLAlchemy 连接数据库：引擎、驱动与会话

### 安装与 MySQL 侧准备

在 anjubao 环境安装两件套（装完记得 `pip freeze > requirements.txt`——第 1 章的习惯）：

```bash
pip install sqlalchemy pymysql
```

- **sqlalchemy**：ORM 框架本体；
- **pymysql**：**驱动**——SQLAlchemy 自己不会说 MySQL 的"方言"，驱动负责真正的网络通信与协议翻译（本课程选 PyMySQL：纯 Python 实现、无需编译、对 MySQL 5.6 支持完好）。

然后用第 1 章 1.6.6 节的 root 账号登录 MySQL，建库：

```sql
CREATE DATABASE anjubao CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

> ⚠️ **为什么是 utf8mb4 而不是 utf8**：MySQL 的 `utf8` 是历史遗留的残缺实现（最多 3 字节），存不下 emoji 和部分生僻字；`utf8mb4` 才是完整的 4 字节 UTF-8。这是 MySQL 圈最著名的坑之一，建库时一步到位。

### 连接 URL 解剖

SQLAlchemy 用一个 URL 字符串描述"连哪台库、用什么驱动、以谁的身份"：

```text
mysql+pymysql://root:你的密码@127.0.0.1:3306/anjubao?charset=utf8mb4
└────┬────┘   └─┬─┘ └──┬──┘└────┬────┘└─┬─┘ └───┬───┘└────┬────┘
 协议+驱动     用户   密码      主机    端口    库名     字符集
（告诉 SQLAlchemy 用 pymysql 说 MySQL 方言）
```

### db.py：引擎、会话工厂、get_db

```python
# db.py —— 数据库连接与会话管理（整个应用的数据入口）
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

# 连接 URL：把"你的密码"换成你的 root 密码
DATABASE_URL = "mysql+pymysql://root:你的密码@127.0.0.1:3306/anjubao?charset=utf8mb4"

# 引擎：进程级的"连接池总管"，整个应用只创建一次
# echo=True 会把自动生成的 SQL 打印到终端——本章调试的利器，正式用时可关
engine = create_engine(DATABASE_URL, echo=True)

# 会话工厂：每次调用产出一个新会话（Session）
SessionLocal = sessionmaker(bind=engine)


class Base(DeclarativeBase):
    """全部 ORM 模型的共同基类（第 3 节的模型都继承它）"""
    pass


def get_db():
    """依赖函数：每个请求借一个会话，请求结束自动归还"""
    db = SessionLocal()
    try:
        yield db          # 把会话交给接口函数使用（yield：函数在此"暂停"，请求结束后继续往下走）
    finally:
        db.close()        # 无论接口成功还是报错，会话必归还
```

**引擎与会话的分工**（**图 6-1**）——用"公司总机与分机"来记：

```mermaid
flowchart TB
    R["接口函数<br/>db: Session = Depends(get_db)"] --> S1["会话 Session<br/>（分机：一次请求的工作对话）"]
    R2["另一个请求"] --> S2["会话 Session<br/>（每请求一个，互不干扰）"]
    S1 --> P["引擎 + 连接池<br/>（总机：统一管理到 MySQL 的连接，复用避免反复握手）"]
    S2 --> P
    P --> M[("MySQL 5.6<br/>anjubao 库")]
```

**图 6-1 引擎管连接池，Session 管一次工作对话，每个请求一个会话**

> 💡 **先睹为快：Depends（依赖注入的第一次实战）**
> 接口形参写 `db: Session = Depends(get_db)`，FastAPI 会在**每个请求进来时**自动执行 get_db——拿到会话塞给你的函数，请求结束后自动执行 close。这个机制叫**依赖注入**，它能注入的不只是数据库会话（第 9 章的"当前登录用户"就是同一个机制），**第 9 章正式深入**，本章照写即可——与第 2 章 HTTPException、第 5 章 async 同款的"先睹为快"待遇。

### 常见坑

- ⚠️ **密码含特殊字符**：URL 里的 `@`、`:`、`/` 等字符会破坏 URL 解析（密码里有 `@` 会被当成主机名的开头）。密码含特殊字符时要么改密码，要么对密码做 URL 编码（`urllib.parse.quote_plus("p@ss")`）。
- ⚠️ **create_engine 不会立刻连接**：它只是"造好了总机"，真正拨号发生在第一次执行 SQL 时（懒连接）。所以连接信息写错时，启动服务**不报错**，第一个接口请求才炸——排错时别被"没报错"迷惑。
- ⚠️ **忘写 `?charset=utf8mb4`**：中文能存但可能乱码（驱动默认字符集与库不一致时）。URL 尾巴上的这个参数别省。
- ⚠️ **每个请求一个会话，不是一个全局会话**：全局共享一个 Session 是并发灾难（多个请求交错操作同一个对话）。get_db 的"每请求一会话"就是为此而生。

### 小结

- 建库用 utf8mb4；URL = 协议+驱动://用户:密码@主机:端口/库名?charset。
- 引擎（进程级、连一次建好）管连接池；Session（请求级）管一次工作对话。
- `Depends(get_db)` 自动开会话/自动关会话——依赖注入的第一次实战，第 9 章深入。

---

## 3. 声明式模型：5 张表翻译成 Python 类

### ORM 的三层映射

先立总纲（**图 6-2**）——后面所有代码都遵循这三行对应关系：

```mermaid
flowchart LR
    A["Python 类<br/>class Notice"] <-->|"声明式映射"| T["MySQL 表<br/>notice"]
    B["类的实例<br/>Notice(...)" ] <-->|"一行数据"| R["表的一行<br/>一条公告"]
    C["实例属性<br/>notice.title"] <-->|"一列"| L["行的字段<br/>title 列"]
```

**图 6-2 ORM 三层映射：类↔表、实例↔行、属性↔列**

SQLAlchemy 的声明式写法（2.x 版本）：类属性用 `Mapped[类型]` 注解 + `mapped_column(...)` 描述列——你会发现它与 Pydantic 模型**形似而神异**，本节末尾有防混淆对照表。

### models.py：5 张表逐张翻译

```python
# models.py —— 5 张表的 ORM 模型（字段与第 1 章表设计逐字一致）
from datetime import datetime

from sqlalchemy import ForeignKey, String, Text, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column

from db import Base


class User(Base):
    """用户表：三种角色共用一张表，role 字段区分"""
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)   # 第 8 章存哈希值，列先建好
    phone: Mapped[str | None] = mapped_column(String(20))
    role: Mapped[str] = mapped_column(String(10), nullable=False, default="owner")
    created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)


class House(Base):
    """房屋表"""
    __tablename__ = "house"
    __table_args__ = (
        # 楼栋+单元+房号 联合唯一：一套房不能登记两次（第 1 章表设计的约定）
        UniqueConstraint("building", "unit", "room", name="uq_house_location"),
    )

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("user.id"), nullable=False)
    building: Mapped[str] = mapped_column(String(20), nullable=False)
    unit: Mapped[str] = mapped_column(String(20), nullable=False)
    room: Mapped[str] = mapped_column(String(20), nullable=False)


class RepairOrder(Base):
    """报修工单表（第 7 章的主场，本章先把表建好）"""
    __tablename__ = "repair_order"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    house_id: Mapped[int] = mapped_column(ForeignKey("house.id"), nullable=False)
    worker_id: Mapped[int | None] = mapped_column(ForeignKey("user.id"))   # NULL = 尚未派单
    type: Mapped[str] = mapped_column(String(10), nullable=False)
    description: Mapped[str] = mapped_column(String(500), nullable=False)
    status: Mapped[str] = mapped_column(String(10), nullable=False, default="待受理")
    score: Mapped[int | None] = mapped_column()                            # NULL = 尚未评价
    created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
    finished_at: Mapped[datetime | None] = mapped_column()                 # NULL = 尚未完工


class Attachment(Base):
    """附件表（文件本体在磁盘 uploads/ 目录，这里只存目录记录）"""
    __tablename__ = "attachment"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("repair_order.id"), nullable=False)
    filename: Mapped[str] = mapped_column(String(255), nullable=False)
    stored_path: Mapped[str] = mapped_column(String(255), nullable=False)
    file_type: Mapped[str] = mapped_column(String(10), nullable=False)
    uploaded_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)


class Notice(Base):
    """公告/规章表（content 是后续 RAG 课程的检索语料）"""
    __tablename__ = "notice"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"), nullable=False)
    title: Mapped[str] = mapped_column(String(100), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)   # 长文本用 Text，不受 String(n) 长度限制
    category: Mapped[str] = mapped_column(String(20), nullable=False)
    created_at: Mapped[datetime] = mapped_column(nullable=False, default=datetime.now)
```

语法要点逐条过：

| 语法 | 作用 | 第 1 章表设计的对应 |
|------|------|---------------------|
| `__tablename__ = "user"` | 表名（不写则用类名小写——**显式写清是工程习惯**） | 表名 |
| `primary_key=True, autoincrement=True` | 主键自增 | "主键，自增" |
| `String(50)` / `Text` | 定长上限字符串 / 长文本 | VARCHAR(50) / TEXT |
| `unique=True` | 唯一约束 | "唯一"（username） |
| `nullable=False` | 非空（Mapped[int] 默认非空，Mapped[int \| None] 默认可空——**写出来更醒目**） | "非空" / "可空" |
| `default="owner"` | 插入时的默认值 | "默认 'owner'" |
| `default=datetime.now` | 动态默认值：**每次插入现算**（写成 `datetime.now()` 就是"建表时刻"的静态值——第 3 章 default_factory 的 SQLAlchemy 版） | "注册时间" |
| `UniqueConstraint(...)` | 联合唯一 | building+unit+room 联合唯一 |
| `ForeignKey("user.id")` | 外键列（本章只建列；**跨表自动关联查询的深玩法是第 7 章主场**） | "外键 → user.id" |

> 💡 **`Mapped[int | None] = mapped_column()` 与 `Mapped[int] = mapped_column(nullable=False)`**：类型注解里带 `| None`，SQLAlchemy 就知道这列可空——worker_id、score、finished_at、phone 四个"业务上允许还没有"的字段都是这个形态，与第 1 章表设计里"可空"标注一一对应。

### 建表与核对：亲眼看它生成的 SQL

在项目根目录新建 `create_tables.py`（一次性脚本）：

```python
# create_tables.py —— 建表脚本（只需在表结构变化后重跑）
import models                     # noqa: F401  导入即注册：确保 5 个模型都被 Base 认识
from db import Base, engine

Base.metadata.create_all(engine)  # 把所有模型翻译成 CREATE TABLE，发给 MySQL
print("建表完成")
```

运行 `python create_tables.py`——因为 db.py 开着 `echo=True`，终端会打印出全部生成的 SQL，其中 user 表部分：

```text
CREATE TABLE user (
    id INTEGER NOT NULL AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    role VARCHAR(10) NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id),
    UNIQUE (username)
)
```

拿它与第 1 章的 user 表字段明细表**逐字段核对**——类型、非空、唯一、自增全部对上。这就是"ORM 不是黑盒"的证据时刻：你写的每一个 `mapped_column`，都变成了你认识的 DDL。再用 MySQL 命令行双向验证：

```sql
USE anjubao;
SHOW TABLES;      -- 应看到 5 张表
DESC user;        -- 字段结构核对
```

> 📌 注意建表顺序无关紧要：外键依赖的表（如 user）即使后定义，create_all 也会自动按依赖排序生成——这是它替你操心的又一个细节。

### ORM 模型 vs Pydantic 模型：全课程最易混淆对照

两套"类 + 类型注解"在项目里并存，从本章起必须分得清清楚楚：

| | ORM 模型（models.py） | Pydantic 模型（schemas.py） |
|---|----------------------|---------------------------|
| 管什么 | **数据库**的表与行 | **接口**进出的数据格式 |
| 继承谁 | `Base`（DeclarativeBase） | `BaseModel` |
| 一个类代表 | 一张表；一个实例=一行数据 | 一份请求体或响应的形状 |
| 声明工具 | `Mapped[...] = mapped_column(...)` | `字段: 类型 = Field(...)` |
| 命名 | 名词裸奔：`Notice`、`House` | 动作后缀：`NoticeCreate`、`NoticeOut` |
| 谁消费它 | SQLAlchemy（翻译成 SQL） | FastAPI（校验与序列化） |

一句话：**ORM 模型是"账本上的行"，Pydantic 模型是"进出的单据"**——账本不外借（内部字段要过滤），单据不落库（受控字段不收）。两套各司其职，第 4 节会看到它们在一次请求里如何接力。

### 常见坑

- ⚠️ **String 必须给长度**：MySQL 的 VARCHAR 需要长度，`Mapped[str]` 不写 `String(n)` 建表直接报错（Text 除外）。
- ⚠️ **忘写 `__tablename__`**：能跑，但表名默认取类名——将来类改名，表名跟着变，灾难现场。永远显式写。
- ⚠️ **default 写成 `datetime.now()`**：所有行的 created_at 都是"进程启动时刻"——第 3 章 default_factory 陷阱原样复刻，传函数本身，不要调用它。
- ⚠️ **create_all 不会改已存在的表**：它只"缺哪张建哪张"。改了模型列定义后再跑，已存在的表**不会更新**——开发期最简单的办法是 DROP TABLE 后重建（数据反正是种子脚本造的），正式的表结构演进工具（Alembic）见附录 C。

### 小结

- 声明式模型三件套：继承 Base、`__tablename__`、`Mapped[类型] = mapped_column(列约束)`。
- 5 张表全部落地，字段与第 1 章逐字一致；ForeignKey 本章只建列，关联查询第 7 章。
- echo=True 让自动生成的 SQL 现形——ORM 不是黑盒。
- ORM 模型=账本行，Pydantic 模型=进出单据，命名体系天然区分。

---

## 4. 心脏移植（上）：房屋与公告接口数据库化

### 移植前的两份升级声明

按升级纪律，凡超出"换数据访问行"的变化必须先声明。本次移植有两处附带升级，**都是数据类型层面的规范化**：

> 📌 **声明一：时间字段从 str 升级为 datetime。** 内存时代为了省事，created_at 存的是 `"2026-10-22 09:15"` 格式的**字符串**；落库后数据库列是真正的 DATETIME，Out 模型的对应字段也随之改为 `datetime` 类型。**响应格式会有一个可见变化**：`"2026-10-22 09:15"` → `"2026-10-22T09:15:00"`（ISO 8601 标准格式，秒数出现，日期与时间之间用 T 分隔）。前端解析这种格式反而更方便（JS 的 `new Date()` 直接认）。
>
> 📌 **声明二：Out 模型新增 `from_attributes` 配置。** 安检（response_model）过去只会"从字典读键"，现在数据源变成了 ORM 对象，要教会它"从对象读属性"：`model_config = ConfigDict(from_attributes=True)`。没加它，接口必 500（ResponseValidationError 老朋友）。本章只给 HouseOut、NoticeOut 加；RepairOrderOut 等工单系模型第 7 章移植时再加。

schemas.py 的两处改动：

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict


class HouseOut(BaseModel):
    """房屋响应模型：owner_id 是内部字段，不放行"""
    model_config = ConfigDict(from_attributes=True)    # ← 新增：允许从 ORM 对象读属性

    id: int
    building: str
    unit: str
    room: str


class NoticeOut(BaseModel):
    """公告的出门规矩：客户端只该看到这些字段"""
    model_config = ConfigDict(from_attributes=True)    # ← 新增

    id: int
    title: str
    content: str
    category: str
    created_at: datetime                               # ← 声明一：str → datetime
```

### 第一个移植示例：GET /houses/{house_id}（最详尽）

改造前后并排对比（左：第 5 章末的内存版；右：本章数据库版）：

```python
# ── 改造前（内存版）──────────────────────────
@router.get("/{house_id}", response_model=HouseOut)
def get_house(
    house_id: int = Path(
        ge=1,
        title="房屋ID",
        description="要查询的房屋编号，从 1 开始",
        examples=[1, 3],
    )
):
    """房屋详情：按编号查询某套房屋的信息"""
    for house in HOUSES:                    # ← 数据访问（遍历内存）
        if house["id"] == house_id:
            return house
    raise HTTPException(status_code=404, detail=f"房屋 {house_id} 不存在")
```

```python
# ── 改造后（数据库版）────────────────────────
@router.get("/{house_id}", response_model=HouseOut)
def get_house(
    house_id: int = Path(
        ge=1,
        title="房屋ID",
        description="要查询的房屋编号，从 1 开始",
        examples=[1, 3],
    ),
    db: Session = Depends(get_db),          # ← 改动①：新增会话形参
):
    """房屋详情：按编号查询某套房屋的信息"""
    house = db.query(House).filter(House.id == house_id).first()   # ← 改动②：内存遍历 4 行 → 数据库查询 1 行
    if house is None:                       # （校验、错误码、docstring 原样未动）
        raise HTTPException(status_code=404, detail=f"房屋 {house_id} 不存在")
    return house
```

**改动清单**（升级纪律的验收材料）：

| # | 位置 | 改动 |
|---|------|------|
| ① | 形参 | 新增 `db: Session = Depends(get_db)`（文件头相应新增 `from fastapi import Depends`、`from sqlalchemy.orm import Session`、`from db import get_db`、`from models import House`，并删除 `from store import HOUSES`） |
| ② | 数据访问 | 4 行内存遍历 → 1 行 `db.query(House).filter(House.id == house_id).first()` |
| — | 校验/错误码/docstring/Path 签名 | **一字未动** |

新语法拆解：

- `db.query(House)`——"我要查 House 表"，对应 `SELECT * FROM house`；
- `.filter(House.id == house_id)`——条件，对应 `WHERE id = %s`（参数化，**天然免疫 SQL 注入**——裸 SQL 痛点一的终结）；
- `.first()`——取第一条，没有则返回 **None**（正好衔接 `if house is None` 的 404，连判空的写法都和内存版 `target is None` 同构）；
- 返回的 `house` 是 **ORM 对象**（House 实例），出门安检（HouseOut + from_attributes）直接从它身上读属性——裸 SQL 痛点二的终结。

### 公告模块五连移植

同样的套路，逐个过。此处只列每个接口的**数据访问行改动**，其余一律原样（完整代码见各改动清单拼装）：

**list_notices（列表）**：

```python
@router.get("", response_model=list[NoticeOut])
def list_notices(category: str | None = None, db: Session = Depends(get_db)):
    """公告列表：可按分类筛选（公告/规章），不传则返回全部"""
    if category is None:            # 没传筛选条件 → 全部返回（原样）
        return db.query(Notice).all()                       # ← 原：return NOTICES
    return db.query(Notice).filter(Notice.category == category).all()   # ← 原：5 行遍历筛选
```

**search_notices（搜索）**——内存版 `in` 子串判断的 SQL 亲戚登场：

```python
@router.get("/search", response_model=list[NoticeOut])
def search_notices(
    keyword: str | None = Query(
        default=None,
        min_length=1,
        max_length=20,
        title="搜索关键词",
        description="在公告标题和正文中模糊搜索，1~20 个字符",
        examples=["停水", "装修"],
    ),
    db: Session = Depends(get_db),
):
    """公告搜索：按关键词模糊匹配标题与正文，不传关键词则返回全部"""
    if keyword is None:
        return db.query(Notice).all()
    query = db.query(Notice).filter(
        # contains 生成 SQL 的 LIKE '%关键词%'；| 是"或"（对应 SQL 的 OR）
        # ——与内存版 in 子串判断、or 连接一一对应
        Notice.title.contains(keyword) | Notice.content.contains(keyword)
    )
    return query.all()
```

**create_notice（发布）**——本章最有仪式感的一刻：

```python
@router.post("", response_model=NoticeOut, status_code=201)
def create_notice(notice: NoticeCreate, db: Session = Depends(get_db)):
    """发布公告（请求体版——正式版本）"""
    data = Notice(                          # ← 原：手工拼 id/max+1/created_at 的 8 行字典
        author_id=1,                        # 简化处理，第 9 章改为当前登录用户
        title=notice.title,                 # 点语法访问模型字段
        content=notice.content,
        category=notice.category,
        # id 由 AUTO_INCREMENT 生成、created_at 由列默认值生成——都不用我们操心
    )
    db.add(data)        # 记账：登记一条待插入的 INSERT
    db.commit()         # 结账：真正写入 MySQL
    db.refresh(data)    # 回读：拿回数据库生成的 id、created_at（响应里要用）
    return data
```

第 3 章埋的 `max(...)+1` 并发隐患伏笔，在这里正式退场：**AUTO_INCREMENT 由数据库在内部加锁分配，两个请求同时 INSERT 也绝不会拿到同一个 id**——第 4 章附录 B 幂等讨论的悬念在此落了一半（另一半"防重复提交"仍需业务层，第 9 章再谈）。

**update_notice（修改）**——`dict.update` 的 ORM 亲戚是 `setattr` 循环：

```python
@router.put("/{notice_id}", response_model=NoticeOut)
def update_notice(
    notice_id: int = Path(ge=1, description="要修改的公告ID"),
    update: NoticeUpdate,             # 请求体模型（形参名随意，类型注解决定它从请求体读取）
    db: Session = Depends(get_db),
):
    """修改公告：只更新客户端传了的字段（部分更新）"""
    # 第一步：找公告，找不到 404（原样；数据访问行更换）
    target = db.query(Notice).filter(Notice.id == notice_id).first()   # ← 原：4 行遍历
    if target is None:
        raise HTTPException(status_code=404, detail=f"公告 {notice_id} 不存在")

    # 第二步：取出"客户端真正传了的字段"（原样）
    changes = update.model_dump(exclude_unset=True)
    if not changes:
        raise HTTPException(status_code=400, detail="未提供任何要修改的字段")

    # 第三步：逐字段覆盖（dict.update 的 ORM 版：setattr 循环）→ 提交
    for field, value in changes.items():          # ← 原：target.update(changes)
        setattr(target, field, value)
    db.commit()                                    # ← 新增：改动必须 commit
    db.refresh(target)                             # ← 新增：回读最新数据供出门安检
    return target
```

**delete_notice（删除）**：

```python
@router.delete("/{notice_id}", status_code=204)
def delete_notice(notice_id: int = Path(ge=1, description="要删除的公告ID"), db: Session = Depends(get_db)):
    """删除公告：成功返回 204（无响应体）"""
    target = db.query(Notice).filter(Notice.id == notice_id).first()   # ← 原：4 行遍历
    if target is None:
        raise HTTPException(status_code=404, detail=f"公告 {notice_id} 不存在")

    db.delete(target)      # ← 原：NOTICES.remove(target)
    db.commit()            # ← 新增：删除也要 commit
    return None            # 204：必须空响应体（原样）
```

### Session 三铁律（贴在显示器上）

| 铁律 | 说明 | 违反后果 |
|------|------|---------|
| **查询不用 commit** | query 只是读，不改库 | （多写 commit 无害但多余） |
| **增删改必须 commit** | add/delete 只是"记账"，commit 才"结账"入库 | 忘 commit：响应 201/200 一切正常，**库里根本没有数据**——"假成功"，本章第一大坑 |
| **commit 之前未真正入库** | commit 前发生异常，记账自动作废（回滚） | 这正是"事务"的雏形——第 7 章正式展开 |

### store.py 瘦身与 seed_data.py 上岗

移植完成后，`store.py` 里的 HOUSES、NOTICES **整个删除**（心脏移植完成的仪式感）；WORKERS、REPAIR_ORDERS、ATTACHMENTS 保留——工单模块还在用，第 7 章移植。

但数据库现在是空表，接口查出来全是空的——需要**种子数据**。新建 `seed_data.py`，插入与原内存版**完全一致**的数据：

```python
# seed_data.py —— 开发期种子数据：可重复执行（先清后插，跑一次数据复位一次）
from datetime import datetime

from db import SessionLocal
from models import Attachment, House, Notice, RepairOrder, User


def dt(s: str) -> datetime:
    """把 '2026-10-22 09:15' 样式的字符串转成 datetime 对象（内存时代的格式就此退休）"""
    return datetime.strptime(s, "%Y-%m-%d %H:%M")


def main():
    db = SessionLocal()
    try:
        # ── 先清后插：保证脚本可重复执行 ──
        db.query(Attachment).delete()
        db.query(RepairOrder).delete()
        db.query(Notice).delete()
        db.query(House).delete()
        db.query(User).delete()
        db.commit()

        # ── 用户：1 管理员 + 2 维修工 + 3 业主 ──
        # 说明：内存时代 house.owner_id 用 11/12/13 占位；落库后外键生效，
        # 占位编号必须换成真实用户——三位业主以 id 4/5/6 登场（显式指定 id，与房屋对应）
        db.add_all([
            User(id=1, username="admin",    password_hash="占位-第8章换哈希", role="admin",  created_at=dt("2026-10-01 09:00")),
            User(id=2, username="wang",     password_hash="占位-第8章换哈希", role="worker", phone="13800000002", created_at=dt("2026-10-01 09:00")),
            User(id=3, username="li",       password_hash="占位-第8章换哈希", role="worker", phone="13800000003", created_at=dt("2026-10-01 09:00")),
            User(id=4, username="zhangsan", password_hash="占位-第8章换哈希", role="owner",  phone="13900000001", created_at=dt("2026-10-01 09:00")),
            User(id=5, username="lisi",     password_hash="占位-第8章换哈希", role="owner",  phone="13900000002", created_at=dt("2026-10-01 09:00")),
            User(id=6, username="wangwu",   password_hash="占位-第8章换哈希", role="owner",  phone="13900000003", created_at=dt("2026-10-01 09:00")),
        ])

        # ── 房屋：3 套（owner_id 对应上面的业主 4/5/6）──
        db.add_all([
            House(id=1, owner_id=4, building="3栋", unit="2单元", room="501"),
            House(id=2, owner_id=5, building="3栋", unit="2单元", room="502"),
            House(id=3, owner_id=6, building="5栋", unit="1单元", room="1802"),
        ])

        # ── 工单：6 张，覆盖全部六种状态（与第 2 章内存版一致，第 7 章接口移植后即刻可用）──
        db.add_all([
            RepairOrder(id=1, house_id=1, worker_id=None, type="水电", description="厨房水管漏水，橱柜底下一直滴水",
                        status="待受理", score=None, created_at=dt("2026-10-22 09:15"), finished_at=None),
            RepairOrder(id=2, house_id=2, worker_id=2, type="电梯", description="1单元电梯上行时有异响",
                        status="已派单", score=None, created_at=dt("2026-10-21 16:40"), finished_at=None),
            RepairOrder(id=3, house_id=3, worker_id=3, type="门窗", description="阳台推拉门脱轨关不上",
                        status="维修中", score=None, created_at=dt("2026-10-20 10:05"), finished_at=None),
            RepairOrder(id=4, house_id=1, worker_id=2, type="水电", description="卫生间地漏堵塞下水慢",
                        status="已完成", score=None, created_at=dt("2026-10-18 14:20"), finished_at=dt("2026-10-19 11:00")),
            RepairOrder(id=5, house_id=2, worker_id=3, type="其他", description="楼道感应灯不亮",
                        status="已评价", score=5, created_at=dt("2026-10-15 19:30"), finished_at=dt("2026-10-16 09:00")),
            RepairOrder(id=6, house_id=3, worker_id=None, type="水电", description="重复提交的报修单",
                        status="已取消", score=None, created_at=dt("2026-10-14 08:00"), finished_at=None),
        ])

        # ── 附件：1 条（幽灵记录，文件不在磁盘——第 5 章的老梗，第 7 章继续用）──
        db.add(Attachment(id=1, order_id=5, filename="维修报告.pdf",
                          stored_path="uploads/docs/5f3ab2c4e7d9.pdf",
                          file_type="pdf", uploaded_at=dt("2026-10-16 09:30")))

        # ── 公告：5 条（与第 2 章内存版一致）──
        db.add_all([
            Notice(id=1, author_id=1, title="停水通知：10月25日3栋临时停水",
                   content="因市政管网检修，10月25日8:00-18:00期间3栋全天停水，请各户提前储水。",
                   category="公告", created_at=dt("2026-10-20 10:00")),
            Notice(id=2, author_id=1, title="电梯年检通知",
                   content="10月28日上午对全部电梯进行年检，期间电梯暂停使用，请走楼梯并注意安全。",
                   category="公告", created_at=dt("2026-10-19 15:30")),
            Notice(id=3, author_id=1, title="住宅装修管理规定",
                   content="装修施工时间为工作日8:00-12:00、14:00-18:00；周末及法定节假日禁止产生噪音的施工；装修垃圾须袋装并投放至指定点位。",
                   category="规章", created_at=dt("2026-10-01 09:00")),
            Notice(id=4, author_id=1, title="消防通道管理规定",
                   content="消防通道内禁止停放车辆、堆放杂物；违者物业将拍照留证并上报消防部门。",
                   category="规章", created_at=dt("2026-10-01 09:05")),
            Notice(id=5, author_id=1, title="高空抛物监控启用公告",
                   content="小区已启用高空抛物监控摄像头，覆盖全部楼栋立面，请广大业主共同监督。",
                   category="公告", created_at=dt("2026-09-28 17:00")),
        ])
        db.commit()
        print("种子数据就绪：6 用户 / 3 房屋 / 6 工单 / 1 附件 / 5 公告")
    finally:
        db.close()


if __name__ == "__main__":
    main()
```

运行 `python seed_data.py`，然后 `SELECT username, role FROM user;` 抽查——6 个用户就位。

> 📌 三点说明：①**先清后插**让脚本可无限重跑，开发期随时"数据复位"；②工单/附件数据也一并入库备好（表建了、数据齐了，第 7 章只差接口移植这一步）；③`password_hash` 现在存的是占位文本——**第 8 章讲密码哈希时正式替换**，这个列先建好就是给那一章留的座位。

### 回归测试：历史性的一刻

按用例表回归（重点用例）：

| # | 操作 | 预期 |
|---|------|------|
| 1 | `GET /notices?category=规章` | 2 条（seed 的 3、4 号） |
| 2 | `POST /notices` 发布一条新公告 | 201；**注意 created_at 变为 `"2026-10-23T17:20:31"` 样式**（声明一的升级效果） |
| 3 | `GET /houses/1` | 正常返回 501（HouseOut 安检通过） |
| 4 | `PUT /notices/1` 只改 title | 200，其他字段未动 |
| 5 | `DELETE /notices/2` | 204 |
| 6 | **重启服务**（Ctrl+C 再启动），再查 `GET /notices` | **数据全部还在**——包括刚才发布、修改、删除的结果 |

用例 6 请专门体会一下：五个章节的"土拨鼠之日"结束了。

### 常见坑

- ⚠️ **忘 commit 的"假成功"**：add 之后直接 return，响应一切正常，库里没数据，重启后才穿帮。铁律：**增删改必 commit**。
- ⚠️ **create 忘 refresh**：响应里 id 为 null、created_at 为 null（数据库生成的值还没回读到对象上）。
- ⚠️ **Out 模型忘加 from_attributes**：500 + 终端 ResponseValidationError——错误信息里能看到 "Input should be a valid dictionary or instance of NoticeOut"，其实是在提示"我不会从对象读值"。
- ⚠️ **同时保留内存与数据库双数据源**：某个接口忘了切、还在读 store.py——现象诡异（改了库不生效/重启行为不一致）。移植完成的资源，其内存数据集必须**立刻删除**（本节已删 HOUSES/NOTICES），让编译器替你抓漏网之鱼（NameError）。
- ⚠️ **filter 写成 `filter(House.id == house_id)` 漏引 House**：这是 SQLAlchemy 表达式不是普通比较，等号两边都要 ORM 列参与——抄错列名启动不报错、查询时报 AttributeError。

### 小结

- 移植套路四件套：加 `db: Session = Depends(get_db)` 形参 → 数据访问行换 query/filter → 增删改加 commit（create 后 refresh）→ Out 模型配 from_attributes。
- 校验、错误码、docstring 原样保留——升级纪律的"心脏移植"条款。
- Session 三铁律：查询不 commit、增删改必 commit、commit 前未入库。
- 种子脚本先清后插可重跑；内存数据集用完即删，杜绝双数据源。

---

## 5. 心脏移植（下）：全流程贯通与工程细节

### 完整数据流收官图

移植后的一个请求走完全程是这样（**图 6-3**）——请注意 Pydantic 与 ORM 两套模型在哪里接力：

```mermaid
flowchart TB
    A["客户端请求"] --> D["Depends(get_db)<br/>自动开会话"]
    D --> V{"进门安检<br/>NoticeCreate（Pydantic）"}
    V -->|合格| B["业务逻辑<br/>校验 404/400（原样）"]
    B --> O["ORM 操作<br/>query / add / commit（SQLAlchemy）"]
    O --> S["出门安检<br/>response_model=NoticeOut + from_attributes<br/>（从 ORM 对象读属性）"]
    S --> J["JSON 响应"]
    J --> C["请求结束<br/>get_db 自动关会话"]
```

**图 6-3 一次请求的完整数据流：Pydantic 管进出的单据，ORM 管账本的行**

### 数据库分页的必要性认知

第 2 章的分页是"全量拿进内存再切片"。数据落库后，这个思路升级为**数据库分页**：让 MySQL 只回传本页那几行——

```python
db.query(Notice).order_by(Notice.id).offset((page - 1) * page_size).limit(page_size).all()
# 对应 SQL：... ORDER BY id LIMIT 3 OFFSET 3
```

`offset` 跳过前若干行、`limit` 只取若干行——**切片逻辑下推给数据库**，全表百万行也只传一页的流量。公告列表不实际改造（与工单列表同理），工单列表第 7 章移植时正式上数据库分页。这里先把反模式钉在墙上：**落库之后还"全表拉进内存再切片"，等于买了数据库却让它当仓库用**。

### 工程细节三则

1. **连接信息不要硬编码进业务文件**：本章 DATABASE_URL 写在 db.py 顶部（密码明文）——开发期可接受；第 10 章 Settings + .env 配置管理正式收编（换密码、换环境不改代码）。这是刻意留到第 10 章的伏笔，不是疏忽。
2. **echo 开关**：调试时开着（亲眼看 SQL），日常关掉（终端不刷屏）。生产必关。
3. **连接失败三连查**（第一个接口就 500 时）：①MySQL 服务启动了吗（services.msc）；②密码对吗（URL 里的密码）；③库建了吗（`SHOW DATABASES;`）。90% 的"连不上"是这三个。

### 常见坑

- ⚠️ **seed 重复执行数据翻倍**：没有"先清后插"的脚本，跑两次数据 ×2。本课程 seed 已内置清库逻辑——自己写种子脚本时别省。
- ⚠️ **改了 models.py 就以为表变了**：create_all 只建缺的表，不改已有的表（第 3 节的坑，再钉一次——因为本章你一定会改模型玩）。
- ⚠️ **在 /docs 看到老数据**：浏览器缓存 + 服务没重启二选一，先 Ctrl+C 重启再看。

### 小结

- 数据流全景：Depends 开会话 → Pydantic 进门 → 业务校验 → ORM 读写 → Pydantic 出门 → 关会话。
- offset/limit 数据库分页替代内存切片；反模式：全表进内存。
- 连接信息硬编码是过渡形态，第 10 章 Settings 收编。

---

## 6. 本章收官：总表 v0.6.0 与双章蓝图

### 总表 v0.6.0

接口清单 14 行与 v0.5.0 **逐字相同**（复贴纪律），本章升级的是数据层——用表后标注记录过渡状态：

> 📌 **v0.6.0 数据源标注**：houses 模块（1 个接口）与 notices 模块（5 个接口）已切换至 **MySQL**；repairs 模块（5 个接口）与 files 模块（3 个接口）仍走内存数据——第 7 章全库化后本标注撤销。响应格式有一处已声明的变化：Out 模型时间字段 str → datetime（ISO 8601 格式）。

### 表 1-1 更新

「登记房屋信息」行：房屋表已落库、种子数据就位。登记接口属于低频管理操作，课程主线不单独实现（房屋档案由 seed 维护）——该行在功能矩阵中标注为「数据层就绪」。

### 遗留钩子（全部指向第 7 章）

1. 工单与附件接口还在读内存——第 7 章移植：ForeignKey 的深玩法（relationship）、连表查询（"我的报修记录"）、事务（派单+写 worker_id 的原子性）；
2. 「查看自己房屋的报修记录」「查看维修工绩效统计」两行功能等连表查询解锁；
3. WORKERS、REPAIR_ORDERS、ATTACHMENTS 三个内存数据集将在第 7 章末尾集体下岗——store.py 清空倒计时。

```mermaid
mindmap
  root((第 6 章<br/>SQLAlchemy 入门))
    连接
      utf8mb4 建库
      连接 URL 解剖
      create_engine 连接池
      SessionLocal 会话工厂
      Depends get_db 先睹为快
    声明式模型
      Mapped mapped_column
      String 必须给长度
      default datetime.now 动态默认
      UniqueConstraint 联合唯一
      ForeignKey 建列
      create_all 与 echo 核对
    两套模型
      ORM 管账本的行
      Pydantic 管进出的单据
      from_attributes 接力
    心脏移植
      只换数据访问行
      query filter first all
      add commit refresh
      delete commit
      setattr 循环更新
    Session 三铁律
      查询不 commit
      增删改必 commit
      commit 前未入库
    工程
      seed 先清后插
      数据库分页 offset limit
      内存数据集用完即删
      第 10 章 Settings 收编连接配置
```

**本章你为安居宝交付了什么**：5 张表落进 MySQL 5.6（与第 1 章设计逐字一致）、6 个接口数据库化、db.py/models.py/seed_data.py 三件套、store.py 瘦身一半——**"重启即失忆"时代终结**。

**下一章预告**：第 7 章《数据库进阶：关联查询与业务落地》——relationship 让工单"认识"自己的房屋和维修工，连表查询解锁"我的报修记录"与"维修工绩效统计"，事务保住状态流转的原子性；最后一个内存数据集下岗，安居宝全面数据库化。

---

## 附录

### 附录 A：SQLAlchemy 常用列类型与约束速查

| SQLAlchemy 写法 | MySQL 对应 | 用途 |
|-----------------|-----------|------|
| `Mapped[int] = mapped_column(primary_key=True, autoincrement=True)` | INT AUTO_INCREMENT PK | 主键 |
| `Mapped[str] = mapped_column(String(50))` | VARCHAR(50) | 定长上限字符串（**必须给长度**） |
| `Mapped[str] = mapped_column(Text)` | TEXT | 长文本（公告正文、规章） |
| `Mapped[datetime] = mapped_column()` | DATETIME | 时间 |
| `Mapped[int \| None] = mapped_column()` | INT NULL | 可空整数 |
| `unique=True` | UNIQUE | 唯一约束 |
| `nullable=False` | NOT NULL | 非空 |
| `default=值` / `default=函数` | DEFAULT / INSERT 时填充 | 默认值（动态默认传函数本身） |
| `ForeignKey("user.id")` | FOREIGN KEY | 外键列 |
| `UniqueConstraint("a", "b")` | 联合唯一 | 组合唯一 |

### 附录 B：query 方法速查

| 方法 | 生成/行为 | 例 |
|------|----------|-----|
| `db.query(Notice)` | SELECT * FROM notice | 查询起点 |
| `.filter(Notice.category == "公告")` | WHERE category = '公告' | 条件（==、!=、>、<） |
| `.filter_by(category="公告")` | 同上（关键字参数写法，适合等值） | 简写 |
| `Notice.title.contains(kw)` | LIKE '%kw%' | 模糊匹配 |
| `Notice.id.in_([1, 2])` | IN (1, 2) | 集合成员 |
| `A \| B`（条件间） | OR | 或 |
| `.first()` | 取第一条，无则 None | 详情接口标配 |
| `.all()` | 取全部为列表 | 列表接口标配 |
| `.count()` | COUNT(*) | 总数（分页 total） |
| `.order_by(Notice.id)` | ORDER BY id（加 `.desc()` 倒序） | 排序 |
| `.offset(n).limit(m)` | LIMIT m OFFSET n | 数据库分页 |
| `db.add(obj)` / `db.delete(obj)` / `db.commit()` | INSERT/DELETE 登记 + 结账 | 增删改 |

### 附录 C：逃生舱——什么时候该直接写 SQL

ORM 覆盖 95% 的日常，但总有它不顺手的时候：复杂统计报表、多表 UPDATE、数据库特有语法。SQLAlchemy 的逃生舱：

```python
from sqlalchemy import text

# 直接执行原生 SQL（参数用 :名字，同样防注入）
rows = db.execute(text("SELECT category, COUNT(*) AS cnt FROM notice GROUP BY category")).all()
for row in rows:
    print(row.category, row.cnt)     # 结果行支持按列名取值
```

**什么时候下舱**：ORM 写出来绕三层的时候（如分组统计）、需要 EXPLAIN 调优的时候、迁移旧 SQL 的时候。原则：**先用 ORM，不顺手再用 text()，且 SQL 集中放在数据层**（第 10 章分层后不许散落在路由里）。

另两条认知级线索：**改已存在的表结构**用 `ALTER TABLE`（开发期 DROP 重建也行）；正式的表结构版本管理工具叫 **Alembic**（SQLAlchemy 官方搭档，实习时大概率遇到，知道名字即可）。
