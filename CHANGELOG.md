# Changelog

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
