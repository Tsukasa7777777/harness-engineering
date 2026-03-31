# Integration Tests for Tool/MCP Connections

This file defines integration tests that verify each tool and MCP connection is working correctly. These tests should be run:

- After initial setup of a new MCP server
- After any credential rotation or auth change
- When a tool starts returning unexpected errors
- As part of a periodic health check (weekly recommended)

## How to Use This File

1. **Find the tool** you want to test in the appropriate section below.
2. **Run the test** by executing the described operation (manually or via script).
3. **Compare** actual output against expected output.
4. **Record** the result in the `Actual Output`, `Pass/Fail`, and `Last Tested` columns.
5. **If a test fails**, check `mcp-registry.md` for auth status and the tool's error messages for recovery steps.

## Test Result Legend

| Result | Meaning |
|---|---|
| PASS | Output matches expected; tool is operational |
| FAIL | Output does not match or tool returns error |
| SKIP | Test cannot be run (e.g., missing credentials, service unavailable) |
| FLAKY | Passes sometimes, fails other times (investigate) |

---

## 1. GitHub

### Test 1.1: List Repository Issues

| Field | Value |
|---|---|
| **Tool** | `github_list_issues` / `list_issues` |
| **MCP Server** | `@modelcontextprotocol/server-github` |
| **Purpose** | Verify GitHub authentication and read access |
| **Preconditions** | GitHub PAT is valid; target repository exists and has at least 1 issue |
| **Test Input** | `owner: "<your-org>", repo: "<your-repo>", state: "open", per_page: 1` |
| **Expected Output** | JSON array with at least 1 issue object containing `id`, `title`, `state`, `created_at` fields. HTTP 200. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | If 401, check PAT expiry. If 404, check owner/repo spelling. |

### Test 1.2: Create Issue

| Field | Value |
|---|---|
| **Tool** | `github_create_issue` / `create_issue` |
| **MCP Server** | `@modelcontextprotocol/server-github` |
| **Purpose** | Verify GitHub write access |
| **Preconditions** | GitHub PAT has `repo` scope; target repository allows issue creation |
| **Test Input** | `owner: "<your-org>", repo: "<test-repo>", title: "[TEST] Integration test - delete me", body: "Automated integration test. Safe to delete.", labels: ["test"]` |
| **Expected Output** | JSON object with `id`, `number`, `html_url` fields. Issue appears in the repository. HTTP 201. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Cleanup** | Close and delete the test issue after verification. |
| **Notes** | Use a dedicated test repo to avoid polluting production. |

### Test 1.3: Search Code

| Field | Value |
|---|---|
| **Tool** | `github_search_code` / `search_code` |
| **MCP Server** | `@modelcontextprotocol/server-github` |
| **Purpose** | Verify code search functionality |
| **Preconditions** | Target repository is indexed by GitHub search |
| **Test Input** | `q: "README repo:<your-org>/<your-repo>"` |
| **Expected Output** | JSON with `items` array containing at least 1 result with `path`, `repository` fields. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | GitHub search index can lag by minutes. If no results, wait and retry. |

---

## 2. File System / Local Tools

### Test 2.1: Read File

| Field | Value |
|---|---|
| **Tool** | `Read` (built-in) |
| **MCP Server** | N/A (native tool) |
| **Purpose** | Verify file read access |
| **Preconditions** | Target file exists at known path |
| **Test Input** | `file_path: "<absolute-path-to-known-file>"` |
| **Expected Output** | File contents returned with line numbers. No permission error. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Test with both small and large files. Verify line number format. |

### Test 2.2: Write File

| Field | Value |
|---|---|
| **Tool** | `Write` (built-in) |
| **MCP Server** | N/A (native tool) |
| **Purpose** | Verify file write access |
| **Preconditions** | Target directory exists and is writable |
| **Test Input** | `file_path: "/tmp/tool-integration-test.txt", content: "Integration test content.\nLine 2."` |
| **Expected Output** | File created successfully. Reading it back returns the exact content. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Cleanup** | Delete `/tmp/tool-integration-test.txt` after verification. |
| **Notes** | Verify multi-line content, special characters, and unicode. |

### Test 2.3: Edit File

