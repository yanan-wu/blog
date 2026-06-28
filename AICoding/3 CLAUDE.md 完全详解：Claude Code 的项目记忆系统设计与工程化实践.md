---
layout: default
---
# CLAUDE.md 完全详解：Claude Code 的项目记忆系统设计与工程化实践

---

## 一、什么是 CLAUDE.md？

**CLAUDE.md 是 Claude Code 的持久化项目上下文文件**——本质上，它是一个放在项目目录树中的 Markdown 文件，Claude Code 在每个会话启动时自动读取，将其内容注入对话上下文，作为 Claude 理解你项目的"入职培训手册"。

用一句话概括它的定位：

> **README.md 写给人类看，CLAUDE.md 写给 AI 看。**

没有 CLAUDE.md 时，Claude Code 每次启动都像一个刚入职的聪明新人——它能读懂代码，但不知道你们的规矩：用什么包管理器、测试怎么跑、哪些架构决策是不能碰的、为什么这里用 Server Actions 而不是 API Routes。你得反复纠正它。有了 CLAUDE.md，这些上下文在第一次对话前就已经就位。

---

## 二、分层加载机制：不止"一个文件"

很多人以为 CLAUDE.md 就是一个文件，但底层设计是一套**分层指令体系**，从多个位置收集规则，按优先级合并。理解这套加载链，是写好它的前提。

### 2.1 四个层级（由低到高优先级）

| 层级 | 路径 | 作用域 | 是否提交 Git | 典型用途 |
|------|------|--------|-------------|----------|
| **Managed**（组织级） | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS）或 `/etc/claude-code/` | 全组织所有项目 | IT/DevOps 管理 | 强制安全扫描、合规策略 |
| **User**（用户全局） | `~/.claude/CLAUDE.md` | 你机器上所有项目 | ❌ 不提交 | "我用 pnpm 不用 npm"、"回复用中文" |
| **Project**（项目级） | `./CLAUDE.md`（项目根目录） | 当前项目 | ✅ **提交，团队共享** | 技术栈、架构约束、编码规范 |
| **Local**（本地覆盖） | `./CLAUDE.local.md`（自动被 gitignore） | 当前项目，仅你自己 | ❌ 不提交 | "我正在调 auth 模块，先别碰 login.ts" |

**优先级越高的层级越后加载，模型对后出现的内容注意力更高**——所以 Local 覆盖 Project，Project 覆盖 User，就像 CSS 的层叠。

### 2.2 目录树向上遍历

如果你在 `myapp/packages/api/src/routes/` 下工作，Claude Code **不只看当前目录**，而是逐层向上爬：

```
myapp/packages/api/src/routes/   ← 最高优先级（最近加载）
myapp/packages/api/src/
myapp/packages/api/
myapp/                            ← 根目录 CLAUDE.md 也生效
```

每层都找 `CLAUDE.md` 和 `.claude/rules/`。这意味着 **Monorepo 天然得到支持**：根目录放通用规则，子包放自己的覆盖规则。

### 2.3 它是怎么注入的？

源码层面，CLAUDE.md 的内容不是嵌在 system prompt 内部的——而是作为附件用 `<system-reminder>` 标签包裹后，以**高优先级用户消息**的形式注入，并带有一段硬编码前缀：

> **IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.**

这就是你在 CLAUDE.md 里写"禁止写注释"能真正覆盖默认行为的原因——它的优先级被设计为**高于** Claude 的默认指令。

### 2.4 缓存与成本

官方说明中提到：Claude Code 对 CLAUDE.md 应用 **Anthropic 的 prompt caching**。第一个请求支付完整 input token，后续 5 分钟内的请求命中缓存，按大幅折扣的 cache-read 费率计费。所以 CLAUDE.md 的成本特征是**一次性开销 per session**，而非 per message——但仍值得保持精简以保护上下文窗口的信号质量。

---

## 三、CLAUDE.md 里**应该写什么**、**不该写什么**

这是最核心的工程判断。Anthropic 官方的指导非常明确：

### ✅ 值得写进去的（高信号内容）

| 类别 | 示例 |
|------|------|
| **构建/测试/lint 命令** | `pnpm test -- --run` vs `npm test`；需要哪些 env var |
| **"我们用 X 不用 Y"的决策** | "用 Zod 不用 Joi"、"用 Vue 不用 React"、"用 pnpm 不用 npm" |
| **架构约束（三句话说清）** | "api/ 层只做参数校验和序列化，业务逻辑在 services/" |
| **硬红线（hard constraints）** | "NEVER write to production DB from tests"、"generated/ 目录禁止手动编辑" |
| **已知坑（gotchas）** | "本地开发必须先 `docker compose up -d redis` 否则缓存层 silently fail" |
| **项目特有约定** | branch 命名规则、commit message 格式、PR checklist |

