# devstack

An opiniated guide for agentic programming best practices. This is what I'm currently using/moving towards to for development.

> Forked from [lhl/devstack](https://github.com/lhl/devstack) with a couple of personal additions: [`@gotgenes/pi-anthropic-auth`](https://github.com/gotgenes/pi-anthropic-auth) (Claude Pro/Max OAuth via pi's native `/login anthropic`) and [`pi-repoprompt-mcp`](https://github.com/w-winter/dot314/tree/main/packages/pi-repoprompt-mcp) (RepoPrompt MCP integration). All credit for the original setup, framing, and writeups belongs to lhl.

## Coding Agent

[Pi Coding Agent](https://pi.dev/) is an open source "minimal terminal coding harness" that is designed be customized and adapt to the way you want to work. You can start with Claude Code, OpenAI Codex, OpenCode or any harness you want, but if you're looking to start really customizing your workflow/experience, or looking for the best tool to use with multiple models (or just looking for a harness that won't introduce ridiculous regressions constantly), I believe Pi Agent's flexibility and ease of customization actually makes it the current best option. 

You should visit their nicely designed website to get a better idea of what it's all about, but if you just want to install my current setup (pi + my plugins):

```bash
git clone https://github.com/danielraffel/devstack
cd devstack
./pi-setup.sh
```

(Or clone the upstream [lhl/devstack](https://github.com/lhl/devstack) if you don't want the additions noted above.)

### My daily-driver setup

After `pi-setup.sh` finishes, log into Anthropic OAuth via `/login` (pick **Claude Pro/Max**) inside pi, then alias the Opus 4.6 + xhigh-thinking launch into your shell so you don't retype it:

```bash
alias pi='env -u AWS_ACCESS_KEY_ID -u AWS_SECRET_ACCESS_KEY -u AWS_SESSION_TOKEN -u AWS_PROFILE -u AWS_DEFAULT_PROFILE -u AWS_REGION -u AWS_DEFAULT_REGION -u AWS_BEARER_TOKEN_BEDROCK -u AWS_ENDPOINT_URL_BEDROCK_RUNTIME command pi --provider anthropic --model claude-opus-4-6 --thinking xhigh'
```

Add to `~/.zshrc` (or `~/.bashrc`) and `source` it. Use `command pi …` if you ever need the unaliased form (e.g. `command pi --provider google …`). Heads up: subscription auth from third-party harnesses bills against [Anthropic extra usage](https://claude.ai/settings/usage) per token, not your flat plan limits — Opus + `xhigh` burns through it fast, so reserve it for the work that needs it.

**Why the `env -u` chain:** pi auto-loads any cloud provider whose env vars happen to be set in your shell. So if (say) your `~/.zshrc` exports `AWS_ACCESS_KEY_ID` for an unrelated workload like SES, pi loads the `bedrock` provider on launch and the model picker fills with `amazon.nova-…` and `anthropic.claude-…-bedrock` entries that fail with `AccessDeniedException` unless your IAM principal has `bedrock:InvokeModelWithResponseStream`. `--provider anthropic` only pins which provider serves your *current* model — the picker still shows every auto-loaded provider's catalog. Stripping the offending vars *before* pi runs prevents the provider from loading at all, keeping the picker clean.

**Diagnose your own situation:** run this in the shell where you launch pi to see which auto-loader env vars are present. Output is just variable names, not values, so it's safe to share.

```bash
env | grep -oE '^(AWS|AZURE_OPENAI|GOOGLE_CLOUD|GOOGLE_APPLICATION|CLOUDFLARE|HF_TOKEN|GROQ|XAI|OPENAI|DEEPSEEK|MISTRAL|CEREBRAS|FIREWORKS)[A-Z_]*' | sort -u
```

For each line that comes back, either drop the export from your shell rc if you don't actually need it, or add `-u <var>` to the alias above to scrub it for pi only. The alias already covers the AWS family because that's the most common SES/IAM leak; the others are one-flag adds when you hit them.

## Why Pi Is Neat

- **extensibility**: it's not *just* open source (Codex and OpenCode are too) — pi is expressly designed to be easily customized. anything you don't like? tell pi to change itself. Almost everything can be refreshed with `/reload` without a restart — see my list for how w/ minimal yak-shaving, you can customize something to be *very* specific to your preferences
- **models**: scope your model list to your fav providers, check out new/free stuff on openrouter, swap between different models
- **non-blocking**: unlike Claude Code/Codex, most UI/commands (switching models, effort, getting status) are *not* blocked while the model is running. this is obvious and how it should be???
- **advanced history**: I don't know if other harnesses have these, but pi makes it easy to time-travel, fork, clone your sessions/rollouts which is very convenient (`/tree`, `/fork`, `/clone`)
- **caveat**: Pi moves relatively fast and there have been a lot of breaking changes — you sort of need to be on top of things with updates. The upside is that maintenance can be handled mostly by your coding agent and all the source is available, so you don't get Claude Code-style breaking where you're stuck (e.g., Claude got dumb for a month and you get gaslit by Anthropic about it)

## My Pi Extensions

Oftentimes there are multiple options for a feature; these are the ones I've picked. My preference is for composability (doing one thing well). Some things don't exist and I've made my own; some are unmaintained or part of monorepos and in those cases I've forked for maintainability/updatability.

### Web Access

This is probably the biggest feature you're going to need. `pi-web-access` is the most popular and robust plugin, and the others augment the capabilities with better data extraction or more robust browsing

- [nicobailon/pi-web-access](https://github.com/nicobailon/pi-web-access) - web search, content extraction, video/YT, GitHub clone, PDF
- [Thinkscape/agent-smart-fetch](https://github.com/Thinkscape/agent-smart-fetch) (pi-smart-fetch) - browser-like TLS fingerprints + Defuddle site extractors
- [MonsieurBarti/camoufox-pi](https://github.com/MonsieurBarti/camoufox-pi) - stealth web access via Camoufox anti-fingerprinting Firefox fork (requires `npx camoufox fetch` + `/reload`)
  - [Camoufox](https://github.com/daijro/camoufox) - there are a few different builds, but basically, it's a Firefox fork designed for AI agents

### Automation & Workflow

There are also a bajillion sub-agent extensions, I mostly would rather start new sessions or control my sub-agents rather than having them spawned willy-nilly, but if I find a good extension I actually use, I'll add it here.

- [lhl/pi-multiloop](https://github.com/lhl/pi-multiloop) - my autoloop. If you want your agent to grind away for days on something, this is what I've personally been extensively battle-testing and am actively improving. A from-scratch implementation from the things I learned from my [codex-autoresearch](https://github.com/lhl/codex-autoresearch/) fork and from my experience working with autoloops since mid-2025. Published to npm and installed via `npm:pi-multiloop`.
- [tintinweb/pi-schedule-prompt](https://github.com/tintinweb/pi-schedule-prompt) - if you just want an easy heartbeat (recurring cron-like tasks, or one-shot tasks) this does the job 

### Context Management 

For saving tokens.

- [MasuRii/pi-rtk-optimizer](https://github.com/MasuRii/pi-rtk-optimizer) - the most mature/complete `rtk` plugin (`rtk` is a standalone rust binary that dynamically filters and compresses command outputs before they reach LLM context for huge token savings)
  - [rtk](https://github.com/rtk-ai/rtk) — if you pay for tokens or have a quota, I've found this to be the easiest way to reduce token consumption
- [nicobailon/pi-boomerang](https://github.com/nicobailon/pi-boomerang) - this allows launching subagents for tasks that deliver just summarized outputs to your harness 
- [sting8k/pi-vcc](https://github.com/sting8k/pi-vcc) - zero-LLM algorithmic compaction. Replaces pi's default single-pass LLM summarization with deterministic extraction (goal / files / commits / outstanding / preferences + rolling transcript). We install this with `overrideDefaultCompaction: true` (see `pi-setup.sh`) because pi core's default compaction can fail with `400 status code (no body)` when the summarization span exceeds the summarizer LLM's input window; pi-vcc never makes that LLM call. Prior history stays searchable via `vcc_recall` / `/pi-vcc-recall`. Evaluation and alternatives (`pi-grounded-compaction`, `@pi-unipi/compactor`, `pi-agentic-compaction`) documented in [`wiki/tools/pi-agent.md`](wiki/tools/pi-agent.md#compaction-landscape).

### Account & Quota Management

- [gotgenes/pi-anthropic-auth](https://github.com/gotgenes/pi-anthropic-auth) — compatibility shim that lets you use your Claude Pro/Max subscription with pi's native `/login anthropic` flow (OAuth) instead of an API key; API-key behavior is unchanged
  - Debug: `PI_ANTHROPIC_AUTH_DEBUG=all` (or `tool-use`) emits structured logs from the OAuth shaping layer
- [lhl/pi-multicodex](https://github.com/lhl/pi-multicodex) — fork of victor-software-house/pi-multicodex with fixes; automatic ChatGPT Codex account rotation when quota limits or rate limits are hit
  - Keeps its own `~/.pi/agent/codex-accounts.json` (separate from pi's native `auth.json`) and patches into existing model resolution so `/model` and provider config work unchanged
  - Recommended: do not use pi's native `/login` for Codex if you're using multicodex; the two auth systems are independent and mixing them causes confusion
- [pi-codex-status](https://www.npmjs.com/package/pi-codex-status) - CLI + pi extension for ChatGPT Codex quota visibility (`/status`, `pi-codex-status statusline`, normalized JSON export); source: [lhl/pi-codex-status](https://github.com/lhl/pi-codex-status)
  - Auth resolution: tries MultiCodex `codex-accounts.json` first, then pi `auth.json`, then Codex CLI `.codex/auth.json`

### Editor Integrations

- [asyrjasalo/pi-extensions — pi-repoprompt-mcp](https://github.com/asyrjasalo/pi-extensions/tree/main/packages/pi-repoprompt-mcp) — exposes [RepoPrompt](https://repoprompt.com/)'s MCP tools to pi behind a single `rp` tool with branch-safe window/tab binding (auto-attaches by `cwd`, persists across `/tree` rewinds and `/fork`), syntax + diff highlighting, edit/delete guardrails, and an `/rp oracle` shortcut to send the current selection to RepoPrompt Chat
  - Requires the RepoPrompt macOS app installed and running; `pi-setup.sh` auto-writes `~/.pi/agent/extensions/repoprompt-mcp.json` pointing at `/Applications/Repo Prompt.app/Contents/MacOS/repoprompt-mcp` (the bundled MCP server) so pi spawns the right binary instead of trying `npx rp-mcp-server`
  - Gotcha: edits to `~/.pi/agent/extensions/repoprompt-mcp.json` only take effect after a full `pi` restart — `/rp reconnect` keeps using the in-memory config loaded at session start, so you'll still see `npm error code ENOVERSIONS` until you exit and relaunch
  - Upstream monorepo: [w-winter/dot314](https://github.com/w-winter/dot314/tree/main/packages/pi-repoprompt-mcp)

### Task Management

- [tintinweb/pi-tasks](https://github.com/tintinweb/pi-tasks) - Claude Code-style task tracking with 7 LLM-callable tools, dependency management, and a persistent visual widget

### UX

- [lhl/pi-skill-dollar](https://github.com/lhl/pi-skill-dollar) - `$` autocomplete shortcut that triggers skill suggestions in the input area
- [lhl/pi-zentui](https://github.com/lhl/pi-zentui) - my personal fork of a status-line that fits my preferences
- [mattleong/pi-code-previews](https://github.com/mattleong/pi-code-previews) - for better syntax-highlighting from tool calls

### Skills

Combo skill + CLI tool (both a pi skill and a standalone command-line tool):

- [`outline-edit`](https://github.com/lhl/outline-edit) — CLI for Outline knowledge base with local markdown cache (pip, mambaforge)
- [`realitycheck`](https://github.com/lhl/realitycheck) — rigorous source analysis workflow: fetch, analyze, extract claims, register, and validate

## Models (as of May 2026)

Pi supports a number of providers OOTB including most first-party frontier model providers as well as Bedrock, Vertex, and HuggingFace and OpenRouter.

### Custom Providers

(I run Claude Pro/Max OAuth via [`pi-anthropic-auth`](#account--quota-management) and don't use a custom provider extension. If you do want Vertex/Gemini-via-Vertex, see [`@lhl/pi-vertex`](https://www.npmjs.com/package/@lhl/pi-vertex) — install with `pi install npm:@lhl/pi-vertex` and set `GOOGLE_CLOUD_PROJECT` + `GOOGLE_APPLICATION_CREDENTIALS`.)

My current best coding models (date is last time I looked at/updated the model):
- GPT-5.5 xhigh (2026-05-09) — brand new, better personality to talk to than 5.3/5.4, the best coder, still can be myopic
- GPT-5.4 xhigh (2026-05-09) — previous best coder, all-around good, terrible writer, meh design skills, very detail oriented/rules stickler
- Opus-4.7 xhigh (2026-05-09) — unpleasant to talk to, upgrade as code reviewer, frontier analysis, way more expensive than 4.6 (2x+ token cost usually) and basically a prick
- Opus-4.6 max (2026-05-09) — my overall fav planner/analyst/swiss army frontier model
- GPT-5.3-codex xhigh (2026-05-09) — coder aspie
- GPT-5.2 xhigh (2026-05-09) — non-code specialist and it shows, oftentimes OOTB, step back thinker makes it useful, very slow

I haven't used enough of the latest open models but these should be good (Sonnet 4.x level?):
- DeepSeek V4 Pro (2026-05-09) — capable, easy to work with, but sort of feels undone/undertrained? HF served, has tokenizer/DSML tool issues, maybe bad provider quant
- MiniMax M2.7 (2026-05-09) — OK, fast, but not frontier
- Kimi K2.6 (2026-05-09) — of the non-frontier models the most dependable/easiest to work with? Also somehow a monster at GPU kernel optimization
- MiMo V2.5 Pro
- Qwen3.6 Max Preview

### Local Models

(The open models above can also be run locally if you have hundreds of GB of memory; these are the ones that actually fit on a consumer GPU.)

- Qwen 3.6 27B/35B-A3B (2026-05-09) — 20GB for Q4 quants, community favorite, but thinking token usage out of control; dense model much smarter but much slower
- Gemini 4 31B/26B-A4B (2026-05-09) — 20GB for Q4 quants, also quite good, way more reasonable tokens, smart even w/o reasoning, less benchmaxxed, but lots of tool call issue reports


## Guides & Writeups

Things I've written about agentic coding and dev workflows:

- [Agentic Coding](writing/20260415%20Agentic%20Coding.pdf) — talk/slides on agentic coding practices (April 2026)
- [Supply Chain Security for Software Developers](writing/202604-supply-chain-security.md) — practical supply chain security for devs (April 2026)
- [My Workflow](writing/202602-lhl-workflow.md) — personal AI-assisted coding workflow notes (February 2026)

## This Repo

This repo is also an LLM Wiki (Karpathy pattern) — a personal knowledge base where an agent ingests sources, compiles synthesized wiki pages, and maintains cross-links. Detailed docs for each component at `wiki/tools/`. Extension evaluations and comparisons at `wiki/tools/pi-agent.md`.

### Tools

| Tool | Version | Install | Purpose |
|---|---|---|---|
| `qmd` | 2.1.0 | npm (global) | Local semantic search engine for markdown/code collections |

## Repo Structure

```
devstack/
├── README.md              # This file — overview + setup
├── inbox/                 # Drop zone for unprocessed material
├── sources/               # Immutable archive of ingested material
│   ├── gists/             # External specs, gists
│   ├── conversations/     # Exported chat sessions, research transcripts
│   ├── articles/          # Web clippings, blog posts
│   └── papers/            # PDFs, academic papers
├── wiki/                  # Agent-maintained compiled knowledge
│   ├── index.md           # Page catalog (agent-maintained)
│   ├── log.md             # Chronological operations log
│   ├── concepts/          # Ideas, patterns, comparisons
│   ├── tools/             # qmd, RTK, Claude Code, Codex, etc.
│   ├── practices/         # Workflows, playbooks, how-tos
│   └── projects/          # Pi Agent, devstack, etc.
├── writing/               # Your authored content — writeups, talks, slides
├── docs/                  # Project working docs for this repo
├── projects/              # Software submodules / subdirs
└── tools/                 # Scripts, configs, utilities
```

### Directory roles

**inbox/** — Drop anything here: URLs, PDFs, conversations, screenshots, half-formed notes. The agent processes items into `wiki/` pages and archives originals into `sources/`. Unprocessed items live here until ingested.

**sources/** — Karpathy's "raw" layer. Immutable once filed. The agent reads from here but never modifies these files. Organized by type. Binary files (PDFs, images) can use Git LFS via `.gitattributes`.

**wiki/** — Agent-owned. The agent creates, updates, and cross-links pages here. Humans read it; the agent writes it. Every page should use `[[wikilinks]]` for cross-references. `index.md` is the entry point — a categorized catalog of all pages with one-line summaries. `log.md` is an append-only chronological record of operations (ingests, queries, lint passes).

**writing/** — Your own authored content: workflow writeups, talks, slide decks, presentations. Not ingested sources or agent wiki pages — these are things you wrote for external audiences.

**docs/** — Working documents for this repo itself — development notes, research, planning. May be human-authored or agent-authored. Not part of the wiki; these are project-specific artifacts.

**projects/** — Actual software (pi-agent, etc.) as submodules or subdirs. Not indexed by qmd — code, not knowledge.

**tools/** — Scripts and configs that support the devstack workflow.

## Wiki Schema

See [`docs/WIKI.md`](docs/WIKI.md) for the full wiki operational schema (operations, page conventions, frontmatter format, index/log formats, subdirectories, filing rules).

## Sources & References

- [sources/gists/karpathy-llm-wiki.md](sources/gists/karpathy-llm-wiki.md) — Karpathy's original LLM Wiki gist
- [sources/gists/rohitg00-llm-wiki-v2.md](sources/gists/rohitg00-llm-wiki-v2.md) — LLM Wiki v2 spec (lifecycle, knowledge graphs, scale)
- [sources/conversations/RESEARCH-llmwiki.md](sources/conversations/RESEARCH-llmwiki.md) — Survey of LLM Wiki ecosystem and project recommendations

### Related

- [github.com/lhl/agentic-memory](https://github.com/lhl/agentic-memory) — deeper research on agentic memory systems (benchmarks, architectures)
