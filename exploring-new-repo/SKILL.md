---
name: exploring-new-repo
description: "Help users quickly understand and navigate an unfamiliar codebase. Use this skill whenever the user opens a new repository for the first time, asks 'what does this project do', 'help me understand this codebase', 'give me an overview', 'onboard me to this repo', 'explore this project', or expresses any desire to get oriented in unfamiliar code — even if they just say 'I'm new here' or 'where do I start'. Also trigger when the user asks about project architecture, directory structure, key modules, entry points, or how to set up the development environment for a project they clearly haven't worked in before."
---

# Exploring a New Repository

Help a new developer go from "I just cloned this repo" to "I understand the architecture and can start contributing" as quickly as possible. The key difference from a generic code summary: **adapt to the user's actual knowledge level**, not the AI's.

## How This Works

The process has three phases and two reactive modes:

1. **Phase 0: Calibration** — Understand the user's background before scanning anything
2. **Phase 1: Reconnaissance** — Scan the repo efficiently
3. **Phase 2: Synthesize** — Produce an onboarding document adapted to the user's level
4. **Phase 2.5: Terminology Checkpoint** — Extract unfamiliar concepts and explain them
5. **Deep-Dive Mode** (reactive) — On-demand module walkthroughs
6. **Concept Coach Mode** (reactive) — Explain concepts in context when the user asks

Always start with Phase 0 unless you already have strong signals about the user's background from the conversation.

---

## Phase 0: Calibration (Understand the User)

Before scanning any code, understand who you're writing for. This takes one question — don't turn it into an interrogation.

Ask something like:

> "在我开始分析这个项目之前，想快速了解一下你的背景，这样我能调整讲解的深度：
> - 你最熟悉的技术栈是什么？（比如前端/后端/ML/系统编程等）
> - 对这个项目的领域（[根据 README 粗判]）了解多少？"

Based on the response, mentally categorize the user's position on two axes:

| | Domain novice | Domain experienced |
|---|---|---|
| **Tech novice** | Explain everything: concepts + code patterns + domain terms | Explain code patterns, skip domain terms |
| **Tech experienced** | Skip code patterns, explain domain terms and project-specific jargon | Focus on project-specific conventions and architecture decisions only |

Carry this calibration through all subsequent phases. When in doubt, err toward explaining more — it's less costly than leaving the user confused.

---

## Phase 1: Reconnaissance (Gather Signals)

Scan the repo to collect raw signals. Do this efficiently — aim for 5-8 tool calls, not 20.

### Step 1: Read the README and project metadata

- Read `README.md` (or `README.rst`, etc.)
- Read config files that reveal the tech stack: `pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, `Makefile`, `CMakeLists.txt`, etc.
- Read `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, or similar contributor guides if they exist
- Check for `.github/workflows/` or `Jenkinsfile` to understand CI

### Step 2: Map the directory structure

- Use `list_dir` on the project root and key subdirectories (source root, tests, docs)
- Annotate what each top-level directory likely contains — don't just list names

### Step 3: Analyze code structure (use the best available tool)

**If codegraph is available** (check with `codegraph_status` — pass `projectPath` if needed):
- Use `codegraph_search` to find key classes and entry points
- Use `codegraph_callers`/`codegraph_callees` to trace important call chains
- Use `codegraph_explore` to read the source of 3-5 core symbols at once
- Always pass `projectPath` (the project root) in codegraph calls if the workspace has multiple projects

**If codegraph is not available**, fall back to:
- `grep_search` for common patterns: `def main`, `if __name__`, `app.`, `export default`, `func main`, entry-point patterns
- `view_file` on files that look like entry points or core modules
- If a `.codegraph/` directory doesn't exist, mention it to the user and offer to initialize it

### Step 4: Identify the development workflow

