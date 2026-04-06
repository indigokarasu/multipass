---
name: ocas-multipass
description: Accomplish tasks that need tools the agent doesn't have. Multipass plans approaches, fills capability gaps with sandboxed tools, and executes autonomously within an isolated session. No check-ins, no approval gates, no escalation. Output is the task result and a replay script. No global installs, no real identity, clean state in and out.
---

# Multipass

Accomplish tasks that require tools you don't have. Fire and forget. User invokes `multipass.run {task}`, Multipass runs to completion, user gets the output. No check-ins, no approval gates, no escalation.

Everything happens inside a session directory. Nothing touches the rest of OpenClaw.

## When to use

- Agent is about to say "I can't do that" but the blocker is a missing tool
- Task requires an external API, MCP server, or skill not currently installed
- User says "figure out how to do X" where X needs external tooling
- User wants a reproducible recipe for an uncommon workflow

## When NOT to use

- Task is solvable with installed OCAS skills or tools
- User wants to permanently install a skill (use `openclaw skills install`)
- User wants to build a new skill (use Forge)
- General web research (use Sift)

## Isolation rules

- All files in session dir: `~/openclaw/data/ocas-multipass/sessions/{session_id}/`
- No global skill installs. No global MCP config changes. No real identity.
- No env vars, secrets, or config written outside session dir.
- If a capability requires breaking isolation, skip it silently and find an alternative.

## Autonomy rules

Once invoked, Multipass runs to completion without user interaction.

- No "Proceed?" gates. No "Should I...?" questions. No escalation.
- Every decision is made by the skill. If a choice is ambiguous, pick the safest sandboxable option and log the reasoning.
- If everything fails, produce a failure report documenting what was tried and why each path failed. A useful failure report is still an output.
- The only user-facing moments are invocation and the final result.

## Loop prevention

LLMs get stuck. They retry the same action with meaningless variations, invent theories about why something failed ("too fast," "data isn't ready yet"), and wait for things that aren't coming. Multipass has hard circuit breakers. Read `references/resilience.md` §Circuit breakers for the full logic.

Rules every worker must follow:

- **Same action, same result = move on.** If an endpoint returned an error, calling it again with slightly different headers or timing will return the same error. Two identical failures on the same endpoint is the maximum. After that, the endpoint is dead. Skip it.
- **A failure is an answer, not a mystery.** A 403 means you're not authorized. A 404 means it doesn't exist. A connection refused means the server is down. Do not theorize about why. Do not invent timing explanations. Read the error, log it, move on.
- **No waits longer than 30 seconds.** The only legitimate wait is polling for a verification email (10s intervals, 2min max). Everything else is the agent stalling. If a step needs "waiting," it's failed.
- **No folk theories.** Do not reason about whether the server is "busy," "processing," "eventually consistent," or "warming up." If the response isn't what you expected, it failed. Try a different approach or a different service.
- **Track every action.** Before performing any action, check `log.jsonl`: have I done this before? If the same URL+method appears 2+ times with failures, do not try it again.
- **Budget per gap.** Maximum 5 total candidates attempted per capability gap. If 5 candidates all fail, the gap is unsolvable with available tools. Report it and move on.

## Workflow

Four phases. Task first, tools second, execution third, report last.

### Phase 1: Plan

Understand the task. What outcome does the user need?

Generate 2-3 approaches ranked by self-sufficiency. Lead with the approach that needs the least external tooling. Many tasks have a no-signup path.

Check runtime config (see `references/orchestration.md` §Config check). If `maxSpawnDepth < 2`, run single-threaded. If mcporter is not installed, mark MCP-by-URL as unavailable and fall back to direct HTTP for MCP servers.

Write `manifest.json` with status `planned`.

### Phase 2: Fill gaps

If the selected approach has no gaps, skip to Phase 3.

**Adaptive complexity:**
- Simple (1 gap, obvious): inline search, no workers.
- Moderate (2-3 gaps): 1-2 targeted workers.
- Complex (4+): full parallel worker set.

If `sessions_spawn` fails when spawning a worker, fall back to inline execution for that surface. Do not stop.

Discovery workers search surfaces (see `references/surfaces.md`), score candidates (see `references/scoring.md`), and write to `checkpoints/`. Orchestrator polls and merges.

**Resolution preference:**

1. Free API, no auth → `web_fetch`
2. Free API, key via signup → throwaway identity + `web_fetch`
3. MCP server → `mcporter call {url}.{tool}` or direct HTTP
4. Downloaded skill → read SKILL.md as context

**All network calls use retry logic** (see `references/resilience.md` §Retry). 3 attempts, exponential backoff, before skipping.

**Identity provisioning** (only when needed): create throwaway inbox from cascade in `references/resilience.md` §Identity. Use BotEmail first (longest-lived). If all providers fail, proceed without signup capability -- restrict to no-auth candidates only.

**When a candidate fails** (signup blocked, gate hit, endpoint down):
- Skip to next candidate. If no candidates remain, try the next approach from Phase 1.
- If all approaches exhausted, apply reframing tactics from `references/resilience.md` §Reframing.
- If reframing fails, proceed to Phase 4 and produce a failure report.

