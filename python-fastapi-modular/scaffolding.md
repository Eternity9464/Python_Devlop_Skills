# 搭建框架

从零建模块化单体，或从参考仓库裁剪复制。占位符见 [SKILL.md](SKILL.md) §2。

---

## 决策：复制 vs 自建

| 情况 | 做法 |
|------|------|
| 团队已有本规范的参考仓库（含 `{app}/framework`） | **复制 framework + main 空壳**，改占位符 |
| 全新团队、无参考实现 | 按下方「最小集」自建，先跑通一条 CRUD，再补 middleware |
| 已有单体但无分层 | 先落地文档 + Rule + 新功能按规范；旧代码触达再改 |

**不要**把业务模块（订单、支付……）一并复制到新域工程。

---

## 阶段 0：占位与工程初始化

```bash
uv init {app}
cd {app}
uv add fastapi uvicorn[standard] sqlalchemy alembic pydantic-settings python-jose passlib
uv add --dev pytest httpx pytest-asyncio ruff mypy
```

确定并写进项目绑定（见 [adapt.md](adapt.md)）：

- `{app}` `{base}` `{admin-prefix}` `{app-prefix}`
- Python 3.11、FastAPI 0.110+、SQLAlchemy 2.x

```text
{root}/
├── pyproject.toml
├── alembic/
│   ├── env.py
│   └── versions/
├── {app}/
│   ├── main.py
│   ├── framework/
│   └── module_{domain}/    # 第一个业务域，阶段 2 再建
└── tests/
```

`main.py` 只做组装：

```python
from fastapi import FastAPI
from {base}.framework.web.exception_handler import register_exception_handlers
from {base}.framework.web.middleware import register_middlewares

def create_app() -> FastAPI:
    app = FastAPI(title="{app}", docs_url="/swagger-ui.html")
    register_exception_handlers(app)
    register_middlewares(app)
    # 在此 include_router，不在此写业务逻辑
    return app

app = create_app()
```

禁止在 `main.py` 堆业务初始化逻辑。

---

## 阶段 1：framework 最小集（必须先有）

没有这 3 块，后面的编码规范无法落地。

### 1.1 `framework/common/`

```python
# result.py
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class CommonResult(BaseModel, Generic[T]):
    code: int = 0
    data: T | None = None
    msg: str = ""

    @classmethod
    def success(cls, data: T = None) -> "CommonResult[T]":
        return cls(code=0, data=data, msg="")

    @classmethod
    def error(cls, code: int, msg: str) -> "CommonResult":
        return cls(code=code, data=None, msg=msg)

# page.py
class PageParam(BaseModel):
    page_no: int = 1
    page_size: int = 10

class PageResult(BaseModel, Generic[T]):
    list: list[T]
    total: int

# error_code.py
from dataclasses import dataclass

@dataclass(frozen=True)
class ErrorCode:
    code: int
    msg: str

GLOBAL_ERROR_CODES = {
    0: "成功",
    400: "请求参数错误",
    401: "未登录",
    403: "无权限",
    500: "系统内部错误",
}

# exception.py
class ServiceException(Exception):
    def __init__(self, error_code: ErrorCode, *params):
        self.code = error_code.code
        self.msg = error_code.msg.format(*params) if params else error_code.msg

def raise_exception(error_code: ErrorCode, *params) -> None:
    raise ServiceException(error_code, *params)

def raise_invalid_param(msg: str) -> None:
    raise ServiceException(ErrorCode(400, msg))
```

### 1.2 `framework/web/`

```python
# exception_handler.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from {base}.framework.common.exception import ServiceException
from {base}.framework.common.result import CommonResult

def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(ServiceException)
    async def service_exception_handler(request: Request, exc: ServiceException):
        return JSONResponse(
            status_code=200,
            content=CommonResult.error(exc.code, exc.msg).model_dump(),
        )

    @app.exception_handler(Exception)
    async def global_exception_handler(request: Request, exc: Exception):
        # log here
        return JSONResponse(
            status_code=200,
            content=CommonResult.error(500, "系统内部错误").model_dump(),
        )

# middleware.py  —— 按包路径自动加前缀（在 main 注册 router 时传 prefix 参数）
# 推荐做法：include_router 时显式传 prefix，而非运行时动态改写
```

### 1.3 `framework/db/`

