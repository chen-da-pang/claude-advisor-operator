# Claude Advisor Operator

[![Skill](https://img.shields.io/badge/type-Codex%20skill-111827)](./SKILL.md)
[![Claude Code](https://img.shields.io/badge/runtime-Claude%20Code-5E5CE6)](https://www.anthropic.com/claude-code)
[![Mode](https://img.shields.io/badge/default-read--only-16A34A)](#safety-model)
[![Repository](https://img.shields.io/badge/status-public-0EA5E9)](https://github.com/chen-da-pang/claude-advisor-operator)

Bring Claude Opus 4.8 into Codex as an external advisor for analysis,
decisions, and final review gates.

`claude-advisor-operator` is a Codex skill for moments when Codex should not be
the only mind in the room. Use it when a problem is too complex, the user is
stuck between competing choices, Codex has already reasoned itself into a
corner, or a Goal-mode task needs a second gate after Codex's own review.

The skill packages a reliable bridge from Codex to local Claude Code running
Opus: explicit context bundles, stdin prompt transport, stream-json progress,
fixed session IDs, transcript lookup, debug logs, and a no-downgrade patience
contract. Claude is not just a code reviewer here. Claude is the outside
advisor: a model with a different reasoning path that can analyze the situation,
challenge Codex's assumptions, recommend a decision, and catch omissions before
the user moves on.

## Why This Exists

Codex is powerful, but long-running work has failure modes that are not solved
by asking Codex to think harder:

- The active context can become small, compressed, or lossy after many turns.
- Complex decisions can require more context than a single Codex thread can
  comfortably preserve.
- Codex can miss issues during its own final review, especially near the end of
  a Goal-mode task.
- The user may be undecided and need an outside model to make a direct call.
- A hard problem can benefit from a second model that is not trapped in Codex's
  current assumptions.

Claude advisor calls also need patience. Opus can be working, reading files, or
waiting on an API response while the caller only sees an empty output file. That
ambiguity often leads to premature retries, smaller fallback prompts, or lost
review sessions.

This skill solves both sides of the problem: it makes the external advisor easy
to invoke, and it makes long Claude runs observable enough that Codex can wait
instead of panicking.

- **Outside-model judgment** - Claude receives a prepared case file and gives
  Codex a decision, critique, or recommendation from a separate reasoning loop.
- **No hidden context assumptions** - Codex writes a context bundle and tells
  Claude exactly what to inspect.
- **No positional prompt footguns** - prompts go through stdin, avoiding
  `--add-dir <directories...>` swallowing the prompt argument.
- **Live progress signals** - real calls use `--output-format stream-json`,
  `--verbose`, `--include-partial-messages`, and `--debug-file`.
- **Recoverable sessions** - every run uses a fixed `--session-id`, plus a meta
  file that records output, stderr, debug, project, and transcript paths.
- **Read-only by default** - Claude gets `Read,Grep,Glob,LS`, not write or shell
  tools.
- **Patience built in** - Codex waits on process liveness, stream events,
  transcript growth, and debug-log movement before declaring a stall.

## When To Use It

Use this skill when you want Claude Opus to act as a serious advisor inside a
Codex workflow:

| Situation | What Claude should do |
| --- | --- |
| Codex is stuck or looping | Reframe the problem and identify the missing assumption |
| The user is unsure between options | Recommend a decision with tradeoffs and confidence |
| The task has outgrown the current context | Read a curated context bundle and restore the big picture |
| Goal-mode work is ending | Run a second gate after Codex's own review |
| A code/design/architecture choice is high-risk | Challenge the plan and point out hidden risks |
| Codex produced a solution but something feels off | Audit the reasoning, not just the diff |
| A long conversation has been compacted | Use an explicit bundle so the advisor sees what still matters |

It is especially useful when the active Codex model has limited remaining
context, has gone through automatic compaction, or has already spent many turns
building local assumptions. Claude does not inherit the Codex conversation by
magic; that is the point. Codex has to prepare the case clearly, and Claude gets
to evaluate it from the outside.

## Installation

Clone the repository into your Codex skills directory:

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/chen-da-pang/claude-advisor-operator.git \
  ~/.agents/skills/claude-advisor-operator
```

Or install it into an existing skills folder manually:

```bash
cp -R claude-advisor-operator ~/.agents/skills/
```

The skill expects a working local `claude` CLI from Claude Code.

```bash
which claude
claude --version
```

## Usage

Ask Codex to use the skill when you need Claude Code as an external advisor,
decision-maker, analyst, or review gate:

```text
Use $claude-advisor-operator to ask Claude which of these two architecture paths we should choose.
```

```text
Codex has been going in circles. Call advisor Claude and ask it to diagnose what we are missing.
```

```text
This is a gate: do not continue until Claude advisor approves or explains the blocker.
```

```text
We are near the end of the Goal. After Codex's review, call Claude Opus as the second gate.
```

Codex will prepare a focused context bundle, invoke Claude through the local
CLI, monitor the run, and return both Claude's judgment and Codex's synthesis.

## Operating Model

The skill turns an advisor call into a structured run:

1. Create `work/` artifacts for the prompt, output stream, stderr, debug log,
   and run metadata.
2. Build a context bundle with the user request, current decisions, relevant
   files, commands, risks, and constraints.
3. Invoke Claude Code with Opus and a fixed session ID.
4. Watch stream-json, stderr, debug logs, and Claude's project transcript.
5. Parse the final `result` event and report model usage when available.
6. Have Codex compare Claude's answer against local evidence before acting.

The core invocation shape is:

```bash
claude -p \
  --safe-mode \
  --verbose \
  --session-id "$SESSION_ID" \
  --model opus \
  --effort high \
  --input-format text \
  --output-format stream-json \
  --include-partial-messages \
  --debug-file "$DBG" \
  --permission-mode bypassPermissions \
  --tools Read,Grep,Glob,LS \
  --add-dir "$PWD" \
  < "$PROMPT" \
  > "$OUT" \
  2> "$ERR"
```

See [SKILL.md](./SKILL.md) for the full workflow, live-check commands, smoke
test, and return contract.

## Safety Model

By default, advisor Claude is intentionally scoped as an advisor, not an
operator with write access:

| Concern | Default |
| --- | --- |
| Model | `opus` |
| Effort | `high` |
| Custom hooks/plugins/MCP | Disabled with `--safe-mode` |
| Prompt transport | stdin |
| Output | `stream-json` |
| Write access | Not granted |
| Shell access | Not granted |
| Tools | `Read,Grep,Glob,LS` |

`--permission-mode bypassPermissions` is used to prevent non-interactive
permission prompts from blocking `claude -p`. It is not the safety boundary.
The safety boundary is the tool allowlist. Do not include `Write`, `Edit`,
`MultiEdit`, or `Bash` unless the user explicitly asks for a non-read-only
Claude run.

Read-only still means Claude can read files inside directories you expose with
`--add-dir`. Do not include secrets in context bundles, and do not add
directories that contain sensitive material unless the review truly requires
them.

## Observability

Every substantial advisor run should leave enough evidence for Codex to decide
whether Claude is healthy, stuck, or finished:

- `$OUT` grows with stream-json events.
- `$ERR` captures CLI errors and warnings.
- `$DBG` records lower-level debug activity when enabled.
- `$META` records the session ID and run-specific paths.
- `~/.claude/projects/<project-key>/<session-id>.jsonl` records Claude's
  transcript.

The skill's patience contract is simple: do not kill, retry, or downgrade while
Claude is alive and there is heartbeat evidence. For Opus high, wait at least
20 minutes before declaring a stall unless the process exits or stderr shows a
hard error. For deeper review modes, expect 30 to 60 minutes.

## Troubleshooting

| Symptom | Likely cause | What to check |
| --- | --- | --- |
| Output file is 0 bytes | Prompt was not delivered, or process has not emitted yet | Check `$ERR`, `$DBG`, process liveness, and transcript growth |
| `claude -r` shows no conversation | Print-mode sessions may not appear under the current resume picker project | Locate the fixed `session_id` under `~/.claude/projects` |
| Claude appears idle | Long Opus run, no first chunk yet, or API wait | Follow the patience contract before retrying |
| "Input must be provided..." | `--add-dir` consumed a positional prompt argument | Use stdin for prompt transport |
| Claude asks for permissions | Invocation is not fully non-interactive | Use `--permission-mode bypassPermissions` plus a narrow `--tools` allowlist |
| Need a quick health check | Verify CLI/model/stdin path | Run the smoke test in [SKILL.md](./SKILL.md#invocation-workflow) |

## Repository Layout

```text
.
├── SKILL.md
└── agents/
    └── openai.yaml
```

`SKILL.md` is the executable operating guide that Codex loads when the skill
triggers. `agents/openai.yaml` provides UI-facing metadata for skill pickers and
default prompts.

## Design Principles

- Make the advisor call explicit, inspectable, and recoverable.
- Use Claude Opus when Codex needs an outside mind, not just another summary.
- Prefer structured context over raw repository dumps.
- Keep Claude's default role advisory and read-only.
- Treat advisor approval as a real gate when the user says it is a gate.
- Preserve model independence: Claude analyzes and decides; Codex verifies and
  synthesizes.
- Do not hide failures behind smaller fallback prompts.

## Validation

Run the Codex skill validator:

```bash
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py \
  ~/.agents/skills/claude-advisor-operator
```

Expected result:

```text
Skill is valid!
```

## Status

This skill is intentionally small: one operating guide and one metadata file.
It is designed to be copied, audited, and adapted without a framework install.