**When a gate is hit** (CAPTCHA, browser login, SMS):
- Search installed skills and MCP servers for a handler (see `references/resilience.md` §Gates). Max search depth: 1 (do not search for handlers of handlers).
- If handler found, use it. If not, skip the service.

Write `manifest.json` with status `resolved`.

### Phase 3: Execute

Spawn `worker:execute` with the resolved tools and the fallback approach baked into the task description so it can pivot without returning to the orchestrator.

The worker:
- Calls APIs via `web_fetch` (with retry)
- Calls MCP servers via `mcporter call` by URL or direct HTTP
- Uses downloaded SKILL.md as context
- Uses session identity for signups
- Writes progress to `log.jsonl` and `checkpoints/execute.json` after every major step
- Writes `status.txt` one-liner for progress visibility

**If the execute worker hits a wall** (API error, rate limit, service down):
- Retry 3 times with backoff.
- If still failing, try the fallback approach.
- If fallback also fails, complete as much as possible and proceed to Phase 4.

### Phase 4: Report

Write to session directory:

**1. Task output** in `output/` -- the deliverable. Even if partial.

**2. replay.md** -- standalone recipe with health checks, version pins, and "what might break" section. See `references/resilience.md` §Replay health.

**3. manifest.json** -- final status: `complete`, `partial`, or `failed`.
- `complete`: task fully accomplished.
- `partial`: some steps succeeded, some failed. Output contains what was accomplished. Manifest documents what remains.
- `failed`: no useful output produced. Manifest documents every approach tried, every candidate evaluated, every failure reason. This is still a useful artifact.

**4. Journal entry.**

Update `status.txt`: `[done] {one-line summary of outcome}`

## Commands

- `multipass.run {task description}` -- full autonomous lifecycle
- `multipass.search {description}` -- discovery only
- `multipass.sessions` -- list recent sessions
- `multipass.replay {session_id}` -- re-execute replay script (new session, new identity)
- `multipass.status` -- config, source health

## Responsibility boundary

**Multipass does**: plan, discover, provision identity, resolve, execute, checkpoint, report. All autonomously.
**Multipass does not**: install anything globally, use real identity, persist secrets, build skills, ask the user for help mid-run.

If a user runs the same replay 3+ times, suggest permanent installation or a Forge-built skill.

## Optional skill cooperation

- **ivangdavila/api**: Check 147-service reference before discovery.
- **ivangdavila/reverse-engineering**: TRACE protocol for probing candidates.
- **Sift**: Deeper candidate research. If absent, use `web_search`.
- **Forge**: Suggest after completion if user needs a permanent solution.
- **mcporter**: Check existing connections. Call by URL for sandboxed access.

## Journal outputs

**Action Journal** -- after every `multipass.run`.
**Research Journal** -- after every `multipass.search`.

Path: `~/openclaw/journals/ocas-multipass/YYYY-MM-DD/{run_id}.json`

## Storage layout

```
~/openclaw/data/ocas-multipass/
  config.json
  search_log.jsonl
  decisions.jsonl
  sessions/
    {session_id}/
      manifest.json
      log.jsonl
      status.txt
      replay.md
      checkpoints/
      skills/
      workspace/
      output/
~/openclaw/journals/ocas-multipass/
  YYYY-MM-DD/{run_id}.json
```

## OKRs

Universal OKRs from spec-ocas-journal.md apply to all runs.

```yaml
skill_okrs:
  - name: tool_invocation_success_rate
    metric: fraction of tool invocations completed successfully
    direction: maximize
    target: 0.92
    evaluation_window: 30_runs
  - name: spawn_depth_efficiency
    metric: average ratio of tasks completed to spawn depth used
    direction: maximize
    target: 0.80
    evaluation_window: 30_runs
  - name: isolation_violation_rate
    metric: fraction of isolated sessions that do not leak context or state outside their boundaries
    direction: maximize
    target: 1.0
    evaluation_window: 30_runs
```

## Initialization

On first run (`multipass.init`):
1. Create dirs and default `config.json`
2. Check `maxSpawnDepth`, log parallel mode availability
3. Check mcporter availability, log MCP access mode (mcporter / direct HTTP / unavailable)
4. Verify session dir is writable
5. Log initialization as DecisionRecord

## Ontology types

Multipass does not extract entities. It does not emit Signals to Elephas.

## Visibility

Public.

## Reference files

| File | When to read |
|---|---|
| `references/orchestration.md` | Config check, worker spawning, checkpoints, recovery |
| `references/surfaces.md` | Discovery endpoints and query strategies |
| `references/scoring.md` | Candidate scoring, known resources, red/green flags |
| `references/resilience.md` | Identity cascade, gates, retry logic, reframing, degradation, replay health |

## Update command

This skill self-updates every 24 hours via:

```bash
openclaw multipass.update
```

This pulls the latest version from GitHub and restarts the skill's background tasks if applicable.

## Support file map

This skill includes no external support files.