### ❌ 不值得写（低信号 / 负信号）

| 类别 | 为什么 |
|------|--------|
| Claude 读代码就能推断的信息 | 比如 "用 TypeScript"——它打开 `.ts` 文件就知道了 |
| 标准语言风格细则 | PEP 8、Airbnb style——交给 ESLint / Prettier / linter 等**确定性工具**管，别指望 prose 规则被稳定遵守 |
| 完整 API 文档 | 链接到文档或用 `@./docs/api.md` 引用，别内联 |
| 冗长的教程式步骤 | 那应该是 Skills / slash commands 的职责 |
| 团队不实际遵守的 aspirational rules | 过时的指令 **比没有指令更糟**——Claude 遵循了反而产生错误预期 |

### 黄金法则：**少即是多**

Anthropic 工程师 Boris Cherny 将自己的 CLAUDE.md 保持在 **~100 行以内**。官方建议目标是 **under ~200 lines**。

每行内容要经过这个筛选：

> **"删掉这一行，会导致 Claude 犯错吗？"** ——如果不能，删掉。

---

## 四、文件位置与工程化布局

### 4.1 推荐的目录结构

```
your-project/
├── CLAUDE.md                    # ★ 主文件，提交 Git，团队共享
├── CLAUDE.local.md              # 自动 gitignore，你的私人临时指令
├── .claude/
│   ├── settings.json            # 权限/工具 allowlist 等运行配置
│   └── rules/                   # 模块化条件规则（高级用法）
│       ├── api.md               # 仅匹配 src/api/** 时加载
│       ├── testing.md           # 仅匹配 tests/** 时加载
│       └── security.md          # 全局安全红线
├── src/
└── package.json
```

### 4.2 `.claude/rules/` 目录：条件加载的精妙之处

对于大型项目，把所有规则塞进一个 CLAUDE.md 会撑爆上下文。**`.claude/rules/*.md` 文件支持 YAML frontmatter 限定作用路径**：

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules
- ALL endpoints MUST validate input with Zod before any processing
- Error responses MUST use `AppError` class, format: `{ error: { code, message } }`
- NEVER bypass auth middleware with `// eslint-disable`
```

只有 Claude 操作匹配 `src/api/**/*.ts` 的文件时，这段规则才会被注入。改 `README.md` 时不加载，节省 token，减少噪音。

### 4.3 `@import` / `@path` 引用语法

CLAUDE.md 支持用 `@./path/to/file.md` 引入外部文件内容（递归展开，最大 ~5 层深度）：

```markdown
# 项目规范

See @README.md for project overview.
@./docs/coding-standards.md
@./docs/git-workflow.md
```

这样 CLAUDE.md 本身保持 ~50 行的入口，详细内容外置维护。

---

## 五、实战模板（可直接落地）

### 模板 1：Next.js 全栈应用

```markdown
# Frontend App – CLAUDE.md

## Stack
- Next.js 15 (App Router, **not** Pages Router)
- TypeScript strict mode
- Tailwind CSS + shadcn/ui
- Drizzle ORM / PostgreSQL on Supabase
- pnpm (NOT npm or yarn)

## Commands
- `pnpm dev` – start dev server (:3000)
- `pnpm build` – production build
- `pnpm test` – vitest run
- `pnpm typecheck` – tsc --noEmit

## Architecture Rules
- **Prefer Server Actions** over ad-hoc API Routes unless the caller is external.
- All server-side code lives under `app/_actions/` or route handlers under `app/api/`.
- `app/` directory: every page.tsx must export `metadata` for SEO.
- Components in `components/` are client-opt-in: default assume "use server".
- Env vars: only `NEXT_PUBLIC_` prefix goes to client; server-only vars go in `.env.local` (never committed).

## Hard Constraints
- NEVER `console.log` in production paths – use the logger utility at `lib/logger.ts`.
- NEVER hardcode secrets. All secrets come from env vars or Supabase Vault.
- Generated types under `types/db/` are auto-generated from Drizzle – DO NOT edit by hand.

## Gotchas
- Running locally needs `supabase start` first (see `.env.local.example`).
- Vitest config uses `environment: "node"` – if you import browser APIs it'll crash.
```

### 模板 2：Python FastAPI 微服务

```markdown
# order-service – CLAUDE.md

A FastAPI + SQLAlchemy + Redis order-processing microservice.
PostgreSQL schema managed via Alembic migrations.