| Field | Value |
|---|---|
| **Tool** | `Edit` (built-in) |
| **MCP Server** | N/A (native tool) |
| **Purpose** | Verify in-place file editing |
| **Preconditions** | Target file exists with known content |
| **Test Input** | `file_path: "/tmp/tool-integration-test.txt", old_string: "Line 2.", new_string: "Line 2 (edited)."` |
| **Expected Output** | File updated. Reading it back shows "Line 2 (edited)." |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Verify that only the target string is changed; rest of file is preserved. |

---

## 3. Google Calendar

### Test 3.1: List Upcoming Events

| Field | Value |
|---|---|
| **Tool** | `gcal_list_events` |
| **MCP Server** | `@cocal/google-calendar-mcp` |
| **Purpose** | Verify Google Calendar read access and OAuth token validity |
| **Preconditions** | OAuth token is valid; user has at least 1 upcoming event |
| **Test Input** | `calendarId: "primary", timeMin: "<today>T00:00:00", timeMax: "<today+7days>T23:59:59", maxResults: 5` |
| **Expected Output** | JSON with `events` array. Each event has `id`, `summary`, `start`, `end` fields. No auth error. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | If 401, run OAuth refresh flow. If empty results, verify the calendar has events. |

### Test 3.2: Find Free Time

| Field | Value |
|---|---|
| **Tool** | `gcal_find_my_free_time` |
| **MCP Server** | `@cocal/google-calendar-mcp` |
| **Purpose** | Verify free/busy query works |
| **Preconditions** | OAuth token valid; calendar has mix of busy and free slots |
| **Test Input** | `calendarIds: ["primary"], timeMin: "<tomorrow>T09:00:00", timeMax: "<tomorrow>T17:00:00", minDuration: 30` |
| **Expected Output** | JSON with `freeSlots` array containing at least 1 slot with `start`, `end`, `duration` fields. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Results depend on actual calendar state. Verify slots do not overlap with known events. |

---

## 4. Gmail / Email

### Test 4.1: Search Emails

| Field | Value |
|---|---|
| **Tool** | `gmail_search` |
| **MCP Server** | `@shinzolabs/gmail-mcp` or `@presto-ai/google-workspace-mcp` |
| **Purpose** | Verify Gmail read access and OAuth token validity |
| **Preconditions** | OAuth token is valid; mailbox has at least 1 email |
| **Test Input** | `query: "is:inbox", maxResults: 3` |
| **Expected Output** | Array of message objects, each with `messageId`, `subject` (or `snippet`), `from`, `date` fields. No auth error. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | If 401, refresh OAuth token. If empty, check that inbox is not empty. |

### Test 4.2: Get Specific Email

| Field | Value |
|---|---|
| **Tool** | `gmail_get` |
| **MCP Server** | `@shinzolabs/gmail-mcp` or `@presto-ai/google-workspace-mcp` |
| **Purpose** | Verify reading a specific email by ID |
| **Preconditions** | A known message ID from Test 4.1 |
| **Test Input** | `messageId: "<id-from-test-4.1>"` |
| **Expected Output** | Full message object with `subject`, `from`, `to`, `body` (or `snippet`), `date` fields. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Verify that body content is returned (not just metadata). |

---

## 5. Notion

### Test 5.1: Search Pages

| Field | Value |
|---|---|
| **Tool** | `notion_search` / `API-post-search` |
| **MCP Server** | `@notionhq/notion-mcp-server` |
| **Purpose** | Verify Notion integration token and search functionality |
| **Preconditions** | Integration token valid; at least 1 page shared with the integration |
| **Test Input** | `query: "test", page_size: 3` |
| **Expected Output** | JSON with `results` array containing page objects with `id`, `title`, `url` fields. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Only pages explicitly shared with the integration are returned. If empty, check sharing settings. |

### Test 5.2: Read Page Content

| Field | Value |
|---|---|
| **Tool** | `notion_get_block_children` / `API-get-block-children` |
| **MCP Server** | `@notionhq/notion-mcp-server` |
| **Purpose** | Verify reading page content blocks |
| **Preconditions** | A known page ID from Test 5.1 |
| **Test Input** | `block_id: "<page-id-from-test-5.1>"` |
| **Expected Output** | JSON with `results` array of block objects, each with `type` and content fields. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Paginated; check `has_more` field for multi-page content. |

---

## 6. Slack

### Test 6.1: List Channels

