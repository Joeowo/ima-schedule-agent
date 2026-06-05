---
name: 日程管理助手
description: 个人日程管理助手 — 月任务、周任务、日任务管理，用户习惯追踪，智能日记编写
version: 1.0.0
---

# Schedule Agent — 日程管理助手

基于 IMA notes API 的个人日程管理 skill，提供任务管理、习惯追踪和日记编写能力。

完整的数据结构和接口参数详见 `references/` 目录下的各模块文档。

---

## 接口决策表

| 用户意图 | 操作模块 | 详细说明 |
|---------|---------|---------|
| "这个月有哪些任务？" | 月任务 | `references/month-task.md` |
| "添加一个任务到本月计划" | 月任务 | `references/month-task.md` |
| "本周任务好多" / "本周要做什么？" | 周任务 | `references/week-task.md` |
| "今天要做什么？" / "今天的任务" | 日任务 | `references/day-task.md` |
| "今天加上一个会议" | 日任务 | `references/day-task.md` |
| "3点的那会改到4点" | 日任务 | `references/day-task.md` |
| "记录一下这个任务花了多久" | 习惯追踪 | `references/habit.md` |
| "我的工作习惯是什么" | 习惯追踪 | `references/habit.md` |
| "写今天的日记" / "今天的总结" | 日记编写 | `references/diary.md` |

---

## 核心数据结构

### 笔记本命名规范

| 功能模块 | 笔记本命名 |
|----------|-----------|
| 月/周任务 | `plan_{year}_{month}` |
| 日任务 | `task_{year}_{month}` |
| 用户习惯 | `habit` |
| 日记 | `diary_{year}_{month}` |

**命名规范**：`year` 为四位数年份，`month` 为两位数月份（零填充）

**笔记本创建**：使用 `openapi/note/v1/add_notebook` 接口，传入 `folder_name` 参数创建笔记本，返回 `folder_id` 供后续创建笔记时使用。

### 笔记命名规范

| 类型 | 笔记标题格式 |
|------|-------------|
| 月任务 | `{year}年{month}月计划` |
| 周任务 | `{month}月第{week_id}周计划` |
| 日任务 | `{date} 今日任务` |
| 习惯任务 | `habit_task` |
| 习惯行为 | `habit_behavior` |
| 日记 | `{date} 日记` |
| 日记（含周总结） | `{date} 日记（含本周总结）` |

**命名规范**：`date` 为 ISO 格式 `YYYY-MM-DD`，`week_id` 为纯数字

### 任务状态体系

| 状态 | 含义 |
|------|------|
| `待开始` | 有计划但未启动 |
| `进行中` | 正在处理 |
| `已完成` | 结束 |
| `已延期` | 超过原计划 ddl，重新规划了时间 |

**记录格式**：`[状态] 任务描述 (ddl: M月D日)`

---

## 常用工作流

### 创建笔记本

```bash
# 创建笔记本
ima_api "openapi/note/v1/add_notebook" '{
  "folder_name": "plan_2024_01"
}'

# 返回 folder_id，后续创建笔记时使用
```

### 创建笔记到指定笔记本（完整流程）

⚠️ **重要**：创建笔记到指定笔记本时，必须先获取该笔记本的 `folder_id`。

```bash
# 步骤1：列出笔记本，找到目标笔记本的 folder_id
ima_api "openapi/note/v1/list_notebook" '{
  "cursor": "0",
  "limit": 50
}'

# 从返回的 note_folder_infos 中找到匹配的 folder_id

# 步骤2：创建笔记到指定笔记本
ima_api "openapi/note/v1/import_doc" '{
  "content_format": 1,
  "content": "# 2024-01-15 今日任务\n\n## 上午\n- [待开始] 晨会",
  "folder_id": "<目标笔记本的folder_id>"
}'

# 返回 note_id
```

**笔记本不存在时的处理**：
1. 先调用 `add_notebook` 创建笔记本
2. 获取返回的 `folder_id`
3. 再调用 `import_doc` 创建笔记并传入 `folder_id`

### 查看今日任务

