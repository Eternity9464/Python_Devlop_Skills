# 编码规范

日常写 Python 时遵守。包路径里的 `{base}` `{domain}` `{resource}` 见 [SKILL.md](SKILL.md) §2。

---

## 1. biz 包结构

```text
{base}/module_{domain}/biz/
├── router_admin.py       管理端路由
├── router_app.py         C 端路由（可选）
├── router_{开放名}.py    开放路由（可选，勿与 admin 合并）
├── service.py            业务逻辑
├── repository.py         数据库 CRUD
├── models.py             ORM model
├── convert.py            ORM ↔ Schema
├── middleware.py         域内 Middleware（可选）
└── config.py             域内 Settings + Depends
```

**依赖**：Router → Service → Repository。禁止 Router 直接注入 Repository 写业务（例外须在 PR 说明）。禁止 Repository 调 Service。跨模块只 import `*/api/`。

---

## 2. Router（管理端）

```python
from fastapi import APIRouter, Depends
from {base}.framework.common.result import CommonResult
from {base}.framework.security.deps import require_permission
from {base}.module_{domain}.api.schemas import (
    XxxCreateReqSchema, XxxUpdateReqSchema, XxxPageReqSchema, XxxRespSchema
)
from {base}.framework.common.page import PageResult
from .service import XxxService

router = APIRouter(tags=["管理后台 - {资源中文名}"])

@router.post("/create", summary="创建")
async def create(
    req: XxxCreateReqSchema,
    service: XxxService = Depends(),
    _=Depends(require_permission("{domain}:{resource}:create")),
) -> CommonResult[int]:
    return CommonResult.success(await service.create_xxx(req))
```

| 操作 | Method | 路径 | 返回 |
|------|--------|------|------|
| 创建 | POST | `/create` | `CommonResult[int]` |
| 更新 | PUT | `/update` | `CommonResult[bool]` |
| 删除 | DELETE | `/delete` | `CommonResult[bool]` |
| 详情 | GET | `/get` | `CommonResult[XxxRespSchema]` |
| 分页 | GET | `/page` | `CommonResult[PageResult[XxxRespSchema]]` |
| 导出 | GET | `/export` | `StreamingResponse` |

- 写操作 Pydantic 自动校验 body；查询参数用 `Query`
- 只调 Service；禁止事务、Repository、吞异常
- main.py 注册时传 `prefix`，router 内**只写资源路径**

在 `main.py` 注册：

```python
from {base}.module_{domain}.biz.router_admin import router as {domain}_admin_router
app.include_router(
    {domain}_admin_router,
    prefix=f"{settings.admin_api_prefix}/{domain}/{resource}"
)
```

开放接口：

```python
# router_{开放名}.py
router = APIRouter(prefix="/external/{系统}", tags=["开放 - {系统}"])

@router.post("/notify")
async def notify(req: XxxNotifyReqSchema, _=Depends(verify_{系统}_signature)):
    ...
```

路径 `/external/...`；独立 `Depends` 或 Middleware 鉴权；**不用**管理端 Bearer。

---

## 3. Schema 与 Convert

```python
# api/schemas.py
from pydantic import BaseModel, Field
from datetime import datetime

class XxxBaseSchema(BaseModel):
    name: str = Field(..., description="名称")

class XxxCreateReqSchema(XxxBaseSchema):
    pass

class XxxUpdateReqSchema(XxxBaseSchema):
    id: int = Field(..., description="ID")

class XxxPageReqSchema(PageParam):
    name: str | None = Field(None, description="名称")
    create_time: list[datetime] | None = Field(None, description="创建时间范围")

class XxxRespSchema(XxxBaseSchema):
    id: int
    create_time: datetime
    model_config = {"from_attributes": True}
```

| 类型 | 类名 |
|------|------|
| 创建 | `XxxCreateReqSchema(XxxBaseSchema)` |
| 更新 | `XxxUpdateReqSchema(XxxBaseSchema)` + `id` |
| 分页 | `XxxPageReqSchema(PageParam)` |
| 响应 | `XxxRespSchema` |
| 导出 | `XxxExportSchema` |

```python
# biz/convert.py
from {base}.module_{domain}.api.schemas import XxxCreateReqSchema, XxxRespSchema
from .models import XxxDO

def req_to_do(req: XxxCreateReqSchema) -> XxxDO:
    return XxxDO(**req.model_dump())

def do_to_resp(do: XxxDO) -> XxxRespSchema:
    return XxxRespSchema.model_validate(do)

def do_list_to_resp(dos: list[XxxDO]) -> list[XxxRespSchema]:
    return [do_to_resp(d) for d in dos]
```

只做映射，不写 if / 业务判断。

---

## 4. Service

```python
# biz/service.py
import logging
from {base}.framework.common.exception import raise_exception
from {base}.module_{domain}.api.error_codes import XXX_NOT_EXISTS
from {base}.module_{domain}.api.schemas import XxxCreateReqSchema, XxxUpdateReqSchema
from .repository import XxxRepository
from .convert import req_to_do, do_to_resp

logger = logging.getLogger(__name__)

class XxxService:
    def __init__(self, repo: XxxRepository = Depends()):
        self.repo = repo

    async def create_xxx(self, req: XxxCreateReqSchema) -> int:
        do = req_to_do(req)
        await self.repo.insert(do)
        return do.id

    async def update_xxx(self, req: XxxUpdateReqSchema) -> bool:
        await self._validate_exists(req.id)
        do = req_to_do(req)
        do.id = req.id
        await self.repo.update(do)
        return True

    async def _validate_exists(self, id: int) -> None:
        if not await self.repo.get_by_id(id):
            raise_exception(XXX_NOT_EXISTS)
```

