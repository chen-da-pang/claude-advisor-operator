---
name: claude-advisor-operator
description: Use when the user asks Codex to call, consult, or summon "顾问 Claude", "advisor Claude", "Claude advisor", "ask Opus", or "use Claude Code as advisor"; or when Claude advisor approval is a required review gate.
---

# Claude Advisor Operator

## Overview

Use this skill as an atomic bridge from Codex to Claude Code advisor mode. It converts the useful parts of the Advisor Strategy into a Codex workflow: collect the right context, invoke Claude Code with Opus, and return the advisor's judgment with the exact model/effort evidence when available.

This is a skill, not an MCP tool or standalone callable tool. Do not search for a tool named `claude-advisor-operator`. When this skill triggers, use the normal shell/tooling available to Codex to run the local `claude` CLI.

This skill assumes `claude-code-operator` is enabled and the local `claude` CLI works. If either is missing, stop and tell the user instead of silently using a Codex subagent.

## Defaults

- Advisor model: `opus`
- Advisor effort: `high`
- Stronger effort: use `xhigh` or `max` when the user explicitly asks for deeper thinking, high-stakes review, security review, architecture review, or "覆盖打击式" context review.
- Output: for real advisor calls, use `--output-format stream-json --verbose` and parse the final `result` event so Codex can observe progress while Claude runs. Use single-result `json` only for tiny smoke tests.
- Isolation: use `--safe-mode` by default for advisor calls so global Claude Code hooks, plugins, MCP servers, and auto-loaded customizations do not distort a focused review. Omit it only when the user explicitly wants those customizations.
- Tools: keep the advisor read-only by default with `--tools Read,Grep,Glob,LS`.
- Permissions: use `--permission-mode bypassPermissions` only to avoid non-interactive permission prompts in `claude -p`; read-only safety is enforced by the `--tools` allowlist. Never include `Write`, `Edit`, `MultiEdit`, or `Bash` in the advisor toolset unless the user explicitly asks for a non-read-only Claude run.

## Context Rule

Claude Code does not automatically inherit the current Codex conversation. Always pass context explicitly.

Create a context bundle under `work/`, for example:

```text
work/claude-advisor-context-YYYYMMDD-HHMMSS.md
```

Include:

- Current user request and the specific advisor question.
- Concise Codex conversation state: decisions made, commands run, files changed, blockers, assumptions.
- Project facts: `pwd`, relevant `git status`, package/config files, file tree summary.
- Relevant code excerpts with absolute paths and line numbers.
- Failed approaches or uncertainties.
- Constraints: user preferences, deadlines, risk tolerance, allowed/disallowed actions.

Do not include secrets, API keys, `.env` contents, token values, browser cookies, keychains, or unrelated full logs.

If the user requests broad or "覆盖打击式" context, broaden the sweep but still filter:

- Use `rg --files` for a file manifest.
- Read manifests/configs first: `package.json`, lockfiles, `pyproject.toml`, `README`, app config, tests, main entrypoints.
- Exclude `.git`, `node_modules`, `dist`, `build`, caches, binary/media files, and generated artifacts unless directly relevant.
- Prefer a structured bundle over raw-dumping the whole repository.

## Invocation Workflow

1. Announce that you are calling Claude Code as an Opus advisor and state the planned model/effort.
2. Create `work/` if needed, choose a `SESSION_ID`, and set run-specific paths (`PROMPT`, `OUT`, `ERR`, `DBG`, `META`) before writing files:

```bash
mkdir -p work
SESSION_ID="$(uuidgen | tr 'A-Z' 'a-z')"
PROMPT="work/claude-advisor-prompt-${SESSION_ID}.md"
OUT="work/claude-advisor-result-${SESSION_ID}.jsonl"
ERR="work/claude-advisor-result-${SESSION_ID}.err"
DBG="work/claude-advisor-debug-${SESSION_ID}.log"
META="work/claude-advisor-run-${SESSION_ID}.meta"
{
  printf 'session_id=%s\n' "$SESSION_ID"
  printf 'prompt=%s\n' "$PROMPT"
  printf 'out=%s\n' "$OUT"
  printf 'err=%s\n' "$ERR"
  printf 'debug=%s\n' "$DBG"
  printf 'project=%s\n' "$PWD"
} > "$META"
```

3. Build the context bundle under `work/`.
4. Build an advisor prompt and write it to the run-specific `$PROMPT` path. The prompt asks Claude to:
   - act as a senior technical advisor,
   - read the context bundle and any referenced project files,
   - identify missing context before guessing,
   - provide assessment, recommendation, implementation guidance, risks, and confidence.
5. Invoke Claude Code through the operator pattern / local CLI with the same run variables from step 2. Prefer stdin instead of passing the prompt as a positional argument, because `--add-dir <directories...>` is variadic and can accidentally consume a following prompt argument. For non-trivial advisor calls, use stream-json, a fixed session id, stderr, and a debug log:

