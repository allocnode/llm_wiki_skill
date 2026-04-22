# LLM Wiki Programming

> A Skill-based system for building persistent programming knowledge bases using LLMs.
>
> Inspired by [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

## What is this?

LLM Wiki Programming helps you build a **persistent, structured, compounding** knowledge base for your programming projects.

Instead of rediscovering knowledge every time you ask a question (like RAG), this system **incrementally builds and maintains** a Markdown wiki as you work. The wiki grows richer with every commit, document, and query.

### Key Features

- **Pure Markdown** — No databases, human-readable, git-tracked
- **Skill-driven** — Agent uses SOPs (Standard Operating Procedures) to do all the work
- **Collaborative confirmation** — Agent proposes, you guide and confirm
- **No custom MCP required** — Uses Agent's built-in tools (bash/read/write/edit)
- **Optional commands** — Trigger workflows via `/wiki-init`, `/wiki-ingest`, etc.

## Architecture

```
User
  ↓
Agent (LLM)
  ├─ Loads Skill (llm_wiki_skill)
  ├─ Reads wiki-schema.md (templates & conventions)
  ├─ Detects context → triggers SOP workflow
  ├─ Uses bash/read/write/edit to execute
  └─ Collaborates with user (propose → feedback → confirm)
       ↓
    ~/.llm-wiki/ (Markdown files, git-tracked)
```

### Three Layers

1. **Raw Sources** — Your project code (immutable, Agent reads only)
2. **The Wiki** — `~/.llm-wiki/` (Agent-generated, maintained)
3. **Schema** — This Skill (defines structure, templates, workflows)

## Installation

### Method 1: Manual Installation (Any Platform)

1. Copy the `llm_wiki_skill/` skill directory to your agent's skills folder:
   - **OpenCode**: `~/.config/opencode/skills/`
   - **Claude Code**: Reference in `CLAUDE.md`
   - **Other agents**: Follow your agent's skill loading mechanism

2. The Skill will be automatically loaded when the agent detects wiki-related tasks.

### Method 2: OpenCode Auto-Setup (Recommended for OpenCode Users)

For **OpenCode** users, you can set up commands and skill automatically:

1. Copy the entire repository to your project or config directory:
   ```bash
   git clone <repo-url> ~/.config/opencode/skills/llm_wiki_skill
   ```

2. **Enable commands** by copying the command files:
   ```bash
   # For global access (all projects)
   mkdir -p ~/.config/opencode/commands
   cp ~/.config/opencode/skills/llm_wiki_skill/.opencode/commands/*.md ~/.config/opencode/commands/

   # Or for a specific project only
   mkdir -p .opencode/commands
   cp ~/.config/opencode/skills/llm_wiki_skill/.opencode/commands/*.md .opencode/commands/
   ```

3. **Restart OpenCode** or reload configuration.

4. **Verify** — Type `/` in the TUI, you should see:
   - `/wiki-init` — Initialize project wiki
   - `/wiki-ingest` — Record recent changes
   - `/wiki-query` — Ask technical questions
   - `/wiki-file` — Ingest external documents
   - `/wiki-lint` — Check wiki health

### Available Commands

| Command | Description | Triggers SOP |
|---------|-------------|--------------|
| `/wiki-init` | Initialize wiki for current project | `init-project-sop` |
| `/wiki-ingest` | Record latest commit changes | `ingest-changes-sop` |
| `/wiki-query <question>` | Query the wiki | `query-sop` |
| `/wiki-file <path>` | Ingest external document | `ingest-file-sop` |
| `/wiki-lint` | Run health check | `lint-sop` |

## How It Works

### 1. Project Initialization

```
User: "Help me understand this project"
  ↓
Agent loads Skill → reads wiki-schema.md
  ↓
Agent scans project structure (find, read package.json, etc.)
  ↓
Agent proposes wiki structure
  ↓
User provides feedback ("focus on auth", "skip deprecated")
  ↓
Agent generates pages → writes to ~/.llm-wiki/projects/{name}/
  ↓
Git commit → update index
```

### 2. Recording Changes

```
User: git commit -m "feat(auth): migrate to JWT"
  ↓
Agent detects commit (soft event)
  ↓
Agent analyzes diff → identifies affected entities
  ↓
Agent proposes updates
  ↓
User confirms
  ↓
Agent updates entity pages → creates change record
  ↓
Git commit
```

### 3. Asking Questions

```
User: "How does auth work?"
  ↓
Agent searches wiki pages
  ↓
Agent reads relevant entities
  ↓
Agent synthesizes answer with citations
  ↓
(Optional) Agent asks: "Save this analysis to wiki?"
```

## Directory Structure

```
llm_wiki_skill/           # Skill directory
├── SKILL.md                    # Main skill (triggers, confirmation, delegation)
└── references/
    ├── wiki-schema.md          # Templates & conventions (read first)
    ├── init-project-sop.md     # Initialize project wiki
    ├── ingest-changes-sop.md   # Record code changes
    ├── ingest-file-sop.md      # Ingest external documents
    ├── query-sop.md            # Query & answer
    └── lint-sop.md             # Health check

.opencode/commands/             # OpenCode command files
├── wiki-init.md
├── wiki-ingest.md
├── wiki-query.md
├── wiki-file.md
└── wiki-lint.md
```

## Wiki Storage Structure

```
~/.llm-wiki/
├── .git/                           # Git version control
├── .schema/
│   └── wiki-schema.md              # This schema file
├── global/                         # Cross-project knowledge
│   ├── index.md
│   ├── log.md
│   ├── concepts/                   # Concepts (microservices, etc.)
│   ├── patterns/                   # Design patterns
│   └── technologies/               # Tech stack knowledge
└── projects/
    └── {project-name}/
        ├── index.md                # Project catalog
        ├── log.md                  # Operation log
        ├── overview.md             # Project overview
        ├── architecture.md         # Architecture
        ├── entities/               # Code entities
        ├── api/                    # API docs
        ├── changes/                # Change history
        ├── sources/                # External docs
        ├── decisions/              # ADRs
        └── analyses/               # Query synthesis (answer filing)
```

## Core Principles

1. **Skill is the core** — Agent uses built-in tools, no custom MCP needed
2. **Thin commands** — Commands only trigger SOPs, don't duplicate logic
3. **Collaborative confirmation** — Agent proposes, user guides, then writes
4. **Source attribution** — Every entity page tracks its source code + commit
5. **Git tracked** — All changes versioned, recoverable, auditable
6. **Compounding** — Insights from queries can be filed back into the wiki

## Platform Support

| Platform | Skill Loading | Command Support | Status |
|----------|--------------|-----------------|--------|
| **OpenCode** | `~/.config/opencode/skills/` | `.opencode/commands/*.md` | ✅ Fully supported |
| **Claude Code** | `CLAUDE.md` reference | Slash commands in CLAUDE.md | ✅ Supported |
| **Cline** | Settings configuration | VS Code extension settings | ✅ Supported |
| **Other Agents** | Follow agent's skill mechanism | Varies | ⚠️ Adapt needed |

## When to Add MCP (Optional)

This system works without any custom MCP. However, as your wiki grows:

| Scale | Pain Point | Optional MCP |
|-------|-----------|--------------|
| < 100 pages | None | Not needed |
| 100-500 pages | Search becomes slow | `wiki_search` |
| > 500 pages | Health check is tedious | `wiki_check_health` |

## Contributing

This is a pattern, not a rigid framework. Adapt it to your workflow:

- Modify templates in `wiki-schema.md`
- Add new SOPs for your domain
- Customize confirmation prompts
- Add new command shortcuts

## License

MIT

## Acknowledgments

- [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) for the original LLM Wiki idea
- [Model Context Protocol](https://modelcontextprotocol.io/) for the tool ecosystem
