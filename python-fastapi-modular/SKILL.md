---
name: python-fastapi-modular
description: >-
  Guides scaffolding and daily coding of a Python FastAPI modular monolith:
  pyproject.toml, shared framework package, app shell, domain modules,
  three API channels, Router/Service/Repository layering, specific ErrorCode
  per failure, logging, and config governance. Use when creating a new Python
  backend, adding a module or API, reviewing architecture, or when the user
  mentions 单体架构、模块化单体、搭建框架、工程规范、代码规范、Python后端、FastAPI.
---

# Python 模块化单体规范（通用）

规定 **怎么搭框架、怎么写代码**，不绑定具体业务域。  
技术栈：Python 3.11+ · FastAPI · SQLAlchemy 2.x · Pydantic v2 · Alembic · uv（包管理）。

| 任务 | 先读 |
|------|------|
| 从零搭工程 / 迁框架 | [scaffolding.md](scaffolding.md) |
| 新模块 / 新 CRUD / 日常编码 | [coding.md](coding.md) |
| 日志、配置、开放对接 | [reference.md](reference.md) |
| 自检 / Review | [checklist.md](checklist.md) |
| 复制到其它仓库 | [adapt.md](adapt.md) |

---

## 1. 何时启用

- 新建 FastAPI 后端、拆模块、加路由、加开放通道
- 新管理端 / C 端 / 外部接口，或 Code Review 分层与错误码
- 把本规范迁到另一个工程
- 用户提到：单体架构、模块化单体、搭建框架、工程规范、Python后端、FastAPI

不用于：纯业务字段含义、前端 Vue、具体表设计讨论（除非同时改后端分层）。

---

## 2. 占位符

| 占位 | 含义 | 示例 |
|------|------|------|
| `{app}` | 工程前缀（包名 / 配置根） | `acme` |
| `{base}` | Python 根包路径 | `acme` |
| `{domain}` | 业务域 | `order` |
| `{resource}` | 资源（URL 与类名） | `store_order` / `StoreOrder` |
| `{admin-prefix}` | 管理端 URL 前缀 | `/admin-api` |
| `{app-prefix}` | C 端 URL 前缀 | `/app-api` |

复制到新工程时只改这些值，目录形态不变。

---

## 3. 架构决策（必须遵守）

**形态**：一个进程、一个可执行入口。`{app}/main.py` 聚合各 `module_*` 路由；横切能力进 `{app}/framework`。

```text
{app}/
├── main.py               启动入口：挂载各模块 router + middleware
├── framework/            横切平台（response、exception、db、security…）
└── module_{domain}/      业务域（可多个）
    ├── api/              契约：ErrorCode、enum、schema（无 ORM 依赖）
    └── biz/              实现：router、service、repository、model、convert
```

| 决策 | 意图 | 禁止 |
|------|------|------|
| 模块化单体 | 运维简单、本地事务 | 把模块当微服务乱加 HTTP 调用 |
| api / biz 分离 | 跨模块只依赖契约 schema | biz 直接 import 另一模块的 ORM model |
| framework 平台化 | 横切可复制 | 业务代码进 framework |
| 包路径驱动 URL prefix | 约定优于配置 | router 手写 `{admin-prefix}` |
| 配置跟域走 | main.py 保持干净 | main.py 堆业务初始化 |

**一句话**：main 是容器，framework 是平台，module 是插件。

---

## 4. 设计原则

| 原则 | 说明 |
|------|------|
| 契约与实现分离 | 跨模块只暴露 `api/`（ErrorCode、enum、Pydantic schema） |
| 单向依赖 | Router → Service → Repository |
| 框架与业务解耦 | 通用进 framework，业务进 `module_*` |
| 接口分通道 | 管理端 / C 端 / 外部对接，鉴权隔离 |
| 统一契约 | `CommonResult`、ErrorCode、RespSchema 命名一致 |
| 具体错误具体消息 | 每种失败独立 ErrorCode；禁止运行时拼 msg |
| 外部数据源容错 | 上游不可用时返回旧数据 + `stale_error` 标记，不让页面崩溃 |
| 缓存线程安全 | 所有共享缓存对象必须用 `threading.Lock` 保护读写 |
| 配置与代码分离 | 密钥、阈值、URL 全部走 `.env`；禁止硬编码业务节点数据 |