- 多表写 / 退款 / 流水：使用 `async with session.begin()`
- 新接口返回 RespSchema，不返回 ORM model
- 禁止空 except；禁止 `raise HTTPException` 表达业务错误

---

## 5. Repository

```python
# biz/repository.py
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import Depends
from {base}.framework.db.session import get_session
from {base}.framework.common.page import PageParam, PageResult
from .models import XxxDO
from {base}.module_{domain}.api.schemas import XxxPageReqSchema

class XxxRepository:
    def __init__(self, session: AsyncSession = Depends(get_session)):
        self.session = session

    async def get_by_id(self, id: int) -> XxxDO | None:
        result = await self.session.execute(
            select(XxxDO).where(XxxDO.id == id, XxxDO.deleted == False)
        )
        return result.scalar_one_or_none()

    async def insert(self, do: XxxDO) -> None:
        self.session.add(do)
        await self.session.flush()

    async def update(self, do: XxxDO) -> None:
        await self.session.merge(do)

    async def select_page(self, req: XxxPageReqSchema) -> PageResult[XxxDO]:
        stmt = select(XxxDO).where(XxxDO.deleted == False)
        if req.name:
            stmt = stmt.where(XxxDO.name.ilike(f"%{req.name}%"))
        if req.create_time and len(req.create_time) == 2:
            stmt = stmt.where(XxxDO.create_time.between(*req.create_time))
        total_stmt = select(func.count()).select_from(stmt.subquery())
        total = (await self.session.execute(total_stmt)).scalar()
        stmt = stmt.offset((req.page_no - 1) * req.page_size).limit(req.page_size)
        rows = (await self.session.execute(stmt)).scalars().all()
        return PageResult(list=list(rows), total=total)
```

- 条件用 `if field:` 判断再加，等价于 Java `xxxIfPresent`
- 批量插入先判空；软删除置 `deleted=True`
- 少量复杂统计可直接写 SQLAlchemy Core，避免大段原生 SQL

---

## 6. ORM Model

```python
# biz/models.py
from sqlalchemy import String, Boolean
from sqlalchemy.orm import Mapped, mapped_column
from {base}.framework.db.base import BaseDO

class XxxDO(BaseDO):
    __tablename__ = "xxx_table"

    name: Mapped[str] = mapped_column(String(64), comment="名称")
    status: Mapped[int] = mapped_column(default=0, comment="状态")
```

- 继承 `BaseDO`（含 id、create_time、update_time、deleted）
- `deleted` 字段用 `True/False`；查询默认加 `XxxDO.deleted == False`
- 审计字段（creator / updater）在 Middleware 或 Depends 中填充

---

## 7. 错误码

**定义**（`module_{domain}/api/error_codes.py`，按域分文件）：

```python
from {base}.framework.common.error_code import ErrorCode

XXX_NOT_EXISTS = ErrorCode(1008007000, "xxx不存在")
OPEN_DEVICE_ID_BLANK = ErrorCode(4001, "deviceId 不能为空")
OPEN_TIMESTAMP_SKEW = ErrorCode(4002, "timestamp 超出允许时间范围（±{} 秒）")
OPEN_SIGN_FAILED = ErrorCode(4003, "签名验证失败")  # 仅签名不匹配
```

**抛出**（Service）：

```python
from {base}.framework.common.exception import raise_exception, raise_invalid_param

raise_exception(XXX_NOT_EXISTS)
raise_exception(OPEN_TIMESTAMP_SKEW, skew_seconds)
raise_invalid_param("参数 xxx 无效")  # 全局 400
```

| 场景 | 文件 | code |
|------|------|------|
| 管理端 / 域内业务 | `error_codes.py` | 域号段 |
| 开放对接 | `{域}_open_error_codes.py` | 对接方约定 |
| 通用参数非法 | 无单独常量 | `raise_invalid_param` → 400 |

禁止：

```python
# ❌ 运行时拼 msg
raise ServiceException(ErrorCode(4001, f"deviceId {id} 不能为空"))

# ❌ 多种原因共用笼统码
raise_exception(OPEN_SIGN_FAILED)   # 实际是 timestamp 超出范围
```

---

## 8. 日志（应用层）

```python
import logging
logger = logging.getLogger(__name__)

# 格式：[{domain}] {动作} key={}
logger.info("[order] 创建订单 order_id=%s", order_id)
logger.warning("[order] 鉴权失败 app_id=%s", app_id)
logger.error("[order] 退款异常 order_id=%s", order_id, exc_info=True)
```

- 禁止空 except；禁止打印 password、secret、sign、token
- **warn**：可恢复 / 鉴权失败 / 重试；**error**：需人工关注；**info**：关键状态变更

---

## 9. 安全

| 能力 | 管理端 | 开放接口 |
|------|--------|----------|
| 鉴权 | Header `Authorization: Bearer <token>` | 独立 `Depends` 或 body `token` |
| 权限 | `Depends(require_permission("…"))` | Service 层校验对接方身份 |
| 数据范围 | `get_current_user` 获取租户/门店 | 对接方 appId / device 范围 |

密钥、证书不进源码；走 `.env` 或环境变量。
