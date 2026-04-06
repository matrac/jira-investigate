---
name: jira-investigate
description: Investigate the latest JIRA Code Defect — reads the ticket, analyzes source code, identifies the root cause, and posts findings back to JIRA. Optionally pass a specific ticket key like ABC-123.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(curl *)
  - Bash(find *)
  - Bash(git *)
  - Bash(powershell *)
---

# /jira-investigate — JIRA Code Defect Investigator

You are investigating the latest JIRA Code Defect ticket. Follow these steps precisely.

IMPORTANT: Do not use python (any version). Use powershell for JSON parsing where needed.

## Input
Optional argument: `$ARGUMENTS` (a specific ticket key like `ABC-123`). If empty, auto-detect the latest unhandled ticket.

## Step 0: Load Environment & Knowledge Base

1. Read the `.env` file at `jira-knowledge/.env` and extract all JIRA variables. Use these for all API calls. If the file doesn't exist, tell the user to run `/jira-investigate-setup init` first and stop.
2. Read these knowledge base files (skip if they don't exist yet):
   - `jira-knowledge/_architecture.md`
   - `jira-knowledge/_index.md`
   - `jira-knowledge/_repos.md`
   - `jira-knowledge/_common-issues.md`
3. These give you accumulated context about the codebase, component locations, and recurring bug patterns. Use them to guide your investigation.

## Step 1: Identify the Target Ticket

**If a ticket key was provided in `$ARGUMENTS`**, use that directly.

**If no ticket key was provided**, find the latest unhandled Code Defect:

1. Query JIRA for recent Code Defects with status Open or Investigating, use powershell for JSON parsing:
   ```
   curl -s -u "${JIRA_EMAIL}:${JIRA_TOKEN}" \
     "${JIRA_BASE_URL}/rest/api/3/search/jql?jql=project%3D${JIRA_PROJECT}%20AND%20issuetype%3D%22Code%20Defect%22%20AND%20status%20in%20(Open%2C%20Investigating)%20ORDER%20BY%20created%20DESC&maxResults=30&fields=key,summary,status,created" | powershell -Command "\$json = \$input | ConvertFrom-Json; \$json.issues | ForEach-Object { \$_.key + '\t' + \$_.fields.status.name + '\t' + \$_.fields.summary }"
   ```

2. Check which tickets already have a `tickets/ABC-1234.md` file in `jira-knowledge/tickets/`.

3. For each candidate (newest first), fetch its comments and check if any comment was authored by `JIRA_MY_EMAIL`. Skip tickets you've already commented on.

4. Pick the first (newest) ticket that has NO comment from your account AND no existing ticket .md file.

**Re-investigation**: If the specified ticket already has a .md file in tickets/, you are RE-INVESTIGATING.
- Read the existing ticket file before starting investigation
- Keep the existing structure but update sections with new findings
- Append a `## Re-investigation ({today's date})` section documenting what changed
- Do NOT post a new JIRA comment unless you found something materially different from the existing analysis

## Step 1.5: Check for Related Prior Tickets

1. Read `jira-knowledge/_index.md` (already loaded in Step 0)
2. Extract component names and key terms from the target ticket's summary
3. Search the index for tickets matching the same component or pattern keywords
4. For each match (max 3), read the first 20 lines of the ticket .md file (Summary + Root Cause header only)
5. Note related ticket keys — include these in your output and use them to inform your investigation

## Step 2: Deep-Read the Ticket

Fetch full ticket details including description, all comments, and attachment list:

```
curl -s -u "${JIRA_EMAIL}:${JIRA_TOKEN}" \
  "${JIRA_BASE_URL}/rest/api/3/issue/{TICKET_KEY}?fields=summary,description,comment,attachment,status,priority,labels,components"
```

Parse and understand:
- **Description**: The full bug report (in ADF format — extract the text content)
- **All comments**: Read every comment for additional context, reproduction steps, logs, investigation notes
- **Attachments**: List all attachments. For each relevant attachment (logs, screenshots, configs):
  - Download via: `curl -s -u "${JIRA_EMAIL}:${JIRA_TOKEN}" -o jira-knowledge/tmp/{filename} "{attachment_content_url}"`
  - For **compressed files** (.gz, .zip, .tar.gz): Decompress first using powershell:
    - `.gz`: `powershell -Command "[System.IO.Compression.GzipStream]::new([System.IO.File]::OpenRead('jira-knowledge/tmp/{filename}'), [System.IO.Compression.CompressionMode]::Decompress).CopyTo([System.IO.File]::Create('jira-knowledge/tmp/{filename_without_gz}'))"`
    - `.zip`: `powershell -Command "Expand-Archive -Path 'jira-knowledge/tmp/{filename}' -DestinationPath 'jira-knowledge/tmp/' -Force"`
    Then treat the decompressed file as a log/text file per the rules below.
  - For **log files** (can be huge): Do NOT read the whole file. Instead:
    1. First check file size with `ls -la`
    2. If > 500 lines, use Grep to search for: `Exception`, `ERROR`, `WARN`, `Caused by`, and keywords from the ticket summary
    3. Read only the surrounding context (-C 20) of matches
    4. If no grep hits, read just the first 50 and last 50 lines
  - For **screenshots/images**: Read them (Claude can view images)
  - For **config/text files**: Read them fully if small, grep if large

Produce a clear understanding of: What is the reported problem? What are the symptoms? What component is affected?

## Step 3: Code Investigation

Use the `_repos.md` component-to-source map and keyword hints to identify which repo(s) and source directories to search.

**If a needed repo is marked "Not cloned"**:
- Try to clone it: `git clone {remote_url} {local_path}`
- If clone fails (permissions, etc.), note this in findings and investigate what you can from other repos

**Investigation strategy**:
0. Spin up 5 agents specialized and do 3 iterations of the below 6 points and debate
1. Extract key error messages, class names, method names, or identifiers from the ticket
2. Use Grep to search across the relevant source paths
3. Use the Explore agent for deep dives when needed — give it specific search objectives
4. Compare HTML/web implementation with Java/Swing implementation if the bug is about behavioral differences
5. Cross-reference with `_common-issues.md` — does this match a known pattern?
6. Trace the code path from the symptom to the root cause

**Important**: Be thorough but focused. Don't read the entire codebase — use the knowledge base to zero in on the right area quickly.

## Step 3.5: Quality Gate

Before producing outputs, verify your findings:

1. **Root cause check**: Can you point to a specific line of code or identifiable code pattern? If not, go back to Step 3 with more targeted searches.

2. **Fix specificity check**: Is the suggested fix concrete — naming a specific file and a specific change? "Needs further investigation" is NOT an acceptable fix.

3. **Confidence assessment**:
   - **HIGH**: Found exact line(s) of code, traced the execution flow, fix is specific and clear
   - **MEDIUM**: Found the likely code area and plausible root cause, but could not line-confirm every detail
   - **LOW**: Could not locate the relevant code or working from symptoms/logs only

4. **If LOW confidence**:
   - Do NOT post a JIRA comment
   - Still create the ticket .md file, marking Confidence as LOW
   - List specific open questions in the Investigation Notes section
   - Report to user that a confident conclusion was not reached

5. **If MEDIUM confidence**: Post the JIRA comment but prefix root cause with "Probable root cause:" and note the uncertainty.

## Step 4: Produce Outputs

### 4a. JIRA Comment

Compose a concise, well-formatted ADF (Atlassian Document Format) comment with:
- **Root cause** in 2-3 sentences maximum (prefix with "Probable root cause:" if MEDIUM confidence)
- **Code snippet** if it helps clarify (use ADF `codeBlock` node)
- **Suggested fix** in 1-2 sentences
- **Key files** involved (as a short list)
Sign it as "mAItraBOTi"

Post with the configured visibility:

```
curl -s -u "${JIRA_EMAIL}:${JIRA_TOKEN}" \
  -X POST -H "Content-Type: application/json" \
  "${JIRA_BASE_URL}/rest/api/3/issue/{TICKET_KEY}/comment" \
  -d @jira-knowledge/tmp/jira_comment.json
```

The comment JSON must use this visibility block:
```json
{
  "visibility": {
    "type": "group",
    "identifier": "{JIRA_COMMENT_VISIBILITY_ID}"
  },
  "body": { ... ADF body ... }
}
```

Use ADF format for the body. Example structure:
```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "Root cause: ", "marks": [{"type": "strong"}] },
        { "type": "text", "text": "Description of root cause here." }
      ]
    },
    {
      "type": "codeBlock",
      "attrs": { "language": "java" },
      "content": [
        { "type": "text", "text": "// relevant code snippet" }
      ]
    },
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "Fix: ", "marks": [{"type": "strong"}] },
        { "type": "text", "text": "Description of fix here." }
      ]
    }
  ]
}
```

**Always write the JSON payload to `jira-knowledge/tmp/jira_comment.json` first**, then use `-d @jira-knowledge/tmp/jira_comment.json` to avoid shell escaping issues.

### 4b. Ticket Knowledge File

Create/update `jira-knowledge/tickets/{TICKET_KEY}.md` with:
```markdown
# {TICKET_KEY}: {Summary}

- **Created**: {date}
- **Investigated**: {today's date}
- **Status**: {status}
- **Type**: Code Defect
- **Confidence**: HIGH | MEDIUM | LOW
- **Related tickets**: ABC-1234, ABC-5678 — or "None found"

## Summary
{What the bug is about, in your own words}

## Root Cause
{Detailed technical explanation}

## Fix
{What needs to change and where}

## Key Files
- {list of relevant source files with paths relative to repo root}

## Related Tickets
- **ABC-1234**: {one-line explanation of the relationship}
(or "None found")

## Investigation Notes
{Any additional context, open questions if confidence < HIGH}
```

### 4c. Update Knowledge Base Files

Update ONLY if you discovered something genuinely new and reusable:

- **`_architecture.md`**: Add ONLY if you discovered a cross-cutting architectural pattern that would help investigate a DIFFERENT component's bug in the future. Per-ticket findings do NOT belong here. The ticket .md file IS the detailed record.

- **`_index.md`**: ALWAYS append one row for the new ticket to the table (mandatory, never skip).

- **`_common-issues.md`**: Update ONLY if:
  (a) This bug matches an existing pattern -> add the ticket key to that pattern's "Seen in" list, OR
  (b) This bug represents a genuinely NEW recurring pattern -> add a new 4-line entry (Pattern + Symptom + Fix + Seen in)
  Do NOT add one-off bugs that are specific to a single method/class.
  **Pruning**: If `_common-issues.md` exceeds 25 patterns, remove the oldest pattern that has been "Seen in" only 1 ticket AND was investigated more than 60 days ago. The ticket file serves as cold storage.

- **`_repos.md`**: Add new component rows to the table if a previously unknown component was investigated. Add keyword hints ONLY for functional-area terms (not specific class/method names — those go in the table Notes column). Check for duplicates against the table Notes column before adding.

IMPORTANT: Do NOT create per-ticket learning entries in `_architecture.md`. The ticket .md file already captures all investigation detail.

**Do NOT rewrite these files from scratch** — use the Edit tool to append or update specific sections. Preserve all existing content.

### 4d. Clean Up Temporary Files

Remove all files from tmp/ to prevent stale data accumulation:
```
find jira-knowledge/tmp/ -type f -delete
```

## Step 5: Report to User

Summarize what you found and did:
1. Which ticket you investigated (and any related tickets found)
2. The root cause (1-2 sentences)
3. The fix (1 sentence)
4. Confidence level (HIGH/MEDIUM/LOW)
5. Confirm the JIRA comment was posted (or explain why not if LOW confidence)
6. List which knowledge files were updated
