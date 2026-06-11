Read a JIRA ticket, summarize it, propose an implementation plan, await approval, then implement the changes, create a branch, and move the ticket to In Progress. Uses the Atlassian MCP server when available, falls back to REST API automatically.

The argument is a JIRA ticket ID (e.g. `RRP-1234`). If no argument is provided, ask the user for the ticket ID before proceeding.

Follow these steps in order. Do not skip any step.

---

## Step 1: Load Jira profile and detect transport

Read `~/.claude/jira.json`.

- Parse the ticket prefix from the ticket ID (everything before the `-`). Example: `RRP` from `RRP-1234`.
- Find the matching profile under `profiles`. If no match, use the `default` profile key. If neither matches, stop and tell the user: "No Jira profile found for prefix `<PREFIX>`. Add it to `~/.claude/jira.json`."
- Extract `base_url` from the profile. You will need it throughout for browse URLs.

**Detect which transport to use:**

Check whether `mcp__atlassian__getJiraIssue` is available as a tool in the current session.

- If **available**: use MCP for all Jira API calls (Steps 2, 6). Note this as `transport = mcp`.
- If **not available**: use REST API for all Jira API calls (Steps 2, 6). Note this as `transport = rest`.
  - For REST, also extract `email` and `token_env` from the profile.
  - Read the API token: `echo "${JIRA_API_TOKEN}"` (or the value of `token_env`).
  - If the token is empty, stop and tell the user: "Environment variable `<TOKEN_ENV>` is not set. Export it in your shell profile."

---

## Step 2: Fetch the ticket

**If transport = mcp:**

Call `mcp__atlassian__getJiraIssue` with the ticket ID. If the call fails, show the error and stop.

**If transport = rest:**

```bash
curl -s -f \
  -u "<EMAIL>:<TOKEN>" \
  -H "Accept: application/json" \
  "<BASE_URL>/rest/api/3/issue/<TICKET_ID>?fields=summary,description,issuetype,status,priority,assignee,reporter,subtasks,issuelinks,labels,fixVersions,comment,customfield_10016"
```

(`customfield_10016` is the standard story-points field — include it if present, ignore if missing.)

If the curl fails (non-zero exit), stop and show the HTTP error to the user.

---

## Step 3: Parse and display the ticket summary

Display in a clean readable format:

- **Ticket**: ID and URL (`<BASE_URL>/browse/<TICKET_ID>`)
- **Type**: issue type (Story, Bug, Task, Sub-task, etc.)
- **Status**: current status
- **Priority**: priority level
- **Summary**: the ticket title
- **Description**:
  - MCP: display as-is (MCP returns pre-rendered text). If empty, show "No description."
  - REST: render the Atlassian Document Format (ADF) body as plain text. If the description uses ADF (`"type": "doc"`), recursively walk the `content` tree:
    - `paragraph` → its text on its own line
    - `heading` → `## <text>`
    - `bulletList` / `orderedList` → each `listItem` as `- <text>` or `1. <text>`
    - `codeBlock` → wrap in triple backticks
    - `blockquote` → prefix lines with `>`
    - `text` nodes → their `text` value (apply `bold`/`italic` marks if present)
    - Any unknown node type → extract any nested `text` nodes and concatenate
    - If plain text (not ADF), show as-is. If null or empty, show "No description."
- **Acceptance Criteria**: look for a section heading containing "acceptance criteria" (case-insensitive) in the description; extract the content under it. If not found, note "Not specified."
- **Subtasks**: list each subtask ID + summary, or "None."
- **Linked Issues**: list each link type + linked issue ID + summary (e.g. `blocks RRP-100: Some other task`), or "None."
- **Story Points**: value of story points field, or "Not set."

---

## Step 4: Load project context

Read the project `CLAUDE.md` in the current working directory if it exists.

Then check if an `agent_docs/` directory exists. If it does, read any files inside it that are relevant to implementation (e.g. architecture rules, coding standards, feature patterns, domain overview, API conventions). Only read files that exist — skip any that don't.

If neither `CLAUDE.md` nor `agent_docs/` exists, ask the user:
> "I couldn't find any project documentation (`CLAUDE.md` or `agent_docs/`). Could you briefly describe the tech stack, architecture patterns, and any coding conventions I should follow? Or I can generate a `CLAUDE.md` for this project — just say the word."

Save any context the user provides in memory for future use in this project. Internalize all loaded context — it constrains every decision in the plan and implementation.

---

## Step 5: Propose an implementation plan

Before proposing a plan, check if the ticket description is clear enough to act on:
- If the description is missing, too vague, or contradictory — stop and ask the user to clarify before proceeding. List the specific questions you need answered.
- If the description is clear enough to proceed, continue below.

