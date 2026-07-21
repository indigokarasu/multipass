# multipass

<p align="center">
<img src="./assets/readme/hero.jpg" width="100%" alt="Accomplish tasks that need tools the agent does not have. Autonomous, isolated, fire-and-forget.">
</p>

multipass — Accomplish tasks that need tools the agent does not have. Autonomous, isolated, fire-and-forget.


> Tell it what you need. It does the work.

## Dependencies

**Optional skill cooperation**
- [Sift](https://github.com/indigokarasu/sift) -- deeper candidate research during discovery; falls back to web_search if absent
- [Forge](https://github.com/indigokarasu/forge) -- suggested after completion when a permanent skill would be more appropriate
- [mcporter](https://github.com/steipete/mcporter) -- MCP server access; falls back to direct HTTP JSON-RPC if not installed
- [ivangdavila/api](https://github.com/ivangdavila/api) -- 147-service reference checked before discovery to avoid redundant searches

**External**
- None required. Multipass provisions temporary throwaway identities for services that require signup.

---

*multipass is part of the [OCAS Agent Suite](https://github.com/indigokarasu).*