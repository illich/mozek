# Open-source / self-hostable platforms vs. the "Zuri Context Service" requirements

Requirements used for scoring (from `zuri-context-service.md`):
- **R1** per-file curation of Drive files (Docs/Sheets/Slides), M:N link to projects
- **R2** one owner/maintainer per doc, notified on AI-found inconsistencies
- **R3** auto-synced Markdown mirror with diff/history (git-like), human-accessible
- **R4** auto-rebuilt per-project summary .md on major changes
- **R5** remote MCP server usable as a Claude.ai custom connector, per-user OAuth, per-project roles (admin/member/reader)

Repo stats were pulled from the GitHub REST API on 2026-10-29 (stars, licence SPDX, last push, latest release). Claims marked **(unverified)** come from prior knowledge and were not re-checked against a primary source in this session.

## 1. Which candidates cover how many requirements? (overview / scorecard)

### Takeaway
No open-source project found covers all five requirements, and none covers R1 (per-file curation), R2 (doc maintainer plus notifications) and R3 (git-style Markdown history) together. The closest is **Onyx**: it covers R5 (remote MCP with OAuth, per-user permissions, all editions) and part of R1 (Drive connector with Sheets, document sets as "projects"). R2, R3 and R4 are missing, and group RBAC is Enterprise-only. Everything else is either a RAG/chat platform with an MCP *client* (Open WebUI, LibreChat, AnythingLLM), or a Markdown/MCP memory tool with no Drive sync (Basic Memory, llm-wiki projects).