Based on the ticket content, analyze the codebase as needed (read relevant files, search for related code) and propose a concrete implementation plan. Structure it as:

### Understanding
One paragraph explaining what this ticket is asking for in your own words.

### TODO
Numbered list of concrete implementation steps.

### Dependencies
- Any tickets this blocks or is blocked by (from linked issues)
- Any code modules, APIs, or files that need to be touched or understood first
- Any open questions you have for the user before starting

### Proposed Branch Name

Check the issue type against these rules:

- `feat/` — Story, Task, Sub-task (including Japanese variants: サブタスク, 軽微なサブタスク, バグ（サブタスク）treated as sub-task), New Feature, Improvement, Change Request
- `bugfix/` — Bug (including Internal Bug and Japanese variants)
- **Any other type** (Epic, Initiative, Release, Test, Incident, Q&A, Ringi, Report, Workstream, Sprint goal, Precondition, etc.) — stop and tell the user: "This ticket type (`<TYPE>`) is not a coding ticket. No branch will be created." Do not proceed with Steps 6–7.

For valid types, suggest: `<type>/<TICKET-ID>-<short-kebab-slug>`

End with: "Does this plan look correct? Please answer any open questions above, then approve to proceed."

**Do not proceed further until the user explicitly approves.**

---

## Step 6: On user approval — transition ticket to In Progress

Check the current status from the ticket data fetched in Step 2.

- If the status already contains "In Progress" (case-insensitive), skip and tell the user: "Ticket is already In Progress."
- Otherwise:

**If transport = mcp:**

Call `mcp__atlassian__getTransitionsForJiraIssue` with the ticket ID. Find the transition whose name contains "In Progress" (case-insensitive). If multiple matches, prefer the closest. If none found, list available names and ask the user which to use. Execute via `mcp__atlassian__transitionJiraIssue`.

**If transport = rest:**

```bash
curl -s -f \
  -u "<EMAIL>:<TOKEN>" \
  -H "Accept: application/json" \
  "<BASE_URL>/rest/api/3/issue/<TICKET_ID>/transitions"
```

Find the transition whose `name` contains "In Progress" (case-insensitive). If none found, list available names and ask the user which to use.

```bash
curl -s -f -X POST \
  -u "<EMAIL>:<TOKEN>" \
  -H "Content-Type: application/json" \
  -d "{\"transition\": {\"id\": \"<TRANSITION_ID>\"}}" \
  "<BASE_URL>/rest/api/3/issue/<TICKET_ID>/transitions"
```

Confirm to the user: "Ticket moved to In Progress."

---

## Step 7: Create the branch

If the ticket type was invalid (not in the feat/bugfix list), this step was already aborted in Step 5.

Use the branch name proposed in Step 5 (or the name the user corrected it to).

Determine the base branch:

```bash
git remote show origin | grep 'HEAD branch' | awk '{print $NF}'
```

Fall back to `git branch -r | grep -E 'origin/(main|develop)'` if that fails; prefer `develop` over `main`.

Fetch and check branch existence:

```bash
git fetch origin
git branch --list <BRANCH_NAME>
git branch -r --list origin/<BRANCH_NAME>
```

- If it exists locally: `git checkout <BRANCH_NAME>` — tell the user: "Branch `<BRANCH_NAME>` already exists — switched to it."
- If it exists on remote only: `git checkout -b <BRANCH_NAME> origin/<BRANCH_NAME>` to track it.
- If it does not exist: `git checkout -b <BRANCH_NAME> origin/<BASE_BRANCH>` to create from base.

Confirm: "Branch `<BRANCH_NAME>` ready."

---

## Step 8: Implement the plan

Execute every item in the TODO list from Step 5, following all rules loaded in Step 4 (architecture, coding standards, screen implementation rules, etc.).

Work through the items in order. For each item:
- Read existing files before editing them.
- Apply all architecture patterns, coding standards, and conventions loaded in Step 4.
- Write no comments unless the WHY is non-obvious.

After completing all items, if the total number of changed files exceeds 10, run a compile check appropriate for the project type:

- **Android / KMP**: `./gradlew assembleDebug`
- **iOS / Swift**: `xcodebuild -scheme <SCHEME> build`
- **Node.js / TypeScript**: `npm run build` or `tsc --noEmit`
- **Python**: `python -m py_compile <changed files>` or run the project's lint/type-check command
- **Other**: look for a build or type-check script in `package.json`, `Makefile`, or the project's `CLAUDE.md` / documentation

If no compile check is applicable or the project type is unclear, skip this step.

Fix any build errors before continuing.