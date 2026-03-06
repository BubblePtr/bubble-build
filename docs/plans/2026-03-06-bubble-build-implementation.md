# bubble-build 实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 构建 bubble-build 仓库的完整基础设施，包括目录结构、JSON Schema、GitHub Actions workflow 和文档

**Architecture:** 采用独立 Schema 文件 + 单 workflow 的设计，使用 ajv-cli 进行 JSON Schema 校验，使用 Node.js 脚本进行业务逻辑校验，通过 curl 调用 GitHub API 触发 ZenBlog 的 repository_dispatch

**Tech Stack:** JSON Schema (draft-07), GitHub Actions, ajv-cli, ajv-formats, Node.js 20, curl

---

## Task 1: 创建基础目录结构

**Files:**
- Create: `entries/2026/.gitkeep`
- Create: `schemas/.gitkeep`

**Step 1: 创建 entries 目录**

```bash
mkdir -p entries/2026
touch entries/2026/.gitkeep
```

**Step 2: 创建 schemas 目录**

```bash
mkdir -p schemas
touch schemas/.gitkeep
```

**Step 3: 验证目录结构**

Run: `ls -la entries/2026/ schemas/`
Expected: 看到 .gitkeep 文件存在

**Step 4: 提交**

```bash
git add entries/ schemas/
git commit -m "chore: 创建基础目录结构"
```

---

## Task 2: 创建 JSON Schema 文件

**Files:**
- Create: `schemas/diary-entry.schema.json`

**Step 1: 创建 Schema 文件**

创建 `schemas/diary-entry.schema.json`，内容如下：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Bubble Diary Entry",
  "description": "Schema for Bubble's daily diary entries",
  "type": "object",
  "required": ["entry_id", "date", "title", "content_markdown"],
  "properties": {
    "entry_id": {
      "type": "string",
      "pattern": "^bubble-\\d{4}-\\d{2}-\\d{2}$",
      "description": "Unique identifier in format: bubble-YYYY-MM-DD"
    },
    "date": {
      "type": "string",
      "pattern": "^\\d{4}-\\d{2}-\\d{2}$",
      "description": "Date in YYYY-MM-DD format"
    },
    "title": {
      "type": "string",
      "minLength": 1,
      "maxLength": 200,
      "description": "Entry title"
    },
    "summary": {
      "type": "string",
      "minLength": 1,
      "maxLength": 500,
      "description": "Brief summary of the entry"
    },
    "content_markdown": {
      "type": "string",
      "minLength": 1,
      "description": "Main content in Markdown format"
    },
    "tags": {
      "type": "array",
      "maxItems": 10,
      "items": {
        "type": "string",
        "pattern": "^[a-z0-9-]+$"
      },
      "uniqueItems": true,
      "description": "Tags for categorization"
    },
    "mood": {
      "type": "string",
      "enum": ["steady", "excited", "reflective", "milestone", "challenging"],
      "description": "Mood or significance indicator"
    },
    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp"
    }
  },
  "additionalProperties": false
}
```

**Step 2: 验证 Schema 文件格式**

Run: `cat schemas/diary-entry.schema.json | python3 -m json.tool > /dev/null && echo "Valid JSON"`
Expected: 输出 "Valid JSON"

**Step 3: 提交**

```bash
git add schemas/diary-entry.schema.json
git commit -m "feat: 添加 diary entry JSON Schema 定义"
```

---

## Task 3: 创建测试用 entry 文件

**Files:**
- Create: `entries/2026/2026-03-06.json`

**Step 1: 创建测试 entry**

创建 `entries/2026/2026-03-06.json`，内容如下：

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

**Step 2: 验证 JSON 格式**

Run: `cat entries/2026/2026-03-06.json | python3 -m json.tool > /dev/null && echo "Valid JSON"`
Expected: 输出 "Valid JSON"

**Step 3: 提交**

```bash
git add entries/2026/2026-03-06.json
git commit -m "feat: 添加测试用 entry 文件"
```

---

## Task 4: 创建 GitHub Actions workflow

**Files:**
- Create: `.github/workflows/notify-zenblog.yml`

**Step 1: 创建 workflow 目录**

```bash
mkdir -p .github/workflows
```

**Step 2: 创建 workflow 文件**

创建 `.github/workflows/notify-zenblog.yml`，内容如下：

```yaml
name: Notify ZenBlog

on:
  push:
    paths:
      - 'entries/**/*.json'
  workflow_dispatch:
    inputs:
      entry_path:
        description: '要触发同步的 entry 路径，例如 entries/2026/2026-03-06.json'
        required: true
        type: string

