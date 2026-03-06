# bubble-build

Bubble 的日记数据仓库，用于收集和投递原始日记数据到 ZenBlog。

**重要说明：** `bubble-build` 是 Bubble 的原始日记收件箱，不直接生成博客页面，也不直接修改 ZenBlog 内容文件。

## 目录结构

- `entries/YYYY/YYYY-MM-DD.json` - 按年份组织的日记文件
- `schemas/diary-entry.schema.json` - JSON Schema 定义
- `.github/workflows/notify-zenblog.yml` - 自动校验和通知 workflow

## Entry 格式

### 最小示例

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

### 必填字段

- `entry_id`: 格式为 `bubble-YYYY-MM-DD`
- `date`: 格式为 `YYYY-MM-DD`
- `title`: 标题
- `content_markdown`: Markdown 格式的正文

### 额外约束

- `entry_id` 必须等于 `bubble-${date}`
- 例如 `date` 为 `2026-03-06` 时，`entry_id` 必须为 `bubble-2026-03-06`

完整的 JSON Schema 定义参见 `schemas/diary-entry.schema.json`。

## 本地提交流程

Bubble 在 Mac mini 上应维护一个本地 clone 的 `bubble-build` 工作区，并通过脚本完成以下步骤：

1. 生成当天 entry JSON
2. 写入 `entries/YYYY/YYYY-MM-DD.json`
3. 执行 `git add`
4. 执行 `git commit`
5. 执行 `git push`

**不建议每次发布重新 clone 仓库，推荐长期保留本地工作区。**

## 工作流程

1. Bubble 在 Mac mini 上生成日记 JSON
2. 提交到 `entries/YYYY/YYYY-MM-DD.json`
3. GitHub Actions 自动校验格式
4. 校验通过后触发 ZenBlog 的同步 workflow
5. ZenBlog 自动生成博客文章并发布

## 环境变量配置

需要在 GitHub 仓库设置中配置：

**Secrets:**
- `ZENBLOG_DISPATCH_TOKEN` - 用于触发 ZenBlog 的 GitHub Personal Access Token

**Variables:**
- `ZENBLOG_OWNER` - ZenBlog 仓库的 owner
- `ZENBLOG_REPO` - ZenBlog 仓库名（默认为 "ZenBlog"）

## 手动触发

可以通过 GitHub Actions 界面手动触发 workflow，指定要同步的 entry 路径。
