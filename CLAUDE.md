# TokenSip operating prompt

Copy this file as `CLAUDE.md` (Claude Code, global at `~/.claude/CLAUDE.md` or per-project) or as an always-on rule file in Cursor (`.cursor/rules/tokensip.mdc` with `alwaysApply: true`). It is written to be concrete enough that copying it alone, even without installing every tool below, changes agent behavior for the better. Installing the tools it references (CodeGraph, graphify, RTK, context-mode, the caveman suite) makes every rule fully actionable instead of partially actionable.

Every command, hook snippet, and threshold below is real, taken from a working setup, not illustrative pseudo-code. Swap tool names if your local install differs, but keep the specificity: an agent given "use a graph tool when appropriate" behaves nothing like an agent given the exact command and the exact trigger condition.

**Repo:** [github.com/ssnivlek/tokensip](https://github.com/ssnivlek/tokensip). If this file got copied on its own, the full write-up (what each tool does, measured numbers, diagrams) lives in that repo's `README.md`.

---

## 0. The one number that matters most: model tier and reasoning effort

Before any tool below, this is the single biggest lever, bigger than every other rule combined. Model tier and reasoning effort are a straight price multiplier on every token in the session, compression tools reduce token *count*, tier/effort changes the *price per token*, and the two stack multiplicatively.

Check the real default with `npx ccusage@latest daily` and `npx ccusage@latest session` (reads Claude Code's local JSONL logs directly, no API call needed). Top-tier models cost several times more per token than mid-tier for comparable day-to-day work, and the most expensive individual sessions tend to be the ones left on a top-tier model by default rather than by deliberate choice.

Concrete rule:
- Set a cheap, capable model as the actual default, not just an intention. In Claude Code, that means an explicit `"model"` key in `~/.claude/settings.json` (global) or the project's `.claude/settings.json`, e.g. `"model": "claude-sonnet-5"`. Without an explicit default, a session falls back to whatever was last manually selected, or to a "fast"/highest-tier mode if one exists, and that default silently becomes the expensive one.
- Set reasoning effort low by default too (`"effortLevel": "low"` in the same file, if the harness supports it), for the same reason: effort is another cost multiplier independent of model tier.
- Escalate manually, per task, only when the task is genuinely hard: architecture decisions, gnarly debugging, ambiguous multi-step design. Don't escalate for routine ops, research, or well-scoped implementation work.
- Sub-agents and background workflow runs inherit the parent session's model tier. A session left on a top-tier model doesn't just cost more itself, every agent it fans out to (parallel benchmarks, dispatched sub-tasks) bills at that same tier too. Check this before fanning out many agents.
- Re-check actual spend periodically with `npx ccusage@latest daily` (per day) and `npx ccusage@latest session` (per session, to find outliers) rather than assuming the default is being respected. It reads local logs, costs nothing to run, and gives real numbers instead of a guess.

## 1. Context cost, the core rule everything else serves

Session cost does not scale by prompt count. It scales by `model tier × effort × turns × accumulated context size × output size`. The single most expensive recurring mistake is letting a large chunk of raw content sit in the conversation, because it gets resent (and re-billed) on every following turn, not paid for once.

Everything below exists to keep accumulated context size down without losing the information actually needed.

## 2. Code exploration: repo-scoped graph before grep

**Install:** `npm install -g @colbymchenry/codegraph`, then `codegraph init` inside any repo to index it. If this command is not available, install it before following the rest of this section.

Before any exploratory `grep`/`find`/`Read` in a directory that has a `.codegraph/` folder at its root:

```bash
codegraph explore "<question, e.g. 'who calls processPayment'>"
```

or, via MCP tool, `codegraph_explore("<question>")`. This returns the relevant symbols' verbatim, line-numbered source plus the call paths between them in one round trip, including dynamic-dispatch hops plain grep cannot follow. Measured on a real query (a RUM instrumentation flow): ~2.8K tokens with the graph (1 call, 30 symbols, 4 files, blast radius) versus ~14-15K tokens without it (3-5 grep/search calls, 4 full files read, one wrong file read along the way), ~80% savings, measured on that call, not estimated.

If a directory has no `.codegraph/`, don't force indexing mid-task, that's a setup decision (`codegraph init`), not something to trigger reactively. Fall back to normal exploration there.

**Hook wiring** (Claude Code, `~/.claude/settings.json`): a `UserPromptSubmit` hook that runs `codegraph prompt-hook` auto-injects a relevant hint on every prompt, so this rule partially enforces itself even without the agent remembering it:

```json
"UserPromptSubmit": [
  { "hooks": [ { "type": "command", "command": "codegraph prompt-hook" } ] }
]
```

## 3. Cross-project or non-code knowledge: graph query before broad search

**Install:** `pipx install graphifyy` (real PyPI package name has a double `y`, `graphify` alone is a different, unrelated package; the CLI itself is invoked as `graphify`).

Before any exploratory search that spans more than one repo, or touches docs/config rather than pure code, in a directory that has `graphify-out/graph.json`:

```bash
graphify query "<question>"                                        # scoped to current project
graphify query "<question>" --graph ~/.graphify/global-graph.json  # scoped to every merged project
graphify path "<A>" "<B>"                                            # relationship between two concepts
graphify explain "<concept>"                                         # a focused concept
```

Estimated cost: a scoped query, ~1-3K tokens (a small subgraph) versus ~10-20K tokens reading a full `GRAPH_REPORT.md` (5-50KB) or looping grep+read, ~75-85% estimated savings.

**In Claude Code, install the real hook, don't rely on remembering.** Running `graphify claude install` (from the user's home directory, so it targets the global `~/.claude/settings.json`, not a project-local one) registers a `PreToolUse` hook that blocks `Bash|Grep` and `Read|Glob` via `graphify hook-guard search`/`graphify hook-guard read` until a `graphify query` has actually run, the same automation tier as CodeGraph's hook above. If that command has never been run, or was silently lost (installing another tool's hooks on top can overwrite `settings.json` instead of merging into it, check `hooks.PreToolUse` after any hook-installing tool runs), this rule degrades to "has to be called on purpose, every time," and an agent that reads the rule once and forgets over a long session will quietly regress to raw grep, that has actually happened. In Cursor, there is no equivalent: `graphify cursor install` only writes an `alwaysApply: true` rule file, which reminds but does not block, confirmed in the tool's own source, not a gap that more configuration closes. Don't confuse "there's a rule about it" with "it's enforced," verify which one is actually true before trusting this rule to self-enforce.