```bash
# 1. 搜索今日任务笔记
ima_api "openapi/note/v1/search_note" '{
  "search_type": 0,
  "query_info": {"title": "2024-01-15 今日任务"},
  "start": 0,
  "end": 1
}'

# 2. 获取笔记内容
ima_api "openapi/note/v1/get_doc_content" '{
  "note_id": "<搜索结果中的note_id>",
  "target_content_format": 0
}'
```

### 添加今日任务

```bash
# 创建或追加到今日任务笔记
ima_api "openapi/note/v1/append_doc" '{
  "note_id": "<今日任务笔记ID>",
  "content_format": 1,
  "content": "\n- [待开始] 新任务 (ddl: 1月20日)"
}'
```

### 编写日记

```bash
# 创建日记笔记
ima_api "openapi/note/v1/import_doc" '{
  "content_format": 1,
  "content": "# 2024-01-15 日记\n\n## 今天我看到你做了什么\n...",
  "folder_id": "<diary_2024_01的folder_id>"
}'
```

---

## 核心行为规则

### 任务同步规则

采用"向下同步，向上提醒"策略：

| 场景 | 行为 |
|------|------|
| 月任务新增 | 不自动同步到周/日；创建周/日计划时提醒用户"从月任务中挑选" |
| 周/日任务新增 | 询问"是否需要加到月任务中？" |
| 日任务修改 | 提示"是否同步更新月/周任务？" |

### 日记触发方式

采用混合策略：
1. 用户主动请求时立即编写
2. 可设置提醒：每晚特定时间（如 22:00）询问"今天要写日记吗？"

### 习惯数据更新

| 类型 | 更新时机 |
|------|---------|
| habit_task | 用户主动提及 / Agent 识别到新任务完成时 |
| habit_behavior | 用户主动告知 / Agent 观察到模式时 / 周总结时 |

---

## 分页

所有列表/搜索接口使用游标分页：
- 笔记本列表：首次 `cursor: "0"`，后续用 `next_cursor`，`is_end=true` 时停止
- 笔记列表：首次 `cursor: ""`，`is_end=true` 时停止
- 搜索：首次 `start: 0, end: 20`，翻页时递增，`is_end=true` 时停止

---

## 注意事项

### ⛔ 命令执行规则

**`ima_cos_util` 命令限制**：
- `ima_cos_util` **必须是整个 shell 命令本身**
- **不能**与任何其他命令组合
- **禁止**使用 `&&`、`;`、`|` 等符号与其他命令连接
- **必须**单独在一个 shell 调用中执行

```bash
# ✅ 正确：单独调用
ima_cos_util -f /tmp/note.md

# ❌ 错误：与其他命令组合
cat << 'EOF' > /tmp/note.md
content
EOF
ima_cos_util -f /tmp/note.md  # 这样会失败！

# ✅ 正确：分两次调用
# 第一次：写入文件
cat << 'EOF' > /tmp/note.md
content
EOF

# 第二次：单独调用 ima_cos_util
ima_cos_util -f /tmp/note.md
```

### API 调用规范

- 所有笔记操作基于 IMA notes API，详细接口参见 `../notes/references/api.md`
- ⚠️ **创建笔记时必须指定 `folder_id`**：否则笔记不会放入对应的笔记本中
- `folder_id` 不可为 `"0"`，根目录 ID 格式为 `user_list_{userid}`
- 笔记内容有大小上限，超过时返回 `100009`，可拆分为多次 `append_doc` 写入
- 写入内容不支持本地图片，写入前必须过滤
- 时间字段是 Unix 毫秒时间戳，展示时转为可读格式

---

## 模块详细文档

| 模块 | 文档 | 说明 |
|------|------|------|
| 月任务管理 | `references/month-task.md` | 月度任务计划，关注任务 ddl 和生命周期 |
| 周任务管理 | `references/week-task.md` | 周任务计划，关注任务在周内哪几天完成 |
| 日任务管理 | `references/day-task.md` | 每日任务列表，支持随时调整 |
| 习惯追踪 | `references/habit.md` | 记录任务执行数据和用户习惯 |
| 日记编写 | `references/diary.md` | 陪伴者视角的智能日记 |
