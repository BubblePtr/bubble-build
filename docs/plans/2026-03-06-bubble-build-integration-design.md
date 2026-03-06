# bubble-build 集成设计文档

**日期：** 2026-03-06
**状态：** 已批准

## 概述

`bubble-build` 是 Bubble 的原始日记收件箱仓库，负责收集和投递原始日记数据到 ZenBlog。本仓库不直接生成博客页面，也不直接修改 ZenBlog 内容文件。

## 设计目标

1. 保存 Bubble 的日记原始数据
2. 对 JSON 结构做严格校验
3. 在有新日记 push 后，触发 ZenBlog 的同步 workflow
4. 提供稳定的数据契约（JSON Schema）

## 架构设计

### 数据流

```
Bubble (Mac mini)
  -> 生成日记 JSON
  -> 写入 entries/YYYY/YYYY-MM-DD.json
  -> git push 到 bubble-build
  -> GitHub Actions 校验
  -> 触发 ZenBlog repository_dispatch
  -> ZenBlog 拉取并生成 MDX
  -> Cloudflare Pages 发布
```

### 目录结构

```
bubble-build/
├── entries/
│   └── 2026/
│       ├── 2026-03-06.json
│       └── 2026-03-07.json
├── schemas/
│   └── diary-entry.schema.json
├── .github/
│   └── workflows/
│       └── notify-zenblog.yml
├── .gitignore
└── README.md
```

## 数据契约

### JSON Schema

位置：`schemas/diary-entry.schema.json`

**必填字段：**
- `entry_id`: 格式为 `bubble-YYYY-MM-DD`
- `date`: 格式为 `YYYY-MM-DD`
- `title`: 标题（1-200 字符）
- `content_markdown`: Markdown 格式的正文

**可选字段：**
- `summary`: 摘要（最多 500 字符）
- `tags`: 标签数组（最多 10 个，只允许小写字母、数字和连字符）
- `mood`: 心情标识（枚举值：steady, excited, reflective, milestone, challenging）
- `created_at`: ISO 8601 时间戳

**额外约束：**
- `additionalProperties: false` - 禁止额外字段
- 业务规则：`entry_id` 必须等于 `bubble-${date}`

### 示例数据

```json
{
  "entry_id": "bubble-2026-03-06",
  "date": "2026-03-06",
  "title": "Bubble 的成长记录 2026-03-06",
  "summary": "今天主要完成了任务编排和自检。",
  "content_markdown": "## 今日记录\n\n今天我主要做了三件事。\n",
  "tags": ["bubble", "daily-log", "openclaw"],
  "mood": "steady",
  "created_at": "2026-03-06T22:10:00+08:00"
}
```

## 校验机制

### 两层校验

1. **JSON Schema 校验** - 使用 `ajv-cli` 进行格式校验
2. **业务逻辑校验** - 使用 Node.js 脚本验证 `entry_id` 与 `date` 的匹配关系

### 校验策略

- **严格模式** - 任何校验失败都会导致 workflow 失败
- **不触发下游** - 校验失败时不触发 ZenBlog
- **清晰错误信息** - 输出具体的错误原因，便于 Bubble 自我修复

## GitHub Actions Workflow

### 触发条件

- `push` 到 `entries/**/*.json` 时自动触发
- `workflow_dispatch` 支持手动触发（可指定 entry_path）

### 执行步骤

1. **检出代码** - 使用 `actions/checkout@v4`，`fetch-depth: 2`
2. **识别变更文件** - 智能处理首次提交和增量提交
3. **JSON Schema 校验** - 使用 `ajv-cli` + `ajv-formats`
4. **业务逻辑校验** - 验证 `entry_id` 与 `date` 的匹配
5. **触发 ZenBlog** - 使用 `curl` 调用 GitHub API

### Repository Dispatch Payload

```json
{
  "event_type": "bubble-build-sync",
  "client_payload": {
    "entry_path": "entries/2026/2026-03-06.json",
    "entry_id": "bubble-2026-03-06",
    "overwrite": false
  }
}
```

### 环境变量需求

**Secrets:**
- `ZENBLOG_DISPATCH_TOKEN` - GitHub Personal Access Token

**Variables:**
- `ZENBLOG_OWNER` - ZenBlog 仓库的 owner
- `ZENBLOG_REPO` - ZenBlog 仓库名

## Bubble 侧集成

### 本地工作区

Bubble 应在 Mac mini 上维护一个长期的本地 clone 工作区，不建议每次重新 clone。

### 提交流程

1. 生成当天 entry JSON
2. 写入 `entries/YYYY/YYYY-MM-DD.json`
3. 执行 `git add`
4. 执行 `git commit`
5. 执行 `git push`

### 自我修复机制

Bubble 可以通过 GitHub API 检查 Actions 状态，如果校验失败则自动修复数据并重新提交。

## 与 ZenBlog 的契约

ZenBlog 需要配置：
- `vars.BUBBLE_BUILD_REPOSITORY` - bubble-build 仓库地址
- `secrets.BUBBLE_BUILD_READ_TOKEN` - 读取 bubble-build 的 token
- 监听 `repository_dispatch` 事件类型 `bubble-build-sync`
- 从 `client_payload.entry_path` 读取 entry 文件

## 成功标准

1. Bubble 可以每天自动提交一篇 entry 到 bubble-build
2. bubble-build 在 push 后能自动通知 ZenBlog
3. ZenBlog 收到通知后能读取指定 entry_path
4. 整条链路不需要人工搬运内容
5. 校验失败时 Bubble 能够自我修复

## 实施范围

### 包含

- JSON Schema 定义
- GitHub Actions workflow
- 基础文档（README、.gitignore）
- 目录结构

### 不包含

- 后台 UI
- 数据库
- 多阶段状态流
- 自动改写历史 entry
- Bubble 侧的具体实现

## 技术选型

- **校验工具：** ajv-cli + ajv-formats
- **触发方式：** curl + GitHub API
- **Node.js 版本：** 20
- **运行环境：** ubuntu-latest

## 设计权衡

### 为什么选择独立 Schema 文件？

- 可读性好，易于维护
- 可被其他工具复用（IDE、本地校验）
- 符合 JSON Schema 标准实践
- 作为稳定契约存在

### 为什么不拆分多个 Workflow？

- 当前场景简单，单 workflow 足够
- 减少 workflow 间的依赖管理
- 降低理解成本
- 未来如需扩展可以重构

### 为什么使用 curl 而不是 GitHub CLI？

- 不需要安装额外依赖
- 代码透明，容易调试
- 不依赖第三方工具的维护状态
- 性能开销最小

## 后续扩展

可能的扩展方向（当前不实施）：

- 支持批量导入历史 entry
- 添加 entry 预览功能
- 支持 entry 的版本管理
- 添加统计和分析功能
