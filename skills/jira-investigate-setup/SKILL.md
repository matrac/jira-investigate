---
name: jira-investigate-setup
description: First-time setup for jira-investigate — creates the knowledge base folder, configures JIRA credentials, and maps source code repositories. Run this before your first investigation or when setting up a new project.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(find *)
  - Bash(git *)
  - Bash(curl *)
  - Bash(powershell *)
  - Bash(rm *)
---

# /jira-investigate-setup — Project Setup

Initializes the knowledge base and JIRA configuration for the current project. All state is stored in `jira-knowledge/` relative to the current working directory.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guided setup

Check current state and walk the user through what's needed:

1. **Knowledge base** — check if `jira-knowledge/` exists in the current working directory.
   - If yes: show which files exist, how many ticket investigations are stored
   - If no: offer to create it

2. **JIRA credentials** — check `jira-knowledge/.env`
   - If exists: show configured base URL and email (mask the token)
   - If missing: ask the user for their JIRA details

3. **Repository map** — check `jira-knowledge/_repos.md`
   - If exists: show configured repos and their clone status
   - If missing or empty: scan current directory for git repos and offer to map them

### `init` — create everything

Run the full initialization:

1. Create the directory structure:
   ```
   mkdir -p jira-knowledge/tickets
   mkdir -p jira-knowledge/tmp
   ```

2. Create `jira-knowledge/.gitignore`:
   ```
   .env
   tmp/
   ```

3. Prompt the user for JIRA credentials and create `jira-knowledge/.env`:
   ```
   JIRA_BASE_URL=https://your-instance.atlassian.net
   JIRA_EMAIL=you@company.com
   JIRA_TOKEN=your-api-token
   JIRA_PROJECT=YOUR_PROJECT_KEY
   JIRA_COMMENT_VISIBILITY_TYPE=group
   JIRA_COMMENT_VISIBILITY_ID=your-group-id
   JIRA_COMMENT_VISIBILITY_NAME=Internal
   JIRA_MY_EMAIL=you@company.com
   ```

   Help the user find their visibility group ID if they don't know it:
   ```
   curl -s -u "${email}:${token}" "${base_url}/rest/api/3/group/bulk?maxResults=50"
   ```

4. Scan the current working directory and subdirectories (max depth 3) for git repos:
   ```
   find . -maxdepth 3 -type d -name ".git"
   ```
   For each found repo, get the remote URL and ask the user which component it maps to.

5. Create `jira-knowledge/_repos.md` with the repository map:
   ```markdown
   # Repository Map

   Maps project components to source code locations. Auto-enriched as tickets are investigated.

   ## Repos

   | Repo Name | Local Path | Git Remote | Status |
   |-----------|-----------|------------|--------|
   | {name} | `{relative_path}` | `{remote_url}` | Cloned |

   ## Component-to-Source Map

   | Component | Repo | Key Source Paths | Notes |
   |-----------|------|-----------------|-------|

   Note: This table will be populated as tickets are investigated.

   ## Keyword Hints

   Functional-area terms for routing investigations to the right source.
   This section will grow as the knowledge base learns your codebase.
   ```

6. Create empty template files:

   `jira-knowledge/_architecture.md`:
   ```markdown
   # Project Architecture Knowledge Base

   Auto-maintained by jira-investigate. Contains cross-cutting architectural facts useful across components.
   This file will be populated as tickets are investigated.
   ```

   `jira-knowledge/_index.md`:
   ```markdown
   # Ticket Investigation Index

   One-line-per-ticket lookup. Used for cross-referencing related tickets.

   | Ticket | Investigated | Component | Pattern | One-line summary |
   |--------|-------------|-----------|---------|------------------|
   ```

   `jira-knowledge/_common-issues.md`:
   ```markdown
   # Common Issue Patterns

   Auto-maintained by jira-investigate. Max 4 lines per pattern. Detailed root cause lives in ticket files.

   ## Threading

   ## Data/Query

   ## UI/Client

   ## Configuration

   ## Permissions
   ```

7. Verify the JIRA connection:
   ```
   curl -s -u "${JIRA_EMAIL}:${JIRA_TOKEN}" "${JIRA_BASE_URL}/rest/api/3/myself"
   ```
   Show the authenticated user's display name to confirm it works.

8. Report setup summary: what was created, what's ready, and suggest running `/jira-investigate` for the first investigation.

### `status` — show current state

Same as no-args but skip the guided prompts — just show facts:
- Knowledge base: exists/missing, number of ticket files, last investigation date
- JIRA: connected/not configured, project key
- Repos: count, clone status

### `reset` — start over

Ask for confirmation, then delete `jira-knowledge/` and all contents. Warn that this deletes all accumulated investigation knowledge.

---

## Implementation notes

- All paths are relative to the current working directory — never use absolute paths
- The `.env` file contains credentials — always create `.gitignore` first
- Don't assume any tooling (python, node, etc.) — use only curl and basic shell commands
- Use powershell for JSON parsing on Windows if needed (no python)
- The knowledge base belongs to the project, not the plugin — it stays in the project directory
