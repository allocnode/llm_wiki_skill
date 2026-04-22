# LLM Wiki

> A Skill-based system for building persistent programming knowledge bases using LLMs. Inspired by [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

## What It Does

Instead of rediscovering knowledge every time you ask a question (like RAG), this system **incrementally builds and maintains** a Markdown wiki as you work. The Agent queries the wiki first before answering any project question — no training-data hallucinations.

## Key Features

- **Mandatory Skill loading** — Agent MUST load this Skill for all wiki operations. No bypassing.
- **Wiki-first queries** — Agent MUST query the wiki before answering. No training-data hallucinations.
- **Language consistency** — One language per wiki (user-confirmed at init).
- **Pure Markdown** — No databases, human-readable, git-tracked.
- **Collaborative confirmation** — Agent proposes, you guide, then writes.

## Installation

### OpenCode (Recommended)

```bash
# 1. Install the skill
git clone <repo-url> ~/.config/opencode/skills/llm_wiki_skill

# 2. Enable commands (global)
mkdir -p ~/.config/opencode/commands
cp ~/.config/opencode/skills/llm_wiki_skill/.opencode/commands/*.md ~/.config/opencode/commands/

# 3. Restart OpenCode
```

### Other Agents

Copy the `llm_wiki_skill/` directory to your agent's skills folder and follow its skill loading mechanism.

## Commands

| Command | Description |
|---------|-------------|
| `/wiki-init` | Initialize wiki for current project |
| `/wiki-ingest` | Record latest commit changes |
| `/wiki-query <question>` | Query the wiki |
| `/wiki-file <path>` | Ingest external document |
| `/wiki-lint` | Run health check |

## How It Works

```
User asks a question
  ↓
Agent loads Skill → queries wiki first (mandatory)
  ↓
Wiki has answer → answer with citations
Wiki has no answer → offer to initialize/update
```

## Repository Structure

```
llm_wiki_skill/                 # Skill directory
├── SKILL.md                    # Main skill
└── references/
    ├── wiki-schema.md          # Templates & conventions
    ├── init-project-sop.md     # Initialize project
    ├── ingest-changes-sop.md   # Record changes
    ├── ingest-file-sop.md      # Ingest documents
    ├── query-sop.md            # Query & answer
    └── lint-sop.md             # Health check

.opencode/commands/             # OpenCode commands
├── wiki-init.md
├── wiki-ingest.md
├── wiki-query.md
├── wiki-file.md
└── wiki-lint.md
```

## Wiki Storage

```
~/.llm-wiki/
├── global/                     # Cross-project knowledge
└── projects/{name}/
    ├── index.md                # Catalog
    ├── overview.md             # Overview
    ├── architecture.md         # Architecture
    ├── entities/               # Code entities
    ├── changes/                # Change history
    ├── sources/                # External docs
    ├── decisions/              # ADRs
    └── analyses/               # Query synthesis
```

## Platform Support

| Platform | Skill | Commands | Status |
|----------|-------|----------|--------|
| **OpenCode** | `~/.config/opencode/skills/` | `.opencode/commands/*.md` | ✅ Full |
| **Claude Code** | `CLAUDE.md` | Slash commands | ✅ Yes |
| **Cline** | Settings | Extension settings | ✅ Yes |
| **Others** | Varies | Varies | ⚠️ Adapt |

## Core Principles

1. **Mandatory** — Agent MUST load this Skill for wiki operations.
2. **Wiki-first** — Query the wiki before answering project questions.
3. **Consistent** — One language per wiki.
4. **Attributed** — Every entity tracks its source code + commit.
5. **Versioned** — All changes git-tracked.
6. **Compounding** — Query insights filed back into the wiki.

## License

MIT

## Acknowledgments

- [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) for the original LLM Wiki idea
