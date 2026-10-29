# Zuri Context Service + Claude.ai Team

Detailed design for the recommended solution (Option I in `solutions.md`), structured along the three phases of [master-plan.md](master-plan.md):

| Master phase | What | Covered in |
|---|---|---|
| **Phase 1: MVP** | Context service (git-based) + minimal web UI for projects, users and Drive documents. **No chat of our own.** Chatting happens in Claude.ai and in Slack via Claude Tag | §1–§5 |
| **Phase 2: Full functionality, Zuri only** | Our own chat (human↔human and human↔AI). **Zuri leaves Slack**, and Claude.ai likely stays in parallel. Single tenant, hard-coding allowed | §6 |
| **Phase 3: Public availability** | Offered to third parties as a service, with billing at roughly **$5/user/month** | §7 |

Status: proposal, 2026-10-29 (revision 5: folded in the identity note in master-plan.md (Google + Zoho OAuth) and the Phase 2+ UI layout in [design.md](design.md); revision 4: second research round re-checked every vendor and platform claim against primary sources; revision 3 aligned with master-plan.md; revision 2 added the build-vs-buy research in [existing-solutions-context-service.md](existing-solutions-context-service.md)). Items marked **(verify)** have to be tested before we rely on them. Items marked **(unverified)** rest on third-party sources only.

