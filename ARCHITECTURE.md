# Django-DDD 架构说明图

> **Django 5.0 + Django Ninja + Pydantic 2.x | 领域驱动设计（DDD）分层架构实践**

---

## 一、全局架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              HTTP 请求进入                                  │
│                          POST /api/todos/                                   │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Presentation 层（表现层）                                                   │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌────────────┐               │
│  │  api.py  │  │request.py │  │response.py│  │containers.py│              │
│  │ (路由)   │  │(请求格式) │  │(响应格式) │  │ (依赖组装) │               │
│  └────┬─────┘  └───────────┘  └───────────┘  └─────┬──────┘               │
│       │                                             │                      │
│       │         从 containers 取出 command/query 实例  │                      │
│       ◄─────────────────────────────────────────────┘                      │
└───────┬─────────────────────────────────────────────────────────────────────┘
        │
        │  调用 command.create_todo() 或 query.get_todos()
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Application 层（应用层）                                                    │
│  ┌─────────────────┐  ┌─────────────────┐                                  │
│  │   Command 类     │  │    Query 类      │                                  │
│  │ (写操作：增删改)  │  │ (读操作：查询)    │                                  │
│  └────────┬────────┘  └────────┬────────┘                                  │
│           │                    │                                            │
│           │  调用 Entity 的业务方法 + Repository 的存取方法                     │
│           ▼                    ▼                                            │
└───────────┬────────────────────┬────────────────────────────────────────────┘
            │                    │
   ┌────────▼────────┐  ┌───────▼──────────┐
   │                 │  │                  │
   ▼                 ▼  ▼                  ▼
┌──────────────────┐  ┌────────────────────────────────────────────────────────┐
│  Domain 层       │  │  Infra 层（基础设施层）                                  │
│  (领域层)        │  │  ┌──────────┐  ┌────────────┐  ┌──────────────────┐   │
│  ┌────────────┐  │  │  │models.py │  │ mapper.py  │  │    rdb.py        │   │
│  │ entity.py  │  │  │  │(DB 表)   │  │(格式转换)  │  │  (数据库操作)    │   │
│  │ (业务实体) │  │  │  └──────────┘  └────────────┘  └──────────────────┘   │
│  ├────────────┤  │  │                                                       │
│  │exception.py│  │  │  Entity ←──mapper──→ Django Model ←──ORM──→ Database  │
│  │ (业务异常) │  │  │                                                       │
│  └────────────┘  │  └────────────────────────────────────────────────────────┘
└──────────────────┘
```

### 依赖方向（核心原则）

```
Presentation  →  Application  →  Domain  ←  Infra
   (外层)          (编排)        (核心)      (外层)

✅ 所有层都依赖 Domain 层
✅ Domain 层不依赖任何其他层（纯 Python，零框架）
✅ Infra 层的 Repository 输入输出都是 Domain Entity
```

---

## 二、模块拆解

项目有 3 个模块，每个模块内部结构一致：

```
src/
├── shared/          ← 🔧 共享内核（基础设施 + 公共抽象）
├── todo/            ← 📝 待办事项（业务模块）
├── user/            ← 👤 用户认证（业务模块）
└── tests/           ← 🧪 测试
```

---

## 三、shared 模块 — 共享内核

> 提供所有模块共用的基类、工具和全局配置，**不含具体业务逻辑**。

```
shared/
├── domain/                          ← 基础领域抽象
│   ├── entity.py                    ← Entity 基类（id + __eq__ + __hash__）
│   └── exception.py                 ← 异常基类 + JWT/认证异常
│
├── infra/                           ← 基础设施
│   ├── authentication.py            ← 🔐 认证服务（bcrypt + JWT）
│   ├── django/
│   │   └── settings.py              ← ⚙️ Django 配置
│   └── repository/
│       ├── mapper.py                ← 📐 Mapper 接口（Entity ↔ Model 转换规范）
│       └── rdb.py                   ← 💾 Repository 基类（通用 save 方法）
│
└── presentation/                    ← 全局 API 入口
    └── rest/
        ├── api.py                   ← 🌐 NinjaAPI 根实例 + 路由注册 + urlpatterns
        ├── containers.py            ← 📦 全局依赖容器（auth_service 单例）
        └── response.py              ← 📤 统一响应封装（response / error_response）
```

### 详细结构

#### Entity 基类

```python
# shared/domain/entity.py
@dataclass
class Entity:
    id: int | None = None          # 所有实体共有的标识符

    def __eq__(self, other):       # 基于 id 判断"是不是同一个东西"
        return self.id == other.id

    def __hash__(self):            # 支持放入 set / dict
        return hash(self.id)
