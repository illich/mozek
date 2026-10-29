# Building blocks for a custom Zuri Context Service

Scope: reusable components for (1) curated Drive registry + Picker admin UI, (2) Drive→Markdown→git sync, (3) incremental LLM summarizer, (4) remote MCP server with Google-login OAuth as a Claude.ai Team custom connector. Research date: 2026-10-29. About 20 tool calls; some areas are thin and are listed under Gaps.

## Q1. Official and community Google Workspace MCP servers: can they be restricted to hand-picked files, use per-user OAuth, and work as Claude.ai connectors?

### Takeaway
Google now hosts official remote MCP servers for Drive, Docs and Sheets (Developer Preview). They use per-user OAuth, and Google documents them as Claude.ai custom connectors. None of the official or community servers has a "registered file list" allowlist: they see whatever the user's Google account sees. So they don't replace the registry/curation layer, but they're a useful complement for ad-hoc Drive access.

### Cited Findings
- The Google Drive MCP server is hosted at `https://drivemcp.googleapis.com/mcp/v1`. Status is **Developer Preview**, which requires membership in the Google Workspace Developer Preview Program. — [Google: Configure the Drive MCP server](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server) (page updated 2026-10-12)
- Drive MCP tools: `copy_file`, `create_file`, `download_file_content`, `get_file_metadata`, `get_file_permissions`, `list_recent_files`, `read_file_content`, `search_files`. Scopes are `drive.readonly` and `drive.file`. It inherits "the same permissions and data governance controls as the user". — [Google Drive MCP guide](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)
- Google's own Claude setup steps: create a Web-application OAuth client with "Use this client for an AI-powered agent" selected, and add redirect URI `https://claude.ai/api/mcp/auth_callback`. Then in Claude, go to Settings/Admin settings > Connectors > Add custom connector, enter the URL, and put the client ID and secret under Advanced settings. Requires Claude Pro/Max/Team/Enterprise. — [Google Drive MCP guide](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)
- Gotchas in Google's guide: requests are rejected unless the OAuth client has the "AI-powered agent" flag, so old clients may need to be recreated. Without `access_type=offline`, users may have to sign in about once an hour. Google also warns about indirect prompt injection and recommends Model Armor or similar. — [Google Drive MCP guide](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)
- There are parallel Docs (`docsmcp.googleapis.com`) and Sheets servers with their own configure guides. Google's index page is [Configure the Google Workspace MCP servers](https://developers.google.com/workspace/guides/configure-mcp-servers) and the Docs guide is [Configure the Docs MCP server](https://developers.google.com/workspace/docs/api/guides/configure-mcp-server). A third-party listing says the Sheets server is in "Developer Preview" — [developersdigest listing](https://www.developersdigest.tech/mcp-servers/google-workspace-sheets) (secondary source)
- **taylorwilsdon/google_workspace_mcp**: MIT, about 3.4k stars. Supports OAuth 2.1 multi-user auth, a stateless mode, streamable HTTP, tool tiers (core/extended/complete) and a `--read-only` flag, and it works as a Claude.ai connector in HTTP mode. There's no documented allowlist of Drive file IDs or folders (`ALLOWED_FILE_DIRS` applies to local files only). — [GitHub](https://github.com/taylorwilsdon/google_workspace_mcp)
- **isaacphi/mcp-gdrive**: MIT, about 282 stars, 11 commits. A local `npx` server (apparently stdio) with a Desktop-app OAuth client and `drive.readonly` + `spreadsheets` scopes. It exports Docs→Markdown, Sheets→CSV, Slides→text, and has no file allowlist. — [GitHub](https://github.com/isaacphi/mcp-gdrive)

### Inferences
- To keep the "curated, not crawled" principle (design §2.1), the official Drive MCP would have to be used with `drive.file` scope only. Even then, its reach depends on which files each user's OAuth grant covers. It has no notion of projects, maintainers or summaries. It is useful as a second connector for "open the live doc", not as the context service.
- taylorwilsdon's server is the best code to fork if you want a Python/FastMCP Google-Workspace server with multi-user OAuth already wired. Adding a registry-based allowlist check in front of its Drive tools would be a small patch, but its scope covers far more than Zuri needs.
- Savings: the official servers save little for the core service. They can stand in for the future `suggest_edit`/`add_comment` write tools (Phase 7), which would avoid writing Docs-API code.

### Gaps
- I couldn't confirm whether the Google-hosted MCP servers will reach GA, or what Workspace-admin controls exist for them. The page only mentions OAuth log events.
- Activity and release cadence of taylorwilsdon's repo: the GitHub page showed no releases or dates.

## Q2. MCP server frameworks with built-in OAuth for Claude.ai remote connectors (Google login in front of Postgres + git)

### Takeaway
**FastMCP (Python) with `GoogleProvider`** is the shortest path. It's an OAuth proxy that adds DCR emulation and CIMD in front of Google (Google itself has no DCR), and exposes the user's email to tools via `get_access_token()`. **Cloudflare `workers-oauth-provider`** plus the `remote-mcp-google-oauth` template is the TypeScript/edge equivalent. Claude's connector docs list several strict requirements that either option has to meet.

### Cited Findings
- FastMCP `GoogleProvider` uses the OAuth Proxy pattern "because Google doesn't support Dynamic Client Registration". You register `base_url` + `/auth/callback` in the Google Console. Required settings are `client_id`, `client_secret`, `base_url` and `required_scopes` (e.g. `openid`, `userinfo.email`). For production you also set `jwt_signing_key` and persistent `client_storage` wrapped in `FernetEncryptionWrapper`. Inside a tool, the user's email is `get_access_token().claims.get("email")`. GoogleProvider is new in 2.12.0, and the production options arrived in 2.13.0. — [FastMCP Google integration](https://gofastmcp.com/integrations/google)
- FastMCP OAuth Proxy has **CIMD support (new in 3.0.0, `enable_cimd` default True)** and DCR emulation: `/register` returns the fixed upstream credentials and records the client callback. `allowed_client_redirect_uris` can be pinned, with `https://claude.ai/api/mcp/auth_callback` given as the example. A consent screen is shown by default. `extra_authorize_params` passes extra parameters upstream. Production needs a shared store such as Redis or DynamoDB, and `MemoryStore` is for testing only. — [FastMCP OAuth Proxy](https://gofastmcp.com/servers/auth/oauth-proxy)
- Claude's connector auth requirements:
  - It supports `oauth_dcr`, `oauth_cimd`, Anthropic-held creds, and (beta) static headers. CIMD is chosen only if the AS metadata advertises `client_id_metadata_document_supported: true` **and** `none` in `token_endpoint_auth_methods_supported`; otherwise Claude falls back to DCR.
  - S256 PKCE is mandatory.
  - The server must return `401` with a `WWW-Authenticate: Bearer resource_metadata=...` header. A `WWW-Authenticate` header on a `200` is ignored.
  - The PRM `resource` must exactly equal the URL the user entered, and only the first `authorization_servers` entry is used.
  - Redirect URI is `https://claude.ai/api/mcp/auth_callback`. Claude Code uses port-agnostic loopback for both `localhost` and `127.0.0.1`.
  - `/token` must accept form-urlencoded bodies.
  - Timeouts are 10 s for discovery/registration/token and 30 s for refresh.
  - Use `invalid_grant` on refresh failure, and rotate refresh tokens for public clients.
  - Egress comes from `160.79.104.0/21`, and a WAF in front of the IdP can break discovery.
  - `client_credentials` is not supported.
  - Anthropic says to scope per-user access with your own per-user permissions, not per-tenant OAuth apps.
  
  — [Claude docs: Authentication for connectors](https://claude.com/docs/connectors/building/authentication)
- Static-header auth (an Owner enters an API key once) is beta for a limited set of orgs, and every member's requests carry the same key, so it "don't use it to identify which user is calling". Per-user roles therefore need OAuth. — [Claude docs: Authentication](https://claude.com/docs/connectors/building/authentication)
- Community issues report Claude.ai OAuth flows failing after the authorization-code redirect, before code exchange, and org-level connectors failing to complete OAuth. — [anthropics/claude-ai-mcp #313](https://github.com/anthropics/claude-ai-mcp/issues/313), [#212](https://github.com/anthropics/claude-ai-mcp/issues/212) (no official resolution found)
- Cloudflare `workers-oauth-provider`: MIT, about 1.9k stars, 337 commits, 1.x line (migration guide from 0.10). It provides OAuth 2.1 AS and resource server, DCR (RFC 7591), the CIMD draft (`clientIdMetadataDocumentEnabled: true`), RFC 8707/9728, and KV storage (`OAUTH_KV`) with hashed tokens and encrypted props. Google is handled through a "you own `/authorize`" pattern (docs/upstream-sign-in.md). — [GitHub](https://github.com/cloudflare/workers-oauth-provider)
- Template: `npm create cloudflare@latest -- my-mcp-server --template=cloudflare/ai/demos/remote-mcp-google-oauth`. It needs `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`, and one community copy shows an optional `HOSTED_DOMAIN` restriction. — [community copy on glama](https://glama.ai/mcp/servers/@ross-jill-ws/cloudflare-mcp-lab/blob/58e69fe9cc583cd48c0d7e600a34421af48845c6/README.md); [Cloudflare launch blog](https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/) (official repo README not verified)
- Official MCP Python SDK: `mcp.server.auth` makes a server an OAuth 2.1 **resource server** only. You implement one `TokenVerifier` method, and it serves RFC 9728 metadata. There's a `validate_token_resource` option, and the reference example is `examples/servers/simple-auth/`. — [MCP Python SDK authorization docs](https://py.sdk.modelcontextprotocol.io/authorization/). The TypeScript SDK v2 beta has an `OAuthTokenVerifier` interface and `@modelcontextprotocol/express` auth middleware — [TS SDK v2 reference](https://ts.sdk.modelcontextprotocol.io/v2/interfaces/_modelcontextprotocol_express.auth_types.OAuthTokenVerifier.html)

### Inferences
- With the official SDKs you still need a separate authorization server that does DCR or CIMD, because Google provides neither. That's exactly the part FastMCP's OAuth Proxy or Cloudflare's provider gives you. A Python stack (Postgres + git + summarizer in Python) points to **FastMCP 3.x + GoogleProvider**. Estimate: auth reduces to about 20 lines of config plus a Redis/Postgres-backed store, saving roughly 1 week of OAuth plumbing and spec-compliance debugging.
- Domain restriction to the Zuri Workspace: FastMCP's docs don't list an `hd` option. Pass `hd=zuri...` via `extra_authorize_params` (a hint only), **and** enforce the email domain (or `hd` claim) server-side in the role check. Making the Google OAuth consent screen "Internal" also restricts sign-in to the Workspace org.
- Cloudflare is attractive if you'd rather run the MCP edge on Workers. But the backend (Postgres, git repo) would then live elsewhere, which adds a hop, so it's less "small and boring" than a single Python container.
- Hosted auth vendors (Stytch, WorkOS AuthKit, Auth0, Descope) all advertise MCP auth (e.g. [Descope MCP SDKs](https://docs.descope.com/mcp/sdks)). For 30 users that's an extra vendor and contract, and FastMCP's proxy already covers the need.

### Gaps
- I didn't check each hosted vendor's (Stytch/WorkOS/Auth0/Descope) current MCP feature set, pricing or Claude.ai compatibility.
- I found no first-party confirmation that FastMCP GoogleProvider + Claude.ai Team custom connector works end-to-end. Test it in the pilot. Claude's [troubleshooting page](https://claude.com/docs/connectors/building/troubleshooting.md) exists but I didn't fetch it.

## Q3. Drive → Markdown sync tooling and native export quality; Sheets to Markdown

### Takeaway
The native `files.export` with `text/markdown` is officially supported for Google Docs and is the right default. Its handling of tables and images isn't documented by Google, so test it in the pilot. Sheets has no Markdown export, and CSV/TSV export covers the first sheet only, so per-tab export through the Sheets API (as the design already plans) is needed. No off-the-shelf tool does "hand-picked file list → Markdown → git" well. wikiGDrive is the closest, but it works on a whole Shared Drive.

### Cited Findings
- The Drive export-format table lists **Markdown (.md) `text/markdown` for Google Docs only**. Sheets exports are xlsx/ods/pdf/zip/csv/tsv, and **CSV/TSV are "first-sheet only"**. Slides exports are pptx/odp/pdf/txt. — [Google: Export MIME types](https://developers.google.com/workspace/drive/api/guides/ref-export-formats)
- Docs Markdown import/export (File > Download > Markdown) launched 2024-07-16. The post doesn't list which elements are supported. — [Workspace Updates blog](https://workspaceupdates.googleblog.com/2024/07/import-and-export-markdown-in-google-docs.html)
- Third-party claims about export quality (unverified): one says the native export "often struggles with anything beyond simple text and headings" and doesn't handle images — [file2markdown blog](https://www.file2markdown.ai/blog/google-docs-to-markdown) (vendor, promotional). Another says `text/markdown` gives no usable image files and suggests docx + `pandoc --extract-media` — [resolve247](https://resolve247.ai/tools/google-doc-to-markdown) (vendor).
- The Docs to Markdown add-on (GD2md-html) creates image links that need re-hosting and turns tables into HTML tables. — [Gd2md-html wiki](https://github.com/evbacher/Gd2md-html/wiki)
- **wikiGDrive** (mieweb): MIT, about 22 stars, 1,306 commits. It syncs Google Docs and Drawings (Drawings→SVG) to Markdown, scans a **whole Shared Drive**, rewrites Drive-ID links to paths, and polls for changes. It ships as Docker. Git isn't mentioned on the README. — [GitHub](https://github.com/mieweb/wikiGDrive)
- `gcmd` (PyPI) is a CLI wrapper around the Markdown export that can save each tab of a multi-tab Doc separately. — [PyPI gcmd](https://pypi.org/project/gcmd/) (not verified in depth)
- isaacphi/mcp-gdrive already contains working Docs→Markdown and Sheets→CSV export code (MIT). — [GitHub](https://github.com/isaacphi/mcp-gdrive)

### Inferences
- The sync worker is mostly glue: `files.get` (modifiedTime/version) → `files.export(text/markdown)` → front-matter → `git commit`. That's about 200–400 lines with google-api-python-client plus GitPython or the git CLI. Off-the-shelf tools save little, and wikiGDrive's whole-drive model goes against the curated design.
- Multi-tab Google Docs: check whether `files.export` includes all tabs (gcmd's per-tab feature suggests it may not, or that tabs need separate handling). This should go in the pilot test list.
- Sheets: `spreadsheets.get` (tab list) + `values.batchGet` per tab → a small Markdown-table renderer. No library is needed beyond something like `tabulate`/`pandas.to_markdown`. I couldn't verify a dedicated maintained tool.
- Images: for an LLM context service, dropping images or replacing them with alt-text placeholders is probably fine. If diagrams matter, fall back to docx export + pandoc for those files.

### Gaps
- No primary-source statement on how `text/markdown` export renders tables, images, comments, suggestions, footnotes or multi-tab docs. Run an empirical test on 3–5 real Zuri docs.
- I didn't verify rclone's current behaviour for exporting Docs as Markdown (it supports export formats for Google Docs, but I didn't confirm `md` support in this session).
- Export size limit: one secondary source mentions 10 MB, which I didn't confirm against Google's reference.

## Q4. "LLM-maintained wiki / summary" patterns usable as the summarizer

### Takeaway
Karpathy's "LLM Wiki" gist (April 2026) is the closest published pattern: immutable raw sources → an LLM-maintained Markdown wiki with `index.md` and an append-only `log.md` → a schema file (CLAUDE.md/AGENTS.md), run through ingest / query / lint operations, with lint finding contradictions. It maps almost one-to-one onto the Zuri design. It's a pattern document, not a library.

### Cited Findings
- The gist (created 2026-04-04) has three layers: raw sources (immutable), the wiki (LLM-written Markdown: summaries, entity/concept pages, overview) and the schema (CLAUDE.md/AGENTS.md defining structure and workflows). — [Karpathy gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- Operations:
  - **Ingest**: read a new source, write a summary, update the index and related pages, append to the log. One source may touch 10–15 pages.
  - **Query**: answer with citations, optionally filing the answer back into the wiki.
  - **Lint**: check for contradictions, stale claims, orphan pages and missing cross-references.
  
  `index.md` is described as working well "at moderate scale without embedding-based RAG". Contradictions are flagged both during ingest and lint, "still benefit from human review". — [Karpathy gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- The gist is framed as an "idea file" to paste into Claude Code or a similar agent. — [VentureBeat coverage](https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an) (secondary)

### Inferences
- Zuri mapping: `docs/` = raw sources, `projects/*/SUMMARY.md` + `INDEX.md` = wiki, `CHANGES.md` = log, and a repo-level `CLAUDE.md` = schema (summary template, contradiction-report format). The summarizer can then be a Claude Code headless run or a scheduled routine on the mirror repo, which would cost little code. Alternatively, a ~150-line script using the Messages API with structured output for `issues` (contradictions as JSON → `issues` table).
- Savings: the pattern saves design and prompt work, not code. Most of the effort is the trigger and diff selection plus the issue-extraction schema.

### Gaps
- I didn't evaluate packaged implementations of the LLM-wiki pattern (many community "llm-wiki" skills and repos exist) for maturity or license.
- No sources found on drift and error accumulation in repeated incremental summarization. The design's weekly full rebuild is a reasonable mitigation, but it isn't validated by a source.

## Q5. Google Picker + `drive.file` in a small admin web app

### Takeaway
Google maintains an official Picker web sample (`googleworkspace/browser-samples`, `drive/picker/helloworld.html`) and a guide. Together with the `drive.file` scope, this is a well-trodden, low-effort path, roughly 1 day of work.

### Cited Findings
- The official "Code sample for web apps" for Google Picker covers multi-select, hiding the nav pane and using the app's OAuth token. It needs a Client ID, an API key and the Cloud project number as the app ID. — [Google Picker sample](https://developers.google.com/workspace/drive/picker/guides/sample)
- Integration guide: [Integrate the Google Picker into web apps](https://developers.google.com/workspace/drive/picker/guides/web-picker). Sample repo pointer: `googleworkspace/browser-samples` → `drive/picker` — [Google Drive API samples](https://developers.google.com/drive/api/samples)
- The Drive MCP guide lists `drive.file` alongside `drive.readonly` as a Drive scope. — [Google Drive MCP guide](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)

### Inferences
- A `drive.file` grant is tied to the OAuth client and the user who picked the file. The sync worker must therefore run with that user's refresh token (or the admin's), or else files must be shared with a service account. This is design open question §7.1. For a 30-person Workspace, a service account with files shared to it (or Shared Drive membership) is operationally simpler for background sync. The Picker is then just the UI for choosing file IDs.

### Gaps
- I didn't find an official statement on how `drive.file` grants behave when a different user later accesses the same file through the same app.

## Q6. Reference implementations or blog posts of "curated Drive knowledge via MCP for Claude"

### Takeaway
I found no published reference implementation of the exact pattern (hand-picked Drive files → Markdown/git mirror → project summaries → per-user-role MCP for Claude.ai). The nearest artifacts are generic Workspace MCP servers and markdown-knowledge-base MCP servers with git sync.

### Cited Findings
- Search results showed markdown-KB MCP servers with git auto-commit (OKF Knowledge MCP), Drive-synced markdown notes (Knowledge Base MCP server) and git+SQLite-FTS5 memory (Anamnesis). None of them is Drive-curation plus project roles. — [OKF Knowledge MCP](https://glama.ai/mcp/servers/h9wf0n8jtn), [Knowledge Base MCP](https://glama.ai/mcp/servers/utxb3tnwud), [Anamnesis](https://skillselion.com/mcp/tool/io.github.oscardvs/anamnesis) (directory listings only, not evaluated)
- Google's documentation already describes using its official Drive MCP server as a Claude custom connector, which is the "uncurated" baseline. — [Google Drive MCP guide](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)

### Inferences
- Overall build-vs-reuse estimate for the MVP (Phases 0–1):
  - **Auth/MCP**: FastMCP + GoogleProvider (big saving, about 1 week).
  - **Picker**: official sample (about 1–2 days saved).
  - **Sync**: native `text/markdown` export + Sheets API + git CLI. Custom but small, and existing tools don't fit the curated model.
  - **Summarizer**: Karpathy LLM-wiki pattern as the schema, run via Claude Code or the Messages API (pattern reuse, little code).
  - **Search**: SQLite FTS5 / Postgres full-text (no extra component).
  
  The remaining custom code is mostly registry/roles, the admin UI and the glue, which is consistent with the design's 2–3 week Phase 1 estimate, possibly shorter.

### Gaps
- No 2025–2026 blog post found describing such a system end-to-end, so it's absent from my results. A narrower search might still turn one up.
