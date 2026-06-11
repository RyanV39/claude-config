Post a comment on a JIRA ticket. Typically used to link a PR after it is created or merged. Uses the Atlassian MCP server when available, falls back to REST API automatically.

Arguments: `<TICKET_ID> <COMMENT_TEXT_OR_URL>`

Example: `RRP-1234 https://github.com/org/repo/pull/42`

If arguments are missing, ask the user for the ticket ID and the comment text before proceeding.

---

## Step 1: Detect transport and load profile

Check whether `mcp__atlassian__addCommentToJiraIssue` is available as a tool in the current session.

- If **available**: use MCP (`transport = mcp`). No credentials needed.
- If **not available**: use REST (`transport = rest`).
  - Read `~/.claude/jira.json`.
  - Parse the ticket prefix (everything before the `-`) and find the matching profile. If no match, use `default`. If neither matches, stop and tell the user to add the profile to `~/.claude/jira.json`.
  - Extract `base_url`, `email`, `token_env`.
  - Read the API token from the env var named by `token_env` (default: `JIRA_API_TOKEN`). If empty, stop and tell the user to export it.

---

## Step 2: Format the comment

If the comment text looks like a URL (starts with `http`), format it as: `PR: <URL>`

Otherwise use the comment text as-is.

---

## Step 3: Post the comment

**If transport = mcp:**

Call `mcp__atlassian__addCommentToJiraIssue` with the ticket ID and formatted comment text.

**If transport = rest:**

```bash
curl -s -f -X POST \
  -u "<EMAIL>:<TOKEN>" \
  -H "Content-Type: application/json" \
  -d "{\"body\": {\"type\": \"doc\", \"version\": 1, \"content\": [{\"type\": \"paragraph\", \"content\": [{\"type\": \"text\", \"text\": \"<COMMENT_TEXT>\"}]}]}}" \
  "<BASE_URL>/rest/api/3/issue/<TICKET_ID>/comment"
```

If the curl fails, show the error to the user.

Confirm: "Comment posted to <TICKET_ID>."