### Cited Findings
- GitHub stats on 2026-10-29 ([GitHub API](https://api.github.com/repos/)):
  - onyx-dot-app/onyx: 33,067 stars, licence "NOASSERTION" (MIT CE + EE licence), latest release v5.0.1 (2026-10-26), very active.
  - pipeshub-ai/pipeshub-ai: 3,886 stars, Apache-2.0, v1.0.0 (2026-10-27), active.
  - airweave-ai/airweave: 6,550 stars, MIT, v0.9.73. **Last push 2026-06-05**, so it looks stale or moved; verify.
  - open-webui/open-webui: 157,127 stars, custom licence (NOASSERTION), v0.12.5 (2026-10-29).
  - Mintplex-Labs/anything-llm: 67,694 stars, MIT, v1.19.0 (2026-10-26).
  - khoj-ai/khoj: 37,885 stars, AGPL-3.0, last release 2.0.0-beta.28 (2026-03-26), last push 2026-08-02.
  - infiniflow/ragflow: 93,074 stars, Apache-2.0, v1.0.0-rc2 (2026-10-16).
  - langgenius/dify: 159,885 stars, custom licence (NOASSERTION; Apache-2.0-based with extra conditions **(unverified)**), 1.18.0 (2026-10-23).
  - Cinnamon/kotaemon: 25,836 stars, Apache-2.0, v0.12.0 (2026-05-31), last push 2026-07-14.
  - arabold/docs-mcp-server: 1,838 stars, MIT, v3.4.1 (2026-10-25).
  - basicmachines-co/basic-memory: 4,396 stars, AGPL-3.0, v0.24.1 (2026-10-23).
  - mieweb/wikiGDrive: 22 stars, MIT, v2.17.1 (2026-03-24).
  - topoteretes/cognee: 32,884 stars, Apache-2.0, v1.6.6rc1.
  - getzep/graphiti: 32,330 stars, Apache-2.0, v0.31.2.
  - The danny-avila/LibreChat API path returned "Moved Permanently", so the repo was renamed or transferred; stats not retrieved.

- Scorecard (✔ meets, ◐ partial, ✘ not met; details in sections below):

| Project | R1 per-file Drive + M:N projects | R2 owner/notify | R3 MD mirror + history | R4 project summary | R5 remote MCP + OAuth + roles | Licence |
|---|---|---|---|---|---|---|
| Onyx | ◐ (Drive incl. Sheets; selection by drive/folder/user, not documented per file; document sets ≈ projects, M:N) | ✘ | ✘ | ✘ | ✔/◐ (OAuth remote MCP, all editions; group RBAC is EE) | MIT + EE |
| PipesHub | ◐ (Drive incl. Sheets/Slides; per-file selection not documented) | ✘ | ✘ | ✘ | ◐ (MCP server with PAT; OAuth not documented) | Apache-2.0 |
| Airweave | ✘ (whole My Drive/shared drives, no config options) | ✘ | ✘ | ✘ | ◐ (MCP server; per-user OAuth unclear) | MIT |
| Open WebUI | ◐ (Drive picker for individual files reported; knowledge bases with access control) | ✘ | ✘ | ✘ | ✘ (MCP client, not server) | custom |
| Dify | ◐ (Online Drive datasource plugin) | ✘ | ✘ | ◐ (could be built as a workflow) | ◐ (app → MCP server, secret-URL auth, no per-user OAuth) | custom |
| RAGFlow | ◐ (Drive connector **(unverified)**) | ✘ | ✘ | ◐ (RAPTOR tree summaries **(unverified)**) | ◐ (MCP server with API key **(unverified)**) | Apache-2.0 |
| Basic Memory | ✘ (no Drive) | ✘ | ◐ (Markdown files; git is DIY locally; history in paid Cloud) | ✘ | ◐ (remote via paid Cloud; local OSS is stdio) | AGPL-3.0 |
| wikiGDrive | ◐ (whole Shared Drive → Markdown, Docs + Drawings) | ✘ | ◐ (Markdown; git not confirmed in README) | ✘ | ✘ | MIT |
| llmwiki / llm-wiki-mcp | ✘ | ✘ | ◐ (Markdown wiki) | ◐ (LLM maintains wiki pages) | ◐ (llmwiki: Claude via MCP; llm-wiki-mcp: stdio only) | ? |

### Inferences
- **No project meets R2.** Even enterprise tools lack a per-doc maintainer who gets notified of AI-found inconsistencies.
- **No project meets R3 as specified** (git history of a Drive Markdown export). Every RAG platform stores chunks and vectors, not human-browsable versioned Markdown.
- These two requirements, plus R4, are the core of Zuri's value. Adopting a platform would mainly save R5 and the sync plumbing for R1, and would bring a large system (Vespa/OpenSearch, Redis, workers) for a 30-person company.

### Gaps
- LibreChat stats not fetched (repo moved).
- Not checked: Khoj, Kotaemon, Cognee or Graphiti Drive connector details. From prior knowledge **(unverified)**, none offers per-file Drive curation or document maintainers; Cognee and Graphiti are memory/graph libraries with MCP servers, not Drive mirrors.

## 2. Onyx (ex-Danswer): best fit for R5 and partly R1

### Takeaway
Onyx ships an official remote MCP server (Streamable HTTP, OAuth with dynamic client registration, PKCE, `read:search` scope) on **all editions**, and it enforces the signed-in user's document permissions. That is the closest thing to a ready-made R5. Two catches: group-based RBAC, curators and Drive permission sync are Enterprise Edition (paid), and the Drive connector selects shared drives, folders or users, not documented individual files.

### Cited Findings
- **Remote MCP + OAuth:** "Use a client that supports remote Streamable HTTP MCP servers and OAuth." Onyx acts as the OAuth authorization server and supports dynamic client registration and Client ID Metadata Documents. PATs and API keys are also supported for bearer auth. — [Onyx MCP server docs](https://docs.onyx.app/overview/onyx_anywhere/mcp_server.md)
- **Tools:** `search_indexed_documents` (filters: source types, **document sets**, time cutoff, agent), `search_web` and `open_urls`, plus resources listing sources, document sets and agents. There are no "read full document", "history" or "summary" tools. — [Onyx MCP server docs](https://docs.onyx.app/overview/onyx_anywhere/mcp_server.md)
- **Permissions in MCP:** "Connecting an app does not give it access to documents you cannot read." Search returns only documents the connected user can access. OAuth "does not grant administrator access or bypass document permissions." — [Onyx MCP docs](https://docs.onyx.app/overview/onyx_anywhere/mcp_server.md); [Onyx MCP server config](https://docs.onyx.app/deployment/configuration/mcp_server.md)
- **Editions:** "The MCP server is available on all editions." It is enabled by default in Docker Compose and Helm, and OAuth turns on automatically when `WEB_DOMAIN` is HTTPS. Access tokens last up to 15 minutes, refresh tokens rotate, and grants expire after 30 days. On Onyx Cloud, tokens need a paid subscription. — [Onyx MCP docs](https://docs.onyx.app/overview/onyx_anywhere/mcp_server.md); [config](https://docs.onyx.app/deployment/configuration/mcp_server.md)
- **Claude:** the docs describe adding Onyx as a remote custom connector in Claude Desktop. The server must be publicly reachable because "Claude Desktop remote connectors connect from Anthropic's cloud." Claude.ai web is not explicitly mentioned. — [Onyx MCP docs](https://docs.onyx.app/overview/onyx_anywhere/mcp_server.md)
- **Drive connector:** supports Google Docs, Sheets and Slides. The "Specific" mode lists Shared Drive URLs, Folder URLs or My Drive emails, so file-level selection is not documented. "Specific Groups" and "Sync with Google Drive" permission modes need a Business plan or higher, or Onyx Cloud. — [Onyx Google Drive connector](https://docs.onyx.app/admins/connectors/official/google_drive/overview)
- **Licensing / EE:** CE is MIT; the `ee` directories are under the Onyx Enterprise License. "RBAC protection to documents and resources are only available in the Enterprise Edition." Curators are an EE role. Permission sync from external systems is EE. — [Onyx access controls](https://docs.onyx.app/security/architecture/access_controls); [Onyx EE](https://docs.onyx.app/admins/getting_started/enterprise_edition); [wz-it comparison (third-party)](https://wz-it.com/en/blog/onyx-self-hosted-installation/)
- From Onyx 4.7 the permission model splits into "Manage Groups", "Manage Connectors & Document Sets" and "Manage Agents". — [Onyx "what's changing"](https://docs.onyx.app/admins/permissions/whats_changing)
- A community MCP wrapper (lupuletic/onyx-mcp-server, API-token based) also exists, but the official server supersedes it. — [glama listing](https://glama.ai/mcp/servers/lupuletic/onyx-mcp-server)

### Inferences
- **Mapping:** Onyx "document set" ≈ Zuri project (a document can sit in several sets, so M:N). The member/reader distinction would need EE user groups. Onyx has no admin/member/reader distinction for *write* tools because its MCP is read-only (`read:search`).
- **Per-file curation workaround:** put the curated files in a dedicated folder or shared drive (or shortcuts; shortcut handling unverified), or write a small custom connector that takes a list of file IDs. Onyx supports custom connectors in Python **(unverified detail)**.
- **Gaps to fill:** R2, R3 and R4 would be separate services: a sync-to-git worker, a summarizer and a notifier. Onyx would then be only the search/MCP layer, and Claude still wouldn't get full-document reads, history or summaries through Onyx's MCP tools unless we add another MCP server. That means building most of the Zuri service anyway, plus running heavy Onyx infrastructure.
- **Effort estimate:** adopting Onyx for R5 saves about 1–2 weeks (OAuth MCP), but group-scoped access needs an EE licence or custom code.

### Gaps
- Whether the Drive connector accepts single-file URLs in recent versions (5.x) was not confirmed. The docs only list drive, folder or user scopes.
- EE pricing not retrieved.
- Claude.ai *web* Team custom connector compatibility (versus Desktop) is not explicitly confirmed by Onyx docs.

## 3. PipesHub

### Takeaway
PipesHub is Apache-2.0 enterprise search with a Drive connector (Docs/Sheets/Slides) and an MCP server, and it enforces source permissions at query time. Its documented MCP auth is a personal access token, not per-user OAuth for Claude.ai. There is no per-file curation, maintainer, history or summary feature.

### Cited Findings
- Licence: "fully open-source (Apache 2.0) and self-hostable". v1.0.0 released 2026-10-27. — [PipesHub GitHub](https://github.com/pipeshub-ai/pipeshub-ai); [GitHub API](https://api.github.com/repos/pipeshub-ai/pipeshub-ai)
- MCP server repo [pipeshub-ai/mcp-server](https://github.com/pipeshub-ai/mcp-server/). The hosted endpoint is `https://app.pipeshub.com/mcp`; coding agents connect "with a personal access token", which runs "as you". — [PipesHub GitHub](https://github.com/pipeshub-ai/pipeshub-ai)
- Permissions are "resolved when the query runs, against the source system's own permissions." It gives block-level citations. — [PipesHub GitHub](https://github.com/pipeshub-ai/pipeshub-ai)
- The README mentions knowledge bases but no document owner, versioning or summaries. — [PipesHub GitHub](https://github.com/pipeshub-ai/pipeshub-ai)

### Inferences
- PipesHub inherits Drive ACLs rather than having Zuri-style project roles. Per-project roles would need custom work.
- With PAT-only auth, Claude.ai Team users would each paste a token as a request header. That is not proper per-user OAuth.

### Gaps
- Whether the MCP server supports OAuth (DCR) for Claude.ai custom connectors was not found. Per-file Drive selection was not documented.

## 4. Airweave

### Takeaway
Airweave is MIT-licensed and syncs sources into searchable collections exposed via MCP, but its Drive connector syncs whole My Drive or shared drives with "no additional configuration options". The repo's last push was June 2026.

### Cited Findings
- Google Drive connector: "This connector does not have any additional configuration options." It supports My Drive and shared drives, uses OAuth with your own client ID and secret, and has no documented Sheets handling. — [Airweave Google Drive docs](https://docs.airweave.ai/docs/connectors/google-drive.md)
- Last push and release v0.9.73 on 2026-06-05. — [GitHub API](https://api.github.com/repos/airweave-ai/airweave)

### Inferences
- Poor fit for R1, R2, R3 and R4. MCP per-user OAuth with group scoping is unclear. The stale activity is a risk.

### Gaps
- Whether Airweave's MCP server supports OAuth for Claude.ai was not verified. The reason for no pushes since June 2026 (move, rename or slowdown) is unknown.

## 5. Chat/RAG platforms: Open WebUI, Dify, AnythingLLM, LibreChat, RAGFlow, Kotaemon, Khoj

### Takeaway
These are chat front-ends or RAG builders. Mostly they *consume* MCP rather than serve a per-user OAuth MCP endpoint. Dify can publish an app as an MCP server, but auth is a secret URL. None has doc maintainers, Markdown/git history or per-project auto-summaries. Open WebUI remains relevant for the later "Zuri Hub" phase (Section 6 of the design), not for the context service.

### Cited Findings
- **Dify:** an app can be published as an MCP server. "The MCP Server URL contains authentication credentials, so treat it like an API key"; regenerating it invalidates the old one. There is no per-user OAuth. — [Dify publish MCP](https://docs.dify.ai/en/use-dify/publish/publish-mcp)
- **Dify:** knowledge pipelines (data-source plugins since 1.9.0) support "Online Drive (Google Drive, OneDrive, etc.)". — [Dify upload files / pipeline](https://docs.dify.ai/en/cloud/use-dify/knowledge/knowledge-pipeline/upload-files); [Dify datasource plugin](https://docs.dify.ai/en/develop-plugin/dev-guides-and-walkthroughs/datasource-plugin)
- **Dify:** a forum user reports the Drive datasource only searched My Drive, not shared drives. — [Dify forum](https://forum.dify.ai/t/failing-to-sync-google-shared-drive-as-data-source-in-knowledge-pipeline/1410?tl=ja)
- **Open WebUI:** knowledge bases have access control. A model with knowledge attached is limited to those knowledge bases, otherwise it sees all the user can access. — [Open WebUI knowledge docs](https://docs.openwebui.com/features/ai-knowledge/knowledge)
- **Open WebUI:** native Google Drive and OneDrive file picking is reported in a README mirror (third-party copy, so verify). — [Gitee mirror](https://gitee.com/wwjedu/open-webui)
- **Open WebUI:** the companion tool "oikb" syncs a knowledge base from a directory, GitHub, S3, Confluence and other sources (as summarized by search; Drive not confirmed). — [Open WebUI docs llms.txt](https://docs.openwebui.com/llms.txt)
- **Open WebUI** licence is custom (NOASSERTION in the GitHub API); it moved to a branding-restricted licence in 2025 **(unverified detail)**. — [GitHub API](https://api.github.com/repos/open-webui/open-webui)
- **AnythingLLM** (MIT), **LibreChat**, **Khoj** (AGPL; last release a beta from March 2026), **Kotaemon** (Apache; last push July 2026) and **RAGFlow** (Apache; v1.0.0-rc2): not deep-checked. From prior knowledge **(unverified)**:
  - AnythingLLM and LibreChat are MCP clients with workspace/agent RAG and no remote OAuth MCP server for knowledge.
  - RAGFlow added external data-source connectors (incl. Google Drive) and an MCP server with API-key auth in 2025, plus RAPTOR hierarchical summaries.
  - None has per-file curation with maintainers or git history.

### Inferences
- If Zuri later builds the Hub on Open WebUI, Open WebUI would call the Zuri MCP server as a client. That matches the design doc and doesn't change the context-service decision.
- A Dify workflow could implement the R4 summarizer logic, but that's a heavy dependency for a cron script.

### Gaps
- Not verified from primary sources: RAGFlow's MCP auth and Drive per-file selection; Open WebUI's Google Drive picker in current docs (the official RAG page returned 404).

## 6. Markdown/MCP knowledge tools: Basic Memory, docs-mcp-server, LLM-wiki projects, wikiGDrive, Drive MCP servers

### Takeaway
These cover pieces of R3 and R4 (Markdown as the source of truth, LLM-maintained wiki pages) or R1 export (Drive → Markdown), but none combines Drive sync, git history, project summaries and a multi-user OAuth remote MCP. They are useful as *components or reference code*, not as a base to adopt.

### Cited Findings
- **Basic Memory:** AGPL-3.0. Knowledge is Markdown files. Locally it runs as `uvx ... basic-memory mcp`. Git is only a DIY sync option ("Manual (Git, Syncthing, etc.)"). The paid Cloud ($15–19/month) adds remote access, "Snapshots and backups… Browse history", built on WorkOS AuthKit. Teams is a shared cloud workspace. It has no Drive connector and no per-project roles. — [Basic Memory GitHub](https://github.com/basicmachines-co/basic-memory)
- **wikiGDrive:** MIT, 22 stars. It converts "Google Docs and Drawings into markdown", works from a whole Shared Drive, and watches for changes. The README mentions no Sheets/Slides, no per-file selection and no MCP/AI. Git integration isn't described in the README (from prior knowledge it has a git commit/push UI; **unverified**). — [wikiGDrive GitHub](https://github.com/mieweb/wikiGDrive)
- **llmwiki (lucasastorian):** an open-source implementation of Karpathy's "LLM wiki" pattern (upload docs, Claude builds the wiki via MCP). About 800 stars, last push 2026-04-27 (per aggregator). — [gittrend.io](https://gittrend.io/repo/lucasastorian/llmwiki)
- **llm-wiki-mcp (flsteven87):** a stdio MCP server over a local `--wiki-root` Markdown directory. — [getdrio listing](https://www.getdrio.com/mcp/io-github-flsteven87-llm-wiki-mcp/md)
- **Drive MCP servers:** felores/gdrive-mcp-server (MIT) converts Google Docs → Markdown automatically but has no git or history. Other Drive MCP servers (a-bonus/google-docs-mcp, aaronsb/google-workspace-mcp and others) are personal, single-user Drive access. — [felores/gdrive-mcp-server](https://github.com/felores/gdrive-mcp-server); [a-bonus/google-docs-mcp](https://github.com/a-bonus/google-docs-mcp); [aaronsb/google-workspace-mcp](https://github.com/aaronsb/google-workspace-mcp)
- **arabold/docs-mcp-server:** MIT, active (v3.4.1, 2026-10-25). It indexes documentation sources for coding agents (purpose from prior knowledge; **unverified** whether it handles Drive). — [GitHub API](https://api.github.com/repos/arabold/docs-mcp-server)
- No repo was found combining Google Drive → Markdown → git history → MCP. A targeted search returned only Drive MCP servers and rclone. — [search summary incl. rclone](https://en.wikipedia.org/wiki/Rclone)

### Inferences
- The R3 sync worker (Drive export → Markdown → git commit with debounce) is small (a few hundred lines). Existing pieces (Drive's native `text/markdown` export, the Sheets API) make it simpler than adopting wikiGDrive.
- The Karpathy "LLM wiki" pattern validates the R4 approach (an LLM incrementally maintaining Markdown summary pages), but the implementations found are single-user and local.

### Gaps
- No primary-source check on llmwiki's licence or multi-user support. No search done on Cognee or Graphiti Drive ingestion (both are memory/graph libraries with MCP servers; not curated-Drive tools, **unverified**).

## 7. Overall recommendation inputs (build vs. adopt)

### Takeaway
Build the Zuri Context Service as designed (it is small). Borrow from open source tactically: the official MCP SDK's OAuth support for R5 (or study Onyx's MCP/OAuth implementation, MIT CE, as a reference), Drive Markdown export, and the LLM-wiki prompting patterns. Revisit Onyx only if semantic search over many documents becomes necessary (Phase 3). It could then sit behind the Zuri MCP server as a search backend, scoped by document sets.

### Cited Findings
- Onyx is the only candidate with documented remote OAuth MCP enforcing per-user permissions on all editions. — [Onyx MCP docs](https://docs.onyx.app/overview/onyx_anywhere/mcp_server.md)
- Group RBAC and permission sync in Onyx are EE-only. — [Onyx access controls](https://docs.onyx.app/security/architecture/access_controls)
- Dify's MCP uses secret-URL auth; PipesHub's uses PATs. — [Dify](https://docs.dify.ai/en/use-dify/publish/publish-mcp); [PipesHub](https://github.com/pipeshub-ai/pipeshub-ai)

### Inferences
- Effort to fill gaps on Onyx (R2 + R3 + R4 + per-file selection + project roles): roughly the same as the full Zuri MVP plus 2–3 weeks (Phases 0–2 in the design), with more operational load. So adoption doesn't pay off at Zuri's scale (30 people, about 500 files).

### Gaps
- Not checked for parity: commercial (non-OSS) offerings such as Glean, Guru (which *does* have doc verification and owners) and Notion AI connectors. Those are outside this note's open-source scope.
