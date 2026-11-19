# 复制到其它工程

把本规范变成目标仓库的「全员默认」。不要复制业务模块。

---

## 1. 复制这些路径

| 源（本 Skill） | 目标仓库 |
|----------------|----------|
| `.cursor/skills/python-fastapi-modular/` 整目录 | `.cursor/skills/python-fastapi-modular/` |
| [templates/python-backend-standards.mdc](templates/python-backend-standards.mdc) | `.cursor/rules/python-backend-standards.mdc`（`alwaysApply: true`，`globs: "**/*.py"`） |

可选：再写一份 **项目绑定** Skill（示例见下），填占位符，方便 Agent 不再猜包名。

---

## 2. 填写项目绑定

在目标仓库新建 `.cursor/skills/{app}-architecture-standards/SKILL.md`：

```markdown
---
name: {app}-architecture-standards
description: >-
  Project binding for the python-fastapi-modular skill in this repo.
  Use when coding or reviewing this backend, or with 单体架构、工程规范.
---

# 项目绑定

本仓库实例化 `../python-fastapi-modular/SKILL.md`。先读该 Skill，再套用下表。

| 占位 | 本仓库 |
|------|--------|
| `{app}` | （填写） |
| `{base}` | （填写，Python 根包名） |
| `{admin-prefix}` | `/admin-api` |
| `{app-prefix}` | `/app-api` |
| Settings env_prefix | `{APP}_` |
| 权限串 | `{domain}:{resource}:{动作}` |

## 本仓库补充

- 已有模块列表、错误码号段、开放路径
- framework 类的实际包名（若与通用名不同）
```

绑定文件 **只写差异**，不复制通用正文。

---

## 3. 替换占位

在 Rule 与示例代码中替换：

- `{base}` → 目标根包名
- `{app}` → 包名 / YAML 前缀
- 权限示例 `{domain}:{resource}` → 目标第一模块

若目标暂无本规范的 framework：先实现 [scaffolding.md](scaffolding.md) 最小集（`CommonResult` + 全局异常 + 分层包名 + Schema/Service/Repository 命名），再写业务。

| 本规范能力 | 目标没有时 |
|------------|------------|
| `CommonResult` | 实现统一包装，成功码约定为 **0** |
| `GlobalExceptionHandler` | 异常 → 同一 JSON |
| `BaseDO` + 软删除 | 审计字段 + `deleted` 字段 |
| `if_present` 条件 | SQLAlchemy 动态 where |
| prefix 注册 | main.py include_router 时传 prefix（须在绑定里写明） |
| `Depends(require_permission)` | 或自有鉴权，通道仍要隔离 |

---

## 4. 落地顺序

1. Skill + Rule 进仓库，填写绑定表
2. 选一个 **新 CRUD 或新开放接口** 按 [checklist.md](checklist.md) 做样板
3. Review 强制对照清单
4. 旧模块 **改到时再对齐**

---

## 5. 给开发者的使用方式

- 对话中提到「按单体规范 / 搭模块 / 新接口」时，Agent 应自动读本 Skill
- 也可 `@python-fastapi-modular` 或 `@项目绑定`
- Python 文件编辑时 Rule（`.mdc`）始终生效，管日常分层与命名
- Skill 管架构决策、搭框架步骤、开放通道与 Review