---

## 5. 按任务执行

**从零搭框架** → [scaffolding.md](scaffolding.md)

1. 定占位符与技术栈
2. 建 pyproject.toml → framework 最小集 → main 空壳
3. 第一个 `module_{domain}`（api + biz）跑通一条管理端 CRUD
4. 需要再加 redis / job / mq；开放通道按 7 步加

**新业务模块** → [scaffolding.md](scaffolding.md)「新模块」+ [coding.md](coding.md)

1. 新建 `module_{domain}/`（api + biz）
2. api：ErrorCode 段 + enum + schema
3. biz：repository → convert → service → router
4. **只改 main.py 注册 router**，不改其他模块代码

**新管理端 / C 端接口** → [coding.md](coding.md) → [checklist.md](checklist.md)

**新开放接口** → [reference.md](reference.md)「外部对接 7 步」

**Review / PR** → [checklist.md](checklist.md)

**迁到其它工程** → [adapt.md](adapt.md)

---

## 6. 工程与模块（摘要）

```text
{root}/
├── pyproject.toml
├── alembic/
├── {app}/
│   ├── main.py
│   ├── framework/
│   │   ├── common/           CommonResult、PageResult、ErrorCode、ServiceException
│   │   ├── web/              GlobalExceptionHandler、prefix 中间件、Swagger 配置
│   │   ├── db/               Base、BaseDO、get_session、MetaHandler
│   │   ├── security/         Token 验证、get_current_user、权限装饰器
│   │   └── config/           Settings（pydantic-settings）
│   └── module_{domain}/
│       ├── api/              error_codes.py、enums.py、schemas.py
│       └── biz/              router.py、service.py、repository.py、
│                             models.py、convert.py
└── tests/
```

**基线栈**：Python 3.11 · FastAPI 0.110+ · SQLAlchemy 2.x · Pydantic v2 · Alembic · uv · pytest

**api 只放**：ErrorCode、enums、跨模块 Pydantic schema（无 SQLAlchemy model）  
**biz 只放**：router、service、repository、ORM model、convert、域内 middleware

跨 **2+ 模块** 才抽共享 schema；同进程不必 HTTP 化。

---

## 7. 三通道（安全架构）

| 通道 | 文件位置 | URL prefix | 鉴权 |
|------|----------|-----------|------|
| 管理端 | `biz/router_admin.py` | `/{domain}/{resource}` | Bearer Token + 权限装饰器 |
| C 端 | `biz/router_app.py` | 模块约定 | 用户 Token |
| 外部 | `biz/router_{开放名}.py` | `/external/{系统}` | **独立 Middleware**，不走管理端 Bearer |

- main.py 按域前缀自动加 `{admin-prefix}`；router **只写资源路径**
- 管理端 REST：`POST /create` · `PUT /update` · `DELETE /delete` · `GET /get` · `GET /page` · `GET /export`
- 外部鉴权算法可不同（JWT / HMAC），**目录结构必须统一**

---

## 8. 分层（摘要）

| 层 | 做 | 禁止 |
|----|----|------|
| Router | 参数校验、权限、调 Service、`return success(data)` | 事务、Repository、吞异常 |
| Schema | CreateReq / UpdateReq / PageReq / Resp / Export | 业务字段乱塞 BaseSchema |
| Service | 规则、事务、编排、返回 RespSchema | 返回 ORM model、直接 raise HTTPException |
| Convert | Pydantic → ORM 或 ORM → Pydantic | 业务 if |
| Repository | SQLAlchemy CRUD、`if_present` 条件 | 调 Service |