| Field | Value |
|---|---|
| **Tool** | `slack_list_channels` |
| **MCP Server** | `@modelcontextprotocol/server-slack` |
| **Purpose** | Verify Slack bot token and read access |
| **Preconditions** | Bot token (xoxb) is valid; bot is in at least 1 channel |
| **Test Input** | `limit: 5` |
| **Expected Output** | Array of channel objects with `id`, `name` fields. No auth error. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | If empty, ensure the bot has been invited to at least one channel. |

### Test 6.2: Get Channel History

| Field | Value |
|---|---|
| **Tool** | `slack_get_channel_history` |
| **MCP Server** | `@modelcontextprotocol/server-slack` |
| **Purpose** | Verify reading messages from a channel |
| **Preconditions** | Known channel ID from Test 6.1; channel has messages |
| **Test Input** | `channel_id: "<id-from-test-6.1>", limit: 3` |
| **Expected Output** | Array of message objects with `text`, `user`, `ts` fields. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Bot needs `channels:history` scope. If 403, check bot permissions. |

---

## 7. SSH / Remote Server

### Test 7.1: Test Connection

| Field | Value |
|---|---|
| **Tool** | `ssh_test_connection` |
| **MCP Server** | `ssh-client-mcp` |
| **Purpose** | Verify SSH key authentication and server reachability |
| **Preconditions** | SSH key exists at configured path; server is running |
| **Test Input** | `serverId: "<configured-server-id>"` |
| **Expected Output** | Connection successful message. No auth error. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | If timeout, check network/firewall. If auth fail, verify key path and permissions (600). |

### Test 7.2: Execute Remote Command

| Field | Value |
|---|---|
| **Tool** | `ssh_exec` |
| **MCP Server** | `ssh-client-mcp` |
| **Purpose** | Verify command execution on remote server |
| **Preconditions** | Active SSH session from Test 7.1 |
| **Test Input** | `sessionId: "<session-id>", command: "echo 'integration-test-ok'"` |
| **Expected Output** | stdout: `integration-test-ok`, exit code: 0 |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | Use a read-only, non-destructive command for testing. |

---

## 8. Web Automation

### Test 8.1: Screenshot

| Field | Value |
|---|---|
| **Tool** | `computer` (screenshot action) via Playwright / Chrome |
| **MCP Server** | `@playwright/mcp` or Chrome MCP |
| **Purpose** | Verify browser automation is working |
| **Preconditions** | Browser is available; network access to target URL |
| **Test Input** | Navigate to `https://example.com`, then take screenshot |
| **Expected Output** | Screenshot image returned showing the Example Domain page. |
| **Actual Output** | _(fill in when testing)_ |
| **Pass/Fail** | _(fill in)_ |
| **Last Tested** | YYYY-MM-DD |
| **Notes** | If timeout, check that browser process can start. Headless mode may behave differently. |

---

## Batch Test Runner

To run all tests in sequence, use the following checklist. Mark each as you go:

| # | Tool | Test | Result | Date |
|---|---|---|---|---|
| 1.1 | GitHub | List issues | | |
| 1.2 | GitHub | Create issue | | |
| 1.3 | GitHub | Search code | | |
| 2.1 | File System | Read file | | |
| 2.2 | File System | Write file | | |
| 2.3 | File System | Edit file | | |
| 3.1 | Calendar | List events | | |
| 3.2 | Calendar | Find free time | | |
| 4.1 | Gmail | Search emails | | |
| 4.2 | Gmail | Get specific email | | |
| 5.1 | Notion | Search pages | | |
| 5.2 | Notion | Read page content | | |
| 6.1 | Slack | List channels | | |
| 6.2 | Slack | Get channel history | | |
| 7.1 | SSH | Test connection | | |
| 7.2 | SSH | Execute command | | |
| 8.1 | Web | Screenshot | | |

**Last full test run**: YYYY-MM-DD
**Overall result**: _/17 passed

---

## Adding New Tests

When adding a new tool or MCP server:

1. Create a new section in this file following the template above.
2. Define at least one **read test** (verifies auth and connectivity) and one **write test** (verifies mutation capability) if the tool supports writes.
3. Include **cleanup instructions** for any write tests that create artifacts.
4. Add the test to the **Batch Test Runner** table.
5. Run the test immediately and record the results.