```

#### Mapper 接口

```python
# shared/infra/repository/mapper.py
class ModelMapperInterface:
    def to_entity(instance) → Entity        # 数据库模型 → 业务实体
    def to_instance(entity) → Model         # 业务实体 → 数据库模型
    def to_entity_list(instances) → [Entity] # 批量转换
    def to_instance_list(entities) → [Model] # 批量转换
```

#### Repository 基类

```python
# shared/infra/repository/rdb.py
class RDBRepository:
    model_mapper: ModelMapperInterface

    def save(entity):
        instance = self.model_mapper.to_instance(entity)  # Entity → Model
        instance.save()                                    # 写入数据库
        return self.model_mapper.to_entity(instance)       # Model → Entity（带上 DB 生成的 id）
```

#### 认证服务

```python
# shared/infra/authentication.py
class AuthenticationService:
    SECRET_KEY = "..."
    ALGORITHM = "HS256"

    def hash_password(plain_password) → str          # bcrypt 加密
    def verify_password(plain, hashed) → bool        # bcrypt 验证
    def create_jwt(user) → str                       # 生成 JWT token
    def get_user_id_from_token(token) → int          # 从 token 解析 user_id
```

#### 统一响应格式

```python
# shared/presentation/rest/response.py
def response(results):        →  {"results": ...}
def error_response(msg):      →  {"results": {"message": "..."}}

class ObjectResponse(Generic[T]):   # 泛型响应，自动生成 OpenAPI Schema
    results: T
```

---

## 四、todo 模块 — 待办事项

> 完整的 CRUD 业务模块，展示 DDD 分层的标准写法。

```
todo/
├── domain/                          ← 领域层
│   ├── entity.py                    ← ToDo 实体（contents, due_datetime, user）
│   └── exception.py                 ← ToDoNotFoundException
│
├── application/                     ← 应用层
│   └── use_case/
│       ├── command.py               ← 写操作：create / update / delete
│       └── query.py                 ← 读操作：get_todo / get_todos
│
├── infra/                           ← 基础设施层
│   ├── database/
│   │   ├── models.py                ← Django ORM Model（ToDo 表）
│   │   ├── migrations/              ← 数据库迁移文件
│   │   └── repository/
│   │       ├── mapper.py            ← ToDoMapper（含嵌套 UserMapper）
│   │       └── rdb.py               ← ToDoRDBRepository
│   └── django/
│       ├── admin.py                 ← Django Admin 注册
│       └── apps.py                  ← AppConfig（label="todo"）
│
└── presentation/                    ← 表现层
    └── rest/
        ├── api.py                   ← 5 个 REST 端点（全部需要 JWT）
        ├── containers.py            ← 依赖组装（repo → query/command）
        ├── request.py               ← PostToDoRequestBody / PatchToDoRequestBody
        └── response.py              ← ToDoSchema / ToDoResponse / ListToDoResponse
```

### 详细结构

#### Domain — 实体

```python
# todo/domain/entity.py
@dataclass
class ToDo(Entity):
    contents: str                     # 待办内容
    due_datetime: datetime | None     # 截止日期（可选）
    user: User                        # 所属用户（关联实体）

    @classmethod
    def new(cls, user, contents, due_datetime):  # 工厂方法
        return cls(contents=contents, due_datetime=due_datetime, user=user)

    def update_contents(self, contents):         # 业务行为
        self.contents = contents

    def update_due_datetime(self, due_datetime):
        self.due_datetime = due_datetime
```

#### Application — 用例编排

```python
# todo/application/use_case/command.py
class ToDoCommand:
    def __init__(self, todo_repo: ToDoRDBRepository): ...

    def create_todo(user, contents, due_datetime) → ToDo:
        todo = ToDo.new(user, contents, due_datetime)    # 1. 创建实体
        return self.todo_repo.save(todo)                  # 2. 持久化

    def update_todo(todo, contents, due_datetime) → ToDo:
        if contents: todo.update_contents(contents)       # 1. 调用实体方法
        if due_datetime: todo.update_due_datetime(...)    # 2. 修改状态
        return self.todo_repo.save(todo)                  # 3. 持久化

    def delete_todo_of_user(user_id, todo_id):
        self.todo_repo.delete_todo_of_user(user_id, todo_id)
```

```python
# todo/application/use_case/query.py
class ToDoQuery:
    def __init__(self, todo_repo: ToDoRDBRepository): ...

    def get_todo_of_user(user_id, todo_id) → ToDo:
        return self.todo_repo.get_todo_of_user(user_id, todo_id)

    def get_todos_of_user(user_id) → list[ToDo]:
        return self.todo_repo.get_todos_of_user(user_id)