---

## 9. 统一响应与错误码（强制）

```json
{ "code": 0, "data": {}, "msg": "" }
```

- 成功：`code === 0`；HTTP 200，以 body 为准
- ErrorCode 定义在 `module_{domain}/api/error_codes.py`；Service **只** `raise ServiceException(ERROR_CODE)`
- 禁止直接 `raise HTTPException` 表达业务错误；禁止运行时拼 msg
- **一场景一常量**；仅签名 / token 不匹配允许笼统「签名验证失败」

完整条文见 [coding.md](coding.md) §错误码、[reference.md](reference.md) §错误码。

---

## 10. 配置与日志（摘要）

| 层级 | 职责 |
|------|------|
| `main.py` | 只 `create_app()` + 注册 router / middleware |
| 域内 `config.py` | 本域 Settings（pydantic-settings）+ 依赖注入 |
| `Settings` | 绑定环境变量，无业务逻辑 |

日志五层（应用 / 访问 DB / 错误 DB / 操作 DB / 集成审计）见 [reference.md](reference.md)。  
Service 禁止空 except；禁止 log 明文 password / secret / sign / token。

---

## 11. 推荐默认

| 项 | 默认 |
|----|------|
| 软删除 | `deleted_at` 字段；历史业务删除字段触达再统一 |
| 跨模块调用 | 2+ 模块才抽共享 schema |
| 开放接口 | 每个 `/external/{系统}` 必有 Middleware + 审计表 |
| 生产访问日志 | 管理端 enable |
| HTTP 失败 | 对内 200 + body.code；网关可映射 4xx |
| 旧代码 | 触达再改，不一次性重构 |
| 外部数据源 | 上游失败返回旧数据 + `stale_error`，不 crash |
| 缓存对象 | 共享缓存必须配 `threading.Lock`，读写都在 `with lock:` 内 |
| 文件写入 | 写本地文件必须 `tempfile + os.replace()` 原子替换 |
| 启动预热 | 重型查询在应用启动时后台线程预热，第一个用户不等待 |

---

## 12. 反例

| ❌ | ✅ |
|----|-----|
| Router → Repository | Router → Service → Repository |
| main.py 堆业务初始化 | 域内 config.py |
| external 复用 admin Token | 独立 Middleware |
| 以为 external 会进访问日志 DB | 仅管理端前缀；外部走审计表 |
| 空 except / log 打 secret | warn + 脱敏 |
| router 写 `{admin-prefix}/...` | 只写 `/{domain}/{resource}` |
| `raise HTTPException` 表达业务 | api 定义 ErrorCode + `raise ServiceException(...)` |
| 多种失败共用一句笼统 msg | 每场景独立 ErrorCode（安全敏感除外） |
| 业务类放进 framework | 业务进 `module_*` |
| 所有路由堆在单个文件（>500行） | 按域拆分 `router_admin.py`，main.py 只做注册 |
| 节点/站点配置硬编码进源码 | 走 `.env` 或配置文件，代码不改能换环境 |
| 直接写文件（写到一半崩溃→损坏） | `tempfile.mkstemp()` + `os.replace()` 原子替换 |
| 多线程共享缓存无锁直接读写 | `threading.Lock` 保护，读写都在 `with lock:` 内 |
| 上游 API 失败直接 500 / 抛异常 | 返回旧缓存数据 + `stale_error` 标记，页面降级展示 |
| 应用启动后第一个用户触发重型查询 | 启动时后台线程 `prewarm()`，缓存提前就绪 |
| `hmac` 比较密码用 `==` | `hmac.compare_digest()` 防时序攻击 |

---

## 13. 交付前

走 [checklist.md](checklist.md) 对应清单（新工程 / 新模块 / 新接口 / 开放接口 / PR）。

规则文件模板（复制到目标仓库 `.cursor/rules/`）：[templates/python-backend-standards.mdc](templates/python-backend-standards.mdc)