Look for:
- Build/install instructions (`Makefile`, `setup.py`, `pyproject.toml` scripts)
- Test commands and test structure
- Linting/formatting config (`.pre-commit-config.yaml`, `ruff.toml`, `.eslintrc`)
- Environment setup (`.env.example`, `docker-compose.yml`, `requirements.txt`)
- Git submodules (`.gitmodules`) — these often contain critical dependencies
- Native extensions (`csrc/`, `src/` with C++/CUDA files, `setup.py` with `ext_modules`)

---

## Phase 2: Synthesize the Onboarding Document

Produce a Markdown artifact with these sections. **Adapt the language and depth based on Phase 0 calibration.**

### Document Template

```markdown
# [Project Name] — Codebase Onboarding

## 30-Second Summary
[1-3 sentences: What is this project? What problem does it solve? Who is it for?]
[If the user is a domain novice, add 1-2 sentences of domain context:
e.g., "LLM 推理时，模型需要为每个 token 计算 Key-Value 对（KV Cache），这些数据占用大量 GPU 内存..."]

## Tech Stack
- **Language**: [e.g., Python 3.12]
- **Framework**: [e.g., integrates with vLLM/SGLang]
- **Build**: [e.g., pip + setuptools with C++/CUDA extensions]
- **Tests**: [e.g., pytest]
- **CI**: [e.g., GitHub Actions]

## Architecture Overview
[A mermaid diagram showing major components and their relationships.
Focus on the 3-5 most important modules and how data flows between them.]

## Repository Map
[Annotated directory tree. Only include meaningful directories — skip `__pycache__`, `node_modules`, `.git`, etc.
For each directory, add a brief description of what it contains and why it matters.]

## Core Modules
[For each key module (3-5 max):
- **What it does**: 1-2 sentences
- **Key classes/functions**: with file links
- **Depends on / Used by**: relationships to other modules]

## Entry Points
[Where does execution start? List:
- CLI entry points
- Main application entry
- API endpoints (if web service)
- Plugin/integration hooks]

## Development Workflow
### Setup
[Copy-pasteable commands to get from clone to running]

### Build & Test
[How to build, run tests, lint]

### Common Tasks
[How to add a feature, fix a bug, run specific tests]

## Key Conventions
[Project-specific patterns a contributor should know:
- Naming conventions
- Import ordering rules
- Error handling patterns
- Anything in CONTRIBUTING.md that's non-obvious]
```

### Writing Guidelines

- **Be specific, not generic.** "Uses a tiered storage architecture (GPU → CPU → disk → S3)" beats "Uses multiple storage backends."
- **Include file links.** Every class, function, or file you mention should be a clickable `[name](file:///path)` link.
- **Use mermaid diagrams.** An architecture diagram is worth a thousand words. Use `graph TD` or `flowchart LR` for component relationships.
- **Copy-pasteable commands.** Every command in "Development Workflow" should work if the user copies and pastes it.
- **Skip what's obvious.** Don't explain that `tests/` contains tests. Do explain that `tests/v1/` mirrors the `lmcache/v1/` structure and integration tests are in `tests/disagg/`.
- **Layer your explanations based on calibration.** For a domain novice encountering "KV cache tiling", don't just name it — briefly say what it is and why it matters. For someone experienced, just name it and move on.

---

## Phase 2.5: Terminology Checkpoint

After delivering the onboarding document, **don't just move on**. This is the critical step that bridges the gap between AI understanding and user understanding.

### How to do it

1. **Extract key terms** from the onboarding document you just generated. Look for:
   - Domain-specific jargon (e.g., "KV cache", "TTFT", "tiling", "prefill vs decode")
   - Architecture patterns the project uses (e.g., "serde", "disaggregation", "paged attention")
   - Abbreviations and acronyms (e.g., "GQA", "MQA", "TMA", "WGMMA")
   - Project-specific terminology (e.g., "MemoryObj", "CacheEngineKey", "chunk budget")