```

#### Infra — 数据库

```python
# todo/infra/database/models.py（Django ORM 模型，和 Entity 完全独立）
class ToDo(models.Model):
    contents = CharField(max_length=200)
    due_datetime = DateTimeField(null=True)
    user = ForeignKey(User, CASCADE, related_name="todos")
    class Meta:
        db_table = "todo"
```

```python
# todo/infra/database/repository/mapper.py（双向转换）
class ToDoMapper:
    user_mapper = UserMapper()

    def to_entity(db_todo) → ToDoEntity:
        return ToDoEntity(
            id=db_todo.id,
            contents=db_todo.contents,
            due_datetime=db_todo.due_datetime,
            user=self.user_mapper.to_entity(db_todo.user)  # 嵌套转换！
        )

    def to_instance(entity) → ToDoModel:
        return ToDoModel(
            id=entity.id,
            contents=entity.contents,
            due_datetime=entity.due_datetime,
            user=self.user_mapper.to_instance(entity.user)  # 嵌套转换！
        )
```

```python
# todo/infra/database/repository/rdb.py（继承基类，扩展查询）
class ToDoRDBRepository(RDBRepository):
    model_mapper = ToDoMapper()

    def save(entity) → ToDo          # ← 继承自 RDBRepository
    def get_todo_of_user(user_id, todo_id) → ToDo     # 查单条
    def get_todos_of_user(user_id) → list[ToDo]        # 查列表
    def delete_todo_of_user(user_id, todo_id) → None   # 删除
```

#### Presentation — API

```python
# todo/presentation/rest/containers.py（依赖组装）
todo_repo    = ToDoRDBRepository()           # 仓库实例
todo_query   = ToDoQuery(todo_repo)          # 注入到 Query
todo_command = ToDoCommand(todo_repo)        # 注入到 Command
```

```python
# todo/presentation/rest/api.py（5 个端点，全需 JWT）
router = Router(tags=["todos"], auth=auth_bearer)

GET    /{todo_id}  → get_todo_handler      # 获取单条
GET    /           → get_todo_list_handler  # 获取列表
POST   /           → post_todos_handler     # 创建
PATCH  /{todo_id}  → patch_todos_handler    # 更新
DELETE /{todo_id}  → delete_todos_handler   # 删除
```

#### 请求 & 响应 Schema

```python
# todo/presentation/rest/request.py
class PostToDoRequestBody:           # 创建请求
    contents: str
    due_datetime: datetime | None

class PatchToDoRequestBody:          # 更新请求
    contents: str | None
    due_datetime: datetime | None
```

```python
# todo/presentation/rest/response.py
class ToDoSchema:                    # 基础数据
    id: int
    contents: str
    due_datetime: datetime | None

class ToDoResponse:                  # 单条响应
    todo: ToDoSchema

class ListToDoResponse:              # 列表响应
    todos: list[ToDoSchema]
```

---

## 五、user 模块 — 用户认证

> 注册、登录、删除账户，提供 JWT Token 给 todo 模块使用。

```
user/
├── domain/                          ← 领域层
│   ├── entity.py                    ← User 实体（email, password）
│   └── exception.py                 ← UserNotFoundException
│
├── application/                     ← 应用层
│   ├── command.py                   ← 写操作：sign_up / log_in / delete
│   └── query.py                     ← 读操作：get_user
│
├── infra/                           ← 基础设施层
│   ├── database/
│   │   ├── models.py                ← Django ORM Model（User 表）
│   │   ├── migrations/
│   │   └── repository/
│   │       ├── mapper.py            ← UserMapper
│   │       └── rdb.py               ← UserRDBRepository
│   └── django/
│       ├── admin.py
│       └── apps.py                  ← AppConfig（label="user"）
│
└── presentation/                    ← 表现层
    └── rest/
        ├── api.py                   ← 3 个 REST 端点（部分需要 JWT）
        ├── containers.py            ← 依赖组装（auth_service + repo → command/query）
        ├── request.py               ← PostUserCredentialsRequestBody
        └── response.py              ← UserSchema / UserResponse / TokenResponse
```

### 详细结构

#### Domain — 实体

```python
# user/domain/entity.py
@dataclass
class User(Entity):
    email: str            # 邮箱（唯一标识）
    password: str         # 密码（存储的是 bcrypt 哈希值）

    @classmethod
    def new(cls, email, hashed_password):
        return cls(email=email, password=hashed_password)