Merged project graphs can be exported to a human-browsable Obsidian vault (`graphify export obsidian --graph <path> --dir <vault-dir>`), that export is for human review, not an agent token-saving mechanism, the query commands above are.

## 4. Shell output: compress before it enters context

**Install:** repo and release binaries at [github.com/rtk-ai/rtk](https://github.com/rtk-ai/rtk) (Rust, no npm/pip package, grab the release for your platform).

Route ordinary shell commands (`git status`, `grep`, `ls`, test runs) through a compressing proxy instead of letting raw terminal output land in context. Wired via a `PreToolUse` hook matching `Bash` in `~/.claude/settings.json`:

```json
"PreToolUse": [
  { "matcher": "Bash", "hooks": [ { "type": "command", "command": "rtk hook claude" } ] }
]
```

Once wired, this is transparent, `git status` becomes `rtk git status` with no extra typing. Meta-commands for checking the tool itself:

```bash
rtk gain              # cumulative savings, real telemetry from the tool itself
rtk discover           # missed optimization opportunities in command history
rtk proxy <cmd>        # run the command raw, completely unfiltered
rtk -vv <cmd>          # verbose but still filtered
```

Measured, native telemetry, not estimated: 340 commands in one session, 22.6K tokens saved, 70-90% savings per command on `ls`/`grep`, one `ls -la` alone saved 18.9K tokens (69%). This is the only number in this entire document backed by a tool's own telemetry rather than a manual measurement or estimate.

**The failure mode to avoid:** aggressive compression is correct for routine operations but can hide the exact detail that matters mid-investigation. Concrete rule: when actively debugging a failing test, a stack trace, or genuinely unexpected behavior, and the compressed output doesn't make the root cause obvious, escalate to `rtk -vv <cmd>` or `rtk proxy <cmd>` before drawing any conclusion. Never use an "ultra-compact" compression mode while debugging, that mode is for repetitive routine operations only, not investigation. Failed commands typically already have their full unfiltered output saved to disk (e.g. under `~/.local/share/rtk/tee/`), check that file before re-running the command.

## 5. Large data processing: sandbox it, don't inline it

**Install (Claude Code plugin):** `/plugin marketplace add mksglu/context-mode` then `/plugin install context-mode@context-mode`. Repo: [github.com/mksglu/context-mode](https://github.com/mksglu/context-mode).

Before pulling a large tool result, file, or command output directly into the conversation to filter, count, parse, or aggregate it, do that work in an isolated sandbox and bring back only the derived answer:

```
ctx_batch_execute(commands, queries)   # run several commands in parallel, index the output, return matching snippets
ctx_search(queries: [...])              # search everything already indexed, including prior session memory
ctx_execute(language, code)             # run code against data; only what is explicitly printed enters context
ctx_fetch_and_index(url)                # index a large web page instead of pulling it whole into context
```

The deciding question is always: am I about to *process* this output, or just *observe* a short, fixed result? Processing routes through the sandbox tools above. Observing something short and fixed (`git status` on a clean tree, `whoami`, a version string) or mutating state (`git`, `mkdir`, `rm`, any write) is fine as a direct call, sandboxing adds nothing there. Summarizing or extracting from a file goes through a sandboxed file-read equivalent; editing a file needs the exact bytes present in the conversation to match against, so a direct read is correct there, not a sandboxed one.

If the harness auto-captures decisions, errors, and prompts into a searchable timeline (`ctx_search(sort: "timeline")` or equivalent), query it instead of re-deriving context that was already captured, especially right after a context clear or compact.

## 6. Response prose: match compression to the moment

**Install (Claude Code plugin):** `/plugin marketplace add JuliusBrussee/caveman` then `/plugin install caveman@caveman`. Repo: [github.com/JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman).

If a prose-compression mode exists, default to its least lossy setting for ordinary responses, escalate to a more aggressive setting only during clearly routine, repetitive stretches (e.g. running the same command across many directories in sequence, where nuance genuinely doesn't matter), and reserve the most aggressive tier exclusively for internal agent-to-agent handoffs, never for a final message a human is meant to read carefully.

Concrete rule for stepping back down to full, uncompressed language regardless of the configured default: security warnings, irreversible-action confirmations, multi-step sequences where a dropped word risks misreading, and any moment the user asks a clarifying question or seems confused. Resume the compressed mode once that moment has passed.

## 7. Spec-driven and multi-step work: lightweight loop over heavy ceremony, by default

**Install:** repo [github.com/JuliusBrussee/cavekit](https://github.com/JuliusBrussee/cavekit), follow that repo's own install steps (the installer copies skill files into `~/.agents/skills/`, symlinked into `~/.claude/skills/`).

Default to a lightweight, single-file spec/build/verify loop instead of a heavier framework built around separate sub-agent phases for brainstorming, planning, execution, and verification. Concretely, that looks like: `spec` (write or update a single `SPEC.md`) → `build` (implement against it) → `check` (audit code against the spec, report drift), reaching for auxiliary steps (a pre-spec interrogation pass, a research pass, an adversarial review pass, a design-deepening pass, a bug-to-invariant backprop step) only when the task actually earns that ceremony.

This is evidence-based, not a stylistic preference: an isolated benchmark (same synthetic task, one function plus a test suite, run in two separate git repos, one framework forced per repo so results don't contaminate each other) produced this result. Lightweight single-file loop: 71,163 tokens, 23 tool calls, ~57 minutes, 8/8 tests passing, and a ~30-line spec file that stayed a dense, reusable contract. Sub-agent-phased framework: 79,749 tokens, 25 tool calls, ~106 minutes, same passing result, but it generated roughly 170 lines of design-and-plan documentation for about 70 lines of actual code and tests, a 3:1 ceremony-to-code ratio, because the phase-separation and self-approval overhead don't have a lightweight branch for "the spec is fully known, skip the ceremony."

That result is for one small, well-defined task, it does not prove the lightweight loop wins at every scale. Reserve the heavier, sub-agent-phased approach for genuinely large or ambiguous work: multi-feature scope, high requirement uncertainty, where the extra structure has room to pay for itself across a much bigger deliverable. If both approaches are available in the same environment, pick one per task and stick with it, don't let them compete mid-task, that produces duplicate process artifacts describing the same work.

## 8. Memory across sessions: don't re-derive what was already captured

**Install:** `npm install -g cavemem`.

If a persistent-memory mechanism with automatic capture exists (hooked into session start, prompt submit, tool use, and session end), let it run and query it rather than manually re-summarizing context it already captured. If the current environment only offers memory as a manual, query-only mechanism with no automatic capture, that's a real functional gap in that environment specifically, not a false alarm, actively query it at the start of relevant work instead of assuming it already has the session's history the way the fully-hooked environment would.

## 9. Never waste context you already paid for

- Never let a tool result, file dump, or generated report enter the conversation raw once it exceeds roughly 100 lines. Read or generate it through a sandboxed tool or sub-agent and bring back only the conclusion, or save it to a file and reference the path.
- Never re-emit a large artifact (a JSON blob, a report, a dashboard, a long generated file) a second time in the same session. Edit the saved file in place and reference its path. Resending the whole block is the single most expensive repeatable mistake in a long session, it is pure new output cost with zero cache reuse, and it also inflates what has to be recalled on every subsequent turn.
- When switching to a genuinely unrelated task mid-session, clear or compact the conversation. That is the only thing that actually cuts the tax of recalling the entire prior history on every future turn, no compression tool touches that tax.
- Switching model mid-session breaks prompt caching (the already-processed prefix has to be reprocessed from scratch under the new model), so avoid casual model switches inside one continuous session, batch them at session boundaries instead.
- Prefer a scoped call over a broad one everywhere a filter or query narrows the result, whether that's an MCP tool call, a search, or a file read. "Give me everything, I'll filter it myself" is the most common way an agent quietly reintroduces the cost this whole document exists to avoid.