```bash
test -s "$PROMPT" || { echo "Missing advisor prompt: $PROMPT" >&2; exit 1; }
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

Use `--effort xhigh` or `--effort max` when warranted by the user's wording or risk level.

6. Patience contract while Claude runs:

- Do not kill, retry, or downgrade to a smaller prompt while the `claude -p` process is alive and any heartbeat exists.
- Heartbeats include: `$OUT` line count or mtime changes; stream events such as `system/status/requesting`, `message_start`, `content_block_delta`, `tool_use`, `tool_result`, or final `result`; the transcript file `~/.claude/projects/<project-key>/${SESSION_ID}.jsonl` appearing or growing; or `$DBG` showing API request / first chunk / tool activity.
- Poll every 30-60 seconds and give the user short status updates for long advisor runs.
- For Opus high advisor calls, wait at least 20 minutes before declaring a stall unless the process exits or stderr shows a hard error. For `xhigh`, `max`, or broad repo review, expect 30-60 minutes.
- The minimum wait floor takes precedence over the no-heartbeat rule. Before the floor, report lack of heartbeat as concerning but keep waiting unless the process exits, stderr shows a hard error, or the user asks to stop.
- After the minimum wait floor, stop only if the process exits nonzero, stderr has an actionable fatal error, the user asks to stop, or there is no heartbeat for 10+ consecutive minutes and no transcript/debug movement.

Useful live checks:

```bash
META="${META:-$(ls -t work/claude-advisor-run-*.meta 2>/dev/null | head -n 1)}"
test -n "$META" || { echo "No claude advisor META file found" >&2; exit 1; }
SESSION_ID="$(awk -F= '$1=="session_id"{print substr($0, index($0,"=")+1)}' "$META")"
PROMPT="$(awk -F= '$1=="prompt"{print substr($0, index($0,"=")+1)}' "$META")"
OUT="$(awk -F= '$1=="out"{print substr($0, index($0,"=")+1)}' "$META")"
ERR="$(awk -F= '$1=="err"{print substr($0, index($0,"=")+1)}' "$META")"
DBG="$(awk -F= '$1=="debug"{print substr($0, index($0,"=")+1)}' "$META")"
PROJECT="$(awk -F= '$1=="project"{print substr($0, index($0,"=")+1)}' "$META")"
wc -l "$PROMPT" "$OUT" "$ERR" "$DBG"
tail -n 20 "$OUT"
PROJECT_KEY="$(printf '%s' "$PROJECT" | sed -E 's#[^A-Za-z0-9]#-#g')"
TRANSCRIPT="$HOME/.claude/projects/${PROJECT_KEY}/${SESSION_ID}.jsonl"
if [ ! -f "$TRANSCRIPT" ]; then
  TRANSCRIPT="$(find "$HOME/.claude/projects" -maxdepth 2 -name "${SESSION_ID}.jsonl" -print -quit)"
fi
[ -f "$TRANSCRIPT" ] && tail -n 20 "$TRANSCRIPT"
```

7. After the process exits, check result, stderr, and debug log before parsing:

- If `$OUT` is empty and `$ERR` is non-empty, read stderr first. The common failure is `--add-dir <directories...>` swallowing a positional prompt, which produces "Input must be provided either through stdin or as a prompt argument when using --print".
- If `$OUT` has stream events but no final `result` event, inspect `$ERR`, `$DBG`, and the transcript before retrying.
- If stderr is empty and there is no final result after process exit, treat Claude Code as unavailable or buggy for this invocation shape.

8. If needed, run a tiny smoke test with the same model/effort and stdin shape. Prefer a semantic smoke test over an exact-string incantation, because global session-start context can sometimes cause generic acknowledgements:

```bash
printf '%s' 'What is 2+2? Answer with only the numeral.' \
  | claude -p \
      --safe-mode \
      --model opus \
      --effort high \
      --input-format text \
      --output-format json \
      --tools Read,Grep,Glob,LS
```

Treat a JSON result of `4` as a working CLI/model/stdin path. If this works but the advisor call fails, fix the advisor invocation shape, prompt path, or tool/read permissions and retry. If this fails, report Claude Code as unavailable.

9. Parse the final `result` event from `$OUT`. Report:
   - resolved model from `modelUsage` when present,
   - effort requested,
   - the advisor's answer,
   - your own synthesis or disagreement.

## Advisor Prompt Shape

Use this compact structure:

```markdown
You are Claude Code acting as a senior technical advisor for Codex.

## Task
[specific decision or review question]

## Context Bundle
Read: [absolute path to work/claude-advisor-context-...md]

## Project Root
[absolute project path]

## What I Need From You
1. Assessment
2. Recommendation
3. Implementation guidance
4. Risks and edge cases
5. Missing context or assumptions
6. Confidence level

Be direct. Do not modify files unless explicitly asked.
```

## Return Contract

Do not present the advisor as if it saw the entire Codex conversation unless the context bundle actually contains it. Say "I passed a prepared context bundle" when relevant.

If Claude Code returns an error, include the error summary and stop. Do not fall back to a Codex model unless the user asks for that fallback.