## Commands (use `uv`, not pip)
- `uv run pytest -m "not slow"` – run unit tests
- `uv run alembic upgrade head` – apply migrations
- `uv run pre-commit run --all-files`
- `docker compose up -d redis postgres` – local deps

## Architecture
src/
  domain/      — pure models & business rules (NO framework imports)
  services/    — business logic layer (depends on domain + repository interfaces)
  adapters/    — external I/O: DB, cache, message queue
  api/         — FastAPI routes: ONLY validation → service call → serialization


## Rules
- ALL DB access through repository pattern; service layer NEVER writes raw SQL.
- Every public service function MUST have a pytest test covering the error path.
- Use `pathlib.Path`, NEVER `os.path` or string concatenation for paths.
- External HTTP calls MUST set `timeout=` and use the shared `http_client` in `adapters/`.
- Logging via `structlog`, NO `print()` anywhere in `src/`.

## Hard Constraints
- NEVER commit real credentials – `.env` is gitignored; use `.env.example` for docs.
- Migrations are ALWAYS reversible. Every `upgrade()` needs a correct `downgrade()`.
```

---

## 六、工作流建议：如何让 CLAUDE.md 保持"活着"

CLAUDE.md 最大的敌人不是写不好，而是**写完就过期**。

### 推荐的维护节奏

| 时机 | 动作 |
|------|------|
| 首次设置 | 运行 `/init`，让 Claude 扫描代码库生成草稿，然后**人工 review & trim** |
| Claude 犯了同一个错两次 | 那就是缺失规则的信号——加一行进去 |
| 架构变更 / 换框架 / 换测试 runner | 更新对应章节 |
| 每季度 | 扫一遍，删掉团队不再遵守的 aspirational rules |

### 快速起步命令

```bash
# 在项目根目录启动 Claude Code 后：
/init        # 让 Claude 自动生成初始 CLAUDE.md
/memory      # 打开 CLAUDE.md 编辑界面
/compact     # 重新加载（如果你在会话中手动编辑了文件）
```

也可以随时在对话中用 `#` 键（不是 `#` 注释，是快捷键）给 Claude 一条指令，它会自动把这条指令归入合适的 CLAUDE.md。

---

## 七、CLAUDE.md vs Memory：规章制度 vs 笔记

这一点经常被混淆，用一个类比说清：

| | CLAUDE.md | Auto Memory (`~/.claude/projects/.../memory/`) |
|---|-----------|-----------------------------------------------|
| **谁写的** | 你 / 团队（人工） | Claude 自己（自动） |
| **性质** | 法律法规 —— Claude **必须**遵守 | 笔记备忘 —— 参考性上下文 |
| **签 Git** | Project 级 ✓ 提交共享 | ✗ 纯本地 |
| **改了立即生效** | 需要 `/compact` 或新会话 | 实时追加 |

**分工**：CLAUDE.md 是你立的法，Memory 是 Claude 自己记的便签。"法律由人制定，笔记由 AI 自动记录。"

---

## 八、进阶：CLAUDE.md 之外的扩展体系

当你项目大到一定程度，仅靠一个 CLAUDE.md 不够用时，Claude Code 还提供：

| 扩展点 | 用途 |
|--------|------|
| `.claude/rules/*.md`（带 `paths:` frontmatter） | 条件加载的模块化规则 |
| `.claude/skills/`（SKILL.md，YAML frontmatter） | 打包可复用的 repeatable workflows（替代旧的 `/commands/`） |
| Hooks（`settings.json` 里配置） | 在文件编辑后自动格式化、commit 前跑 lint 等 shell 触发 |
| MCP servers（`.mcp.json`） | 让 Claude 连 Google Drive / Jira / Slack 等外部工具 |
| `CLAUDE.local.md` | 个人临时覆盖，不污染团队文件 |

---

## 总结

CLAUDE.md 的设计哲学很优雅：**用一个纯文本的 Markdown 文件，以最低摩擦力赋予团队对 AI 行为的可编程约束。** 它不要求你学新 DSL、不用 JSON Schema、不用写插件——就 Markdown，就 Git，就 `cat CLAUDE.md`。

用好它的关键三原则：

1. **把它当 onboarding doc，不当 spec**——保持鲜活，保持短
2. **写"Claude 猜不到的"，不写"代码已经说了的"**——技术栈决策、硬约束、gotchas、命令
3. **分层组织**——全局偏好在 `~/.claude/CLAUDE.md`，团队规范在 `./CLAUDE.md`，临时状态在 `CLAUDE.local.md`，精细化条件规则进 `.claude/rules/`

做到了这些，Claude Code 就从"每次都要重新教的新人"变成"读过员工手册的老手"——这才是 CLAUDE.md 真正的杠杆价值。

