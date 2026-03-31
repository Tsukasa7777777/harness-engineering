# MCP Server Registry

This file tracks all MCP servers (tool providers) in the system. Every tool the agent can invoke must be registered here. This registry serves as the single source of truth for tool availability, authentication status, and operational constraints.

## How to Use This Registry

- **Adding a new MCP server**: Add a row to the appropriate category table. Fill in all columns.
- **Deprecating a server**: Change status to `deprecated`. Do not remove the row -- keep it for audit history.
- **Troubleshooting**: Check `Status` and `Last Verified` columns first. Expired auth is the most common failure.
- **Rate limit planning**: Check `Rate Limits` before designing workflows that make many calls.

## Status Definitions

| Status | Meaning |
|---|---|
| `connected` | Active, authenticated, and verified working |
| `connected (user)` | Active but requires user-scoped access token (not tenant/bot token) |
| `disconnected` | Configured but not currently authenticated or reachable |
| `deprecated` | No longer in use; kept for reference |
| `planned` | Not yet configured; intended for future use |

---

## Registry

### Code & Project Management

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| GitHub | `@modelcontextprotocol/server-github` | Issue management, code management, PRs, repository operations | `connected` | Personal Access Token (PAT) | 5,000 req/hr (authenticated) | YYYY-MM-DD | Primary code management tool |
| Sentry | `sentry-mcp` (SSE transport) | Error monitoring, performance analysis, issue tracking | `connected (user)` | OAuth / API Token | Varies by plan | YYYY-MM-DD | SSE transport; requires user token |

### Document & Knowledge Management

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| Notion | `@notionhq/notion-mcp-server` | Document management, database CRUD, page creation | `connected` | Integration Token | 3 req/sec | YYYY-MM-DD | Primary knowledge base |
| Google Drive | `@isaacphi/mcp-gdrive` | File search, read, download from Google Drive | `connected` | OAuth 2.0 | 12,000 req/day | YYYY-MM-DD | Read-focused; use Google Workspace MCP for writes |
| Google Docs | `@anthropic/google-workspace-mcp` | Create, read, edit Google Docs | `connected` | OAuth 2.0 | Shared with Drive quota | YYYY-MM-DD | Part of Google Workspace MCP bundle |
| Markitdown | `@junkfactory/markitdown-mcp` | Convert PDF, Excel, Word, etc. to Markdown | `connected (user)` | None (local) | N/A (local processing) | YYYY-MM-DD | Runs locally; no network auth needed |

### Communication

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| Gmail | `@shinzolabs/gmail-mcp` | Email send, receive, search, label management | `connected` | OAuth 2.0 | 250 sends/day (consumer) | YYYY-MM-DD | Requires explicit user permission for sends |
| Slack | `@modelcontextprotocol/server-slack` | Channel history, message posting, reactions | `connected` | Bot Token (xoxb) | Tier 2-4 varies by method | YYYY-MM-DD | Multiple workspace support |
| Google Chat | `@presto-ai/google-workspace-mcp` | Chat space messaging, DMs | `connected` | OAuth 2.0 | Shared with Workspace quota | YYYY-MM-DD | Part of Google Workspace MCP bundle |

### Calendar & Scheduling

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| Google Calendar | `@cocal/google-calendar-mcp` | Event listing, creation, availability check | `connected` | OAuth 2.0 | 1,000,000 req/day | YYYY-MM-DD | Supports multiple calendar IDs |

### Data & Spreadsheets

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| Google Sheets | `google-sheets-mcp` | Spreadsheet read/write, cell updates | `connected` | OAuth 2.0 | 300 req/min per project | YYYY-MM-DD | Used for finance data |

### Web & Search

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| Playwright | `@playwright/mcp` | Web automation, scraping, screenshots | `connected` | None (local) | N/A (local browser) | YYYY-MM-DD | Headless browser; resource-intensive |
| Brave Search | `@modelcontextprotocol/server-brave-search` | Web search | `planned` | API Key | 2,000 req/mo (free) | -- | Requires API key registration |
| Firecrawl | `@anthropic/firecrawl-mcp` | Web data extraction, site crawling | `planned` | API Key | Varies by plan | -- | Requires API key registration |
| Context7 | `@upstash/context7-mcp` | Library documentation lookup | `connected (user)` | None | Rate-limited by upstream | YYYY-MM-DD | Useful for coding tasks |

### Infrastructure & DevOps

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| SSH (Xserver) | `ssh-client-mcp` | Remote server management, file ops, deploy | `connected` | SSH Key (`~/.ssh/xserver_rsa_local`) | N/A | YYYY-MM-DD | Production server access; use with caution |

### Content & Media

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| Google Slides | `@bohachu/google-slides-mcp` | Presentation creation and editing | `connected` | OAuth 2.0 | Shared with Workspace quota | YYYY-MM-DD | -- |
| Remotion | `@remotion/mcp` | React video generation, Remotion docs | `connected` | None (local) | N/A | YYYY-MM-DD | Documentation lookup + video rendering |

### AI & Code Generation

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| CodeX (OpenAI) | `codex-mcp-server` | Code generation, code review, web search | `connected` | OpenAI API Key | Varies by model | YYYY-MM-DD | Sub-agent for parallel code tasks |

### Social Media

| Tool Name | MCP Package | Purpose | Status | Auth Method | Rate Limits | Last Verified | Notes |
|---|---|---|---|---|---|---|---|
| X (Twitter) | `@enescinar/twitter-mcp` | Tweet posting, search, trend collection | `connected` | OAuth 1.0a | 300 tweets/3hr; 180 reads/15min | YYYY-MM-DD | Strict rate limits; batch carefully |

---

## Authentication Reference

| Auth Method | Credential Location | Rotation Schedule | Notes |
|---|---|---|---|
| OAuth 2.0 (Google) | `credentials/gcp-oauth.keys.json` | Auto-refresh | Token refresh handled by MCP server |
| GitHub PAT | Environment variable / `.env` | 90 days recommended | Scoped to repo + issues |
| Slack Bot Token | `.env` | No expiry (unless revoked) | xoxb-prefixed token |
| SSH Key | `~/.ssh/xserver_rsa_local` | Annual rotation | Ed25519 or RSA-4096 |
| API Keys (various) | `.env` | Varies | Never commit to repository |

## Adding a New MCP Server: Checklist

- [ ] Choose or build an MCP server package
- [ ] Configure authentication credentials
- [ ] Add entry to `.mcp.json` (project-level) or `~/.claude/mcp.json` (user-level)
- [ ] Verify connection with a smoke test
- [ ] Register in this file with all columns filled
- [ ] Add integration tests to `integration-tests.md`
- [ ] Document any special setup steps in `setup-guide.md` or equivalent
- [ ] Update agent skill definitions if the tool enables new capabilities
