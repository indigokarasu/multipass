# Multipass

Multipass accomplishes tasks that require tools you don't currently have. Invoke `multipass.run {task}`, Multipass runs to completion autonomously, you get the output. No check-ins, no approval gates, no escalation.


Skill packages follow the [agentskills.io](https://agentskills.io/specification) open standard and are compatible with OpenClaw, Hermes Agent, and any agentskills.io-compliant client.

Everything happens inside a session directory. Multipass plans multiple approaches, discovers and provisions the best available tool for each capability gap, executes with incremental checkpoints and graceful degradation, and writes a task result plus a standalone replay script. If no approach succeeds, it produces a failure report documenting every path tried -- a useful artifact even when the task can't be completed.

---

## Commands

| Command | Description |
|---|---|
| `multipass.run {task}` | Full autonomous lifecycle: plan, discover, execute, report |
| `multipass.search {description}` | Discovery only -- find tools for a capability without executing |
| `multipass.sessions` | List recent sessions with status and one-line outcome |
| `multipass.replay {session_id}` | Re-execute a replay script in a new session with a new identity |
| `multipass.status` | Runtime config, mcporter availability, source health |
| `multipass.update` | Pull latest version from GitHub source (preserves data and journals) |

## Setup

`multipass.init` runs automatically on first invocation. It creates all required directories, writes a default `config.json`, checks `maxSpawnDepth` for parallel mode availability, and checks mcporter installation for MCP server access.

## Dependencies

**Optional skill cooperation**
- [Sift](https://github.com/indigokarasu/sift) -- deeper candidate research during discovery; falls back to `web_search` if absent
- [Forge](https://github.com/indigokarasu/forge) -- suggested after completion when a permanent skill would be more appropriate
- [mcporter](https://github.com/steipete/mcporter) -- MCP server access; falls back to direct HTTP JSON-RPC if not installed
- [ivangdavila/api](https://github.com/ivangdavila/api) -- 147-service reference checked before discovery to avoid redundant searches

**External**
- None required. Multipass provisions temporary throwaway identities for services that require signup.

## Storage

```
{agent_root}/commons/data/ocas-multipass/
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
      workspace/
      output/
{agent_root}/commons/journals/ocas-multipass/
  YYYY-MM-DD/{run_id}.json
```

## Changelog

### v4.0.0 -- April 4, 2026
- Final packaging release: README, CHANGELOG, journal spec, full skill.json

### v3.0.0 -- April 4, 2026
- Initial release: four-phase autonomous workflow (Plan, Fill gaps, Execute, Report)
- Adaptive complexity, circuit breakers, disposable identity cascade, candidate scoring, parallel discovery workers

---

*Multipass is part of the [OpenClaw Agent Suite](https://github.com/indigokarasu) -- a collection of interconnected skills for personal intelligence, autonomous research, and continuous self-improvement.*
