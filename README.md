# TokenSip

**A concrete, hook-driven token-optimization stack for Claude Code and Cursor.**

Session cost in a coding agent does not scale by prompt count. It scales by `model tier × effort × turns × accumulated context size × output size`. The single biggest lever is model/effort tier, the second biggest is context that has already entered the conversation and gets resent (and re-billed) on every following turn. Neither is fixed by wishing, both are fixed by concrete configuration, which is what this repo documents.

Every command, hook snippet, and number below is real, taken from a working setup and, where marked, from actual measurement, not illustrative pseudo-code.

---

## 0. Model tier and reasoning effort (biggest lever, check this first)

Before any tool below: what model and what reasoning effort is a session actually running at, by default, when nobody thinks about it?

**Diagnose it for real**, don't guess:

```bash
npx ccusage@latest daily     # cost per day, split by model
npx ccusage@latest session   # cost per session, to find outlier sessions
```

`ccusage` reads Claude Code's local JSONL logs directly, no API call, no cost to run it. On one real account's local history, this showed a top-tier model (Opus-class) responsible for **~88% of all spend** across every logged day, versus a second-tier model (Sonnet-class) responsible for the remaining ~12%, at roughly **7x lower cost per session** for comparable work. The three most expensive individual sessions in that history ($326, $226, $145) were all top-tier-model sessions. This dwarfs every percentage-savings number quoted for the tools below, model tier is a multiplier on every token, compression tools reduce token count, the two stack multiplicatively.

**Fix the default, don't rely on remembering to pick the cheap model each time.** In Claude Code, set it explicitly in `~/.claude/settings.json` (global) or a project's `.claude/settings.json`:

```json
{
  "model": "claude-sonnet-5",
  "effortLevel": "low"
}
```

Without an explicit `model` key, a session falls back to whatever was last manually selected, or to a "fast"/highest-tier mode if the harness has one, and that silently becomes the expensive default. `effortLevel` (reasoning effort) is a second, independent multiplier, low by default costs meaningfully less than high for the same task.

Escalate manually, per task, only when it's genuinely warranted: architecture decisions, hard debugging, ambiguous multi-step design. Two things easy to miss:

- **Sub-agents and background workflow runs inherit the parent session's model tier.** A session left on a top-tier model doesn't just cost more itself, every agent it fans out to (parallel benchmarks, dispatched sub-tasks, a multi-agent workflow) bills at that same tier too.
- **Switching models mid-session breaks prompt caching**, the already-processed context prefix has to be reprocessed from scratch under the new model. Batch tier changes at session boundaries, not mid-conversation.

Re-run `ccusage` periodically. A default that was correct last month can silently drift if a harness update changes what "default" resolves to.

---

## Works in both Claude Code and Cursor

### CodeGraph

Per-repository symbol/call graph (SQLite), stored in `.codegraph/` at the project root. Answers "who calls X", "where is X defined", "what's the blast radius of changing Y" in one shot instead of a grep-then-read loop.

```bash
codegraph explore "<question>"        # shell
codegraph_explore("<question>")       # MCP tool
```

Returns verbatim, line-numbered source for the relevant symbols plus the call paths between them, including dynamic-dispatch hops that plain grep can't follow.

**Measured on a real query** (RUM instrumentation flow): with CodeGraph, ~2.8K tokens (1 call, 30 symbols, 4 files verbatim, blast radius). Without it, ~14-15K tokens (3-5 grep/search calls, 4 full files read, one wrong file read along the way). **~80% savings**, measured, not estimated.

Auto-injects a hint on every prompt via a `UserPromptSubmit` hook:

```json
"UserPromptSubmit": [
  { "hooks": [ { "type": "command", "command": "codegraph prompt-hook" } ] }
]
```

If a directory has no `.codegraph/`, the tool simply doesn't apply there, indexing is opt-in (`codegraph init`), not something to trigger reactively mid-task.

### graphify

A broader knowledge graph: code, docs, config, and effectively any input (image, video, paper) turned into graph nodes, with community detection clustering related topics. Each project gets its own graph (`graphify-out/graph.json`), and all projects are merged into one global graph (`~/.graphify/global-graph.json`) for cross-project queries.