**What changed in revision 5 (2026-10-29, same day):**
- **Identity.** [master-plan.md](master-plan.md) notes that everyone at Zuri has both a Google account and a Zoho Mail account, and that both offer OAuth. This confirms the design's "no passwords of our own" line. Phase 1 stays **Google-only** (the Picker needs a Google token, and one FastMCP server runs exactly one interactive login provider). **Zoho Accounts is an OpenID Connect provider** (checked: discovery document, `email` + `email_verified` claims), so the Hub gets "Sign in with Zoho" as a second login in Phase 2 and the registry stores `(provider, subject)` per user from day one. §3.1, §3.2, §4, §6.1, §7.4 and §8 were updated.
- **Phase 2 UI.** [design.md](design.md) (user-written) fixes the Hub layout: an icon rail (Home, projects sorted by the user's usage with a "…" overflow after five, Settings, User), a top-bar search, a narrow conversation list with a **Files** item per project, and a wide main pane. §6.2 gained per-user project activity and an events feed, §6.3 a layout paragraph, and §3.1 a shared **sync-status vocabulary** so the Phase 1 admin page and the Hub's Files view show the same thing.

**What changed in revision 4 (2026-10-29):**
- Team plans now include **pooled monthly API credits** ($20 per Standard seat, $100 per Premium seat, capped at $500/month, no rollover) that cover the Messages API, Managed Agents and the Agent SDK. For 30 Standard seats that is up to $500/month, which should pay for the summarizer and much of the Phase 2 Hub chat at **no extra cost**. §3.5 and §6.6 were updated.
- **Claude Tag** is in public beta, Team and Enterprise only. A connector attached to a Slack *channel* runs under one **shared agent identity** (an admin signs in once), so per-user roles in channels only work through **personal connectors** (on by default for Team orgs) or in DMs. The $2,500 launch credit **expired on 1 October 2026**. §3.8 and §5 were updated.
- **FastMCP is at 4.x** (4.1.4, 26 Oct 2026) and the current MCP spec revision is **2026-07-28**, which deprecates Dynamic Client Registration in favour of Client ID Metadata Documents (CIMD). FastMCP has emitted the metadata Claude needs to pick CIMD since June 2026. The two GitHub issues we worried about turned out to be server-side bugs (wrong redirect status, missing `code` parameter); a new one (#476, tools not surfaced on claude.ai web) is open. §3.7 was updated.
- Google: the Docs API now supports **real suggested edits** (`writeMode: SUGGEST`), the Markdown export has a 10 MB cap and embeds images as base64 (unverified), Drive quotas moved to a quota-unit model, and the native `.md` feature (5 Oct 2026) is about editing `.md` files in Docs, not exporting Docs. §3.3 and step 2.4 were updated.
- Market: 18 more vendors checked, verdict unchanged. **Graphlit is shutting down** and Pinecone's Drive connector reached end of life, a warning against buying a small "context layer" vendor. **Supermemory** is the only commercial product with per-file Drive picking plus an OAuth MCP server; it has no maintainers, history, project roles or EU hosting. Details in [existing-solutions-context-service.md](existing-solutions-context-service.md).

**Decision: build it, reusing existing parts.** No open-source or commercial product found covers this service. Each candidate (Onyx, Dust, Guru, Glean) has *permission-aware search through an OAuth MCP server*, but none combines hand-picked files, document maintainers, a Markdown mirror with diffs and self-rebuilding project summaries. Those parts are our actual value. See §2a for what we reuse.

---

## 1. Idea in one paragraph

We build a small **Zuri Context Service** that owns the *knowledge*:
- which Drive files belong to which project;
- who maintains them;
- an always-current Markdown mirror with diff history;
- an up-to-date summary per project.

**Phase 1:** we do **not** build the chat. People chat in **Claude.ai / Claude desktop / mobile** (paid by the Team subscription), and Claude reaches our knowledge through a **custom connector** (a remote MCP server). Group AI chat happens in **Slack via Claude Tag**, using the same connector.

**Phase 2:** the same service gets its own chat (the "Zuri Hub"), which replaces Slack and Claude Tag. **Phase 3:** it's offered to other companies.

Everything built in Phase 1 is reused unchanged in both later phases.

```
                  ┌────────────────────── Zuri Context Service ──────────────────────┐
 Google Drive ───►│  Registry  ──►  Sync worker  ──►  Mirror (git)  ──►  Summarizer   │
 (picked files)   │     ▲                                   │                │        │
                  │     │                                   ▼                ▼        │
                  │  Admin UI                         MCP server  ◄──  Notifier ──────┼──► maintainers
                  └─────────────────────────────────────┬───────────────────────────┘     (Slack/email/Trello)
                                                        │ (OAuth, per user)
                      ┌─────────────────────────────────┼─────────────────────────────┐
                      ▼                                 ▼                             ▼
              Claude.ai / desktop / mobile      Phase 1: Claude Tag (Slack)   Phase 2+: Zuri Hub chat
              personal chats (subscription)     group chats (usage-billed)    (replaces Slack), also
              all phases                        dropped in Phase 2            Claude Code, local models
```

---

## 2. Design principles

1. **Curated, not crawled.** The service only ever touches files someone explicitly registered. Nothing else in Drive is visible to it.
2. **Markdown + git is the source of truth for the AI.** It's human-readable, diffable, and portable to any model or tool.
3. **The chat UI is replaceable.** All logic (permissions, knowledge, maintainer workflow) lives in the service behind MCP. Claude.ai, Slack, or our own UI are just clients.
4. **Subscription first, API where unavoidable.** Interactive chats run on Team seats. Background jobs and group chat use metered usage, kept small and capped. Since 2026 the Team plan includes pooled API credits (up to $500/month for our size), so "API-billed" often means "free until the pool is used up".
5. **Small and boring.** One small server, one database, one git repo. A single developer can run it.
6. **Search is a swappable part.** We start with Postgres/SQLite full-text search. If semantic search is needed later, a search engine (Onyx, Dust, a RAG API) goes *behind* the Zuri MCP server, scoped by project, never in front of it. The registry, mirror, summaries and maintainer loop stay ours.
7. **Don't block Phase 3, but don't build it yet.** Phases 1–2 may hard-code Zuri specifics, except for a few cheap things that would be expensive to retrofit (see §7.4): a `tenant_id` on every table, per-tenant git repos and paths, no Zuri logic inside MCP tools, and MCP tool annotations from day one.

---

## 2a. Build vs. buy: what we reuse

Research summary (full details in [existing-solutions-context-service.md](existing-solutions-context-service.md)):

| Candidate | What it would give us | Why not adopt it |
|---|---|---|
| Onyx (open source) | OAuth MCP server with per-user permissions, Drive connector | Search tools only. Drive scoping by drive/folder, not file. Group RBAC is in the paid Enterprise Edition. Heavy infrastructure. Adopting it ≈ our MVP + 2–3 weeks |
| Dust (SaaS) | OAuth MCP server, Spaces ≈ projects, EU hosting | ~€720/month for 30 seats. Single-file picking not documented. No mirror, summaries or maintainers |
| Guru (SaaS) | Card owners with re-verification reminders, OAuth MCP approved for Claude | Timer-based re-checks of its own cards, not AI-found contradictions in Drive files. No per-file Drive curation |
| Glean (SaaS) | OAuth MCP server, Claude Team supported | Crawls everything a user can see (the opposite of curated). Reported minimum ~$50k/year |
| Claude Team native Drive | Live-synced Docs in projects | Disabled in shared projects |
| Supermemory (SaaS, round 2) | Hosted Google Picker for single files, Markdown conversion, OAuth MCP server | Markdown is internal and lossy, no history, no maintainers, no project roles, no EU region. Scale plan $399/month. A design reference, not a platform |
| Slite / Guru (SaaS, round 2) | OAuth MCP servers that name Claude; Slite is EU-hosted and has review owners, Guru has one verifier per card with AI auto-unverify | Drive scoped by folder; human or timer-based verification, not contradiction detection; no mirror or summaries. $10–25 per seat |
| Dust / Langdock (SaaS, round 2) | EU-hosted chat platforms with granular Drive selection | Phase 2 alternatives at best (€24–30 and €25 per seat). Dust has an OAuth MCP server (US and EU endpoints) but Claude.ai is not named in its docs **(verify)**; Langdock's MCP is API-key only |

**What we reuse:**

| Part of our design | Reused component | Saves |
|---|---|---|
| MCP server + OAuth (§3.7) | **FastMCP 4.x** (Python, pin ≥ 4.1) with `GoogleProvider` / OAuth proxy. It stands in for client registration, which Google lacks, supports CIMD, and gives tools the caller's email | ~1 week of OAuth and spec work |
| (fallback) | Cloudflare `workers-oauth-provider` + `remote-mcp-google-oauth` template (TypeScript), or a hosted auth vendor (WorkOS / Stytch / Descope) | same |
| Admin UI file picking (§3.2) | Google's official Picker web sample (`googleworkspace/browser-samples`, `drive/picker`) + `drive.file`. Open WebUI's `google-drive-picker.ts` (multi-select, `drive.file` only, since v0.12.3) is a second reference implementation | 1–2 days |
| Docs → Markdown (§3.3) | Native Drive `files.export` with `text/markdown` | converter code |
| Sheets/Slides (§3.3) | Sheets API per tab; isaacphi/mcp-gdrive (MIT) as reference code | small |
| Summarizer (§3.5) | Karpathy's **"LLM Wiki"** pattern (gist of 4 April 2026): sources → wiki pages + index + append-only log + schema file + periodic "lint" for contradictions. `lucasastorian/llmwiki` (Apache-2.0) shows the same idea as an MCP tool set (`search/read/lint`) | prompt/design work |
| Later write tools (Phase 2, step 2.4) | Docs API `batchUpdate` with `writeMode: SUGGEST` (real suggested edits) and Drive `comments.create`, both allowed under `drive.file`. Google's hosted Docs/Drive MCP servers stay a reference only: they are Developer Preview and need the restricted `drive.readonly` scope | Docs API code |

**Not reused, still custom:** registry and roles, sync worker (~200–400 lines), issue/maintainer workflow, notifier, admin pages. Nothing on the market does these the way we need.

**Checked in round 2 (2026-10-29):** Graphlit (shutting down), Langdock, Vectara, Supermemory, Contextual AI, Pinecone Assistant, Ragie, Slite, Tettra, Atlassian Rovo, Box, Egnyte, Notion, Glean, Guru, Onyx EE, Dust, Anthropic's connector directory, plus Khoj, Kotaemon, Cognee, Graphiti and LibreChat. None changes the verdict. Still unchecked: Bloomfire and Fini (vendor claims of contradiction detection, no Drive), and Coda.

---

## 3. Components

### 3.1 Registry (database)

Postgres (or SQLite for the pilot).

```
users(id, email, name, slack_id)                 -- one row per person; the verified @zuri.com email
user_identities(user_id, provider, subject,      -- provider: google | zoho (Phase 2+) | ... ;
                email, email_verified, last_login_at) -- key on the provider's `sub`, never on email alone
projects(id, slug, name, color, emoji, description)
project_members(project_id, user_id, role)       -- role: admin | member | reader
documents(id, drive_file_id, title, mime_type,
          maintainer_user_id, ai_access,         -- ai_access: read | suggest | write
          status, last_synced_at, last_revision)  -- status: not_fetched | fresh | outdated | error
project_documents(project_id, document_id,       -- M:N: one doc in many projects
                  added_by, added_at, note)
proposals(id, project_id, drive_url, proposed_by, -- members propose, admins approve
          reason, status, decided_by, decided_at)
issues(id, document_id, reported_by,             -- "complaints" to maintainers
       source, description, status, created_at)  -- source: summarizer | chat | human
summaries(project_id, version, git_commit, built_at, kind)  -- kind: incremental | full
audit_log(id, user_id, action, target, at)
```

**Phase 3 readiness:** add `tenants(id, name, plan, google_customer_id)` and a `tenant_id` column to every table from day one, even though it is always `zuri` in Phases 1–2. Retrofitting it later is costly. Nothing else multi-tenant is built yet.

**Why M:N matters:** the ground-station spec is registered **once**, with one maintainer, and linked to the "big demonstrator", "small drones" and "ground station" projects. It's synced once and appears in all three summaries.

**Sync-status vocabulary.** `documents.status` is shown next to the maintainer on the Phase 1 admin page and, unchanged, in the Hub's Files view ([design.md](design.md)): `not_fetched` (registered, never exported), `fresh` (mirror matches the latest Drive revision), `outdated` (Drive changed; export pending or debounced), `error` (last export failed: over 10 MB, access lost, unsupported type). Keep the list short and stable; the UI only colours it.

### 3.2 Admin UI (small web page)

- Google sign-in, restricted to the Zuri domain. Google is the only login in Phase 1 because the Picker needs a Google token; Zoho sign-in is added with the Hub (§4).
- **Add document:** a Google Picker dialog with the `drive.file` scope. The service gets access only to the picked files. Then set the maintainer, the project(s) and the AI access level.
- **Project page:** members and roles, the document list with maintainer and sync status (same columns and status words as the Hub's Files view in [design.md](design.md), so the page can be reused), the latest summary, open issues and pending proposals.
- **My documents:** each maintainer sees their files and the issues reported against them.

This can be very plain (server-rendered HTML). It's an admin tool, not the chat. Start from Google's official Picker web sample.

**Picker and background sync.** A `drive.file` grant belongs to the user who picked the file: Google's Picker guide ties Drive access to "the OAuth 2.0 token... in which the user previously granted access", and `drive.file` covers files "the user shares with an app while using the Google Picker API". Google documents no official pattern for background access to picked files, only the two general routes **(verify in the pilot: expect a 404 when another principal reads a picked file)**:
- the picking user's stored **refresh token** (Google's `access_type=offline`), or
- the picked file shared with a **service account**. Service accounts can't own files or hold storage since 2025, but they can read anything shared with them and can be members of a Shared Drive, which is the cleanest setup. The admin page can do the sharing automatically after picking.

Decide in the pilot. Avoid domain-wide delegation: it would need the restricted `drive`/`drive.readonly` scopes.

### 3.3 Sync worker

Every 5 minutes it checks `modifiedTime`/`version` of all registered files. Under Drive's 2026 quota-unit model a `files.get` costs 5 units against 1,000,000 units per minute per project, so 500 files every 5 minutes is negligible; one `changes.list` call per cycle is cheaper still. Optionally it adds Drive push notifications (`changes.watch` channels last up to 7 days, `files.watch` up to 1 day; both are payload-less "wake up" calls) to react faster.

**Debounce:** a file counts as "changed" only after **20 minutes without further edits**, because Google Docs autosave constantly.

**Export:**
| Source | Method | Output |
|---|---|---|
| Google Docs | `files.export` `mimeType=text/markdown` (Docs only; exports are capped at **10 MB**) | `docs/<slug>--<id>.md` |
| Google Sheets | Sheets API `spreadsheets.values.batchGet`, one range per tab (Drive's CSV/TSV export covers **only the first tab**, and there's no Markdown export for Sheets). The Sheets API allows 60 reads per minute per user, which bounds a single service identity | `docs/<slug>--<id>.md` with one Markdown table per tab (very large tabs also as `.csv` next to it) |
| Google Slides | `files.export` `text/plain` | `.md` with one section per slide |
| PDF / docx in Drive | download + converter (e.g. pandoc, pdftotext) | `.md` |
| Comments (optional) | Drive `comments.list` | appended section "Open comments" (exports don't contain comments) |

Google doesn't document how the Markdown export handles tables, images, footnotes or multi-tab documents. Third-party reports **(unverified)** say headings get Pandoc-style `{#id}` anchors and images are embedded as **base64 `data:` URIs**, which count toward the 10 MB cap, so the worker should strip or relocate images before committing. Google Docs **tabs** are a separate risk: per-tab export is officially unsupported (issue 379170478), the Docs API needs `includeTabsContent=true` to return more than the first tab, and whether the Markdown export concatenates all tabs is undocumented **(verify)**. Test all of this on 3–5 real Zuri docs in the pilot. **Fallback** for difficult files: export as docx and convert with pandoc. Google's native `.md` support (5 Oct 2026) only lets people edit `.md` files stored in Drive inside Docs; it doesn't change Docs → Markdown export.

Each file gets a small front-matter header: title, Drive link, maintainer, projects, last modified, revision. That way the AI can always cite the source and the person responsible.

**Commit:** one git commit per changed file, with a message like `sync: <title> (rev 123, edited by X)`. The git history *is* the diff store.

**Retention:** a nightly job squashes or prunes history older than N days (e.g. 90). Summaries and issue records keep the long-term memory.

### 3.4 Mirror repository layout

```
zuri-knowledge/
  docs/
    ground-station-spec--1AbC.md
    flight-control-params--9XyZ.md
    ...
  projects/
    demonstrator/
      SUMMARY.md           -- current project summary
      INDEX.md             -- list of docs with maintainer + one-line description
      CHANGES.md           -- human-readable log of recent major changes
    small-drones/
      ...
  registry.yaml            -- export of the registry (read-only, for transparency)
  CLAUDE.md                -- "schema": rules for how summaries, index and change log are written
```

This follows Karpathy's "LLM Wiki" layout:
- immutable sources → `docs/`
- LLM-maintained wiki → `SUMMARY.md` (+ per-doc summaries later)
- index → `INDEX.md`
- append-only log → `CHANGES.md`
- schema → `CLAUDE.md`

It can be mirrored to a private GitHub repo. That gives people a browsable history and enables the low-tech Claude.ai GitHub integration route.

### 3.5 Summarizer

**What counts as a major change:** any of these triggers a rebuild.
- The diff touches more than X % of a document or more than N lines.
- A document was added to or removed from the project.
- A maintainer clicks "this is a major change" in the admin UI.
- A daily catch-up if anything changed at all.

**Incremental update** (the default):
- *Input:* the previous `SUMMARY.md`, the `INDEX.md`, and the diffs since the last build, each with document title and maintainer.
- *Prompt:* update the summary; keep its structure; record what changed in `CHANGES.md`; **list contradictions** between the changed text and other documents or the summary (doc, passage, conflicting doc, why).
- Contradictions become `issues` records and are sent to the **maintainer** (R2).

**Lint pass** (the LLM Wiki pattern's contradiction check): run nightly or after each incremental update. It compares changed passages against the rest of the project's docs and summaries and produces candidate inconsistencies. As the pattern itself notes, these "still benefit from human review". Here that review is the maintainer loop: every finding becomes an `issue` for the doc's maintainer, who marks it *fixed / not an issue / discuss*.

**Full rebuild** (weekly, or when the summary drifts): summarize each document separately (cached, rebuilt only when that document changed), then merge them into the project summary. This avoids errors piling up from repeated incremental updates.

**Summary structure** (template):
1. Purpose and current status of the project
2. Key decisions and parameters, each with its source doc
3. Components and the shared dependencies with other projects
4. Open questions and known inconsistencies
5. Recently changed
6. Document map: what to read for which topic

**How it runs:**
- **Production:** an API key with a monthly spend cap, paid first from the Team plan's **pooled API credits** ($20 per Standard seat per month, capped at $500/month for the org, no rollover; they cover the Messages API, Managed Agents and the Agent SDK, but not Claude Code or Claude.ai extra usage). An Owner claims them under Organization settings > Billing. Use Sonnet 5.5 ($2/$10 per million tokens in/out) or Haiku 5.5 ($0.10/$0.50) for incremental updates and Opus 5.5 ($4/$20) for weekly full rebuilds and the inconsistency check; the Batch API halves the price of the weekly jobs. Rough estimate: a weekly full rebuild of 12 projects with about 50 docs each is on the order of 3–5 million input tokens, so roughly $15–30/month on Opus 5.5, well inside the credit pool. Measure the real token counts in the pilot before fixing the model mix.
- **Scheduling options:** our own cron job; Managed Agents **scheduled deployments** (cron, per-run budget caps, $0.08 per session-hour on top of tokens, public beta) if we want Anthropic to host the loop; or Claude Code **scheduled tasks** (formerly routines, research preview, available on Team) working on the mirror repo. Scheduled tasks draw on the creating developer's seat limits and belong to that person's account, so they suit the pilot more than production.
- **Not allowed:** a self-hosted `claude -p` server running on a subscription login (see `solutions.md`). Anthropic paused the planned Agent SDK billing change in June 2026, but its legal page still limits subscription OAuth to "ordinary use" of Anthropic's own apps and forbids intermediating Claude for other users.

### 3.6 Notifier

When an issue is created or a proposal awaits approval, it sends a message:
- Phase 1: **Slack DM** to the maintainer or admin (Slack API) by default. Phase 2: an in-app notification in the Zuri Hub (plus push/email), since Slack goes away. Keep the notifier behind a small `Channel` interface so this is a swap, not a rewrite;
- email as a fallback;
- optionally a Trello card in a "Doc issues" list.

The message includes the quote, the conflicting document, the conversation it came from (if any), and buttons: *fixed / not an issue / discuss*.

A weekly digest per maintainer lists open issues on their documents.

### 3.7 MCP server (the custom connector)

A remote MCP server using streamable HTTP. It authenticates with **OAuth (Google sign-in)**, so every call comes with the identity of the actual user and the service can apply project roles.

**Implementation:** **FastMCP 4.x** (Python, pin ≥ 4.1; 4.1.4 was released on 26 Oct 2026) with `GoogleProvider`.
- Google offers no dynamic client registration, so FastMCP's OAuth proxy stands in for it. The current MCP spec (revision **2026-07-28**) deprecates dynamic registration in favour of **Client ID Metadata Documents (CIMD)**. Claude uses CIMD only when the server's metadata advertises both `client_id_metadata_document_supported: true` and `"none"` in `token_endpoint_auth_methods_supported`; FastMCP has done that since PR #4206 (June 2026), older builds make Claude fall back to dynamic registration, which also works. Check the live `/.well-known/oauth-authorization-server` in the pilot.
- Tools read the caller's email via `get_access_token().claims.get("email")` or the dependency form `CurrentAccessToken()` / `TokenClaim("email")`, and the role check looks that email up in the registry. Role gating can use `@mcp.tool(auth=...)` checks (3.0+) or `require_roles` (4.0+).
- Production needs an explicit `jwt_signing_key` and a persistent, encrypted client/token store (`FernetEncryptionWrapper(RedisStore)`); the default disk store is for development only. Set `fastmcp_access_token_expiry_seconds` explicitly, because Google access tokens last about an hour and Claude's refresh timeouts are tight (10 s for discovery and token calls, 30 s for refresh).
- Emit `302`/`303` redirects with a query-string response (never `307` or `form_post`) and an absolute `resource_metadata` URL: those were the actual causes behind the Claude.ai OAuth failures reported in claude-ai-mcp #313 and #212.
- Don't rely on elicitation for confirming writes. In 4.x it's old-protocol-only, and there's no evidence Claude.ai web supports it for custom connectors. Use tool annotations plus Claude's own approval prompts.

**Domain restriction:** FastMCP stores `email` and `email_verified` claims but no `hd` claim, and Google says the `hd` parameter is only a UI hint, so:
- pass `extra_authorize_params={"hd": "zuri.com"}` as a convenience, and **enforce** the domain and `email_verified` in the server-side auth check;
- set the Google OAuth consent screen to "Internal" as a second barrier (and it skips Google app verification entirely in Phases 1–2).

**Claude.ai connector requirements** (from Claude's connector auth docs, checked 2026-10-29; FastMCP covers most of them):
- an unauthenticated request returns `401` with `WWW-Authenticate: Bearer resource_metadata=...` (Claude ignores the header on a `200`);
- the protected-resource `resource` value equals the exact URL entered in Claude, and `resource` is sent on authorization and token requests (RFC 8707);
- PKCE with S256 on every authorization request;
- a form-encoded `/token` endpoint; refresh tokens rotate for public clients; `invalid_grant` for expired refresh tokens;
- 10-second discovery, registration and token timeouts, 30 seconds for refresh;
- redirect URI `https://claude.ai/api/mcp/auth_callback` for web, desktop, mobile and Cowork; Claude Code uses port-agnostic `http://localhost/callback` and `http://127.0.0.1/callback`;
- only the first `authorization_servers` entry is used;
- Claude connects from 160.79.104.0/21.

Claude's "static header" shared-key option (beta, limited orgs) sends the same key for every member, so it can't identify users. Per-user roles require OAuth. Authentication settings of a connector can't be edited after creation; remove and re-add instead.

**Risk:** the two OAuth issues we tracked (claude-ai-mcp #313 and #212) are closed; both were server-side bugs (a method-preserving `307` redirect, and callbacks without a `code` parameter). FastMCP issue #1919 ("Google OAuth doesn't work for Claude.ai") was fixed in late 2025 and the reporter confirmed `GoogleProvider` works on Claude.ai and Claude Code. There is still **no published proof for a Team org-wide connector or for mobile**, only Anthropic's statement that all its clients share one OAuth client. A **new open issue, #476** (since June 2026): custom connectors show "Connected" on claude.ai web but their tools are sometimes never offered to the model; FastMCP and non-FastMCP servers are affected and Anthropic hasn't replied. So the connector is still the **first pilot test**, with Claude Desktop and Claude Code as the fallback chat surfaces while that bug lasts. Framework fallbacks: Cloudflare `workers-oauth-provider` (MIT, now targets MCP 2026-07-28, its Google demo has a `HOSTED_DOMAIN` setting), or a hosted authorization server (WorkOS AuthKit is free to 1M monthly users; Stytch, Descope, Scalekit and Clerk also support CIMD). The official MCP Python and TypeScript SDKs now ship only the resource-server half, so they're not an alternative for the login flow.

**Search backend:** Postgres or SQLite full-text search to start. Semantic search (Onyx, Dust or a RAG API, scoped per project) can be plugged in behind the `search` tool in step 1.3 without changing the tool interface.

| Tool | What it does | Who may call |
|---|---|---|
| `list_projects()` | projects the user belongs to, with role | everyone |
| `get_project_summary(project)` | `SUMMARY.md` + `INDEX.md` | reader+ |
| `list_documents(project)` | docs with maintainer, last change | reader+ |
| `read_document(doc, section?)` | Markdown of a doc (or one section of a long doc) | reader+ of any linked project |
| `search(project, query)` | full-text search (plus semantic search later) across that project's docs | reader+ |
| `recent_changes(project, since)` | diffs + `CHANGES.md` entries | reader+ |
| `doc_history(doc, n)` | last n revisions with diff | reader+ |
| `report_inconsistency(doc, description, quote)` | creates an issue and notifies the maintainer | member+ |
| `propose_document(project, drive_url, reason)` | adds to the admin approval queue | member+ |
| `approve_proposal(id)` / `add_member(...)` | admin actions from chat | admin |
| later: `suggest_edit(doc, ...)`, `add_comment(doc, ...)` | writes to the Google Doc (R3) | member+, doc `ai_access ≥ suggest` |

**Server-side instructions** (in the MCP `instructions` field): "Always start with `get_project_summary` for the project the user is talking about. Quote sources with doc title + link. If you find contradictions, offer to call `report_inconsistency`." Each Claude.ai project also repeats this in its own instructions.

**Prompt-injection hygiene:** document content is returned as data, clearly delimited. Write tools need explicit user confirmation, and Claude.ai asks for tool approval by default. Every tool carries a `title` and `readOnlyHint`/`destructiveHint` annotations (`@mcp.tool(annotations=ToolAnnotations(...))`), and read and write actions are separate tools, which the connector directory requires anyway (§7.1).

### 3.8 How people use it in Phase 1 (UX)

In Phase 2 the Slack parts below move into the Zuri Hub (§6). The Claude.ai part stays.

- **One Claude.ai project per Zuri project**, e.g. "🛰 Ground station", shared with the project members, with instructions that pin the project slug. The project name and emoji are the visual cue for what we're talking about.
- **Personal chat:** open the project in Claude.ai and ask. It works on mobile too, with attachments, long threads and artifacts, all native.
- **Team chat:** a Slack channel per project or topic (`#gs-ai`, `#drones-ai`) with Claude Tag enabled and the Zuri connector in its bundle. You tag @Claude in a thread. **Claude Tag facts (public beta, checked 2026-10-29):** Team and Enterprise only; it requires *Routines* to be enabled for the org; the legacy "Claude in Slack" bot was retired on 5 Oct 2026; channel work is billed from an Owner-funded usage balance at API rates, 1:1 DMs bill to the member's own seat; the $2,500 launch credit expired on 1 Oct 2026. **Identity:** a connector attached to a channel belongs to the **agent identity**, not to a person: an admin signs in once and "what Claude can do never changes based on who asked". Our role checks therefore see one account per channel. Two ways to live with that:
  1. **One service user per channel bundle.** Create a Zuri account per channel (or reuse a project admin's) that is a member of exactly the projects that channel may see, and sign the channel's connector in as that user. Channel membership then has to match project membership, which admins control anyway.
  2. **Personal connectors.** They are on by default for every Team org: when a member asks something in a channel, Claude offers to use that member's own connector, with that member's permissions, after a one-time approval. That gives per-user roles for requests people make themselves. Routines and proactive work never use personal connectors.
  The pilot picks the mix (§5). Setup is an Owner flow at claude.ai/admin-settings/claude-tag plus a Slack admin installing the app from the Slack Marketplace and pairing it with `@Claude connect`.
- **Whisper:** human-only Slack DMs or channels where Claude Tag isn't enabled. The context service is never involved.
- **Late invites to a group:** follows Slack's rules (public channels show full history, so create a new private channel for a "from now" group).

---

## 4. Security and operations

- **Hosting:** one small VM or container (e.g. Cloud Run / Hetzner / a GCP region in the EU) + Postgres + the git repo. Roughly €10–40/month. Backups: a database dump and the git remote.
- **Google access:** the `drive.file` scope via Picker (per-file, non-sensitive). If documents sit in many people's My Drives, use a service account and share the picked files with it instead. Decide this in the pilot.
- **Identity:** no passwords of our own. [master-plan.md](master-plan.md) notes that everyone at Zuri has both a Google account and a Zoho Mail account, each with OAuth. **Phase 1 uses Google only:** the Picker needs a Google token anyway, and a FastMCP server runs exactly one interactive login provider (`MultiAuth` adds token *verifiers*, not a second login screen). **Zoho Accounts supports OpenID Connect** (discovery at `https://accounts.zoho.com/.well-known/openid-configuration`, per data centre; scopes `openid email profile`; the ID token carries `email` and `email_verified`; Zoho's docs say to identify users by `sub`, not by email). So the Hub (Phase 2) can offer "Sign in with Zoho" beside Google through any OIDC library, and FastMCP's generic `OIDCProxy` could front the MCP server with Zoho if Google ever had to be dropped **(verify)**. Neither provider has a Zuri-organisation claim (Google's `hd` is only a hint, Zoho has none), so the domain check stays on the verified `email` claim (§3.7). Both logins map to one `users` row by verified email **(verify that everyone's Google and Zoho addresses are identical; otherwise the Hub needs a one-time account-linking step)**. The Google OAuth app is "Internal" in Phases 1–2, so it needs no Google verification; Phase 3 switches to an external app, which only needs brand verification because `drive.file` is non-sensitive (§7.1). Roles are checked on every MCP call. Every read and write is audit-logged.
- **Data flow:** document text goes to Anthropic only when a user's chat (or the summarizer) asks for it, under Team or commercial terms. Nothing goes to other services unless a model gateway is added later.
- **Failure mode:** if the service is down, Claude.ai still works, just without project knowledge. Nothing is lost, because Drive stays the master.

---

## 5. Phase 1 delivery: context service MVP

| Step | Scope | Duration | Billing |
|---|---|---|---|
| **1.0: Pilot** | `registry.yaml` by hand, sync worker → git → GitHub, LLM-Wiki-style summarizer script, one project. A shared Claude.ai project using the GitHub integration ("Sync now" by hand). Claude Tag enabled in one Slack channel. **Plus a FastMCP + Google OAuth "hello" connector to retire the auth risk early** (see checklist below) | ~1 week | subscription + small API |
| **1.1: Context Service MVP** | Postgres registry, admin UI from the Picker sample, FastMCP server with read tools + role checks, registered as a Team custom connector, added to Claude Tag channels | **1.5–2 weeks** (was 2–3; FastMCP + Picker sample) | same |
| **1.2: Maintainer loop** | issues, lint pass, `report_inconsistency`, notifier (Slack DM / email / Trello), proposals + approvals, weekly digest, full + incremental summaries | 1–2 weeks (nothing reusable exists) | same |
| **1.3: Quality** | semantic search for big projects (pluggable backend behind `search`), per-doc summaries, comments sync, admin dashboards (sync health, issue stats, token usage) | ongoing | same |

**Total to a working Phase 1 (steps 1.0–1.2): about 3.5–5 weeks for one developer** (was 4–6). Step 1.3 continues in parallel with Phase 2.

**Pilot checklist.** These risks must be retired before step 1.1:

| Verify | Why it matters | Fallback if it fails |
|---|---|---|
| FastMCP 4.x `GoogleProvider` completes OAuth as an Owner-added Team connector, on web **and** mobile, Claude picks CIMD (check the server metadata), the email claim reaches tools, and tools actually appear in chats (claude-ai-mcp #476) | All per-project roles depend on it | Cloudflare `workers-oauth-provider`, or a hosted auth vendor; Claude Desktop/Code while #476 lasts |
| The connector works inside *shared* Claude.ai projects for all members (no official statement either way; one third-party site says private projects only, unverified) | UX = one shared project per Zuri project | The GitHub-mirror route with manual "Sync now" |
| Claude Tag: a channel-attached connector runs as one admin account (confirmed by the docs). Test the **service-user-per-channel** setup and the **personal-connector** path, and whether the latter is acceptable UX for members | Group chat roles | Scope each Slack channel to one project |
| `text/markdown` export quality on 3–5 real docs (tables, base64 images, footnotes, **tabs**, comments) + Sheets per-tab rendering via `values.batchGet` | Mirror and summary quality | docx + pandoc for difficult files; Docs API per tab |
| `drive.file` + Picker vs. a service account for background sync (does a service account get a 404 on a picked file it wasn't shared?) | Sync reliability across My Drives | Share picked files with a service account or put them in a Shared Drive |
| Token cost of incremental vs. full summary rebuilds; claim the Team API credits and confirm they cover the summarizer's API key | Summarizer billing | API key with a monthly spend cap |

**Optional parallel check:** a 2-week **Dust trial**. Dust's docs confirm a remote MCP server with OAuth (DCR and CIMD) at `dust.tt/mcp` and `eu.dust.tt/mcp`, but don't name Claude.ai, confirm single-file picking, or say whether one file can sit in several Spaces **(verify)**. Only relevant if EU-only hosting becomes mandatory and nobody can run a server. Even then, Dust would only replace the search/access layer, not the mirror, summaries or maintainer loop. Slite (EU-hosted, OAuth MCP, $10–20 per seat) is the other candidate for that narrow case.

**Exit criteria for Phase 1** (after step 1.2): a member can ask "what's the current ground-station uplink frequency and who owns that doc?" in Claude.ai on their phone and get the right answer with a link, and a reader can't call `report_inconsistency`.

---

## 6. Phase 2: full functionality for Zuri ("Zuri Hub" with our own chat)

**Decided in the master plan:** we build our own chat for human↔human and human↔AI conversations. **Zuri leaves Slack** (and with it Claude Tag). Claude.ai likely stays in parallel for personal AI chats. This is single-tenant: no support for other companies, no payments, and some parts may be hard-coded (but see principle 7).

**What this means for billing:**
- AI chats in our own UI are billed at **API rates**, because subscription logins can't be used in our own app. The Team plan's pooled API credits (up to $500/month for 30 Standard seats) apply to this API usage, so the first part of the Hub's AI traffic is effectively included in the seats.
- Claude Tag's usage cost disappears with Slack, so group AI chat moves from Claude Tag usage to our own API usage. The cost is similar in kind.
- **Personal chats stay cheapest in Claude.ai** (Team subscription + Zuri connector). The Hub's main value is group and project chat, plus replacing Slack.

### 6.1 Architecture additions

```
 Zuri Hub web/PWA ──► Hub backend ──► Model gateway (LiteLLM) ──► Claude API (Opus 5.5)
        │                 │                                  └─► other providers / local (Ollama, vLLM)
        │                 ├─► Conversation store (Postgres + object storage for attachments)
        │                 └─► Zuri Context Service tools (same MCP server, called server-side)
        └─ realtime (WebSocket) for multi-user rooms
```

- **Same registry and identity.** The Hub just adds conversation tables (with `tenant_id`, per principle 7). Login is "Sign in with Google" or "Sign in with Zoho" (OpenID Connect, §4) against the same `users` row; there is no password store.
- **Same MCP tools.** The Hub's agent calls the context service exactly like Claude.ai does, so behaviour is consistent across clients.
- **Model gateway:** one place for API keys, per-project model choice, spend caps per project/user, logging, and switching to local models for sensitive projects (R7). In Phase 3 it also holds per-tenant keys (bring-your-own key). **Choice:** LiteLLM (MIT, per-key/team budgets) had **two supply-chain incidents in 2026** (malicious PyPI releases in March and a compromised 1.107.0 plus Docker `latest` tags on 21 Oct 2026), so pin versions and verify images, or evaluate **Bifrost** (Apache-2.0, hierarchical budgets, no enterprise gating). For calling the Zuri MCP server from the Hub backend, the Messages API's own **MCP connector** (`mcp_servers` + `mcp_toolset`) and the Vercel AI SDK v7 Anthropic provider both support remote MCP servers directly, so the gateway only has to pass requests through.

### 6.2 Conversation model

```
conversations(id, tenant_id, project_id?, kind, title, topic, created_by, ai_enabled)
   -- kind: personal | project | group | whisper | dm
   -- whisper/dm ⇒ ai_enabled = false, enforced in backend; never sent to any model,
   --               excluded from search/summaries/exports to AI
conversation_members(conversation_id, user_id, role, visible_from)
   -- visible_from = conversation start OR join time → "invite with history: from start / from now"
messages(id, conversation_id, author(user|ai), body, attachments, thread_parent, created_at)
ai_runs(id, conversation_id, model, tokens_in, tokens_out, cost, tools_called)
project_activity(user_id, project_id, last_seen_at, score)
   -- per-user usage; orders the project icons in the rail (design.md), top 5 + "…"
events(id, tenant_id, project_id?, kind, ref, actor_id, at)
   -- kind: message | doc_synced | issue_opened | summary_rebuilt | proposal | member_joined
   -- feeds the Home overview; filtered by what the viewer may see (never whisper/dm)
```

### 6.3 Chat features

**Layout (from [design.md](design.md), user-written, applies to Phase 2+).** Three areas, like the chat tools people already use:
- **The frame:** a narrow icon rail on the left (Home = overview of everything the user has access to; DMs possibly as its own icon, to be decided; project icons sorted by that user's usage, with a "…" icon for projects 6+; then Settings and User at the bottom) and a one-line top bar that holds only a search field for now.
- **Left column (narrow):** channels and conversations in roughly chronological order. In a project view it also has a **Files** item. Details to be defined later.
- **Right column (wide, about two thirds of a widescreen monitor):** the main content, usually one conversation. Under Files: the project's Drive documents with their maintainer and sync status (`fresh`, `outdated`, `not_fetched`, `error`, §3.1).

Consequences for the model: `project_activity` for the rail ordering and `events` for the Home overview (§6.2); the Files view reuses the registry's `documents` and `project_documents` directly, so the Phase 1 admin page and the Hub show the same list.

**Built in roughly this order.** Items 1–4 are the AI core. Item 5 is required before Slack can be switched off.
1. The frame and the project-scoped rooms from the layout above, with clear headers: project color and emoji (also used for the rail icon), a visibility badge (🔒 personal / 👥 project / 🧩 group / 🤫 whisper — AI off), and topic. The Files view ships with this item because it is a read-only list over the registry.
2. AI as a participant: `@claude` in group rooms, always-on in personal rooms. Streaming answers, tool calls shown as collapsible "read doc X" chips with links.
3. Attachments (files, images) passed to the model. Long threads with automatic context compaction (summarize older turns).
4. Invites with a "show history: from start / from now" choice (from `visible_from`). The AI only sees messages that *all current members* may see, so a late joiner can't extract hidden history through the AI.
5. **Slack-replacement layer (R10):** human DMs, threads, mentions, reactions, notifications (web push + mobile PWA, email digest), search, file sharing, and a Slack history import. The notifier (§3.6) moves here. The top-bar search field exists from item 1 (design.md puts it in the frame) but starts as plain full-text search over the user's conversations and mirrored documents; the full version lands here.
6. "Promote to doc": turn a conversation outcome into a Google Doc draft or a doc comment (with the write tools from step 2.4).

### 6.4 Build approach for the Hub

Phase 3 means the Hub becomes a **product we sell**, which changes the build choice compared with revision 2.

| Option | Pros | Cons |
|---|---|---|
| Fork/extend Open WebUI | Chat UI, attachments, multi-model and Channels exist already; its Picker code is a good reference | Licence clause 4 forbids altering "Open WebUI" branding in any deployment with more than fifty end users in a rolling 30 days without written permission or an enterprise licence. That's a blocker for a white-labelled product in Phase 3. Its data model also doesn't fit whisper/visible_from/projects |
| Mattermost + Agents plugin | Mature Slack-like chat, mobile apps | Compiled builds are MIT, self-built servers are AGPL v3 or commercial, `server/enterprise` is source-available; the licence FAQ says a derivative must replace the Mattermost name and logo and resale needs a reseller agreement. Weaker Claude-like chat. (Zulip is Apache-2.0 and the cleanest of the Slack clones for white-label, but still the wrong shape; its data model is worth borrowing) |
| **Own app** (e.g. TypeScript: Next.js/SvelteKit + Vercel AI SDK, or Python backend + any SPA; Postgres; WebSockets) | Exactly our UX and data model; no licence constraints on resale; the same code serves Phase 3 | Most work: ~4–8 weeks for the AI core, plus 3–6 weeks for the Slack-replacement layer |

**Recommendation:** build our **own app**. Optionally spend 2–3 days on a **throwaway Open WebUI prototype** (≤50 users, unbranded use is fine) connected to the Zuri MCP server, just to learn which chat features people actually use before building.

### 6.5 Phase 2 steps

| Step | Scope | Duration |
|---|---|---|
| **2.0: Prototype (optional)** | Throwaway Open WebUI + LiteLLM + Zuri MCP tools for one project. Measure API cost per active user | 2–3 days |
| **2.1: Hub AI core** | Own app: Google/Zoho OIDC login, conversation model (kinds, whisper, visible_from), the three-pane layout from design.md (icon rail, conversation list, main pane with Files view), AI participant, attachments, compaction | 4–8 weeks |
| **2.2: Slack replacement** | DMs, threads, mentions, notifications/push, search, Slack import; notifier moves into the Hub. **Switch Slack off** (Claude Tag ends with it) | 3–6 weeks |
| **2.3: Hardening** | Admin dashboards (usage, cost per project), backups/restore drill, audit log review, mobile polish | 1–2 weeks |
| **2.4: Write access & agents** | `suggest_edit` via the Docs API `batchUpdate` with `writeMode: SUGGEST` (real suggested edits, confirmed in Google's docs) and `add_comment` via Drive `comments.create` (API comments render unanchored in the editors), then direct edits for `ai_access=write` docs. All of this works under `drive.file` for picked files and sits behind our registry allowlist. Google's hosted Docs/Drive MCP servers remain reference material only (Developer Preview, restricted scopes, no allowlist). Scheduled agents (e.g. weekly "project health" post per room) via Managed Agents scheduled deployments or our own cron | ongoing |

**Total Phase 2: about 2–4 months for one developer** (rough estimate).

**Exit criteria for Phase 2:** Slack is switched off. Every chat kind (personal, project, group, whisper, DM) works in the Hub on desktop and phone. Whisper conversations are verifiably never sent to a model.

### 6.6 Billing in Phase 2

| Usage | Where | Paid by |
|---|---|---|
| Personal AI chat | Claude.ai / desktop / mobile / Claude Code + Zuri connector (or the Hub if preferred) | Team subscription (Hub: API) |
| Group / project AI chat | Zuri Hub | API usage via gateway, capped per project; the Team plan's pooled API credits (≤ $500/month) are used first |
| Human chat (Slack replacement) | Zuri Hub | hosting only |
| Summaries, inconsistency checks, scheduled agents | Context service | API usage, small and capped; also from the credit pool |
| Sensitive projects (optional) | Hub → local model via gateway | own hardware |

---

## 7. Phase 3: public availability

Offer the context service + Hub to other companies, priced at roughly **$5/user/month**.

### 7.1 What has to be added

| Area | What | Notes |
|---|---|---|
| **Multi-tenancy** | Tenant signup, isolation enforced in every query (or Postgres row-level security), per-tenant git repos, per-tenant Drive credentials and model keys, tenant deletion/export | Cheap only if principle 7 was followed |
| **Self-service onboarding** | "Sign in with Google" (or another OpenID Connect provider, §4) → create a workspace → invite users → pick Drive files (Google only) → connect Claude | The first 10 minutes decide adoption |
| **Billing** | A merchant of record handles EU VAT for us: Paddle (5% + $0.50 per transaction) or **Stripe Managed Payments** (Stripe's own MoR, +3.5% on top of card fees, about 5% + €0.25 in the EEA, Czech sellers eligible, digital products only, subscriptions need Stripe Billing at +0.7%, the buyer sees "Sold through Link, LLC"); or plain Stripe + Stripe Tax with our own VAT filings. Per-seat subscription, trial, seat sync, invoices, dunning | The fixed €0.25 fee matters at $5 invoices; B2B reverse-charge handling in Managed Payments is undocumented **(verify)** |
| **Google OAuth verification** | Stay on **`drive.file`** (non-sensitive scope): Google says verification isn't mandatory for non-sensitive-only apps; **brand verification** (homepage, privacy policy and terms on the same Search-Console-verified domain, 2–3 business days of manual review) is only needed to show our name and logo on the consent screen. Avoid restricted scopes (full `drive`/`drive.readonly`), which require a yearly lab-verified CASA assessment (now called AL1/AL2; roughly $540 per year per app, unverified). Workspace admins can allowlist our app by OAuth client ID; no Marketplace listing is needed | Service-account sharing still works for customers ("share files with sync@…"), but Picker + per-user tokens is smoother |
| **Claude connector directory** | Submit the MCP server to Anthropic's connector directory so customers' Claude orgs can add it in one click. Current criteria: remote server over HTTPS, OAuth 2.0, a title plus `readOnlyHint`/`destructiveHint` on every tool, separate read and write tools (no catch-all `api_request`), 64-character tool names, the server calls only our own first-party APIs, plus a privacy policy URL, documentation URL, support contact, icon and test account. Any Owner on a paid plan can submit at claude.ai/directory/manage; submissions are auto-scanned and listed as "Community", and Anthropic may upgrade them to "Verified" | Annotations and the read/write split are cheap to do from day one |
| **Compliance & trust** | GDPR DPA, sub-processor list (hosting, Anthropic, Google, Stripe/Paddle), EU hosting, data retention controls, security page; SOC 2 later if larger customers ask | Prompt-injection and cross-tenant leakage tests become critical |
| **Operations** | Status page, support channel, per-tenant rate limits (Drive API quotas), monitoring, backup/restore per tenant | |

### 7.2 Pricing economics: $5/user/month can't include AI chat

- **Our cost per user** for the context service is low: hosting, sync, and summaries that only send diffs. That fits inside $5.
- **AI chat at API rates** doesn't fit. One active Opus user can easily cost more than $5/month in tokens.
- **So the $5 covers the context service + Hub (human chat, registry, mirror, summaries, maintainer loop, Claude connector).** AI chat is paid separately, by one of:
  1. **"Bring your own Claude":** customers chat in their *own* Claude.ai subscription via our connector. This is the Phase 1 model and costs us nothing. It's also a strong selling point: "makes the Claude subscription you already pay for know your project documents".
  2. **Bring your own API key** (Anthropic, OpenAI, or a local endpoint) for AI in the Hub, via the model gateway's per-tenant keys.
  3. **Metered AI credits** resold with a margin, for customers who want one bill.
- **Summaries are included but capped** per tenant (number of docs or rebuilds per month), with cheaper models for incremental updates. Measure the real cost per tenant in Phase 2 before fixing the price.
- **Positioning:** Dust (€24–30/seat), Langdock (€25), Guru (~$25/seat, third-party figure), Slite ($10–20), Notion Business (€19.50), Google Workspace Business Standard with Gemini (€13.60) and Glean (enterprise pricing, ~100-seat minimum, unverified) all sell search plus their own chat. Supermemory ($19–399/month flat) and Cognee Cloud ($5 per workspace plus tokens) are the nearest "docs + Claude connector" offerings. Nobody sells hand-picked Drive docs with maintained summaries and an OAuth connector at about $5/user. Two close-fit vendors (Graphlit, Pinecone's Drive connector) shut down during 2026, which argues for our git mirror as the customer's safe, exportable substrate. At $5 we'd be the low-cost, **curation + document-ownership** layer that plugs into the AI the customer already uses.

### 7.3 Phase 3 steps (rough)

| Step | Scope | Duration |
|---|---|---|
| **3.1: Multi-tenant core** | tenant isolation, onboarding, per-tenant Drive/model credentials, data export/delete | 3–5 weeks |
| **3.2: Billing** | Paddle/Stripe, plans, trials, seat sync, AI credits or BYO key | 1–2 weeks |
| **3.3: Trust & listing** | Google OAuth verification, Claude connector directory submission, DPA/privacy/security pages, cross-tenant security tests | 2–4 weeks (partly waiting on reviews) |
| **3.4: Beta** | 3–5 friendly companies, then public launch | ongoing |

### 7.4 Cheap things to do in Phases 1–2 so Phase 3 isn't a rewrite

1. A `tenant_id` on every table and in every query helper (always `zuri` for now).
2. The git mirror path is `tenants/<tenant>/…` (or one repo per tenant), never a single global repo.
3. No Zuri-specific logic in MCP tools or prompts. Project names and prompt templates live in config or the DB.
4. Every MCP tool gets a `title` and `readOnlyHint`/`destructiveHint` annotations from day one.
5. Drive access only through `drive.file` (+ service-account sharing as an option). Never request restricted scopes.
6. The model gateway is keyed per tenant (just one key for now).
7. Audit log and per-tenant usage counters from the start. They become billing and compliance data later.
8. Keep the OAuth layer swappable. FastMCP's proxy fronts one Google OAuth client; if Phase 3 customers need their own identity providers, a hosted authorization server (WorkOS, Descope, Scalekit) or Cloudflare's library takes over without touching the tools. Store `(provider, subject)` per user from the start (§3.1): Zoho in Phase 2 and customers' Microsoft/Okta logins in Phase 3 then only add rows.

---

## 8. Open questions / decisions needed

**Phase 1**
1. Where do the documents live: My Drives or Shared Drives? This decides `drive.file` + Picker vs. a service account.
2. Data residency: is EU-only hosting required? Are local models needed for some projects?
3. Where do maintainer notifications go in Phase 1: Slack DM, email (everyone has Zoho Mail, so email is reliable), or Trello? (In Phase 2 they move into the Hub.)
4. Who are the Claude Team **Owners**? Only they can add the connector and configure Claude Tag.
5. How long to keep diffs (proposed: 90 days)?
6. **(verify)** The custom connector works inside *shared* Claude.ai projects for all members.
7. Claude Tag: the docs confirm that a channel-attached connector runs as one admin account. Decide between a service user per channel bundle and relying on personal connectors (§3.8), and **(verify)** both in the pilot. Only relevant until Slack is switched off in Phase 2.
8. Claude Code scheduled tasks are confirmed available on Team (research preview) and bill to the creating developer's seat. Decide whether to use them for the pilot summarizer or go straight to an API key funded by the Team API credits. **(verify)** that the credits appear under Organization settings > Billing and cover our API key's usage.
9. **(verify)** FastMCP 4.x + Google OAuth works end to end as a Claude.ai Team connector on web and mobile, and tools are actually offered to the model (claude-ai-mcp #476). See the pilot checklist in §5. This is the biggest technical risk.
10. **(verify)** Does the `text/markdown` export include all tabs of a tabbed Google Doc? If not, export per tab through the Docs API.
11. Service account vs. per-user `drive.file` tokens for background sync.

**Phase 2**
12. Tech stack for the Hub (TypeScript full-stack vs. Python backend + SPA). Ideally the same language as the context service.
13. Slack switch-off date, and what Slack history to import.
14. Monthly API budget for Hub AI chat, and whether personal AI chat is encouraged in Claude.ai or in the Hub.
20. Identity: **(verify)** that everyone's Google and Zoho Mail addresses are identical (otherwise the Hub needs an account-linking step). Which login is the default button, and whether Zoho is offered in Phase 2 at all or only kept for Phase 3 customers.
21. UI ([design.md](design.md)): DMs as a separate rail icon or inside Home; what the Home overview aggregates (events kinds in §6.2); the conversation-list ordering and Files-view columns that design.md leaves "to be defined in detail later".

**Phase 3**
15. Product name and brand. It shouldn't be "Zuri" if the company name should stay separate. Legal entity for selling.
16. AI pricing model: bring your own Claude/key, metered credits, or both.
17. Merchant of record (Paddle) vs. Stripe + own VAT handling.
18. Resolved: any Owner on a paid plan can submit to the connector directory; listings start as "Community". Open: whether we want to wait for a "Verified" badge before launch.
19. Merchant of record: Paddle vs. Stripe Managed Payments, pending confirmation of B2B reverse-charge handling in the latter.

---

## Sources added in revision 4 (checked 2026-10-29)

- Anthropic: https://code.claude.com/docs/en/legal-and-compliance , https://support.claude.com/en/articles/15036540 , https://platform.claude.com/docs/en/about-claude/api-credits-for-subscribers , https://support.claude.com/en/articles/11175166 , https://support.claude.com/en/articles/10166901 , https://support.claude.com/en/articles/10167454 , https://github.com/anthropics/claude-ai-mcp/issues/180 , https://claude.com/docs/claude-tag/overview , https://claude.com/docs/claude-tag/admins/setup-overview , https://claude.com/docs/claude-tag/admins/set-spend-limit , https://claude.com/docs/claude-tag/admins/add-connections , https://claude.com/docs/claude-tag/admins/connections/custom , https://claude.com/docs/claude-tag/concepts/agent-identity , https://claude.com/docs/claude-tag/concepts/personal-connectors , https://support.claude.com/en/articles/15575654 , https://code.claude.com/docs/en/slack , https://support.claude.com/en/articles/12005970 , https://platform.claude.com/docs/en/about-claude/pricing , https://code.claude.com/docs/en/routines , https://claude.com/docs/connectors/building/submission , https://claude.com/docs/connectors/building/review-criteria , https://claude.com/docs/directory/publish , https://claude.com/docs/connectors/building/authentication , https://platform.claude.com/docs/en/api/ip-addresses , https://support.claude.com/en/articles/10593882 , https://github.com/anthropics/claude-ai-mcp/issues/313 , https://github.com/anthropics/claude-ai-mcp/issues/212 , https://github.com/anthropics/claude-ai-mcp/issues/122 , https://github.com/anthropics/claude-ai-mcp/issues/476 , https://platform.claude.com/docs/en/managed-agents/scheduled-deployments , https://support.claude.com/en/articles/11845131 , https://support.claude.com/en/articles/9266767
- Google: https://developers.google.com/workspace/drive/api/guides/ref-export-formats , https://developers.google.com/workspace/drive/api/reference/rest/v3/files/export , https://developers.google.com/workspace/docs/api/how-tos/tabs , https://issuetracker.google.com/379170478 , https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/batchGet , https://developers.google.com/workspace/drive/api/guides/api-specific-auth , https://support.google.com/cloud/answer/9110914 , https://support.google.com/cloud/answer/13465431 , https://appdefensealliance.dev/casa/casa-tiering , https://developers.google.com/identity/protocols/oauth2/production-readiness/overview , https://developers.google.com/identity/protocols/oauth2/production-readiness/brand-verification , https://developers.google.com/workspace/drive/api/guides/push , https://developers.google.com/workspace/drive/api/guides/limits , https://developers.google.com/workspace/sheets/api/limits , https://developers.google.com/workspace/drive/api/guides/manage-comments , https://developers.google.com/workspace/docs/api/how-tos/suggestions , https://developers.google.com/workspace/guides/configure-mcp-servers , https://developers.google.com/workspace/drive/api/guides/configure-mcp-server , https://developers.google.com/workspace/release-notes , https://developers.google.com/workspace/drive/picker/guides/web-picker , https://developers.google.com/workspace/drive/api/guides/about-shareddrives , https://developers.google.com/identity/openid-connect/openid-connect , https://workspaceupdates.googleblog.com/2026/10/preview-edit-and-collaborate-on-Markdown-files-natively-across-Drive-and-Docs.html , https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data
- FastMCP and MCP: https://gofastmcp.com/integrations/google , https://gofastmcp.com/servers/auth/oauth-proxy , https://gofastmcp.com/servers/dependency-injection.md , https://gofastmcp.com/servers/storage-backends.md , https://gofastmcp.com/servers/authorization.md , https://gofastmcp.com/servers/tools , https://gofastmcp.com/servers/elicitation , https://github.com/jlowin/fastmcp/releases , https://github.com/jlowin/fastmcp/issues/1919 , https://github.com/jlowin/fastmcp/pull/4206 , https://modelcontextprotocol.io/specification/versioning , https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization , https://modelcontextprotocol.io/specification/2026-07-28/changelog , https://py.sdk.modelcontextprotocol.io/run/authorization/ , https://github.com/cloudflare/workers-oauth-provider , https://github.com/cloudflare/ai/tree/main/demos/remote-mcp-google-oauth , https://workos.com/docs/authkit/mcp , https://workos.com/pricing
- Phase 2/3 stack: https://docs.litellm.ai/blog/security-incident-oct-21-2026 , https://docs.litellm.ai/docs/proxy/users , https://docs.getbifrost.ai/features/governance , https://vercel.com/blog/ai-sdk-7 , https://ai-sdk.dev/providers/ai-sdk-providers/anthropic , https://raw.githubusercontent.com/mattermost/mattermost/master/LICENSE.txt , https://docs.mattermost.com/product-overview/faq-license , https://raw.githubusercontent.com/zulip/zulip/main/LICENSE , https://raw.githubusercontent.com/open-webui/open-webui/main/LICENSE , https://www.paddle.com/pricing , https://docs.stripe.com/payments/managed-payments , https://docs.stripe.com/payments/managed-payments/eligibility , https://stripe.com/en-ie/pricing
- Vendors (round 2): https://www.graphlit.com/legal/sunset , https://docs.pinecone.io/release-notes/2026 , https://supermemory.ai/docs/memory-api/connectors/google-drive , https://supermemory.ai/docs/supermemory-mcp/introduction , https://supermemory.ai/pricing , https://slite.com/changelog/slite-mcp-server , https://slite.com/pricing , https://help.getguru.com/docs/verifying-and-unverifying-cards , https://developer.getguru.com/docs/guru-mcp-server-overview , https://docs.dust.tt/docs/dust-mcp-server , https://dust.tt/pricing , https://www.langdock.com/pricing , https://docs.langdock.com/en/using-langdock/guides/integrations/mcp/langdock-agent-mcp-server , https://onyx.app/pricing , https://docs.onyx.app/overview/onyx_anywhere/mcp_server , https://docs.cognee.ai/cognee-cloud/connections/cloud-mcp.md , https://cognee.ai/pricing , https://github.com/lucasastorian/llmwiki , https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

## Sources added in revision 5 (checked 2026-10-29)

- Zoho Accounts "Sign in using Zoho (OIDC)": https://www.zoho.com/accounts/protocol/oauth/sign-in-using-zoho.html (discovery URL, `openid email profile phone` scopes, ID-token claims incl. `email_verified`, "do not use email as the unique identifier"); data-centre server list: https://accounts.zoho.com/oauth/serverinfo
- FastMCP generic OIDC proxy: https://gofastmcp.com/servers/auth/oidc-proxy ; FastMCP `MultiAuth` (one auth server plus token verifiers, no second login flow): https://gofastmcp.com/servers/auth/multi-auth
- Phase 2+ UI layout: [design.md](design.md) (user-written)
