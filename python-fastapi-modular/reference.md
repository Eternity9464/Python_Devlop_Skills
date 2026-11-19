# 参考：日志、配置、开放对接、框架类型

配合 [SKILL.md](SKILL.md)。类名按参考实现；迁到新工程时换成等价类型。

---

## 错误码与消息（完整）

1. **一场景一常量**：每种失败在 `api/error_codes.py` 独立 `ErrorCode`，msg 中文、可联调定位。
2. **Service 只抛常量**：`raise_exception(ERROR_CODE)` 或带 `{}` 占位的 params。
3. **禁止运行时造 msg**：不直接 `raise ServiceException(ErrorCode(code, "临时字符串"))`。
4. **禁止大类掩盖**：不得把「缺字段 / 格式错 / 不存在 / 签名错」写成同一句。
5. **安全例外**：仅签名、token 不匹配可保留笼统 msg，不泄露校验细节。

开放与内管 **分文件**，避免对接方 code 段与内部号段混淆。

Middleware / 全局异常处理：

```python
@app.exception_handler(ServiceException)
async def handle_service_exception(request: Request, exc: ServiceException):
    # 透传 exc.code + exc.msg，不改回笼统文案
    return JSONResponse(
        status_code=200,
        content=CommonResult.error(exc.code, exc.msg).model_dump(),
    )
```

---

## 框架类型目录（最小集）

新工程应具备同等能力（类名可不同）：

| 能力 | 典型类型 |
|------|----------|
| 统一响应 | `CommonResult` / `PageResult` / `PageParam` |
| 错误 | `ErrorCode` / `ServiceException` / `raise_exception` |
| 持久化 | `BaseDO` / `XxxRepository`（含 `if_present` 条件） |
| Web | `GlobalExceptionHandler` / Settings（prefix 配置） |
| 请求过滤基类 | FastAPI `Depends` 或 `BaseHTTPMiddleware`（按前缀决定） |

框架能力一览（按需，见 [scaffolding.md](scaffolding.md) 阶段 3）：

| 能力 | 实现方式 |
|------|---------|
| 统一响应 / 工具 / 全局 ErrorCode | `framework/common/` |
| 异常 / URL prefix / 访问日志 / Swagger | `framework/web/` |
| ORM BaseDO / 审计填充 | `framework/db/` |
| Token / 权限 / 操作日志 | `framework/security/` |
| 缓存 / 限流幂等 / 任务 / 消息 / Excel / 链路 | 按需独立模块 |

---

## 日志五层

```text
请求
 ├─① 应用日志（logging）      Middleware 摘要 + Service logger → 文件
 ├─② 访问日志 DB              管理端 Middleware → infra 表（仅 {admin-prefix}）
 ├─③ 错误日志 DB              GlobalExceptionHandler → 未捕获异常
 ├─④ 操作日志 DB              装饰器（如 @log_record）→ 管理端敏感写
 └─⑤ 集成审计表               域内 ingest/bind log（开放接口幂等+对账）
```

| 类型 | 何时用 | 存储 |
|------|--------|------|
| 应用日志 | 排障、可恢复异常 | 文件 / 标准输出 |
| 访问日志 DB | 管理端 API 审计 | 框架；可配置开关 |
| 错误日志 DB | 500 / 未预期 | 框架自动 |
| 操作日志 | 管理端关键写 | 显式装饰器 |
| 集成审计 | 外部对接、幂等 | 域内 `xxx_ingest_log` 等 |

**重要**：访问日志 Middleware 若仅覆盖 `{admin-prefix}`，则 `/external/**`、C 端路径**不会**进访问日志表。开放接口必须自建审计表。

Python logging 配置示例：

```python
# framework/config/logging_config.py
import logging.config

LOGGING = {
    "version": 1,
    "formatters": {
        "default": {"format": "%(asctime)s [%(levelname)s] %(name)s - %(message)s"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "default"},
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "logs/{app}.log",
            "maxBytes": 10_485_760,
            "backupCount": 5,
            "formatter": "default",
        },
    },
    "root": {"level": "INFO", "handlers": ["console", "file"]},
}

logging.config.dictConfig(LOGGING)
```

敏感字段日志脱敏：password、token、access_token、secret、sign 不得明文打印。

---

## Middleware 顺序（概念）

```text
CORS → Trace → AccessLog（管理端）
→ 管理端 Bearer Token 验证（Depends / Security）
→ 开放接口鉴权 Depends（自行验签短路）
→ Router
```

开放接口鉴权在 Security 之后，不复用管理端 Bearer。

---

## 外部对接 7 步

1. `biz/router_{开放名}.py` + `APIRouter(prefix="/external/{系统}")`（非 admin）
2. `biz/middleware.py` 或 `biz/{开放名}_auth.py` 实现 `Depends` 验签
3. `biz/config.py` 注册 Settings + 在 `main.py` include_router
4. `.env` 中白名单该前缀（无需 Token 验证的路径）
5. 管理端 Token 验证 Depends 跳过该前缀
6. 域内审计表 + `docs/` 联调文档（code + **具体 msg** 对照表）
7. 幂等：Header/body 约定 `request_id` + 审计表 UNIQUE

审计表建议字段：`request_id`, 对接方身份, `api_type`, `raw_json`, `result_json`, `create_time`。

鉴权失败：HTTP 200 + `CommonResult.error(code, msg)` + `logger.warning("[{domain}Auth] ...", key)`。

---

## Settings（pydantic-settings）

```python
# 域内 config.py
from pydantic_settings import BaseSettings
from pydantic import Field

class XxxApiSettings(BaseSettings):
    model_config = {"env_prefix": "{APP}_XXX_API_", "env_file": ".env"}

    app_id: str = Field(..., description="对接方 appId")
    secret: str = Field(..., description="密钥")

_settings: XxxApiSettings | None = None

def get_xxx_api_settings() -> XxxApiSettings:
    global _settings
    if _settings is None:
        _settings = XxxApiSettings()
    return _settings
```

`.env` kebab 或 SCREAMING_SNAKE_CASE ↔ Python snake_case。同一业务域多个 Settings 可共用一个 `config.py`。

---

## 数据权限（模式）

```python
async def get_scoped_query(current_user=Depends(get_current_user)):
    shop_id = current_user.shop_id
    return shop_id  # 在 Repository 中追加 .where(XxxDO.shop_id == shop_id)
```

字段名按域替换（tenant_id / store_id / shop_id）。无登录上下文的开放接口按对接方身份过滤，不用管理端 current_user。

---

## 新能力落地顺序

1. api：ErrorCode + enums + 跨模块 schema
2. models → repository → convert → service
3. router（选对通道）
4. 外部：鉴权 Depends + config.py + 文档 + 审计表
5. Review：[checklist.md](checklist.md)

---

## 待团队拍板（写成 ADR，勿口头约定）

1. C 端是否补与管理端同级的 URL / Method 表
2. 操作日志覆盖范围（仅系统模块 vs 全管理端写操作）
3. 开放接口鉴权失败是否落审计表
4. APM（OpenTelemetry 等）是否生产启用
5. 是否用 pytest + import 检查校验包路径与依赖方向

---

## 修订

| 日期 | 说明 |
|------|------|
| 2026-11-19 | 从 java-modular-monolith 转换为 Python FastAPI 版本 |