```bash
graphify query "<question>"                                        # scope: current project
graphify query "<question>" --graph ~/.graphify/global-graph.json  # scope: every merged project
graphify path "<A>" "<B>"
graphify explain "<concept>"
```

**Estimated savings:** a scoped query costs ~1-3K tokens (a small subgraph). Without graphify, reading the full `GRAPH_REPORT.md` (5-50KB) or looping grep+read costs ~10-20K tokens. **~75-85% estimated.**

**Key difference from CodeGraph: no automatic hook by default.** It has to be called on purpose, every time. An agent that reads the rule once and forgets over a long session will quietly regress to raw grep, that has actually happened in practice. A Cursor-style always-on editor rule (`alwaysApply: true`) helps as a reminder but does not force the call the way a real hook does, don't mistake "there's a rule about it" for "it's enforced."

A merged global graph can be exported as a browsable Obsidian vault:

```bash
graphify extract . --backend claude-cli --global --as "<project-name>"   # re-index one project
graphify export obsidian --graph ~/.graphify/global-graph.json --dir <vault-dir>
```

That export is for human browsing, it is not itself a token-saving mechanism, the query/path/explain commands above are. Re-indexing many projects back-to-back with a CLI-backed extraction can spawn several concurrent model subprocesses and exhaust memory, do one project at a time and check system memory between runs if extracting more than a couple.

### The Obsidian vault: the human-readable layer on top of the graph

The merged global graph (`~/.graphify/global-graph.json`) is great for an agent to query, but not something a human opens directly. The export command above turns it into an Obsidian vault with ready-made table views (Everything, By community, Code only, Rationale, Concepts, Documents, Communities) for browsing the same graph visually.

This export isn't the token-saving mechanism itself, that's still `graphify query`/`explain`/`path`, it's the bridge for human review: open the vault and browse community by community to understand what the graph actually captured, instead of asking the agent to re-summarize everything (which would cost tokens for no reason).

A scheduled low-priority background job (e.g. a launchd job on macOS, roughly every 4 hours, low CPU/IO priority, skipping the run entirely if an active coding session or high system load is detected) can keep this in sync automatically: re-merge the global graph and re-export the vault only when at least one project's underlying graph actually changed, so it never fights an active session for resources and never does pointless work when nothing changed.

### RTK (token-optimized CLI proxy)

Compresses **shell command output** before it ever enters the model's context (`git status`, `grep`, `ls`, test runs). Not about code structure, about the raw text a terminal command would otherwise return.

Wired via a `PreToolUse` hook matched on `Bash`:

```json
"PreToolUse": [
  { "matcher": "Bash", "hooks": [ { "type": "command", "command": "rtk hook claude" } ] }
]
```

Once wired, this is fully transparent, `git status` becomes `rtk git status` automatically, no extra typing required.

```bash
rtk gain              # cumulative savings, real telemetry from the tool itself
rtk discover           # missed optimization opportunities in command history
rtk proxy <cmd>        # run the raw command, completely unfiltered (for debugging)
rtk -vv <cmd>          # verbose, still filtered
```

**Measured, native telemetry (`rtk gain`):** 340 commands in one session, 22.6K tokens saved, 70-90% savings per command on `ls`/`grep`, a single `ls -la` peaked at 18.9K tokens saved (69%). **This is the only percentage in this document backed by the tool's own telemetry**, everything else is a one-off manual measurement or an estimate.

**The failure mode to actively guard against:** aggressive compression is correct for routine operations but can hide the exact detail that matters mid-investigation. When actively debugging a failing test or genuinely unexpected behavior, and the compressed output doesn't make the root cause obvious, escalate to `rtk -vv <cmd>` or `rtk proxy <cmd>` before drawing any conclusion. Never use an "ultra-compact" mode while debugging, that's for repetitive routine operations only. Failed commands typically keep their full unfiltered output saved to disk already (e.g. under `~/.local/share/rtk/tee/`), check that file instead of re-running the command.

### caveman suite

Same author, one package, three distinct pieces, each solving a different problem:

**caveman** compresses the agent's own reply prose (drops articles, filler, pleasantries, keeps code/commands verbatim). Intensity levels:

| Level | When |
|---|---|
| `lite` | Default, least lossy, still saves tokens |
| `full` | Routine/repetitive stretches where nuance doesn't matter (e.g. running the same command across many directories) |
| `ultra` | Subagent-to-subagent handoffs only, never a final user-facing message |
| `wenyan` | Not recommended, too lossy relative to the gain |

An external benchmark measured ~30-40% savings on response prose at the `full` level. Regardless of configured level, drop back to full uncompressed language for security warnings, irreversible-action confirmations, multi-step sequences where a dropped word risks misreading, or whenever the user asks a clarifying question or seems confused.

**cavekit** runs a `grill → spec → research → review → build → check` loop over a single `SPEC.md` file, no sub-agents. Everyday skills: `spec` (write/update the spec), `build` (implement against it), `check` (audit code against the spec and report drift). Situational skills, only invoked when the task earns them: `grill` (pre-spec interrogation), `research`, `review` (adversarial), `deepen`, `backprop` (turn a bug into a new spec invariant so it can't recur silently).

**An isolated benchmark** (same synthetic task: one Python function plus a pytest suite, run in two separate git repos, one framework forced per repo so results couldn't contaminate each other) pitted cavekit against a sub-agent-based framework with separate brainstorm/plan/execute/verify phases:

| | cavekit | sub-agent-phased framework |
|---|---|---|
| Tokens | 71,163 | 79,749 |
| Tool calls | 23 | 25 |
| Wall time | ~57 min | ~106 min |
| Result | 8/8 tests passing | 8/8 tests passing (same) |
| Process artifacts | ~30-line `SPEC.md`, dense, reused as a contract | ~170 lines of design+plan docs for ~70 lines of code+tests (3:1 ceremony-to-code) |

Same final result either way. The sub-agent-phased framework's phase-separation and self-approval overhead doesn't have a lightweight branch for "the spec is fully known, skip the ceremony," so it paid a disproportionate cost on a small, well-defined task. That result does not generalize to every task size, a genuinely large or ambiguous multi-feature effort is where the heavier framework's structure has room to pay for itself. Pick one per task, don't run both at once, that produces duplicate process artifacts describing the same work.

**cavemem** persists memory across sessions (decisions, errors, blockers, prior context) so an agent doesn't have to re-derive or ask the user to re-explain what was already established. Automation depends on the harness, see the table below.

---

## Claude Code only

### context-mode

Runs a command or data-processing step in an isolated sandbox, indexes the result (full-text + similarity search), and returns only the derived conclusion, never the raw bytes. The distinction from RTK: RTK compresses the output of one shell command, context-mode goes further and lets you **process** (filter, count, aggregate, parse) an entire large dataset without its raw bytes ever touching the conversation.

```
ctx_batch_execute(commands, queries)   # run commands in parallel, index them, return matching snippets
ctx_search(queries: [...])              # search everything already indexed, plus session memory
ctx_execute(language, code)             # process data; only what you print() enters context
ctx_fetch_and_index(url)                # index a large page instead of pulling it whole into context
```

**The deciding question:** are you going to process (filter/count/parse) the output, or just observe a short, fixed result? Processing → context-mode. Observing a clean `git status`, `whoami`, or mutating state (`git`, `mkdir`, `rm`) → a plain shell call is still correct, sandboxing adds nothing there. Summarizing a file → `ctx_execute_file`; editing a file → a plain file read is still correct (an editor needs the exact bytes in context to match against).

`ctx_search(sort: "timeline")` also pulls decisions, errors, and prompts captured automatically across sessions, surviving a context clear/compact, query it instead of re-deriving context already captured.

**cavemem, Claude-Code-specific behavior:** in Claude Code, cavemem hooks into every lifecycle event and captures memory automatically:

```json
"SessionStart":     [{ "hooks": [{ "type": "command", "command": "cavemem hook run session-start" }] }],
"UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "cavemem hook run user-prompt-submit" }] }],
"PostToolUse":      [{ "hooks": [{ "type": "command", "command": "cavemem hook run post-tool-use" }] }],
"Stop":             [{ "hooks": [{ "type": "command", "command": "cavemem hook run stop" }] }],
"SessionEnd":       [{ "hooks": [{ "type": "command", "command": "cavemem hook run session-end" }] }]
```

No manual step required, memory capture just happens.

---

## Cursor only

Nothing in this stack is exclusive to Cursor today. Every tool above either runs identically in both editors, or runs in Claude Code only with a lighter-weight equivalent in Cursor (see the automation table below). If you install this stack in Cursor specifically: caveman and cavekit work the same (as editor rules and symlinked skills respectively), CodeGraph and RTK have full hook parity, graphify has an editor rule instead of nothing (still requires an active call either way), and cavemem is query-only there, without automatic capture.

---

## Real hook vs. rule-that-needs-a-habit

A real hook fires on its own, no agent decision involved, it's wired into the harness's lifecycle events. A "skill" or editor rule is a text instruction asking the agent to invoke itself when it judges the situation relevant, it is not forced execution. Confusing the two is the most common way a stack like this quietly stops working, a tool with no real hook depends entirely on the agent remembering it exists.

| Tool | Claude Code | Cursor |
|---|---|---|
| CodeGraph | Automatic, `UserPromptSubmit` hook | MCP-registered, same automatic hint |
| graphify | **No hook**, requires an active call | Editor rule (`alwaysApply`), doesn't actually block anything |
| RTK | `PreToolUse` hook, always on | Equivalent hook, always on |
| caveman | Active plugin, applies on its own | Editor rule, same effect |
| cavekit | Available via the skill system, invoked by name or context | Symlinked skills, same invocation |
| cavemem | Hook on every lifecycle event | MCP query-only, no automatic capture |
| context-mode | MCP plugin, invoked as the task calls for it | Not configured |

---

## Prompt caching, a separate lever from model tier and from any single tool

Distinct from picking a cheaper model (section 0) and from any single tool's savings: prompt caching is about reusing already-processed context, not shrinking context or changing its price.

- A session inside the cache's freshness window reuses the already-processed prefix and only pays for the new delta.
- Switching models mid-session breaks that cache, a full new prefix has to be processed under the new model. Batch tier changes at session boundaries.
- Tools that shrink what enters context (CodeGraph, graphify, RTK, context-mode) shrink the prefix that has to be cached and resent. cavemem preserves the prefix across sessions via retrieved memory instead of rebuilding context from zero.
- Re-emitting a large artifact (a JSON blob, a report, a dashboard) more than once in the same session is the single most expensive way to break this: it's new output every time, none of it reuses cache, and it inflates what has to be recalled (and potentially recached) on the next turn. Save it to a file, reference it by path, edit in place instead.

## The prompt

[`CLAUDE.md`](./CLAUDE.md) in this repo is the operating prompt that ties this whole policy together as concrete, copy-pasteable rules, not vague advice, real commands, real hook JSON, real thresholds. Drop it into any Claude Code project (`CLAUDE.md`, global or per-project) or Cursor project (an equivalent always-on rule file) to get this decision policy directly, whether or not every tool it references is installed, the rules that don't need a specific tool (model/effort defaults, context hygiene, prose compression discipline) apply regardless.

## Quick decision table

| Situation | What to do |
|---|---|
| Starting a new setup, or haven't checked in a while | Run `ccusage daily`/`session`, confirm model/effort defaults are actually cheap |
| Question about code in one repo | CodeGraph |
| Broad question, cross-project, or about docs | graphify |
| Running a shell command | Let RTK intercept it |
| Actively debugging, RTK's compressed output doesn't add up | `rtk -vv` or `rtk proxy`, never conclude from compressed output alone |
| Processing/filtering/counting a large dataset | context-mode, not a plain shell call |
| Observing a short fixed result, or mutating state | A plain shell call is fine |
| Spec, planning, or implementation work | cavekit, not a sub-agent-heavy framework by default |
| Genuinely large/ambiguous multi-feature work | The heavier sub-agent-phased approach may earn its overhead here |
| Switching to an unrelated task mid-session | Clear or compact the context |
| About to re-emit a large report/JSON again | Don't. Save it to a file and reference the path |
| Task needs real architectural or hard-debugging reasoning | Escalate model/effort manually, per task, then drop back down |
