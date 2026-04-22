# LLM Wiki Programming — 设计文档

> **Version**: 0.3.1-draft  
> **Status**: Design Phase — 极简核心版（已修复格式）  
> **目标**: 构建面向程序开发的 LLM Wiki 知识库系统，核心是 Skill + Agent，MCP 按需增强。

---

## 1. 背景与目标

### 1.1 问题背景

在程序开发过程中，以下场景反复出现：

- **项目熟悉**：接手新项目时，需要大量时间阅读代码、文档、理解架构。每次都要从零开始。
- **知识丢失**：代码改动后，设计意图、决策原因逐渐模糊。新成员难以理解"为什么这样设计"。
- **信息分散**：设计文档、README、代码注释、会议纪要在不同位置，查询困难。
- **上下文断裂**：Agent（OpenCode / Claude / Cline）每次对话都是独立的，无法积累项目知识。这次对话的理解，下次还要重新来过。

### 1.2 核心目标

构建一个**持久化、结构化、可增长**的编程知识库，让 Agent 能够：

1. **自动积累**：在开发过程中自动（或半自动）记录项目知识
2. **结构化存储**：以 Markdown Wiki 形式组织，支持交叉引用和快速导航
3. **智能查询**：基于积累的知识回答技术问题，带引用来源
4. **健康维护**：定期检查知识库质量，发现矛盾、遗漏和过时内容

### 1.3 设计原则

| 原则 | 说明 |
|------|------|
| **纯 Markdown 存储** | 不引入数据库，降低依赖和复杂度。人类可直接阅读。 |
| **轻量确认机制** | 协作式确认：Agent 生成 → 用户反馈 → Agent 调整 → 最终写入。用户始终参与。 |
| **跨平台通用** | 不绑定特定 Agent，核心是 Skill，Agent 自带工具完成所有操作 |
| **项目+全局层** | 支持多项目隔离，同时维护全局共享知识层（概念、模式、技术栈） |
| **软事件驱动** | 通过 Skill 指导 LLM 自主判断触发时机，不需要硬编码钩子 |
| **不造轮子** | Agent 已有能力（bash/read/write/edit）直接完成，初期不需要自定义 MCP |
| **子 Agent 可选** | 简单任务主 Agent 自己做；复杂任务可委派子 Agent，避免上下文膨胀 |
| **极简起步** | 对齐 Karpathy 模式：一个 Schema + 一个对话会话就能工作 |

---

## 2. 角色分工

### 2.1 三个角色

**用户**：提出问题、提供反馈、确认操作。

**Agent（主 Agent / 子 Agent）**：加载 Skill，理解工作流，使用自有工具（bash/read/write/edit）执行操作，与用户协作迭代。

**Skill**：告诉 Agent "这是什么系统"、"何时做什么"、"按什么步骤"、"模板长什么样"。不执行，只指导。

```
用户
  ↓
主 Agent（LLM — 大脑）
  ├─ 加载 Skill（了解工作流、模板、约定）
  ├─ 自主判断触发时机（软事件）
  ├─ 执行工作流：用自己的工具（bash/read/write/edit）干活
  ├─ 复杂任务 → 委派子 Agent（可选）
  └─ 与用户协作迭代（确认 → 调整 → 再确认）
       ↓（需要时）
    子 Agent（LLM — 执行者）
       ├─ 加载分支 SOP
       ├─ 用自己的工具干活
       └─ 返回结果给主 Agent
```

### 2.2 各角色的边界

| 角色 | 能做什么 | 不能做什么 | 工具/能力 |
|------|---------|-----------|-----------|
| **Agent** | 思考、理解、总结、归纳、决策、执行文件操作 | 无 | bash, read, write, edit, glob, webfetch |
| **子 Agent**（可选） | 专注执行单一复杂任务 | 不决定"是否执行" | 同 Agent |
| **Skill** | 告诉 Agent "何时做什么"、"按什么步骤"、"模板长什么样" | 不执行，只指导 | SKILL.md + references/ |
| **MCP**（可选） | 确定性机械操作：搜索、死链检查 | **绝不思考** | 初期不需要 |

