# 🔍 jira-investigate

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-green.svg)](CHANGELOG.md)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-7c3aed.svg)](https://claude.com/claude-code)
[![JIRA Cloud](https://img.shields.io/badge/JIRA-Cloud-0052CC.svg)](https://www.atlassian.com/software/jira)

A Claude Code plugin that investigates JIRA Code Defect tickets by reading the ticket, analyzing source code, identifying root causes, and posting findings back -- with a self-improving knowledge base.

## What it does

On each run, the agent:

1. Picks the latest unhandled Code Defect from your JIRA project
2. Reads everything: description, comments, attachments (decompresses logs, greps for exceptions)
3. Cross-references a knowledge base of past investigations to find related bugs
4. Searches your codebase with parallel agents to trace the root cause
5. Assesses its own confidence before posting (won't comment if it can't find the code)
6. Posts a formatted analysis with code snippets back to JIRA
7. Updates its knowledge base so the next investigation is faster

## Install

```
/plugin marketplace add matrac/jira-investigate
/plugin install jira-investigate
```

Then restart Claude CLI for the skills to load.

## Setup

Run the setup skill first:

```
/jira-investigate-setup init
```

This will:
- Create the `jira-knowledge/` folder in your project
- Ask for your JIRA credentials (base URL, email, API token)
- Scan for git repos and help you map components
- Create the knowledge base skeleton

## Usage

```
/jira-investigate              # auto-pick latest unhandled ticket
/jira-investigate ABC-1234      # investigate a specific ticket
/jira-investigate-setup                    # check status
/jira-investigate-setup init               # first-time setup
```

## Knowledge base

The plugin creates a `jira-knowledge/` folder in your project root:

```
jira-knowledge/
├── .env                    # JIRA credentials (gitignored)
├── .gitignore
├── _architecture.md        # Cross-cutting architectural patterns
├── _index.md               # One-line-per-ticket lookup table
├── _common-issues.md       # Recurring bug patterns by category
├── _repos.md               # Component-to-repo-to-source map
├── tickets/
│   ├── ABC-1234.md          # Per-ticket investigation record
│   └── ...
└── tmp/                    # Temporary files (auto-cleaned)
```

The knowledge base is **self-improving**: each investigation enriches the architecture notes, bug patterns, and component maps. After 5-10 tickets, investigations become significantly faster because the agent already knows where to look.

## How it works

### Tiered knowledge (scales to 100+ tickets)

- **Always loaded** (~170 lines): architecture, index, repos, common issues
- **On-demand** (0-3 files): related ticket details loaded only when relevant
- **Growth**: ~1 line per ticket (index row), not proportional to investigation detail

### Quality gate

The agent self-assesses before posting to JIRA:
- **HIGH confidence**: exact line(s) of code found, concrete fix -- posts normally
- **MEDIUM confidence**: likely area found -- posts with "Probable root cause:" prefix
- **LOW confidence**: can't locate the code -- does NOT post, creates ticket file with open questions

### Re-investigation

Running on a ticket that was already investigated appends a "Re-investigation" section instead of duplicating work.

## Customization

The JIRA query in the investigation skill is preconfigured to search for:

- **Issue type**: `Code Defect`
- **Status**: `Open` or `Investigating`

These may not match your project. Review and edit `skills/jira-investigate/SKILL.md` -- look for the JQL query in Step 1 and adjust the `issuetype` and `status` values to match your JIRA workflow.

Similarly, the comment visibility is set to post to an `Internal` group. If your project uses a different group or no restricted visibility, update the visibility settings in `jira-knowledge/.env` after setup.

## Requirements

- Claude Code
- JIRA Cloud instance with API access
- One or more git repositories with the source code
- Windows (uses powershell for JSON parsing) -- Linux/Mac support: replace powershell commands with jq

## License

MIT
