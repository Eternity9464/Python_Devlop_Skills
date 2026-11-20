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

---

## 内存缓存模式（外部 API 聚合场景）

适用于：调用 Zabbix / Splunk / Confluence 等重型外部 API，单次查询耗时 5-30 秒。

### 标准缓存类

```python
# framework/cache/base.py
import threading

class _Cache:
    """线程安全的单值缓存容器。"""
    def __init__(self):
        self.data = None
        self.ts   = 0.0
        self.lock = threading.Lock()
```

### Stale-While-Revalidate（推荐）

```python
# 适合：数据轻微过期可接受，绝不能让用户等待
import time, threading

_cache = _Cache()
_refreshing = threading.Lock()
TTL = 60  # 秒

def get_data(force=False):
    now = time.time()
    with _cache.lock:
        data, ts = _cache.data, _cache.ts

    if data and not force:
        if now - ts >= TTL:
            _refresh_async()     # 后台刷新，立刻返回旧数据
        return data              # ← 用户 <1ms 拿到数据

    return _fetch_and_store()    # 首次或 force：同步等待

def _refresh_async():
    if not _refreshing.acquire(blocking=False):
        return                   # 已有刷新在跑，不重复启动
    def run():
        try:
            _fetch_and_store()
        except Exception:
            pass                 # 刷新失败：继续用旧数据
        finally:
            _refreshing.release()
    threading.Thread(target=run, daemon=True).start()

def _fetch_and_store():
    data = _call_external_api()  # 实际查询
    with _cache.lock:
        _cache.data = data
        _cache.ts   = time.time()
    return data
```

### 启动预热（prewarm）

应用启动时立即在后台填充缓存，第一个用户不必等待：

```python
# 在 create_app() / lifespan 末尾调用
def prewarm():
    def run():
        try:
            get_data()
        except Exception:
            pass
    threading.Thread(target=run, daemon=True).start()
```

### 后台定时刷新（适合超重型查询）

```python
# 适合：单次查询 >30s，不能放在请求链路里
import time, threading

def start_refresher(interval: int = 120):
    def loop():
        while True:
            try:
                _fetch_and_store()
            except Exception:
                pass
            time.sleep(interval)
    threading.Thread(target=loop, daemon=True).start()
```

**选型指引**：

| 场景 | 策略 |
|------|------|
| 查询 <5s，轻微过期可接受 | Stale-While-Revalidate |
| 查询 >30s | 后台定时刷新 + prewarm |
| 文件内容（xlsx/json），按需读 | mtime 感知（见下节） |

---

## 文件缓存：mtime 感知

适用于：本地 xlsx / JSON 文件，由外部上传替换，内容变化即失效。

```python
import os

_file_cache = {}  # path -> (key, data)

def load_file(path: str):
    st = os.stat(path)
    key = (st.st_mtime_ns, st.st_size)   # 变化信号
    hit = _file_cache.get(path)
    if hit and hit[0] == key:
        return hit[1]                     # 文件未变，直接返回
    data = _parse(path)                   # 重新解析
    _file_cache[path] = (key, data)
    return data
```

---

## 原子文件写入

**禁止**直接 `open(path, "w")` 写文件，写到一半程序崩溃会导致文件损坏。

```python
import os, tempfile, json

def atomic_write_json(path: str, data: dict) -> None:
    dir_ = os.path.dirname(path)
    os.makedirs(dir_, exist_ok=True)
    fd, tmp = tempfile.mkstemp(dir=dir_, suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        os.replace(tmp, path)   # 原子替换：要么旧文件完好，要么新文件完整
    finally:
        if os.path.exists(tmp):
            os.remove(tmp)
```

xlsx 同理：先写临时文件 → `validate()` → `os.replace()`。

---

## 外部数据源降级策略

上游 API（Zabbix / Splunk / Confluence 等）不可用时，**不应让页面直接 500**。

```python
def get_public_ips(force=False):
    now = time.time()
    with _lock:
        data, ts = _cache["data"], _cache["ts"]

    if data and not force and now - ts < TTL:
        return data

    try:
        fresh = parse(_fetch())       # 请求上游
    except Exception as e:
        if data:
            # ✅ 有旧数据：降级返回，标记为 stale
            return dict(data, stale_error=str(e))
        raise                         # 从未成功过：让调用方处理

    with _lock:
        _cache["data"], _cache["ts"] = fresh, now
    return fresh
```

Router / 页面层：

```python
result = get_public_ips()
if result.get("stale_error"):
    # 展示 banner："数据来自缓存，上游暂时不可用"
    ...
```

规则：
- 有旧数据 → 降级展示 + 告警
- 从未有数据 → 抛出，让页面展示明确的错误信息
- 禁止静默吞掉异常后返回空数据（用户不知道数据有问题）

---

## 安全字符串比较（防时序攻击）

比较密码、Token、签名时，**禁止用 `==`**：

```python
# ❌ 危险：长度不同时提前返回，攻击者可枚举
if user_input == stored_password:
    ...

# ✅ 安全：固定时间比较，不泄露信息
import hmac
if hmac.compare_digest(user_input.encode(), stored_password.encode()):
    ...
```

同样适用于 HMAC 签名验证：

```python
expected = hmac.new(secret.encode(), payload.encode(), "sha256").hexdigest()
if not hmac.compare_digest(received_sign, expected):
    raise_exception(SIGN_FAILED)
```



| 日期 | 说明 |
|------|------|
| 2026-11-19 | 从 java-modular-monolith 转换为 Python FastAPI 版本 |
| 2026-11-20 | 补充缓存模式、原子文件写入、降级策略、安全比较（来自生产项目实践） |
