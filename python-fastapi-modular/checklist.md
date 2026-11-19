# 自检清单

按场景勾选后再提 PR 或宣布「框架搭完」。

---

## 新工程（搭建框架）

- [ ] 占位符已写入项目绑定（`{app}` `{base}` 前缀）
- [ ] `pyproject.toml` 完整；依赖锁定（`uv.lock`）
- [ ] `framework/common/`：`CommonResult`（成功 `code === 0`）、ErrorCode、`raise_exception`
- [ ] `framework/web/`：全局异常处理、按 prefix 注册 router、main.py 未手写前缀
- [ ] `framework/db/`：`BaseDO`、`get_session`、审计填充
- [ ] `main.py` 是空壳：只 `create_app()` + 注册 router；无业务初始化堆砌
- [ ] 至少一条管理端 CRUD 返回 `code === 0`
- [ ] 本 Skill + Rule 模板已进仓库 `.cursor/`
- [ ] 密钥未进 Git；`.env.example` 已提交；`.env` 在 `.gitignore`

---

## 新模块

- [ ] `module_{domain}/` 含 api + biz；`__init__.py` 已建
- [ ] `main.py` include_router（未改其他模块）
- [ ] `api/schemas.py` 无 SQLAlchemy import
- [ ] ErrorCode 使用新号段并注释分区
- [ ] 跨模块只 import 对方 `api/` 下的 schema
- [ ] 域内 Settings 在 `biz/config.py`，未改 framework

---

## 新管理端 / C 端接口

### 模块与包

- [ ] 错误码在 `api/error_codes.py`
- [ ] 包路径：`biz/router_admin.py` / `service.py` / `repository.py` / `models.py` / `convert.py`
- [ ] Router **未**直接注入 Repository（例外已在 PR 说明）

### Router

- [ ] `APIRouter(tags=[...])` + `@router.post/put/delete/get` 正确
- [ ] 管理端 router 在 `router_admin.py`；main.py 传 prefix（未硬写 `{admin-prefix}`）
- [ ] HTTP：`POST /create`、`PUT /update`、`GET /get`、`GET /page` 等
- [ ] 写操作 `Depends(require_permission(...))`
- [ ] `summary` 有中文描述
- [ ] `CommonResult` + `CommonResult.success()`；无业务逻辑
- [ ] `Depends(XxxService)` 注入

### Schema / Convert / Service / Repository

- [ ] `CreateReqSchema` / `UpdateReqSchema` / `PageReqSchema` / `RespSchema`；共有字段在 BaseSchema
- [ ] 字段有 `description`；时间范围 `list[datetime]`；分页 `extends PageParam`
- [ ] convert 仅映射，无 if 判断
- [ ] Service 类 + 方法；多表写有 `begin()` 事务
- [ ] `raise_exception(ERROR_CODE)`；更新前 `_validate_exists`
- [ ] model `extends BaseDO`；repository 查询加 `deleted == False`；`if_present` 条件
- [ ] 批量写先判空；软删除置 `deleted=True`

### 行为

- [ ] 成功 `code === 0`
- [ ] 参数错误 400 + 可读 msg
- [ ] 业务不存在等用模块错误码（非 500 裸 raise）
- [ ] 权限串与菜单配置一致
- [ ] 无密钥硬编码；关键路径有日志（非空 except）

---

## 新开放接口

- [ ] 不在 `router_admin.py`；路径 `/external/{系统}`
- [ ] 独立鉴权 `Depends` + 域内 `biz/config.py` + `Settings`
- [ ] main.py 注册时使用独立 prefix
- [ ] 审计表 + `request_id` UNIQUE
- [ ] 联调文档列出 **code + 具体 msg**
- [ ] 鉴权失败：200 + CommonResult + `logger.warning`；透传 ServiceException msg
- [ ] 每种失败独立 ErrorCode（签名不匹配除外）

---

## 错误码专项

- [ ] 新增失败分支已在 `api/error_codes.py` 加常量
- [ ] Service 仅 `raise_exception(...)`，无运行时拼 msg
- [ ] 未用一个笼统码覆盖多种原因
- [ ] 仅安全敏感场景使用笼统 msg

---

## Review 反馈格式

- 必须改：分层反了、错误码违规、开放通道鉴权混用、main.py 堆业务、secret 入日志
- 建议改：命名不一致、日志前缀缺失、可抽共享 schema
- 可选：注释、文档措辞