### 2.3 核心认知

**Skill 是核心，Agent 用自有工具就能工作。**

这是 Karpathy 原始模式：
- 一份 Schema（AGENTS.md / CLAUDE.md）告诉 LLM 结构和工作流
- LLM 在一个对话会话中用 read/write/edit/bash 完成所有操作
- 不需要额外工具、不需要子 Agent、不需要 MCP

我们的增强：
- 将 Schema 扩展为 Skill 体系（Main Skill + 分支 SOP），让工作流更清晰
- 保留子 Agent 可选能力，处理复杂任务时不干扰主 Agent
- MCP 作为**未来可选增强**，初期完全不需要

---

## 3. 存储设计

### 3.1 目录结构

```
~/.llm-wiki/
├── .git/                           # Git 版本控制（Agent 用 bash 操作）
│
├── .schema/
│   └── wiki-schema.md              # 页面模板、目录结构、命名约定
│                                   # Skill 加载时首先读取
│
├── global/                         # 全局共享知识层
│   ├── index.md                    # 全局目录（所有概念、模式、技术、项目索引）
│   ├── log.md                      # 全局操作日志
│   ├── concepts/                   # 跨项目概念（如"微服务"、"事件溯源"）
│   │   └── README.md
│   ├── patterns/                   # 设计模式与最佳实践
│   │   └── README.md
│   ├── technologies/               # 技术栈知识（React, Go, PostgreSQL...）
│   │   └── README.md
│   └── projects-index.md           # 项目索引与快速链接
│
└── projects/                       # 项目层（每个项目独立命名空间）
    └── {project-name}/             # 项目标识名（目录名）
        ├── index.md                # 项目目录（页面列表 + 快速导航）
        ├── log.md                  # 项目操作日志（按时间线）
        ├── overview.md             # 项目概览（目标、技术栈、团队、关键链接）
        ├── architecture.md         # 架构说明（模块关系、数据流、部署图）
        ├── decisions/              # ADR（架构决策记录）
        │   ├── index.md
        │   └── {date}-{decision-title}.md
        ├── entities/               # 代码实体（组件、类、模块、服务）
        │   ├── index.md            # 实体目录
        │   └── {entity-name}.md    # 单个实体详情
        ├── api/                    # API 接口文档
        │   ├── index.md
        │   └── {api-group}.md
        ├── changes/                # 变更历史（按 commit 或时间）
        │   └── {YYYY-MM-DD}-{short-desc}.md
        ├── sources/                # 外部资料摘要（设计文档、RFC 等）
        │   └── {source-title}.md
        └── analyses/               # 查询产生的分析/综合页面（答案回存）
            └── {analysis-title}.md
```

### 3.2 命名规范

| 项目 | 规范 | 示例 |
|------|------|------|
| 项目目录名 | kebab-case，取自项目根目录名或用户指定 | my-api-service, web-dashboard |
| 实体文件名 | kebab-case | auth-service.md, user-repository.md |
| 决策文件名 | YYYY-MM-DD-{kebab-title}.md | 2026-04-22-adopt-kafka.md |
| 变更文件名 | YYYY-MM-DD-{short-desc}.md | 2026-04-22-auth-jwt-refactor.md |
| Wiki 链接 | [[page-name]] 或相对路径 [text](path.md) | [[AuthService]], [API Design](../api/auth.md) |

### 3.3 Git 版本控制

- **初始化**：Agent 用 bash 执行 git init
- **提交**：每次写入后，Agent 用 bash 执行：

  cd ~/.llm-wiki/ && git add -A && git commit -m "[时间] 操作 | 摘要"

- **提交格式**：[2026-04-22 14:30] ingest-changes | my-app: auth JWT refactor
- **提交信息规范**：
  - 开头是 [YYYY-MM-DD HH:MM]
  - 接着是操作类型：init, ingest-changes, ingest-file, query, lint, update
  - 竖线后是项目名和简短摘要
- **用途**：变更追溯、故障恢复、对比历史

---

## 4. Skill 体系设计（核心）