```python
# base.py
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class BaseDO(Base):
    __abstract__ = True
    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    creator: Mapped[str | None] = mapped_column(default=None)
    updater: Mapped[str | None] = mapped_column(default=None)
    create_time: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    update_time: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), onupdate=func.now()
    )
    deleted: Mapped[bool] = mapped_column(default=False)

# session.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from {base}.framework.config.settings import get_settings

engine = create_async_engine(get_settings().database_url)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session():
    async with AsyncSessionLocal() as session:
        yield session
```

### 1.4 `framework/config/`

```python
# settings.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="{app}_")

    database_url: str
    secret_key: str
    admin_api_prefix: str = "/admin-api"
    app_api_prefix: str = "/app-api"
    access_log_enable: bool = True

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`.env.example`（密钥不进 Git）：

```ini
{APP}_DATABASE_URL=postgresql+asyncpg://user:pass@localhost/{app}
{APP}_SECRET_KEY=change-me
```

---

## 阶段 2：第一个业务模块 + 一条 CRUD

```text
{app}/module_{domain}/
├── api/
│   ├── error_codes.py     ErrorCode 号段
│   ├── enums.py
│   └── schemas.py         跨模块 Pydantic schema
└── biz/
    ├── models.py           ORM model
    ├── repository.py       数据库 CRUD
    ├── convert.py          ORM ↔ Schema 转换
    ├── service.py          业务逻辑
    └── router_admin.py     管理端路由
```

顺序（强制）：

1. api：ErrorCode 段（按域分段，如 `1008007xxx`）+ enum + schema
2. models：ORM model extends `BaseDO`
3. repository：CRUD + `if_present` 条件
4. convert：ORM ↔ Schema
5. service：接口 + 实现
6. router：管理端通道
7. `main.py` 注册 router（`prefix=f"{admin_prefix}/{domain}"`）
8. 启动，用 Swagger 验证 `POST {admin-prefix}/{domain}/{resource}/create`，body `code === 0`

编码模板见 [coding.md](coding.md)。自检见 [checklist.md](checklist.md)。

---

## 阶段 3：按需加能力（不要一次上齐）

| 何时 | 加什么 |
|------|--------|
| 要登录、权限、操作日志 | `framework/security/`（JWT 验证、`get_current_user`、权限装饰器） |
| 要缓存 | `framework/redis/`（redis-py / aioredis） |
| 要定时任务 | APScheduler 或 Celery Beat |
| 要 MQ | Celery + RabbitMQ/Redis |
| 要导出 | openpyxl / pandas |
| 要限流 | slowapi 或自定义 Middleware |
| 要链路 trace | opentelemetry-sdk |

新增能力只放横切逻辑；业务模块按需 import，**禁止**业务 model 进 framework。

---

## 阶段 4：开放通道（有外部系统再做）

按 [reference.md](reference.md)「外部对接 7 步」。最低要求：

- router **不在** `router_admin.py`，单独 `router_{开放名}.py`
- 独立 Middleware / Depends + 域内 `config.py` + `Settings`
- `main.py` 注册时 prefix 为 `/external/{系统}`
- 域内审计表（幂等 UNIQUE）+ `docs/` 联调文档

---

## 新模块（已有工程）

已有 framework / main 时，只做：

1. 新建 `module_{domain}/`（api + biz）
2. `api/schemas.py` 只用 Pydantic，不 import 其他模块的 ORM model
3. **跨模块**只 import 对方 `api/` 下的 schema
4. `main.py` 中 `include_router` 增加该域 router
5. 错误码占用新号段，在 `error_codes.py` 注释分区
6. 不改 framework，不把域内 Settings 注册到 main.py

---

## 配置治理（搭框架时一并落地）

```text
main.py          → 只组装 + 注册 router
域内 config.py   → 本域 Settings + Depends
Settings         → pydantic-settings，无逻辑
```

`.env` 用 SCREAMING_SNAKE_CASE，Python 用 snake_case。  
Profile：`.env`（默认值）、`.env.local` / `.env.prod` 覆盖。密钥不进 Git。

---

## Alembic 迁移

```bash
alembic init alembic
# alembic/env.py 中导入所有 model 确保自动检测
alembic revision --autogenerate -m "init"
alembic upgrade head
```

---

## 复制参考仓库时的改名清单

全局替换（先文档后代码）：

| 参考仓库常见值 | 换成 |
|----------------|------|
| 旧 `{app}` | 新 `{app}` |
| 旧 `{base}` 包名 | 新 `{base}` |
| 权限串 `旧域:旧资源:动作` | 新域权限 |
| 示例业务表 / 开放路径 | **删除**，不要带进新域 |

复制后立刻删掉：业务 `module_*`、业务 SQL migration、业务联调文档。