```

#### Application — 用例编排

```python
# user/application/command.py
class UserCommand:
    def __init__(self, auth_service, user_repo): ...

    def sign_up_user(email, plain_password) → User:
        hashed = self.auth_service.hash_password(plain_password)  # 1. 密码加密
        user = User.new(email, hashed)                             # 2. 创建实体
        return self.user_repo.save(user)                           # 3. 持久化

    def log_in_user(email, plain_password) → str:
        user = self.user_repo.get_user_by_email(email)            # 1. 查用户
        if not auth_service.verify_password(plain, user.password): # 2. 验证密码
            raise NotAuthorizedException()
        return self.auth_service.create_jwt(user)                  # 3. 生成 JWT

    def delete_user_by_id(user_id):
        self.user_repo.delete_user_by_id(user_id)
```

#### Infra — 数据库

```python
# user/infra/database/models.py
class User(models.Model):
    email = EmailField(unique=True)
    password = CharField(max_length=200)
    class Meta:
        app_label = "user"
        db_table = "user"
```

```python
# user/infra/database/repository/rdb.py
class UserRDBRepository(RDBRepository):
    model_mapper = UserMapper()

    def save(entity) → User              # ← 继承自 RDBRepository
    def get_user_by_id(user_id) → User   # 按 ID 查
    def get_user_by_email(email) → User  # 按 Email 查
    def delete_user_by_id(user_id)       # 删除
```

#### Presentation — API

```python
# user/presentation/rest/containers.py
user_repo    = UserRDBRepository()
user_query   = UserQuery(user_repo)
user_command = UserCommand(auth_service, user_repo)  # 注入认证服务！
```

```python
# user/presentation/rest/api.py
router = Router(tags=["users"])

POST   /           → sign_up_user_handler     # 注册（无需认证）
POST   /log-in     → log_in_user_handler      # 登录（无需认证）
DELETE /me          → delete_user_me_handler   # 删除自己（需要 JWT）
```

---

## 六、依赖组装全图

```
                    ┌────────────────────────────────┐
                    │  shared/presentation/rest/      │
                    │  containers.py                  │
                    │                                 │
                    │  auth_service = AuthService()   │ ─── 全局单例
                    └──────────┬──────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                                 ▼
┌─────────────────────────────┐  ┌─────────────────────────────────┐
│ user/presentation/rest/     │  │ todo/presentation/rest/         │
│ containers.py               │  │ containers.py                   │
│                             │  │                                 │
│ user_repo = UserRDBRepo()   │  │ todo_repo = ToDoRDBRepo()       │
│ user_query = UserQuery(     │  │ todo_query = ToDoQuery(         │
│     user_repo               │  │     todo_repo                   │
│ )                           │  │ )                               │
│ user_command = UserCommand( │  │ todo_command = ToDoCommand(     │
│     auth_service,           │  │     todo_repo                   │
│     user_repo               │  │ )                               │
│ )                           │  │                                 │
└─────────────────────────────┘  └─────────────────────────────────┘
```

---

## 七、API 接口全览

| # | 方法 | 完整路径 | 认证 | 模块 | 功能 | 成功状态码 |
|---|------|---------|------|------|------|-----------|
| 1 | `GET` | `/api/health-check/` | ❌ | shared | 健康检查 | 200 |
| 2 | `POST` | `/api/users/` | ❌ | user | 用户注册 | 201 |
| 3 | `POST` | `/api/users/log-in` | ❌ | user | 用户登录 | 200 |
| 4 | `DELETE` | `/api/users/me` | ✅ JWT | user | 删除当前用户 | 204 |
| 5 | `GET` | `/api/todos/{todo_id}` | ✅ JWT | todo | 获取单条 ToDo | 200 |
| 6 | `GET` | `/api/todos/` | ✅ JWT | todo | 获取用户所有 ToDo | 200 |
| 7 | `POST` | `/api/todos/` | ✅ JWT | todo | 创建 ToDo | 201 |
| 8 | `PATCH` | `/api/todos/{todo_id}` | ✅ JWT | todo | 更新 ToDo | 200 |
| 9 | `DELETE` | `/api/todos/{todo_id}` | ✅ JWT | todo | 删除 ToDo | 204 |

---

## 八、数据流图示例 — "创建一条 ToDo"

```
客户端                                          服务端
──────                                          ──────

POST /api/todos/
Headers: Authorization: Bearer <jwt_token>
Body: {"contents": "学习 DDD", "due_datetime": "2026-05-01T10:00:00"}

    │
    ▼
┌─ Presentation 层 ──────────────────────────────────────────────────────┐
│                                                                       │
│  1. auth_bearer 拦截 → 验证 JWT → 提取 user_id                         │
│  2. api.py 接收请求 → 解析 PostToDoRequestBody                         │
│  3. 调用 containers 中的 todo_command                                   │
│                                                                       │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
                                ▼