2. **Present them to the user** as an interactive checklist:

   > "这份概览中涉及了一些领域术语和项目概念。我列出来，你可以告诉我哪些不太熟悉，我来详细解释：
   >
   > **领域概念**: KV Cache, TTFT, Prefill vs Decode, Tiling, Paged Attention, ...
   > **架构模式**: Serde, PD Disaggregation, Tiered Storage, ...
   > **项目术语**: MemoryObj, CacheEngineKey, StorageBackendInterface, ...
   >
   > 哪些你想让我展开讲讲？"

3. **For each concept the user flags**, provide a layered explanation:

   **Level 1 — Intuition** (1-2 sentences, use analogy):
   > "KV Cache 就像是一本'读书笔记'——LLM 在处理之前的文字时已经做过一次计算，把结果（Key 和 Value）记下来，之后遇到新文字时就不用重新读整本书了。"

   **Level 2 — Technical** (how it works in this project):
   > "具体来说，Transformer 的每一层在做 attention 时会产生 Key 和 Value 张量。LMCache 把这些张量从 GPU 搬到 CPU/磁盘/远程存储，下次相同前缀的请求来时直接复用，跳过重复计算。"

   **Level 3 — In the code** (where to find it):
   > "在这个项目中，KV Cache 的存取逻辑在 [cache_engine.py](file:///path) 的 `CacheEngine` 类中。存储到不同介质的实现在 [storage_backend/](file:///path) 下。"

   Don't dump all three levels at once. Start with Level 1, and if the user asks "能再详细说说吗？", go to Level 2 and 3.

---

## Phase 3: Deep-Dive Mode

After delivering the onboarding document, the user may ask follow-up questions like:
- "How does module X work?"
- "Walk me through the request flow"
- "What would I need to change to add feature Y?"

For these, provide a **focused code walkthrough**:

1. Use codegraph (or file reading) to trace the relevant code path
2. Show key code snippets with annotations
3. Draw a sequence diagram (mermaid `sequenceDiagram`) if the flow involves multiple components
4. Identify extension points ("to add a new storage backend, implement this interface and register it here")

Keep deep-dives focused. Answer the specific question — don't regenerate the whole overview.

**Remember your calibration**: if the user is a domain novice, explain *why* the code does what it does, not just *what* it does. "This function tiles the attention computation into blocks because GPU SRAM is too small to hold the full attention matrix" is far more useful than "This function performs tiled attention computation."

---

## Adapting to Different Project Types

The template above is a starting point. Adapt based on what you find:

| Signal | Adaptation |
|--------|-----------|
| Monorepo (multiple `package.json` / `go.mod`) | Add a "Packages Overview" section listing each package's purpose |
| Web service (Flask/Express/etc.) | Add "API Endpoints" section, show route → handler mapping |
| Library/SDK | Focus on public API surface, show usage examples |
| CLI tool | Show command structure, subcommand hierarchy |
| ML/Data project | Highlight model architecture, training pipeline, data flow |
| Has design docs (`docs/design/`) | Reference them, don't duplicate their content |
| CUDA/C++ extensions (`csrc/`, `ext_modules`) | Map the Python binding → C++ → kernel layers separately; note per-GPU-architecture kernel variants |
| Git submodules (`.gitmodules`) | List submodule purposes; these are often large dependencies (CUTLASS, etc.) |

## Anti-Patterns to Avoid

- **Don't just dump `tree` output.** Annotate directories or don't include them.
- **Don't read every file.** Be strategic — README, config, entry points, and 3-5 core modules is enough.
- **Don't over-abstract.** "Handles data processing" is useless. "Compresses KV caches using CacheGen codec for storage/transmission" is useful.
- **Don't ignore existing docs.** If there's a `CONTRIBUTING.md` or `docs/` with good content, reference it instead of rewriting.
- **Don't make the document too long.** The overview should be scannable in 2-3 minutes. Save depth for deep-dives.
- **Don't assume the user knows domain terminology.** If Phase 0 revealed they're new to the domain, explain terms inline or flag them for the terminology checkpoint.
- **Don't skip Phase 2.5.** The terminology checkpoint is what transforms a "correct but opaque" document into a genuine learning experience.
