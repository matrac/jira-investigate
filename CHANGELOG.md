# Changelog

## [0.3.0] - 2026-10-01

### Changed
- Multi-agent investigation is now concrete and mandatory: five named roles (evidence analyst, code-path tracer, change historian, sibling comparer, skeptic) launched in parallel, a fixed answer format per agent, a merge rule, a debate round when agents disagree and a verification round if still open. Investigation Notes record the hypotheses considered and why alternatives were rejected.
- Hardened for unattended runs: Bash-tool-only shell rules, grep-only loading of `_index.md` and `_common-issues.md`, read-only tool constraints for sub-agents.

### Evaluation
- On 54 resolved Code Defects with tested fixes (2 runs each), root cause matched the developers' fix in 90% of runs vs 84% for 0.2.1, with fewer wrong answers (1 vs 3). Cost and time per ticket are about 3x and 2.5x higher.

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