### 4.1 体系结构

```
llm_wiki_skill/                   # 主 Skill（全局指导）
├── SKILL.md                            # 系统介绍、触发规则、工作流 SOP、协作确认机制
└── references/
    ├── wiki-schema.md                  # 页面模板、目录结构、命名约定
    ├── init-project-sop.md             # 项目初始化 SOP
    ├── ingest-changes-sop.md           # 变更灌入 SOP
    ├── ingest-file-sop.md              # 文件灌入 SOP
    ├── query-sop.md                    # 查询 SOP
    └── lint-sop.md                     # 健康检查 SOP
```

### 4.2 Skill 与 Karpathy Schema 的关系

Karpathy 用一个文件（AGENTS.md / CLAUDE.md）定义一切。我们将这个文件扩展为 Skill 体系：

| Karpathy 的 Schema | 我们的 Skill |
|-------------------|-------------|
| 系统介绍 | Main Skill 的 SKILL.md 开头 |
| 目录结构约定 | references/wiki-schema.md |
| 页面模板 | references/wiki-schema.md |
| 工作流步骤 | references/*-sop.md |
| 触发规则 | Main Skill 中的软事件表 |

**优势**：结构更清晰，LLM 按需加载对应 SOP，避免单文件过长。

### 4.3 Main Skill (llm_wiki_skill)

**加载流程**：

Agent 加载 llm_wiki_skill
    ↓
Step 1: 读取 references/wiki-schema.md（了解模板和约定）
    ↓
Step 2: 理解触发规则和工作流 SOP
    ↓
Step 3: 等待用户输入或检测软事件

**触发规则总表**：

| 用户语境 / 软事件 | 匹配信号 | 操作 |
|----------------|---------|------|
| 项目初始化 | "理解这个项目"、"熟悉代码库"、"分析这个项目"、询问架构但 wiki 不存在 | 执行 init-project-sop |
| 变更灌入 | 对话中出现 git commit 输出、用户描述代码变更（"我修复了X"、"重构了Y"） | 询问确认 → 执行 ingest-changes-sop |
| 文件灌入 | 用户上传/引用文档并说"记录"、"存入 wiki"、"整理这份文档" | 询问确认 → 执行 ingest-file-sop |
| 技术查询 | "X 怎么工作"、"为什么这样设计"、"X 和 Y 什么关系" | 执行 query-sop |
| 健康检查 | "检查 wiki"、"知识库健康度"、"发现矛盾"、"清理 wiki" | 执行 lint-sop |

**协作确认机制**：

当 Agent 完成分析并准备写入 wiki 时，不要直接写入。按以下流程：

1. **展示摘要**：向用户展示即将写入的内容摘要
   - 如果是初始化：展示 overview.md 的核心内容、将生成的实体列表
   - 如果是变更：展示变更影响分析、将更新的页面列表
   - 如果是文件灌入：展示提取的核心要点、将创建/更新的页面
   - 如果是查询回存：展示分析摘要、建议的保存路径

2. **请求反馈**：
   "以上内容是否准确？有哪些需要调整或补充的？"

3. **迭代调整**：
   - 如果用户提出修改意见（如"多关注错误处理部分"、"这个实体描述不准确"）
   - Agent 根据反馈重新生成或调整内容
   - 再次展示，直到用户满意

4. **最终确认**：
   - 用户明确说"确认"、"写入"、"没问题"
   - 或用户没有提出修改意见（视为满意）

5. **执行写入**：
   - 使用 write/edit 工具写入文件
   - 使用 bash 执行 git commit
   - 追加 log.md

重要：
- 不要跳过步骤 2-3，即使用户看起来很满意
- 给用户机会指导重点（对齐 Karpathy "stay involved" 理念）
- 如果用户说"跳过"或"不用了"，则记录到 log.md（"用户跳过了 ingest"）

**子 Agent 委派规范（可选）**：

当任务复杂且可能干扰主 Agent 当前主线工作时，可以委派子 Agent：

1. **判断标准**：
   - 需要读取 > 10 个文件进行分析 → 委派
   - 需要生成 > 5 个新页面 → 委派
   - 当前主 Agent 有更重要的对话主线 → 委派

2. **委派方式**：
   - 告诉子 Agent 要执行哪个 SOP 文件
   - 提供必要的上下文（项目路径、变更信息等）
   - 明确交付物

3. **审查**：
   - 子 Agent 完成后，主 Agent 审查内容质量
   - 检查是否遵循 wiki-schema.md 的模板和约定
   - 向用户汇报结果

---

### 4.4 分支 SOP 文件

每个 SOP 是一份详细的工作流说明书，告诉 Agent 如何完成一项具体任务。**教 Agent 用自己的工具干活**。

#### init-project-sop.md — 项目初始化

**目标**：扫描项目结构，生成 overview、architecture、entities 页面。

**步骤**：

**Step 1: 扫描项目结构**

使用 bash tool：

find . -type f -not -path "*/node_modules/*" -not -path "*/.git/*" -not -path "*/vendor/*" -not -path "*/dist/*" -not -path "*/build/*" -not -path "*/.next/*" | sort

**Step 2: 读取关键文件**

使用 read tool：
- README.md
- package.json / go.mod / Cargo.toml / pyproject.toml / pom.xml
- src/ 或 app/ 目录的一级结构

**Step 3: 分析并识别实体**

基于读取内容分析：
1. 项目类型和技术栈
2. 主要模块和目录结构
3. 关键代码实体（组件、服务、类、模块）

**Step 4: 生成 Wiki 页面**

按照 wiki-schema.md 中的模板：

overview.md：
- 项目目标和范围
- 技术栈列表
- 目录结构说明
- 团队信息（如有）

architecture.md：
- 系统整体描述
- 模块关系（文本图或 Mermaid 图）
- 数据流描述
- 关键技术选型

entities/*.md：
- 每个关键实体一个页面
- 包含 Source 字段（指向源码文件路径）
- 包含职责、依赖、接口、变更历史

**Step 5: 写入文件**

使用 write tool 写入到 ~/.llm-wiki/projects/{projectName}/

**Step 6: Git 提交**

使用 bash tool：

cd ~/.llm-wiki/
git init 2>/dev/null || true
git add -A
git commit -m "[$(date +%Y-%m-%d\ %H:%M)] init | {projectName}"

**Step 7: 更新全局索引**

使用 read tool 读取 global/projects-index.md
使用 edit tool 添加新项目链接
使用 bash tool：

cd ~/.llm-wiki/ && git add -A && git commit -m "[$(date +%Y-%m-%d\ %H:%M)] update-index | add {projectName}"

---

#### ingest-changes-sop.md — 变更灌入

**目标**：分析代码变更，更新相关实体页面。

**步骤**：

**Step 1: 获取变更信息**

使用 bash tool：

git log -1 --pretty=format:"%H|%s|%b"
git diff HEAD~1 --stat
git diff HEAD~1

**Step 2: 分析变更影响**

1. 解析 commit message（如果是 conventional commit，提取 type 和 scope）
2. 识别变更文件对应的实体（文件 → 模块/组件映射）
3. 判断变更性质：
   - 接口变更 → 更新实体的 Interface 部分
   - 新增实体 → 创建新的 entity 页面
   - 架构调整 → 更新 architecture.md
   - 修复 bug → 在变更历史中标注

**Step 3: 读取并更新实体页面**

使用 read tool 读取受影响的 entities/*.md
使用 edit tool 更新：
- 在 Change History 段落追加变更记录
- 如果有接口变更，更新 Interface 部分
- 如果有新增依赖，更新 Dependencies

**Step 4: 创建变更记录页**

使用 write tool 在 changes/ 下创建新页面：

# {Change Title}

> **Date**: {YYYY-MM-DD}
> **Commit**: `{hash}`
> **Type**: {feat|fix|refactor|docs}
> **Scope**: {scope}

## Summary
{一句话摘要}

## Changes
- `{file1}` — {做了什么}
- `{file2}` — {做了什么}

## Affected Entities
- [[EntityA]] — {影响说明}
- [[EntityB]] — {影响说明}

## Motivation
{为什么做这个变更}

**Step 5: 更新 index.md**

使用 read + edit 更新项目的 index.md

**Step 6: Git 提交**

使用 bash tool：

cd ~/.llm-wiki/ && git add -A && git commit -m "[{时间}] ingest-changes | {project}: {summary}"

---

#### query-sop.md — 技术查询

**目标**：基于 Wiki 知识回答用户问题，支持答案回存。

**步骤**：

**Step 1: 搜索相关页面**

读取 ~/.llm-wiki/projects/{project}/index.md 定位相关页面
如果 index.md 信息不足，用 bash grep 搜索关键词：

grep -r "关键词" ~/.llm-wiki/projects/{project}/ --include="*.md"

**Step 2: 读取相关页面**

使用 read tool 逐个读取相关页面
如果需要，也读取 global/ 层的相关概念/技术页面

**Step 3: 综合答案**

基于读取的内容，综合回答用户问题
答案必须包含引用来源：

> 来源：[[entities/AuthService]], [[api/auth]]

**Step 4: 判断是否需要回存**

如果答案包含新的综合、分析或比较（而非简单的事实复述）：
- 向用户询问："这次分析涉及多个实体和跨项目的综合，是否保存到知识库以便将来复用？"
- 如果用户确认，使用 write tool 保存到 analyses/ 目录
- 使用 edit tool 更新 index.md 添加链接
- 使用 bash 执行 git commit

**Step 5: 记录日志**

使用 edit tool 在 log.md 追加查询记录

---

#### ingest-file-sop.md — 文件灌入

**目标**：将外部文档（设计文档、RFC、文章）灌入知识库。

**步骤**：

**Step 1: 读取文件**

使用 read tool 读取文件内容

**Step 2: 提取摘要**

分析文件内容，提取：
- 标题和核心要点
- 涉及的技术/概念
- 涉及的实体

**Step 3: 生成 source 页面**

使用 write tool 在 sources/ 下创建摘要页面
包含原始文件路径引用

**Step 4: 更新相关实体**

使用 read + edit 在相关 entity 页面添加引用

**Step 5: 更新 index.md 并提交**

使用 edit + bash(git)

---

#### lint-sop.md — 健康检查

**目标**：检查知识库质量并生成报告。

**步骤**：

**Step 1: 扫描页面**

使用 bash tool 列出所有页面：

find ~/.llm-wiki/projects/{project} ~/.llm-wiki/global -name "*.md" | sort

**Step 2: 检查死链**

使用 bash grep 提取所有 [[link]] 和 [text](path)
验证每个目标是否存在

**Step 3: 检查孤立页**

统计每个页面的入站链接数
找出无入站链接的页面

**Step 4: 分析矛盾（Agent 智能）**

读取 architecture.md 和关键 entity 页面
检查描述是否一致

**Step 5: 生成报告**

向用户汇报问题列表（按严重度分组：error / warning）
对每个问题，提供修复建议

**Step 6: 修复（可选）**

根据用户确认，使用 edit tool 修复死链、合并孤立页等
使用 bash(git) 提交修复

---

## 5. 命令行扩展（可选）

### 5.1 设计原则

**命令是薄的触发器，不重复 SOP 逻辑。**

命令行扩展的目的是让用户可以通过快捷键（如 `/wiki-init`）快速触发某个工作流，但命令本身不应该包含任何业务逻辑。

**正确的设计**：
- 命令文件（如 OpenCode 的 `.opencode/commands/wiki-init.md`）只负责：
  1. 接收用户参数（如项目名）
  2. 告诉 Agent："请加载 `init-project-sop.md` 并执行"
  3. 可选：传递一些上下文（如当前目录路径）
- 真正的逻辑完全由 Skill/SOP 定义，命令只是入口

**错误的设计（避免）**：
- 在命令文件里写一遍 SOP 步骤（导致两边维护）
- 在命令文件里定义页面模板（模板应该在 `wiki-schema.md` 里）

### 5.2 命令与 Skill/SOP 的关系

```
用户输入 /wiki-init
    ↓
命令文件被触发（薄的入口）
    ↓
Agent 读取命令文件 → 知道要执行哪个 SOP
    ↓
Agent 加载对应的 SOP 文件（如 references/init-project-sop.md）
    ↓
Agent 按 SOP 执行（使用自有工具 bash/read/write/edit）
    ↓
完成
```

**关键原则**：
- 命令是「入口」
- Skill/SOP 是「逻辑」
- 两者分离，命令不重复 Skill 的内容

### 5.3 OpenCode 命令配置示例

OpenCode 支持通过 Markdown 文件定义自定义命令，放置在 `.opencode/commands/` 目录下。

**命令文件示例**（`.opencode/commands/wiki-init.md`）：

```yaml
---
description: Initialize LLM Wiki for the current project
subtask: true
---
```

Please initialize the LLM Wiki for the current project by executing the `init-project-sop` workflow from the `llm_wiki_skill` skill.

1. First, read the skill reference file `~/.llm-wiki/.schema/wiki-schema.md` to understand templates and conventions.
2. Then follow the `references/init-project-sop.md` from the `llm_wiki_skill` skill step by step.
3. The project path is the current working directory.
4. Ask the user for confirmation before writing any files.

**命令文件示例**（`.opencode/commands/wiki-ingest.md`）：

```yaml
---
description: Record recent code changes into LLM Wiki
subtask: true
---
```

Please record the recent code changes into the LLM Wiki by executing the `ingest-changes-sop` workflow from the `llm_wiki_skill` skill.

1. Read `references/ingest-changes-sop.md` from the `llm_wiki_skill` skill.
2. Follow the SOP to analyze the latest commit and update wiki pages.
3. Ask the user for confirmation before making any changes.

**注意**：
- 命令文件的 `template`（内容部分）**不重复 SOP 步骤**
- 命令文件只是告诉 Agent "去加载哪个 Skill 的哪个 SOP"
- 所有业务逻辑、模板、约定都在 Skill 里定义
- 如果 SOP 更新了，命令文件不需要修改

### 5.4 其他平台的命令扩展

| 平台 | 命令配置方式 | 命令文件位置 |
|------|-------------|-------------|
| **OpenCode** | Markdown 文件 + YAML frontmatter | `.opencode/commands/*.md` |
| **Claude Code** | `CLAUDE.md` 中定义 slash commands | `CLAUDE.md` 文件 |
| **Cline** | VS Code 插件配置 | 设置面板或配置文件 |

**设计原则跨平台通用**：无论哪个平台，命令都只作为 Skill 的入口，不重复 SOP 逻辑。

---

## 6. MCP（可选增强）

### 6.1 设计原则

**初期完全不需要 MCP。** Agent 用自有工具（bash/read/write/edit）即可完成所有操作。

**当 wiki 规模增长后，可选添加 MCP**：

| 规模 | 痛点 | 可选 MCP Tool |
|------|------|--------------|
| < 100 页 | 无 | 不需要 |
| 100-500 页 | index.md 太长，搜索 token 消耗大 | wiki_search |
| > 500 页 | 跨文件死链检查太繁琐 | wiki_check_health |

### 6.2 可能的 MCP Tools（未来）

**wiki_search（可选）**

场景：wiki 页面增多，Agent 用 grep 搜索效果差。

输入：{ query, projectName?, limit, searchContent }
输出：{ results: [{project, path, title, summary, matchScore, matchContext?}] }

保留理由：Agent 自己做需要 glob 找到所有文件 -> 循环 read 每个文件 -> 正则匹配 -> 排序。步骤多、token 消耗大。

**wiki_check_health（可选）**

场景：跨文件死链/孤立页检查太繁琐。

输入：{ projectName? }
输出：{ issues: [{type, severity, page, description, details?}], stats: {...} }

保留理由：Agent 自己做需要读取几十个文件、提取所有链接、验证目标是否存在、统计入站链接数。步骤极多。

---

## 7. 核心场景数据流

### 场景 1：初始化新项目
```
用户: "帮我理解一下这个项目"
    ↓
Agent 加载 llm_wiki_skill Skill
    ↓
读取 wiki-schema.md 了解模板和约定
    ↓
Skill: "用户要求理解项目 → 检查是否已初始化"
    ↓
Agent 用 bash ls 检查 ~/.llm-wiki/projects/ 是否存在该项目
    ↓
未初始化 → Skill: "执行 init-project-sop"
    ↓
Agent 按 SOP 执行：
  Step 1: bash find 扫描目录
  Step 2: read README.md, package.json
  Step 3: 分析（NestJS 电商后端，模块有...）
    ↓
**协作确认**：
  Agent: "我准备生成以下内容：
    - overview.md：项目目标是... 技术栈是...
    - architecture.md：主要模块有 api/, services/, models/
    - entities/：将生成 8 个实体页面（UserService, AuthService...）
    是否准确？有哪些需要调整？"
  用户: "没问题，但 AuthService 已经废弃了，不要生成它"
  Agent: "好的，移除 AuthService，生成 7 个实体页面"
    ↓
  用户: "确认"
    ↓
  Step 4-7: write 写入文件 → bash git commit → edit index.md
    ↓
Agent: "已创建 my-app 知识库，包含 9 个页面。建议先阅读 overview。"
```
### 场景 2：Commit 后记录变更
```
用户: git commit -m "feat(auth): migrate to JWT"
    ↓
Agent 检测到 git commit 输出（软事件）
    ↓
Skill: "匹配到变更灌入信号"
    ↓
Agent 按 ingest-changes-sop 执行：
  Step 1: bash git log + git diff 获取变更
  Step 2: 分析（conventional commit → feat/auth，影响 AuthService）
    ↓
**协作确认**：
  Agent: "检测到变更：feat(auth): migrate to JWT
    影响分析：
    - auth.go：JWT 验证逻辑变更
    - middleware.go：新增 JWT middleware
    - 将更新 entities/AuthService.md（追加变更历史）
    - 将创建 entities/AuthMiddleware.md（新实体）
    - 将创建 changes/2026-04-22-auth-jwt.md
    是否准确？需要补充什么？"
  用户: "middleware 的验证逻辑很重要，单独记录一下"
  Agent: "好的，在 AuthMiddleware 页面中增加 Verification Logic 段落"
    ↓
  用户: "确认"
    ↓
  Step 3-6: read + edit + write → bash git commit
    ↓
Agent: "已记录变更。更新了 AuthService，新增 AuthMiddleware。"
```
### 场景 3：技术查询（答案回存）
```
用户: "比较一下 AuthService 和 OAuthService 的设计差异"
    ↓
Agent 加载 query-sop
    ↓
Step 1: read index.md → 定位 entities/AuthService.md, entities/OAuthService.md
Step 2: read 读取这两个页面
    ↓
Step 3: Agent 综合答案：
  "AuthService 使用 JWT，负责内部 API 鉴权...
   OAuthService 使用 OAuth 2.0，负责第三方接入...
   关键差异：1) 协议不同 2) 使用场景不同..."
    ↓
Step 4: 判断需要回存（这是新的综合分析）
  Agent: "这次分析涉及多个实体的比较，是否保存到知识库以便将来复用？"
  用户: "保存"
    ↓
  Agent write → analyses/auth-vs-oauth.md
  Agent edit → index.md 添加链接
  Agent bash → git commit
```
---

## 8. 实施路线

### Phase 1: Skill 核心（2-3 天）

**目标**：编写完整的 Skill 体系，Agent 用自有工具就能工作。

**任务**：
- [ ] 编写 references/wiki-schema.md（页面模板、命名约定、目录结构）
- [ ] 编写 SKILL.md（系统介绍、触发规则、协作确认机制、委派指南）
- [ ] 编写 references/init-project-sop.md
- [ ] 编写 references/ingest-changes-sop.md
- [ ] 编写 references/ingest-file-sop.md
- [ ] 编写 references/query-sop.md
- [ ] 编写 references/lint-sop.md
- [ ] 安装到 Agent 中测试加载

**验收标准**：
- Skill 能在 Agent 中正确加载
- Agent 能根据 Skill 自主判断触发工作流
- 协作确认机制正常工作（展示 → 反馈 → 调整 → 确认）

### Phase 2: 实战验证（3-5 天）

**目标**：在真实项目中使用，收集反馈。

**任务**：
- [ ] 在 2-3 个真实项目中测试初始化流程
- [ ] 测试软事件触发（commit 后自动提示）
- [ ] 测试协作确认机制（展示摘要 → 用户反馈 → Agent 调整）
- [ ] 测试查询和答案回存
- [ ] 测试健康检查
- [ ] 根据实战反馈优化 SOP 和模板

### Phase 3: 可选 MCP 增强（未来）

**目标**：当 wiki 规模增大后，按需添加 MCP。

**触发条件**：
- 页面 > 100 个，index.md 搜索效率下降 → 添加 wiki_search
- 页面 > 500 个，死链检查太耗时 → 添加 wiki_check_health

---

## 9. 附录

### 8.1 术语表

| 术语 | 定义 |
|------|------|
| **Skill** | 封装工作流和领域知识的指令集，指导 LLM 行为 |
| **SOP** | Standard Operating Procedure，标准作业流程 |
| **Wiki** | 本系统中指 Markdown 文件组成的知识库 |
| **Entity** | 代码中的关键抽象（组件、服务、类、模块等） |
| **软事件** | LLM 根据语境自主判断触发，非硬编码事件监听 |
| **协作确认** | Agent 生成 → 用户反馈 → Agent 调整 → 最终写入 |
| **子 Agent** | 由主 Agent 创建的专门执行单一任务的 LLM 实例（可选） |
| **Source 溯源** | 在 wiki 页面中标注信息来源（源码文件 + commit） |

### 8.2 查看工具建议

| 工具 | 特点 | 推荐度 |
|------|------|--------|
| **Obsidian** | 原生支持 [[wikilink]]、Graph View（可视化页面关系）、Dataview 插件查询 frontmatter | 强烈推荐 |
| **VS Code** | 配合 Markdown 插件、GitLens 查看版本历史 | 推荐 |
| **Git 客户端** | 查看 ~/.llm-wiki/ 的 git log 和 diff，回溯每次变更 | 必须 |

**Obsidian 使用技巧**：
- 打开 ~/.llm-wiki/ 作为 Vault
- Graph View 可直观看到：哪些页面是 hub（连接多）、哪些是 orphan（孤立）
- 在 Settings → Files and links 中开启 "Use [[Wikilinks]]"

### 8.3 Git 查看示例

cd ~/.llm-wiki/
git log --oneline -10                    # 查看最近 10 次变更
git show HEAD:projects/my-app/entities/AuthService.md   # 查看历史版本
git diff HEAD~1 -- projects/my-app/     # 查看最近一次 ingest 改了什么

### 8.4 参考资源

- [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — 原始灵感来源
- [MCP Specification](https://modelcontextprotocol.io/specification/latest) — 协议规范
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) — 开发 SDK
- [MCP Best Practices](https://modelcontextprotocol.io/docs/concepts/best-practices) — 最佳实践
- [Agent Skills Specification](https://agentskills.io/specification) — Skill 规范

---

## 变更日志

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-22 | 0.1.0 | 初始设计文档 |
| 2026-04-22 | 0.2.0 | 架构纠偏：明确角色分工、MCP 精简为 2 个 Tool、引入子 Agent 协作 |
| 2026-04-22 | 0.3.0 | 极简核心版：Skill 为核心、Agent 自有工具完成所有操作、MCP 变为可选增强、确认机制改为协作式迭代 |
| 2026-04-22 | 0.3.1 | 修复格式问题（移除嵌套代码块），补充 Git 查看示例、Obsidian 使用技巧、更详细的 SOP 步骤 |
| 2026-04-22 | 0.3.2 | 新增「命令行扩展」章节：设计命令作为 Skill 的薄入口，不重复 SOP 逻辑；提供 OpenCode 命令配置示例 |