┌─ Application 层 ──────────────────────────────────────────────────────┐
│                                                                       │
│  4. todo_command.create_todo(user, contents, due_datetime)             │
│     ├─ user_query.get_user(user_id)  → 获取 User Entity              │
│     ├─ ToDo.new(user, contents, due_datetime) → 创建 ToDo Entity     │
│     └─ todo_repo.save(todo) → 持久化                                  │
│                                                                       │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
┌─ Domain 层 ─────────────┐  ┌─ Infra 层 ─────────────────────────────┐
│                          │  │                                        │
│  ToDo Entity:            │  │  5. mapper.to_instance(todo_entity)    │
│  - id: None              │  │     → ToDoModel(contents=..., ...)     │
│  - contents: "学习 DDD"  │  │                                        │
│  - due_datetime: 5/1     │  │  6. todo_model.save()                  │
│  - user: User(id=1)     │  │     → INSERT INTO todo ...              │
│                          │  │     → id = 42（数据库生成）             │
│  纯 Python dataclass     │  │                                        │
│  零框架依赖              │  │  7. mapper.to_entity(saved_model)      │
│                          │  │     → ToDo(id=42, contents=..., ...)   │
└──────────────────────────┘  └────────────────────────────────────────┘
                                │
                                ▼
┌─ 回到 Presentation 层 ────────────────────────────────────────────────┐
│                                                                       │
│  8. ToDoResponse.build(todo) → {"todo": {"id": 42, ...}}             │
│  9. response(result) → {"results": {"todo": {...}}}                  │
│ 10. 返回 HTTP 201                                                    │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

    │
    ▼

HTTP 201 Created
{"results": {"todo": {"id": 42, "contents": "学习 DDD", "due_datetime": "2026-05-01T10:00:00"}}}
```

---

## 九、通俗理解 — 快递站类比

把整个系统想象成一个 **快递站**：

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  📦 Presentation 层 = 前台窗口                                │
│     负责接待客户、检查身份证（JWT）、填写单据（Request Schema）│
│     把结果装信封交给客户（Response Schema）                    │
│                                                              │
│  📋 Application 层 = 调度员                                   │
│     接到前台的单子，安排流程：                                 │
│     "先查有没有这个人 → 创建包裹 → 存到仓库"                  │
│     调度员自己不搬东西，只指挥                                 │
│                                                              │
│  📦 Domain 层 = 包裹本身                                      │
│     包裹有自己的属性（内容、重量、收件人）                     │
│     包裹有自己的规则（"超重不能寄"、"内容不能为空"）           │
│     包裹不关心自己被存在哪里                                   │
│                                                              │
│  🏭 Infra 层 = 仓库 + 搬运工                                 │
│     - Mapper = 搬运工（把包裹从"客户格式"转成"仓库格式"）     │
│     - Repository = 仓库管理员（存取包裹）                     │
│     - Django Model = 货架上的格子（数据库表的结构）            │
│                                                              │
└──────────────────────────────────────────────────────────────┘

一次"寄包裹"的流程：

  客户来到前台 → 出示身份证（JWT）
        ↓
  前台检查身份 → 转交调度员
        ↓
  调度员说："创建一个新包裹，存到仓库"
        ↓
  Domain：创建包裹对象（检查规则：内容不能为空 ✓）
        ↓
  搬运工（Mapper）：把包裹从"业务格式"转成"仓库格式"
        ↓
  仓库管理员（Repository）：放到货架上（写数据库）
        ↓
  搬运工再把"仓库格式"转回"业务格式"（带上自动编号）
        ↓
  调度员把结果交给前台 → 前台装信封（Response）→ 交给客户
```

---

## 十、测试体系

```
tests/
├── conftest.py              ← 自定义 APIClient + session 级 fixture
├── functional/              ← 🎭 功能测试（Mock 应用层，不碰数据库）
│   ├── test_health_check_api.py   GET /api/health-check/
│   ├── test_todo_api.py           5 个 Todo API 端点
│   └── test_user_api.py           3 个 User API 端点
└── integration/             ← 🔗 集成测试（真实数据库，验证 Repository）
    ├── test_todo.py               Todo 的 CRUD + 映射
    └── test_user.py               User 的 CRUD + 映射
```

| 测试类型 | 范围 | Mock 什么 | 验证什么 |
|---------|------|-----------|---------|
| **Functional** | HTTP → Presentation → (Mock) | Application 层 | 路由正确、状态码正确、响应结构正确 |
| **Integration** | Repository → ORM → DB | 无 | Entity ↔ Model 映射正确、CRUD 操作正确 |