jobs:
  validate-and-dispatch:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:
      - name: Checkout bubble-build
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install ajv-cli
        run: npm install --global ajv-cli ajv-formats

      - name: Resolve changed entries
        id: entries
        shell: bash
        run: |
          set -euo pipefail

          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            ENTRY_PATH="${{ inputs.entry_path }}"
            if [ ! -f "$ENTRY_PATH" ]; then
              echo "Entry file not found: $ENTRY_PATH"
              exit 1
            fi

            {
              echo "files<<EOF"
              echo "$ENTRY_PATH"
              echo "EOF"
            } >> "$GITHUB_OUTPUT"
            exit 0
          fi

          BEFORE_SHA="${{ github.event.before }}"
          AFTER_SHA="${{ github.sha }}"

          if [ -z "$BEFORE_SHA" ] || [ "$BEFORE_SHA" = "0000000000000000000000000000000000000000" ]; then
            FILES=$(git ls-files 'entries/**/*.json')
          else
            FILES=$(git diff --name-only --diff-filter=AM "$BEFORE_SHA" "$AFTER_SHA" -- 'entries/**/*.json')
          fi

          if [ -z "$FILES" ]; then
            echo "No changed entry files found."
            {
              echo "files<<EOF"
              echo ""
              echo "EOF"
            } >> "$GITHUB_OUTPUT"
            exit 0
          fi

          {
            echo "files<<EOF"
            printf '%s\n' "$FILES"
            echo "EOF"
          } >> "$GITHUB_OUTPUT"

      - name: Validate changed entries
        if: steps.entries.outputs.files != ''
        shell: bash
        run: |
          set -euo pipefail

          while IFS= read -r entry; do
            [ -z "$entry" ] && continue

            echo "Validating schema: $entry"
            ajv validate \
              -s schemas/diary-entry.schema.json \
              -d "$entry" \
              --spec=draft7 \
              -c ajv-formats

            echo "Running business validation: $entry"
            node --input-type=module -e '
              import fs from "fs";

              const filePath = process.argv[1];
              const raw = fs.readFileSync(filePath, "utf8");
              const entry = JSON.parse(raw);

              if (entry.entry_id !== `bubble-${entry.date}`) {
                console.error(
                  `entry_id/date mismatch in ${filePath}: expected bubble-${entry.date}, got ${entry.entry_id}`
                );
                process.exit(1);
              }

              console.log(`Business validation passed: ${filePath}`);
            ' "$entry"
          done <<< "${{ steps.entries.outputs.files }}"

      - name: Dispatch to ZenBlog
        if: steps.entries.outputs.files != ''
        env:
          ZENBLOG_DISPATCH_TOKEN: ${{ secrets.ZENBLOG_DISPATCH_TOKEN }}
          ZENBLOG_OWNER: ${{ vars.ZENBLOG_OWNER }}
          ZENBLOG_REPO: ${{ vars.ZENBLOG_REPO }}
        shell: bash
        run: |
          set -euo pipefail

          if [ -z "${ZENBLOG_DISPATCH_TOKEN:-}" ]; then
            echo "Missing secret: ZENBLOG_DISPATCH_TOKEN"
            exit 1
          fi

          if [ -z "${ZENBLOG_OWNER:-}" ]; then
            echo "Missing variable: ZENBLOG_OWNER"
            exit 1
          fi

          if [ -z "${ZENBLOG_REPO:-}" ]; then
            echo "Missing variable: ZENBLOG_REPO"
            exit 1
          fi

          while IFS= read -r entry; do
            [ -z "$entry" ] && continue

            entry_id=$(node --input-type=module -e '
              import fs from "fs";
              const raw = fs.readFileSync(process.argv[1], "utf8");
              const data = JSON.parse(raw);
              process.stdout.write(data.entry_id);
            ' "$entry")

            payload=$(node --input-type=module -e '
              const entryPath = process.argv[1];
              const entryId = process.argv[2];
              process.stdout.write(JSON.stringify({
                event_type: "bubble-build-sync",
                client_payload: {
                  entry_path: entryPath,
                  entry_id: entryId,
                  overwrite: false
                }
              }));
            ' "$entry" "$entry_id")

            echo "Dispatching $entry_id to ${ZENBLOG_OWNER}/${ZENBLOG_REPO}"

            curl --fail-with-body --silent --show-error \
              -X POST \
              -H "Accept: application/vnd.github+json" \
              -H "Authorization: Bearer ${ZENBLOG_DISPATCH_TOKEN}" \
              https://api.github.com/repos/${ZENBLOG_OWNER}/${ZENBLOG_REPO}/dispatches \
              -d "$payload"
          done <<< "${{ steps.entries.outputs.files }}"
```

**Step 3: 验证 YAML 格式**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/notify-zenblog.yml'))" && echo "Valid YAML"`
Expected: 输出 "Valid YAML"

**Step 4: 提交**

```bash
git add .github/workflows/notify-zenblog.yml
git commit -m "feat: 添加 GitHub Actions workflow 用于校验和通知 ZenBlog"
```

---

## Task 5: 创建 .gitignore 文件

**Files:**
- Create: `.gitignore`

**Step 1: 创建 .gitignore**

创建 `.gitignore`，内容如下：

```
# macOS
.DS_Store
._*

# Editor
.vscode/
.idea/
*.swp
*.swo
*~

# Node
node_modules/
```

**Step 2: 提交**

```bash
git add .gitignore
git commit -m "chore: 添加 .gitignore 文件"
```

---

## Task 6: 创建 README.md 文件

**Files:**
- Create: `README.md`

**Step 1: 创建 README**

创建 `README.md`，内容如下：

```markdown
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
```

**Step 2: 提交**

```bash
git add README.md
git commit -m "docs: 添加 README 文档"
```

---

## Task 7: 验证完整性

**Step 1: 检查所有文件是否存在**

Run: `ls -la entries/2026/ schemas/ .github/workflows/ && ls -la .gitignore README.md`
Expected: 所有文件都存在

**Step 2: 验证 git 状态**

Run: `git status`
Expected: 工作区干净，没有未提交的文件

**Step 3: 查看提交历史**

Run: `git log --oneline`
Expected: 看到所有的提交记录

---

## 完成标准

- [ ] 目录结构完整（entries/, schemas/, .github/workflows/）
- [ ] JSON Schema 文件创建并格式正确
- [ ] 测试用 entry 文件创建并格式正确
- [ ] GitHub Actions workflow 文件创建并格式正确
- [ ] .gitignore 文件创建
- [ ] README.md 文件创建并内容完整
- [ ] 所有文件已提交到 git

## 后续步骤

完成实施后，需要：

1. 将仓库推送到 GitHub
2. 在 GitHub 仓库设置中配置 Secrets 和 Variables
3. 测试 workflow 是否能正常触发
4. 在 Bubble 侧配置自动提交脚本
