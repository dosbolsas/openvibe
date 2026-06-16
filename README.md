# openvibe

**A multi-agent [OpenCode](https://opencode.ai) setup that splits AI coding into four separate roles — plan, review, build, code-review — each on a different model that checks the others' work.**

The problem with letting one model plan, write, and check its own code is that it grades its own homework. And it passes. Every time. openvibe breaks the work across *different model families* so that mistakes have a real shot at getting caught by something that doesn't share the author's blind spots.

Designed for a **product-minded operator**: you describe features in plain language, the pipeline handles every technical decision, and you make the final "is this what I wanted" call. You are not expected to read code to use this. You are expected to actually run the thing and click the button before shipping — no model can do that for you.

---

## The four roles

| Role | Agent | Job | Model | Why |
|------|-------|-----|-------|-----|
| **Architect** | `plan` | Reads the codebase, decides the architecture, writes `PLAN.md`. Cannot touch source code. | GLM-5.1 | Different model family from the builder — eliminates shared blind spots that survive from plan to implementation. |
| **Builder** | `build` | Implements the plan to the letter, runs tests, leaves changes uncommitted until you say ship. | DeepSeek V4 Pro | DeepSeek's reasoning at high effort — strong execution in the iterative build loop. |
| **Plan Reviewer** | `@1-plan-review` | Independently critiques the *plan* for judgment flaws **before any code is written**. Reads the real codebase itself — does not trust the architect's description of it. | Kimi K2.7 | Different lab → decorrelated blind spots. |
| **Code Reviewer** | `@2-code-review` | After the build: conformance to the plan (omissions, deviations, excess) **and** code quality (bugs, security via Semgrep, anti-patterns) in a single pass. Bipartite verdict. | Qwen 3.7 Max | Different lab from both planner and builder — 4 distinct model families with zero overlaps. Backed by Semgrep (5000+ deterministic rules). |

**The single most important thing:** the planner, builder, and reviewers are on four distinct model families. A checker that shared the builder's training distribution would share its blind spots. Decorrelation is the whole game.

### What each stage actually catches

- **Plan Review** — judgment errors: a plan that would compile, pass its own tests, and solve the subtly wrong problem. Catches this *before* any build tokens are spent.
- **Code Review** — two things in one pass: (1) conformance errors (omissions, deviations, things built in excess of the spec) and (2) implementation quality (bugs, security holes, anti-patterns). Backed by Semgrep. A bipartite verdict tells you whether to go back to the architect (drift) or the builder (issues).
- **Running it yourself** — whether it's actually what you wanted. This is the realest test and it's on you.

---

## The files

| File | What it is |
|------|------------|
| `opencode.jsonc` | Minimal project config: MCP servers, default agent. No agent definitions — those live in markdown. |
| `.opencode/agents/plan.md` | Markdown agent for `plan`. YAML frontmatter + system prompt. Overrides the built-in plan agent. |
| `.opencode/agents/build.md` | Markdown agent for `build`. YAML frontmatter + system prompt. Overrides the built-in build agent. |
| `.opencode/agents/1-plan-review.md` | Markdown subagent for `@1-plan-review`. YAML frontmatter + system prompt. |
| `.opencode/agents/2-code-review.md` | Markdown subagent for `@2-code-review`. YAML frontmatter + system prompt. |
| `.gitignore` | Keeps `PLAN.md` out of commits. |
| `PLAN.md` | **Generated at runtime** by the architect. The handoff artifact. Not authored by you. |

Agents are defined as **markdown files** in `.opencode/agents/` — OpenCode's native agent format. Each file has YAML frontmatter (model, temperature, permissions) followed by the system prompt body. The file name becomes the agent name (`plan.md` → `plan` agent). No separate prompt files, no sync problems, no `{file:}` references to maintain. Everything travels with the repo.

---

## How the work flows

```
You describe a task (plain language)
        │
        ▼
 ┌─────────────┐   reads repo, writes PLAN.md,
 │  plan (GLM) │   shows you a plain-English summary
 └─────────────┘
        │
        ▼
   @1-plan-review .... (optional, for non-trivial tasks)
        │           reads PLAN.md + the actual code,
        │           returns SOUND or REVISE
        │           └─ if REVISE: architect fixes PLAN.md, re-review
        ▼
   Tab to build .... shared session: builder has the plan as live context
        │
        ▼
 ┌────────────────┐   reads PLAN.md, implements exactly
 │ build (DeepSeek) │   that, runs tests,
 └────────────────┘   leaves changes UNCOMMITTED
        │
        ▼
   @2-code-review . reads PLAN.md + git diff + changed files + Semgrep,
        │           returns bipartite verdict: CONFORMANCE + QUALITY
        ▼
   You run it / click it / use it  ← the only check that counts
        │
        ▼
    Tell build "commit and push" → it shows you exactly what it's
        committing → commits → pushes
```

`plan` and `build` are **primary agents** — switch with Tab, shared session, no lossy re-paste. The two checker agents are **subagents** — invoke with `@1-plan-review` / `@2-code-review` from whichever primary agent you're in. They run in isolated child sessions, which is *by design*: a reviewer that inherited the architect's reasoning would just agree with it. The checkers read `PLAN.md` from disk instead.

### When to use the full chain vs. not

Full chain for large or hard-to-reverse changes. For a one-line tweak, the honest minimum is: **plan → build → run it → commit**. The checkers are insurance you add when the stakes justify extra passes. Don't summon a four-model review pipeline to change a button color.

Session context carries operator preferences organically across agent switches. The git log carries project history. No separate memory file needed.

---

## Key design decisions (don't undo these)

**The architect is read-only on source files.** A load-bearing constraint, not a limitation. Forces handoff rather than drift into implementation; keeps the expensive deep-reasoning seat from editing code it was only meant to think about. It can write exactly one file (`PLAN.md`) and run read-only git commands.

**Builder is on a different model family from the planner.** The planner runs on GLM-5.1 (Z.AI), the builder on DeepSeek V4 Pro — separate labs, separate training data. No model family plans and implements its own work. Decorrelation is a stronger safety property than any per-model capability advantage.

**Plans hand off via a file on disk, not via conversation context.** OpenCode's multi-agent context inheritance is fuzzy and version-dependent. `PLAN.md` makes every handoff deterministic and inspectable. The checker prompts are explicitly instructed to trust disk state over inherited context.

**The builder leaves changes uncommitted until you say ship.** It has access to `git commit`/`git push` but waits for your explicit "commit and push" or "ship it." Before acting, it prints the files, the commit message, and the target branch. It halts on secrets, detached HEAD, or anything genuinely anomalous. Destructive operations (force-push, rebase, reset) are permission-denied at the config level.

**Post-build review is a single pass with a bipartite verdict.** Conformance (did the code match the plan?) and quality (is the code any good?) are checked by one agent in one context-load, avoiding the redundant dual context-load of separate drift and code-review passes. The bipartite output format (CONFORMANCE + QUALITY) preserves the accountability routing: drift failures go to the architect, quality failures go to the builder. The tradeoff: a single model provides the only post-build opinion — Semgrep adds a deterministic, non-LLM safety net for security issues.

**Version verification is mandatory.** The architect must never design against a third-party API from memory. It reads the pinned version from dependency files, then checks that version's docs via context7. This rule was added after a real failure where a model confidently planned against a deprecated API. Documenting the verified version in the plan is what actually enforces the check.

**Semgrep is the non-LLM guardrail.** `@2-code-review` runs every changed file through Semgrep's MCP — 5000+ deterministic rules, 35+ languages, fires the same way regardless of the LLM's current mood. Findings are cited as `(Semgrep: <rule-id>)`. Degrades gracefully if Semgrep isn't installed. One-time setup: `brew install semgrep`. Note: `SEMGREP_SEND_METRICS=off` breaks the default `auto` config — leave it unset.

**Markdown agents, not JSONC+separate-prompt-files.** The old architecture used `{file:./architect.md}` references in the JSONC config, which resolved relative to the config file's location — not the repo root — causing sync failures when the config lived outside the project. Markdown agents embed the prompt directly in the file, eliminating the sync problem at the root.

---

## What's in PLAN.md

PLAN.md is the actual product of the pipeline — everything else exists to produce and check it. The architect writes a plain-English header (for you) plus a fenced `<build_specification>` (for the agents).

| Section | Purpose |
|---------|---------|
| IN PLAIN ENGLISH | What you'll get, jargon-free |
| THE BIG PICTURE | How it fits the system + the one tradeoff worth knowing |
| ASSUMPTIONS I MADE | Product choices the architect made for you — correct any that are wrong |
| CONTEXT I VERIFIED | The real files and patterns examined — evidence the plan isn't hallucinated |
| FILES TO TOUCH | path · what changes there |
| COMPONENTS | name · responsibility · verified external versions |
| SEQUENCING | Ordered build steps, when order matters |
| ALTERNATIVES CONSIDERED | What else was evaluated and why it lost (optional) |
| KEY DECISIONS | choice · one-line rationale |
| INTERFACES / CONTRACTS | Data shapes and boundaries the builder must honor |
| HOW TO VERIFY | The exact command(s) that prove it works |
| ACCEPTANCE CRITERIA | What "done" looks like |
| RISKS / WATCH-OUTS | The 1–2 places this is most likely to go wrong |
| BUILD ESCALATION | The task-specific halt condition so the builder can't loop forever |
| OUT OF SCOPE | What was deliberately not built |

**Right-sizing matters.** This is a menu, not a mandatory template. A one-line CSS tweak skips SEQUENCING, INTERFACES, and RISKS. A database migration leans on all of them. CONTEXT I VERIFIED, FILES TO TOUCH, and ACCEPTANCE CRITERIA are always present; the rest earn their place. No "RISKS: n/a" padding.

### Dynamic audit sections (outside `<build_specification>`)

PLAN.md is more than a static spec — it carries the pipeline's audit trail across agent invocations. These sections are appended or replaced during review and build cycles, not authored by the architect up front:

| Section | Who writes it | When |
|---------|--------------|------|
| `## REVIEW HISTORY` | Architect | After revising PLAN.md due to an `@1-plan-review` REVISE verdict. Records each flaw, whether it was ACCEPTED or REJECTED, and the rationale. The latest review history replaces any previous one. |
| `## BUILD FAILURE` | Builder | On hitting the BUILD ESCALATION circuit breaker. Records which step failed, what was built, what was tried, and the error log (with secrets redacted). The architect reads and deletes this section when resolving the failure. |

Both sections live outside the `<build_specification>` tags so they never conflict with the formal plan structure. You can spot whether a plan has been through review or escalation by checking for these headers. The architect and reviewers read them from disk — the operator never needs to copy-paste between agents.

---

## Temperature and models

**Rule: use each vendor's own recommended temperature for thinking mode.** The intuition that "checking should be colder, so lower the temperature" is wrong for reasoning models — it can collapse the chain-of-thought and actively degrade output. Temperature follows the model, not the role.

| Agent | Model | Temp | Why |
|-------|-------|------|-----|
| plan | GLM-5.1 | 1.0 | Z.AI's documented default for GLM-5.1 series per API reference |
| build | DeepSeek V4 Pro | 1.0 | DeepSeek's thinking-mode guidance |
| @1-plan-review | Kimi K2.7 | 1.0 | Moonshot's recommendation for K2.7 thinking mode |
| @2-code-review | Qwen 3.7 Max | 1.0 (top_p: 0.95) | Qwen's own coding agent benchmarks (SWE-Bench, Terminal-Bench 2.0) use `temp=1.0, top_p=0.95` |

**All models run on OpenCode Go** (`opencode-go/`) — flat subscription ($10/month), dollar-denominated limits, zero-retention policy. Single auth, single bill. No mixed providers to debug.

Model strings (verified against the Go endpoint table):
- `opencode-go/glm-5.1`
- `opencode-go/deepseek-v4-pro`
- `opencode-go/kimi-k2.7`
- `opencode-go/qwen3.7-max`

> ⚠️ **Verify every model string** in the `/models` picker before trusting it. The Go docs internally contradict on the Kimi ID (endpoint table says `kimi-k2.7`, but a config example says `kimi-k2.7-code`). If `kimi-k2.7` doesn't resolve, try `kimi-k2.7-code`. If Qwen3.7 Max fails, fall back to `opencode-go/minimax-m2.7` (preserves decorrelation — MiniMax is a 5th distinct lab).

---

## Platform notes (OpenCode ~1.15)

- **Per-agent reasoning effort may not always apply.** The GUI reasoning selector can "stick" across agent switches. If plan and build end up at the same effort, set it manually in the UI.
- **Bash permissions are prefix-matched.** Keep deny patterns broad — especially `git push --force*` and `git push * --force`.
- **Permission `deny` rules behave differently via the SDK** than in interactive desktop use. Interactive use honors them as configured.
- **Context inheritance across subagents is version-dependent.** Handoffs go through `PLAN.md` on disk rather than relying on inherited context — the checker prompts are written around this.
- **Markdown agents in `.opencode/agents/` override built-in agents with the same name.** `plan.md` overrides the built-in plan agent; `build.md` overrides the built-in build agent. The file name IS the agent name.
- **Config precedence:** project `opencode.jsonc` overrides global `~/.config/opencode/opencode.jsonc`. `.opencode/` directories are loaded after project config, so markdown agents take final priority.

---

## Quickstart

1. **Copy the files** (`opencode.jsonc` + `.opencode/agents/` directory + `.gitignore`) into your project root.
2. **Connect OpenCode Go.** Run `/connect` in the TUI, select `OpenCode Go`, paste your API key.
3. **Install Semgrep** once: `brew install semgrep`. Leave `SEMGREP_SEND_METRICS` unset. Skip this and `@2-code-review` falls back to LLM-only analysis with a note.
4. **Verify model strings** in `/models`. Fix any that don't resolve (see Kimi/Qwen notes above).
5. **Set `CONTEXT7_API_KEY`** as an environment variable in your shell profile. The project config uses `{env:CONTEXT7_API_KEY}` for the context7 MCP server.
6. **Use it:**

| Action | How |
|--------|-----|
| Start a task | Describe it to `plan` (default agent) |
| Review the plan | `@1-plan-review` |
| Build it | Tab → `build`, tell it to implement |
| Check conformance + code quality | `@2-code-review` |
| Confirm it's what you wanted | Run/click/use it yourself |
| Commit & push | Tell `build` "commit and push" |
| Plan got flagged | Tell `plan` to address the review; it rewrites `PLAN.md` |
| Builder got stuck | It hits the `BUILD ESCALATION` condition and reports back |
