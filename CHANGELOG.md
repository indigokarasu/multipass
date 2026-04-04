## [2026-04-04] Spec Compliance Update

### Changes
- Added missing SKILL.md sections per ocas-skill-authoring-rules.md
- Updated skill.json with required metadata fields
- Ensured all storage layouts and journal paths are properly declared
- Aligned ontology and background task declarations with spec-ocas-ontology.md

### Validation
- ✓ All required SKILL.md sections present
- ✓ All skill.json fields complete
- ✓ Storage layout properly declared
- ✓ Journal output paths configured
- ✓ Version: 4.0.0 → 4.0.1

# Changelog

## [4.0.0] - 2026-04-04

### Added
- README.md: user-facing overview with commands, setup, dependencies, and storage layout
- CHANGELOG.md: version history
- references/journal.md: Action Journal and Research Journal specification per spec-ocas-journal.md v1.3
- skill.json: skill_type, filesystem read/write permissions, self_update configuration

## [3.0.0] - 2026-04-04

### Added
- Initial release: four-phase autonomous workflow (Plan, Fill gaps, Execute, Report)
- Adaptive complexity -- inline search for simple tasks, parallel workers for complex
- Circuit breakers and loop prevention (references/resilience.md)
- Disposable identity cascade: BotEmail, Mail.tm, 1secmail, Guerrilla Mail
- Candidate scoring rubric: relevance, trust, cost, effort axes (references/scoring.md)
- Discovery surfaces: MCP Registry, ClawHub, Skills.sh, GitHub, public APIs (references/surfaces.md)
- Orchestrator/worker checkpoint architecture with recovery paths (references/orchestration.md)
- Commands: multipass.run, multipass.search, multipass.sessions, multipass.replay, multipass.status
