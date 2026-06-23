# Changelog

## [0.2.1] - 2026-06-23

### Fixed
- Restored the multi-agent investigation default that was unintentionally altered in 0.2.0 (investigation behavior is unchanged from 0.1.0)

## [0.2.0] - 2026-06-23

### Changed
- Renamed the setup skill to `setup` — invoke as `/jira-investigate:setup` (the previously documented `/jira-investigate-setup` did not resolve)
- Comment signature is now configurable via `JIRA_COMMENT_SIGNATURE` in `.env` (default: "AI Investigator"); previously hardcoded
- Generalized investigation guidance — removed the project-specific Java/Swing comparison and the fixed "5 agents" default in favor of complexity-scaled parallel investigation

## [0.1.0] - 2026-04-06

### Added
- Initial release
- `/jira-investigate` skill -- auto-detects latest unhandled Code Defect, reads ticket + attachments, searches codebase with parallel agents, posts findings to JIRA
- `/jira-investigate-setup` skill -- first-time setup with credential config, repo mapping, and knowledge base scaffolding
- Self-improving knowledge base with tiered architecture (always-loaded index + on-demand ticket details)
- Quality gate with HIGH/MEDIUM/LOW confidence assessment before posting
- Compressed attachment handling (.gz, .zip)
- Re-investigation support for previously analyzed tickets
- ADF-formatted JIRA comments with code snippets, signed as "mAItraBOTi"
- Comment visibility control (configurable group restriction)
