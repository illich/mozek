# Solutions for "AI over curated project documents" (Zuri)

Research date: 2026-10-29 (revision 5: identity note from master-plan.md (Google + Zoho OAuth) and the Phase 2+ UI in [design.md](design.md) folded in; revision 4: every platform and vendor claim re-checked against primary sources, 18 more vendors examined; revision 3: recommendation aligned with [master-plan.md](master-plan.md); revision 2 added Markdown mirrors, project summaries and Team-subscription billing). This is based on web research. Vendor features, especially beta ones, change fast. Anything marked **(unverified)** comes from third-party sources only. Anything marked **(verify)** is my inference and still needs testing.

## 1. Requirements recap (from concept.md)

| # | Requirement | Weight |
|---|---|---|
| R1 | Per project: hand-pick 10–100 Google Drive docs/sheets for the AI. No broad Drive access | **Core** |
| R2 | Each document has one **maintainer**, and the AI reports inconsistencies to that person | Core |
| R3 | AI has read-only access now, write access later | Medium |
| R4 | Project roles: admin (people + docs), member (read/write, proposes docs), reader | Core |
| R5 | Chats can be personal (1 human + AI), team, or custom groups. AI sees all project docs | **Core** |
| R6 | "Whisper" chats between humans that the AI never sees | Medium |
| R7 | Claude (Opus 5.5) by default, with the option of other or local models | Medium |
| R8 | Full Claude-like chat: attachments, long threads | Core |
| R9 | Friendly UX with clear cues for project, visibility and topic | Core |
| R10 | Replaces Slack (secondary goal) | Low |
| R11 | Invite people to a group chat later, choosing whether they see history from the start or from now on | Medium |
| **R12** | **Keep a `.md` copy of every connected Google file, update it automatically when the source changes, and keep the diffs for a while** | **Core (new)** |
| **R13** | **One summary `.md` per project covering all its files, rebuilt on every major source change** | **Core (new)** |
| **R14** | **AI chats should run on the Claude Team subscription, not be billed at API prices** | **Important (new)** |
| — | Projects overlap a lot (ground station, flight control shared): documents belong to **several** projects | Important |

Scale: 30 people, 2 big projects and about 10 small ones, roughly 100–500 curated documents in total.

---

## 2. Key findings that shape the decision

### About Claude and billing (R14)

1. **The subscription only covers Anthropic's own apps.** Anthropic's legal page (primary source, checked 2026-10-29) says OAuth authentication "is intended exclusively for purchasers of Claude Free, Pro, Max, **Team**, and Enterprise subscription plans and is designed to support ordinary use of Claude Code and other native Anthropic applications"; developers, "including those using the Agent SDK, should use API key authentication"; and customers "may not pay for, resell, or intermediate Claude usage on their end users' behalf". Anthropic paused the planned Agent SDK billing change in June 2026, so a developer's own `claude -p` runs still draw on their seat, but a server that serves other users is intermediation. **Consequence:** no third-party or self-built chat app (Open WebUI, LibreChat, Onyx, Dust, a custom app) can run on your Team seats. They all bill at API rates or their own pricing. **New in 2026:** Team plans include **pooled monthly API credits** ($20 per Standard seat, $100 per Premium seat, capped at $500/month per org, no rollover) usable for the Messages API, Managed Agents and the Agent SDK. For 30 Standard seats that's up to $500/month of API usage already paid for, which changes the economics of the background summarizer and of Phase 2 Hub chat.
2. **To use the subscription, the chat has to happen in Claude.ai or the Claude apps.** The supported way to bring your own data in is a **custom connector (a remote MCP server)**:
   - Only Owners (and Primary Owners) can add one to a Team org. Owners can trial a connector with "Connect just for me" first.
   - Each member then connects with **their own OAuth login**, so the server can tell who is calling and enforce project roles. Members can set per-tool permissions (always allow / needs approval / blocked); Owners can disable specific tools.
   - Connectors work in every Claude client: claude.ai, Desktop, Cowork and the mobile apps.
   - Not documented either way: whether a connector works inside a *shared* Claude.ai project for all members **(verify)**.

   This is the key building block for R14.
3. **Claude.ai has no group chat with AI.** A chat belongs to one person, and shared chats are view-only snapshots (now also shareable by e-mail invite). Still true in October 2026; Anthropic's group-chat answer is Claude Tag in Slack.
4. **Claude Tag (Slack) uses metered billing and a shared identity in channels.** Public beta, Team and Enterprise only; the legacy "Claude in Slack" bot was retired on 5 Oct 2026. Channel, thread and group-DM work draws from an Owner-funded *usage balance* at **API rates** (Team "usage credits", formerly "extra usage"). 1:1 DMs are billed to the sender's own seat. Owners set spend caps ($500–$5,000 presets, per-channel limits). The one-time **$2,500 launch credit expired on 1 Oct 2026**, so don't budget on it. Claude Tag also requires *Routines* to be enabled for the org. Channel **bundles** can include a custom remote MCP server from the org's connector library, so a Slack channel's Claude can use the Zuri MCP server. **But the connector then belongs to the agent identity:** an admin signs in once and "what Claude can do never changes based on who asked". Per-user identity in channels exists only through **personal connectors** (on by default for Team orgs; Claude asks the requester for approval and uses that person's own connector), and in DMs. **Bottom line:** personal AI chat can be fully subscription-billed, but **group AI chat costs API-rate usage on any route**, and per-user roles in channels need the personal-connector path or one service account per channel.
5. **Claude.ai shared projects can't use the Drive connector.** The Drive option works only in private projects (re-confirmed in the support article, updated October 2026). The GitHub integration in projects syncs only when someone clicks **"Sync now"**; the auto-sync request (claude-ai-mcp #180) was closed "not planned" in May 2026. So "Drive → Markdown in GitHub → Claude project" works, but needs manual refreshes.

### About Markdown mirrors and summaries (R12, R13)

6. **Drive can export Google Docs as Markdown directly**: `files.export` with `mimeType=text/markdown` (Docs only, capped at 10 MB per export). Sheets need the Sheets API (`spreadsheets.values.batchGet`, one range per tab) and you render Markdown tables yourself; Drive's own CSV/TSV export is documented as "first-sheet only". Slides → `text/plain`. Google doesn't document how the Markdown export treats tables, footnotes, images or **tabs**; third-party reports say images come out as base64 data URIs and headings get `{#id}` anchors **(unverified)**, and per-tab export is officially unsupported **(verify whether all tabs are included)**.
7. **Change detection:**
   - `files.watch` (per file, channel lasts at most 1 day) and `changes.watch` (whole change feed, at most 7 days). Both send payload-less "wake up" webhooks, so you still call `changes.list` / `files.get(modifiedTime, version)`.
   - Channels must be renewed.
   - A 5–10 minute polling fallback on `modifiedTime` is simpler and fine for 500 files. Drive's 2026 quota model charges 5 units per `files.get` against 1,000,000 units per minute per project, so polling is free in practice; the Sheets API's 60 reads per minute per user is the tighter limit.
   - Google Docs autosave constantly, so **debounce** changes (e.g. treat a doc as changed after 15–30 minutes without edits).
8. **Diffs come free with git.** Commit each Markdown export into a git repo. You get diffs, history and blame without writing any code, and "keep for a while" becomes a retention job (squash or prune history older than N days) **(design suggestion)**.
9. **Existing tools:** [wikiGDrive](https://github.com/mieweb/wikiGDrive) (MIT, still active in August 2026) converts a whole Shared Drive's Docs to Markdown and watches for changes. gdrivemd is a pair of shell scripts around the `gog` CLI for two-way per-doc sync; gcmd, gdocs-git-sync, md-gdoc and google-docs-markdown-sync are similar small CLIs. None combines Picker-based per-file selection with a git mirror, so the sync worker stays a (small) build. Google's native `.md` support in Drive and Docs (announced 5 Oct 2026) lets people view and edit `.md` files stored in Drive inside Docs; it is not a Docs → Markdown export feature.
10. **No product found maintains a rolling per-project summary from diffs.** This step is always custom. The usual pattern: after a debounced "major change", feed **the previous summary + the diffs** to the model and ask for an updated summary. Run a full rebuild weekly to stop drift. The same pass is a natural place to **detect inconsistencies and notify the maintainer** (R2).
11. **The background summarizer needs an API key, but the Team plan now pays for it.** Summaries are an automated backend job, and under the policy in finding 1 that is API usage, not subscription usage. The Team plan's pooled API credits (finding 1) cover it: at current list prices (Opus 5.5 $4/$20 per million input/output tokens, Sonnet 5.5 $2/$10, Haiku 5.5 $0.10/$0.50, Batch API at half price) a weekly full rebuild of all projects is a rough $15–30/month, far below the $500/month pool. Hosting options: our own cron, Managed Agents scheduled deployments (cron, per-run budgets, public beta), or Claude Code scheduled tasks (research preview, available on Team, billed to the creating developer's seat). A human clicking "rebuild summary" in Claude.ai also works but isn't automatic.

### From revision 1 (still valid)

12. **Google's `drive.file` scope + Google Picker** is the clean way to meet R1: the app sees only the files someone explicitly picked. Google classifies it as non-sensitive and says app verification "is not mandatory" for apps using only non-sensitive scopes; an "Internal" app for our own Workspace needs no verification at all. Restricted scopes (`drive`, `drive.readonly`) would require a yearly lab-verified CASA assessment. The grant is bound to the picking user's token, so background sync needs either stored refresh tokens or files shared with a service account **(verify)**.
13. **Off-the-shelf RAG platforms** (Onyx, Dust, Open WebUI…) scope Drive by **folder / shared drive**, not by file, and keep their own internal text copies rather than user-visible `.md` + diffs.
14. **No product has a document maintainer concept (R2) or invite-with-history (R11).**
15. **Whole-document context beats RAG at this scale.** A project summary plus 10–100 Markdown files fits in the 1M-token context of the current Claude models, which makes cross-document consistency checks feasible.

### New in revision 4 (2026-10-29)

16. **The MCP auth stack moved on.** FastMCP is at 4.x (4.1.4 on 26 Oct 2026). The current MCP spec revision is 2026-07-28: it deprecates Dynamic Client Registration in favour of Client ID Metadata Documents (CIMD), makes Protected Resource Metadata and the RFC 8707 `resource` parameter mandatory, and makes the protocol stateless. FastMCP has advertised the metadata Claude needs to choose CIMD since June 2026. The official MCP Python and TypeScript SDKs now ship only the resource-server half (token verification), so a "Google login as identity" server needs FastMCP's OAuth proxy, Cloudflare's `workers-oauth-provider`, or a hosted authorization server (WorkOS, Stytch, Auth0, Descope, Scalekit, Clerk all support CIMD).
17. **The Claude.ai connector issues we tracked were server-side.** claude-ai-mcp #313 (a method-preserving `307` redirect) and #212/#122 (callback without `code`) are closed with explanations. The open one is **#476**: connectors show "Connected" on claude.ai web but tools are sometimes never offered to the model, since June 2026, no Anthropic reply. FastMCP's own issue #1919 was fixed and its reporter confirmed `GoogleProvider` works on Claude.ai and Claude Code.
18. **Write access to Google Docs is now simple.** The Docs API accepts `writeMode: SUGGEST` on `batchUpdate`, so the AI can make real suggested edits, and Drive `comments.create` adds (unanchored) comments. Both work under `drive.file`. Google's hosted Drive/Docs/Sheets MCP servers remain Developer Preview, need the restricted `drive.readonly` scope and have no file allowlist.
19. **Connector directory is open to any paid plan.** An Owner on a Team plan can submit at claude.ai/directory/manage; listings start as "Community" after an automated scan. Criteria: HTTPS, OAuth, `title` plus `readOnlyHint`/`destructiveHint` on every tool, separate read and write tools, privacy policy, docs, support contact.
20. **Two close-fit vendors died in 2026.** Graphlit is winding down and Pinecone's Picker-based Drive connector reached end of life on 31 Aug 2026. Of 18 newly examined vendors, only Supermemory picks individual Drive files *and* has an OAuth MCP server; nobody has document maintainers with contradiction alerts or a Markdown/git mirror. Details in [existing-solutions-context-service.md](existing-solutions-context-service.md).

---

## 3. Recommended architecture: "Zuri Context Service" + Claude.ai Team (Option I)

**Detailed design and roadmap: [zuri-context-service.md](zuri-context-service.md).**

In short: we build a small service that owns the knowledge:
- a registry of hand-picked Drive files with maintainers and many-to-many project links;
- a Markdown mirror in git, so diffs come for free;
- incremental per-project `SUMMARY.md` files with inconsistency detection and maintainer notifications;
- a remote MCP server (Team custom connector, per-user OAuth, role checks).

It's delivered in the three phases of [master-plan.md](master-plan.md):
- **Phase 1 (MVP):** the context service + a minimal admin web UI, no chat of our own. People chat in Claude.ai (Team subscription), and group AI chat happens in Slack via Claude Tag with the same connector (usage-billed).
- **Phase 2 (Zuri only):** our own chat, the "Zuri Hub" (human and AI, API-billed via a model gateway). It replaces Slack and Claude Tag. Claude.ai stays in parallel.
- **Phase 3:** offered publicly at about $5/user/month. AI chat is paid separately (the customer's own Claude subscription via our connector, their own API key, or metered credits).

Build-vs-buy research ([existing-solutions-context-service.md](existing-solutions-context-service.md), two rounds, about 30 candidates) found no product to adopt. We build, reusing FastMCP 4.x (OAuth connector), Google's Picker sample, Drive's Markdown export, the Docs API suggestion mode and the "LLM Wiki" summary pattern.

**Covers:** R1, R2, R4, R7 (via MCP), R8, R12, R13, and R14 for personal chats. **Gaps until Phase 2:** group chat only via Slack (R5, usage-billed), R11 follows Slack rules, and R10 (replacing Slack) isn't met. Phase 2 closes all three.

**Low-tech pilot (I-lite):** sync worker → Markdown + `SUMMARY.md` in a private GitHub repo → linked into a shared Claude.ai project via the GitHub integration (manual "Sync now"). About 1 week. Covers R12/R13 on the subscription.

---

## 4. Other options (updated for R12–R14)

### A: Claude-native only (Claude.ai Team + Claude Tag), no custom service
Fastest start, but no hand-picked files (only a Drive folder or shared drive as scope), no `.md` mirror or diffs, no project summary (Claude Tag channel memory is the closest thing), no maintainer. Personal chats run on the subscription. Slack channel AI is usage-billed. It's now best seen as **Option I without the context service**.

### B: Dust.tt
Spaces per project, granular Drive selection (per-file not confirmed **(verify)**), multi-model, Slack integration, EU data residency on Business and Enterprise. It has its own remote OAuth MCP server (`dust.tt/mcp`, `eu.dust.tt/mcp`, DCR and CIMD) that Claude could in principle add as a connector, though Claude.ai isn't named in its docs **(verify)**. Its internal index isn't a user-visible `.md` mirror with diffs, and there are no rolling project summaries. Pro seats cost €24/month billed yearly (€30 monthly) *plus* nothing from the Team subscription is reused for chat inside Dust, so it fails R14.

### C: Onyx (formerly Danswer)
Open source (MIT core + Enterprise directory), any LLM including local ones, Drive scoped by shared drive, folder or My-Drive user (no individual files in 5.x), Slack bot. It has an OAuth MCP *server* on all editions (search tools only). Its index isn't Markdown with diffs, there are no summaries, group RBAC is Enterprise-only, and it bills via API, so it fails R14. Onyx Cloud is $20/user/month; self-hosted Enterprise pricing is quote-only.

### D: Open WebUI (self-hosted)
The best off-the-shelf **group chat with AI** (Channels with @model). Model-agnostic, with RBAC. Since v0.12.3 its Google Drive picker uses the `drive.file` scope and multi-select (for chat attachments), a useful reference for our admin UI.
- R12/R13 need the same sync worker and summarizer as Option I, pushed into its Knowledge collections via the API.
- It can also *use the Zuri MCP server* (it's an MCP client, not a server).
- **Fails R14:** every message is billed at API rates.
- **Licence:** clause 4 forbids altering "Open WebUI" branding in deployments with more than fifty end users in a rolling 30 days without written permission or an enterprise licence, which rules it out as the base of a resold product.
- **Role in the plan:** a throwaway prototype or fallback group-chat front-end. To try it without installing anything: Railway one-click template (~$5–20/month) or Elestio managed hosting, plus an Anthropic API key.

### E: Mattermost / Zulip + AI agent plugin
A real Slack replacement with per-channel agents and MCP tool grants, model-agnostic. Fails R14 (API billing). The sync worker and summarizer are still needed. Licences: Mattermost's compiled builds are MIT, self-built servers are AGPL v3 or commercial, and its FAQ requires a reseller agreement for resale; Zulip is Apache-2.0 and the cleanest for white-label, but all three are the wrong shape for a Claude-like AI chat.

### F: NotebookLM / Gemini
Native per-file Drive sources and automatic overviews. But it's Gemini, not Claude, there's no `.md` mirror or diffs, sharing is limited, and it fails R7/R14. Not recommended.

### G: Langdock, Slite, Guru, Supermemory, TypingMind Teams, Groupchats.ai, MultipleChat
Multi-model and some sharing, but no curation + maintainer + mirror combination, and all API-billed or with their own pricing. Notes from round 2: **Langdock** (EU, €25/seat) attaches individual Drive files to an agent as nightly-refreshed attachments but its MCP is API-key only and not for Claude.ai. **Slite** ($10–20/seat, EU-hosted) and **Guru** (custom pricing, ~$25/seat unverified) both have OAuth MCP servers that name Claude and both have human verification owners, but Drive is folder-scoped and there's no contradiction detection, mirror or summaries. **Supermemory** ($19–399/month flat) is the only product with a hosted Picker for single Drive files plus an OAuth MCP server; its Markdown is internal and lossy, with no history, maintainers, project roles or EU region. All are useful as prior art, none as the platform. Not recommended.

### H: Fully custom chat app → now **Phase 2** of the master plan
Meets every functional requirement, including R5/R6/R10/R11 natively, but chats in it **fail R14**: they go through the API. The master plan accepts this for Phase 2 (Zuri leaves Slack), while personal chats can stay in Claude.ai on the subscription. It's built *on top of* Option I: the sync worker, summarizer and MCP server are reused unchanged. Because Phase 3 resells it, it should be our own app rather than a fork of Open WebUI (whose licence restricts rebranding above 50 users) or Mattermost. The layout is fixed by the user-written [design.md](design.md): a Slack-like three-pane app (icon rail with Home and usage-sorted projects, a conversation list with a per-project Files view, a wide main pane, search in the top bar). Login reuses the OAuth everyone at Zuri already has (Google, and Zoho Mail via OpenID Connect), so there's no password store. Details: [zuri-context-service.md §6](zuri-context-service.md).

---

## 5. Comparison

| | R1 hand-picked | R2 maintainer | R5 group AI chat | R7 any model | R8 Claude-like chat | R12 md + diffs | R13 project summary | R14 Team subscription | Effort |
|---|---|---|---|---|---|---|---|---|---|
| **I Context Service + Claude.ai/Tag** | ✓ | ✓ | ◐ Slack, usage-billed | ✓ (MCP) | ✓ native | ✓ | ✓ | ✓ personal / ◐ group | Medium |
| I-lite GitHub repo + Claude projects | ✓ | ◐ | ✗ | ◐ | ✓ | ✓ | ✓ | ✓ | Low |
| A Claude.ai + Tag only | ◐ folder | ✗ | ◐ | ✗ | ✓ | ✗ | ✗ | ✓ / ◐ | Very low |
| B Dust | ◐ folder | ✗ | ◐ | ◐ | ✓ | ✗ | ✗ | ✗ | Low |
| C Onyx | ◐ folder | ✗ | ◐ | ✓ | ✓ | ✗ | ✗ | ✗ | Medium |
| D Open WebUI (+ sync) | ◐→✓ | ◐ | ✓ | ✓ | ✓ | ◐→✓ | ◐→✓ | ✗ | Medium |
| E Mattermost + Agents | ◐ | ◐ | ✓ | ✓ | ◐ | ◐ | ◐ | ✗ | Medium-high |
| F NotebookLM | ✓ | ✗ | ✗ | ✗ | ◐ | ✗ | ◐ | ✗ | Low |
| H Fully custom app | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ | High |
| Supermemory (SaaS, round 2) | ✓ picker | ✗ | ✗ | ◐ | ✓ via Claude.ai | ◐ internal, no history | ✗ | ✓ if its OAuth MCP works in Claude.ai (verify) | Low |
| Slite / Guru (SaaS, round 2) | ◐ folder | ◐ human verifiers | ✗ | ◐ | ✓ via Claude.ai | ✗ | ✗ | ✓ (OAuth MCP names Claude) | Low |

---

## 6. Recommendation (aligned with master-plan.md)

The plan follows [master-plan.md](master-plan.md). Details, durations and checklists are in [zuri-context-service.md](zuri-context-service.md).

1. **Phase 1, MVP (about 3.5–5 weeks):**
   - **Week 1, pilot:** sync worker for one big project (YAML registry, Docs → `text/markdown`, Sheets → per-tab Markdown tables, debounced polling, git commit), an LLM-Wiki-style `SUMMARY.md`, a GitHub-linked shared Claude.ai project, and Claude Tag in one Slack channel. **First, prove that a FastMCP 4.x + Google OAuth connector works in Claude.ai Team on web and mobile, and that its tools actually show up in chats.** Claim the Team API credits for the summarizer's API key.
   - **Then:** a DB-backed registry (with `tenant_id` from day one), a minimal admin web UI for projects, users and Drive documents (Picker), the remote MCP server with role checks registered as a Team connector and added to Claude Tag channels, and the maintainer loop (`report_inconsistency`, nightly lint, notifications).
   - No chat of our own. Chatting happens in Claude.ai and Slack/Claude Tag.
2. **Phase 2, Zuri Hub (about 2–4 months):**
   - Our own chat app, laid out as in [design.md](design.md) (icon rail, conversation list, main pane with a Files view): personal, project, group, whisper and DM conversations; AI as a participant via a model gateway (Claude API, optionally other or local models); invites with history choice. Login via Google or Zoho OpenID Connect, no passwords.
   - Then the Slack-replacement layer (threads, notifications, search, Slack import). **Switch Slack and Claude Tag off.** Claude.ai stays for personal chats on the subscription.
   - Write tools (suggest edits/comments on Google Docs) and scheduled agents.
3. **Phase 3, public service (about 2–3 months):** multi-tenancy, self-service onboarding, billing (~$5/user/month), Google OAuth verification (`drive.file` only, so no CASA assessment), Claude connector directory listing, GDPR/DPA. At $5 the price can't include API-billed AI chat, so AI is "bring your own Claude/key" or metered credits.

**Things to verify early** (each one is roughly a 1-hour test):
- FastMCP 4.x + Google OAuth completes as an Owner-added Team connector (web and mobile), Claude picks CIMD, and tools are offered to the model (open bug claude-ai-mcp #476).
- A custom connector works for all members inside a *shared* Claude.ai project (not documented either way).
- Claude Tag: the channel connector is confirmed to run as one admin account. Test the service-user-per-channel setup and the personal-connector path. This only matters for Phase 1.
- Markdown export quality for your typical docs (tables, base64 images, footnotes, **tabs**). Comments are *not* included in exports; use the Drive `comments` API if they matter.
- Whether your docs live in personal My Drives or Shared Drives. This affects the auth choice: `drive.file` via Picker with stored refresh tokens, or a service account given access to the picked files (a Shared Drive membership is cleanest).
- Whether everyone's Google and Zoho Mail addresses are identical. If not, the Phase 2 Hub needs a one-time account-linking step (the registry keys identities on the provider's `sub`, as Zoho's docs require).

---

## Sources

- Subscription OAuth policy (primary): https://code.claude.com/docs/en/legal-and-compliance , Agent SDK pause: https://support.claude.com/en/articles/15036540 , Team API credits: https://platform.claude.com/docs/en/about-claude/api-credits-for-subscribers ; press coverage of the February 2026 clarification: https://www.theregister.com/software/2026/02/20/anthropic-clarifies-ban-on-third-party-tool-access-to-claude/5014546 , https://gigazine.net/gsc_news/en/20260220-anthropic-third-party-block/ , https://alternativeto.net/news/2026/2/anthropic-officially-bans-using-subscription-authentication-for-third-party-claude-use
- Custom connectors (remote MCP), Team owners, per-user auth: https://support.claude.com/en/articles/11175166-about-custom-connectors-remote-mcp , https://claude.com/docs/connectors/custom
- Claude GitHub integration (manual Sync now): https://support.claude.com/en/articles/10167454-use-the-github-integration , auto-sync request: https://github.com/anthropics/claude-ai-mcp/issues/180
- Claude Google Workspace connectors (Drive not in shared projects): https://support.claude.com/en/articles/10166901-use-google-workspace-connectors , https://claude.com/docs/connectors/google/drive
- Claude group chat limitations: https://usepoly.co/claude-group-chat , https://www.smithstephen.com/p/in-a-shared-chatgpt-project-everyone
- Claude Tag: https://claude.com/docs/claude-tag/overview , setup: https://claude.com/docs/claude-tag/admins/setup-overview , spend limits: https://claude.com/docs/claude-tag/admins/set-spend-limit , connectors in channels: https://claude.com/docs/claude-tag/admins/add-connections , https://claude.com/docs/claude-tag/admins/connections/custom , agent identity: https://claude.com/docs/claude-tag/concepts/agent-identity , personal connectors: https://claude.com/docs/claude-tag/concepts/personal-connectors , launch credit (expired): https://support.claude.com/en/articles/15575654-claude-tag-launch-promo-for-claude-team-and-enterprise , legacy bot retirement: https://code.claude.com/docs/en/slack , Team usage credits at API rates: https://support.claude.com/en/articles/12005970 , API prices: https://platform.claude.com/docs/en/about-claude/pricing , pricing analysis: https://www.eesel.ai/blog/claude-tag-pricing , overview: https://www.digitalapplied.com/blog/anthropic-claude-tag-slack-team-collaboration-2026
- Claude Code scheduled tasks (routines): https://code.claude.com/docs/en/routines , Managed Agents scheduled deployments: https://platform.claude.com/docs/en/managed-agents/scheduled-deployments , Claude Code in Team seats: https://support.claude.com/en/articles/11845131
- Claude connector auth and directory: https://claude.com/docs/connectors/building/authentication , https://claude.com/docs/connectors/building/review-criteria , https://claude.com/docs/directory/publish , OAuth issues: https://github.com/anthropics/claude-ai-mcp/issues/313 , https://github.com/anthropics/claude-ai-mcp/issues/122 , https://github.com/anthropics/claude-ai-mcp/issues/476
- Zoho Accounts OpenID Connect (second login for the Hub): https://www.zoho.com/accounts/protocol/oauth/sign-in-using-zoho.html ; FastMCP generic OIDC proxy and MultiAuth (one login provider per server): https://gofastmcp.com/servers/auth/oidc-proxy , https://gofastmcp.com/servers/auth/multi-auth
- FastMCP and MCP spec: https://gofastmcp.com/integrations/google , https://gofastmcp.com/servers/auth/oauth-proxy , https://github.com/jlowin/fastmcp/releases , https://github.com/jlowin/fastmcp/issues/1919 , https://github.com/jlowin/fastmcp/pull/4206 , https://modelcontextprotocol.io/specification/2026-07-28/changelog , https://py.sdk.modelcontextprotocol.io/run/authorization/
- Drive export formats (text/markdown, first-sheet-only CSV): https://developers.google.com/workspace/drive/api/guides/ref-export-formats , 10 MB export cap: https://developers.google.com/workspace/drive/api/reference/rest/v3/files/export , Docs Markdown import/export: https://workspaceupdates.googleblog.com/2024/07/import-and-export-markdown-in-google-docs.html , Docs tabs: https://developers.google.com/workspace/docs/api/how-tos/tabs , https://issuetracker.google.com/379170478 , Sheets batchGet: https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/batchGet
- Drive push notifications: https://developers.google.com/workspace/drive/api/guides/push , https://fast.io/resources/google-drive-webhook/ , quotas: https://developers.google.com/workspace/drive/api/guides/limits , https://developers.google.com/workspace/sheets/api/limits
- Docs suggestions and comments: https://developers.google.com/workspace/docs/api/how-tos/suggestions , https://developers.google.com/workspace/drive/api/guides/manage-comments ; Google-hosted Workspace MCP servers: https://developers.google.com/workspace/guides/configure-mcp-servers , https://developers.google.com/workspace/drive/api/guides/configure-mcp-server
- Drive scopes (`drive.file`, Picker): https://developers.google.com/workspace/drive/api/guides/api-specific-auth
- Drive→Markdown tools: https://github.com/mieweb/wikiGDrive , https://github.com/zonca/gdrivemd , https://www.zonca.dev/posts/2026-05-24-gdrivemd-tutorial , https://github.com/shanemcd/gcmd , native .md in Drive (5 Oct 2026): https://workspaceupdates.googleblog.com/2026/10/preview-edit-and-collaborate-on-Markdown-files-natively-across-Drive-and-Docs.html
- Sheets per-tab export: https://www.skills.sh/googleworkspace/cli/recipe-backup-sheet-as-csv
- Onyx: https://docs.onyx.app/admins/connectors/official/google_drive/overview , https://docs.onyx.app/enterprise_edition/permissioning , https://docs.onyx.app/overview/onyx_anywhere/mcp_server , https://onyx.app/pricing
- Dust: https://docs.dust.tt/docs/what-data-do-the-agents-have-access-to , https://docs.dust.tt/docs/google-drive-connection , https://docs.dust.tt/docs/dust-mcp-server , https://dust.tt/pricing , https://www.fast.io/resources/dust-ai-workspace.md
- Round-2 vendors: Langdock https://www.langdock.com/pricing , https://docs.langdock.com/en/using-langdock/guides/integrations/mcp/langdock-agent-mcp-server ; Slite https://slite.com/changelog/slite-mcp-server , https://slite.com/pricing ; Guru https://help.getguru.com/docs/verifying-and-unverifying-cards , https://developer.getguru.com/docs/guru-mcp-server-overview ; Supermemory https://supermemory.ai/docs/memory-api/connectors/google-drive , https://supermemory.ai/docs/supermemory-mcp/introduction , https://supermemory.ai/pricing ; Graphlit sunset https://www.graphlit.com/legal/sunset ; Pinecone release notes https://docs.pinecone.io/release-notes/2026
- Open WebUI: https://docs.openwebui.com/features/channels/ , https://docs.openwebui.com/features/authentication-access/rbac/permissions.md , Drive picker with drive.file (v0.12.3): https://github.com/open-webui/open-webui/releases/tag/v0.12.3 , MCP client: https://docs.openwebui.com/features/extensibility/mcp , hosting: https://railway.com/deploy/open-webui , https://elest.io/open-source/openwebui/resources/quickstart , enterprise: https://docs.openwebui.com/enterprise.md
- Mattermost Agents: https://docs.mattermost.com/agents/docs/admin_guide.html , licence: https://raw.githubusercontent.com/mattermost/mattermost/master/LICENSE.txt , https://docs.mattermost.com/product-overview/faq-license ; Zulip licence: https://raw.githubusercontent.com/zulip/zulip/main/LICENSE
- NotebookLM Enterprise: https://docs.cloud.google.com/agentspace/notebooklm-enterprise/docs
- Langdock / TypingMind: https://docs.langdock.com/en/using-langdock/guides/chatbots/teams-bot.md , https://docs.typingmind.com/typingmind-team/getting-started/typingmind-vs-claude-team
- Open WebUI licence (branding clause, clause 4): https://docs.openwebui.com/license/ , https://raw.githubusercontent.com/open-webui/open-webui/main/LICENSE
- Google Drive scopes and verification (non-sensitive vs restricted/CASA): https://developers.google.com/workspace/drive/api/guides/api-specific-auth , https://support.google.com/cloud/answer/9110914 , https://support.google.com/cloud/answer/13465431 , https://appdefensealliance.dev/casa/casa-tiering , https://developers.google.com/identity/protocols/oauth2/production-readiness/overview , https://nango.dev/docs/api-integrations/google-shared/google-security-review
- Claude connector directory submission: https://claude.com/docs/connectors/building/submission , https://support.claude.com/en/articles/11596036-anthropic-mcp-directory-faq
