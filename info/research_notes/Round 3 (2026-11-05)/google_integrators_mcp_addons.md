# Round 3 notes: who integrates with Drive/Docs, MCP servers, Workspace extension points (checked 2026-11-05)

Scope: input for `google-drive-grill.md`. Dates in brackets are the page's visible "updated" stamp. (unverified) = third-party only or fetched from a search snippet.

## 0. Google-imposed ground rules (apply to everyone)

- Drive scope classes (api-specific-auth, undated): `drive`, `drive.readonly`, `drive.metadata(.readonly)`, `drive.meet.readonly`, `drive.scripts` = **restricted**; `drive.apps.readonly` = sensitive; `drive.file`, `drive.appdata` = **non-sensitive**. Restricted scopes need restricted-scope OAuth verification; if restricted-scope data is stored/transmitted via your servers, a **CASA security assessment** is required, revalidated **annually**; only "backup/sync, productivity/education, reporting/security" app types qualify. Google explicitly recommends migrating to `drive.file` + Google Picker (per-file, non-sensitive, "streamlined verification"). [S1][S2][S3]
- CASA (appdefensealliance.dev, 2026-06-27): OWASP ASVS based; AL1 and AL2 are both lab tested/verified by an ADA authorized lab; the level is chosen by Google based on data sensitivity and user count; yearly revalidation. Costs are not published officially (third-party figures about $540–1k+/yr, unverified). [S3][S4]
- Verification exemptions: Apps Script projects whose owner and users are in the same Workspace customer need no verification [S6]; OAuth consent **Internal** user type is GCP-org-bound and only org members can authorize. External + Testing: max 100 test users, consent/refresh tokens expire after 7 days; unverified production apps are capped at 100 new users lifetime. [S5][S6]
- Workspace admin side: admins set per-app access Trusted / Limited (unrestricted scopes only) / specific scopes / Blocked; "Trust internal apps" lets org-built apps call restricted APIs; the default "allow users to access any third-party apps" covers unrestricted data only, so restricted scopes need the app explicitly set Trusted (Claude's own help tells admins to do exactly that). [S7 2026-10-26][S10b]
- Export: `files.export` supports `text/markdown` for Docs; export cap 10 MB; accepted scopes include `drive.file`. [S8][S9]

## A. Products (one line each)

| Product | Type | Scopes (as documented) | Per-file pick | Sync | Src |
|---|---|---|---|---|---|
| Claude.ai Drive connector | "Google builds and runs the connector"; per-user OAuth; Team/Enterprise: Owner enables | **No scope strings published**; tools: search/read Docs, Sheets, Slides, PDF, Office, images, metadata, recent changes, upload, create folders, share/move/trash (approval by default); Docs → Markdown; no comments/suggestions; 10 MB export cap; the admin may have to mark the Claude OAuth app "Trusted" (= restricted-scope access) | Yes (picker in chat and private projects) | Chat: re-reads live each message; project knowledge refreshes periodically | [S10][S10b] |
| Claude for Google Workspace (Docs/Sheets/Slides sidebar, beta) | **Marketplace add-on**, user install or super-admin "Admin install" per OU/group; Pro/Max/Ultra/Team/Enterprise | Two permissions only: "Display and run third-party web content in prompts and sidebars…" + "View and manage documents… that this application has been installed in" (= `*.currentonly` wording); no Drive/Gmail/Calendar; other Drive files only via the separate Drive connector | Open file only | Edits in place, version history under the user's name | [S10c] |
| Claude Tag (Slack) Google connector | admin Google sign-in (no scope choice) or GCP service-account key + DWD with explicit scopes, e.g. `drive.readonly` | admin-set | n/a | live calls | [S11] |
| ChatGPT Google Drive app | per-user OAuth; admin "sync" variant = service account + DWD | user app: `drive.readonly`, `drive.metadata.readonly`, `drive.activity.readonly`, `drive` (for write actions); sync: `drive.readonly`, `drive.metadata.readonly`, `drive.activity.readonly`, `admin.directory.user.alias.readonly` (+ directory scopes). The Workspace admin must approve the OpenAI app for those scopes | user picks or search | the sync variant indexes; the Activity API signals changes (help.openai.com returned 403; from search snippets, partially unverified) | [S12][S13] |
| Notion AI connector for Drive | Google Workspace **Marketplace app**, Super Admin install, one Notion workspace ↔ one Workspace; Business/Enterprise only | scopes not published; respects Drive ACLs | exclude shared drives only | Indexed (embeddings), first sync about 36 h, then about hourly | [S14] |
| Notion import of Google Docs | per-user OAuth + picker, one doc at a time, one-way import (no live sync) | — | Yes | No | [S15] |
| Slack Google Drive app | Slack app, each member connects their own Google account; optional domain-wide auth via Super Admin | scopes not listed | Yes (file select) | notifications (comments, access requests); shared files searchable in Slack | [S16][S17] |
| Zapier | per-user OAuth; polling triggers (New File / Folder / File in Folder / Updated File) | scopes not documented (community: the consent shows only sign-in, unverified) | by ID or search | polling | [S18 2026-10-01] |
| Make | per-user OAuth; "Google Restricted" (Workspace) vs "Google Drive" connection; custom client optional | `drive`, `drive.readonly`, `userinfo.email` | search modules | watch modules; webhook only for "instant" modules | [S19] |
| n8n | self-hosted: your own OAuth client | the credential hardcodes `drive`, `drive.appdata`, `drive.photos.readonly` (full Drive, restricted → your client needs verification/CASA if external) | by ID | triggers poll | [S20] |
| Unito | Enterprise-only, activated on demand; two-way sync of folders (name/description), files read-only | not published | folder subsets | yes, no live triggers | [S21] |
| Glean | service account + DWD, Super Admin setup | `drive.readonly`, `admin.directory.user/group.readonly`, `admin.reports.audit.readonly` (+ optional labels, domain) | no (crawls the whole domain) | full crawl + recrawl on activity | [S22 2026-09-30] |
| Guru | 3 modes: user OAuth, inherited permissions (admin), DWD service account | `drive.readonly`, `admin.directory.user/group.readonly`, `admin.reports.audit.readonly` | folders / whole drives | indexed | [S23 2026-06-05] |
| Dust | user OAuth via a dedicated Workspace user ("dust-admin"); scopes not published | — | the admin selects data in a selection modal (granularity not stated) | initial sync >24 h; incremental; 2 MB text cap per file | [S24] |
| Langdock | per-user "sign in with Google" (delegated OAuth), BYO OAuth client possible; an older doc mentioned DWD for workspace level | not published | search/browse at runtime | live calls | [S25] |
| Coda (now under Superhuman) | import tool: user OAuth; requested "View & Manage Google Drive files" + "Add itself to Drive" (help page 403; from snippet, unverified) | broad Drive (users complained about least privilege) | Yes | one-way import | [S26] |
| Grammarly in Docs | **browser extension**, not a Workspace add-on; no Google OAuth | none | n/a | n/a | [S27] |
| DocuSign eSignature for Workspace | **Marketplace add-on** (Docs, Drive, Gmail; 4M+ installs; listing updated 2025-12-03) | "See, edit, create, and delete all your Google Docs documents" (= `documents`), `drive.file` ("only the specific Drive files you use with this app"), picker metadata | Yes | write-back of completed docs | [S28] |

Answer: yes, non-Google products work with Docs effectively, under two patterns. (1) Per-file `drive.file` + Picker or an add-on (no CASA; DocuSign, Claude's sidebar add-on, Notion import). (2) Broad `drive.readonly`/`drive` with annual CASA, or an admin-configured service account + DWD inside the customer's own Workspace (Glean, Guru, ChatGPT sync), which shifts the trust decision to the customer's admin. Per-vendor CASA status is not published anywhere; it is inferred from the scope class.

## B. MCP servers

- **Google hosted Workspace MCP** (configure-mcp-servers): Developer Preview Program membership required. Drive `https://drivemcp.googleapis.com/mcp/v1` (search/read/create/copy/download/metadata/permissions; preview since 2026-04-22), Docs `docsmcp…` (read_doc, update_doc, comments since 2026-10-01; preview since 2026-07-13), Sheets, Slides, Gmail, Calendar, Chat, People, Classroom. Scopes: Drive `drive.readonly` + `drive.file`; Docs adds `documents(.readonly)`. Auth: your own GCP OAuth web client flagged **"Use this client for an AI-powered agent"** (from **2026-11-11 non-agentic clients are rejected**); no DWD service accounts. Supported clients: Antigravity, **Claude.ai/Desktop custom connector** (redirect `https://claude.ai/api/mcp/auth_callback`, Pro/Max/Team/Enterprise), any Streamable-HTTP client. Needs `access_type=offline` for refresh tokens. No GA date given. [S29][S30]
- **`@modelcontextprotocol/server-gdrive`** (v0.6.2): the repo `modelcontextprotocol/servers-archived` was archived 2025-05-28, read-only, "no security guarantees". Fork dylancaponi/gdrive-mcp-server (unverified). [S31]
- Community (GitHub star counts 2026-11-05):

  | Repo | Stars | Licence | Scopes | File limiting | Auth/transport |
  |---|---|---|---|---|---|
  | taylorwilsdon/google_workspace_mcp | 3,478 | MIT | Drive: `drive`, `drive.readonly`, `drive.file`; `--read-only` → `drive.readonly` only; Docs: `documents(.readonly)` + `drive.readonly` + `drive.file` | No (tool tiers/services only) | own OAuth client; stdio or Streamable HTTP; OAuth 2.1/PKCE bearer; service account DWD; documented as a Claude.ai custom connector |
  | xing5/mcp-google-sheets | 1,016 | MIT | Sheets only | — | — |
  | a-bonus/google-docs-mcp | 683 | MIT | `documents`, `spreadsheets`, `drive`, `gmail.modify`, `calendar.events` | No (domain allowlist only) | OAuth or service account; stdio/httpStream; Cloud Run + OAuth 2.1 documented |
  | isaacphi/mcp-gdrive | 282 | MIT | (last push 2025-05) | — | stdio |
  | piotr-agier/google-drive-mcp | 235 | MIT | — | — | — |

  Neither of the two Drive/Docs servers inspected (taylorwilsdon, a-bonus) can be limited to selected files; both use broad `drive`/`drive.readonly`. [S32][S33]
- Aggregators: **Zapier MCP** (`mcp.zapier.com`, per-user MCP URL; Drive 16 write / 4 search / 5 trigger actions; Zapier holds the Google OAuth app) [S34]; **Composio** (managed OAuth by default or BYO client; notes that Google blocks unverified clients on restricted scopes; `drive.file` vs `drive`; Tool Router session MCP URL; **Rube** = Composio's hosted MCP at `rube.app/mcp`, third-party docs only) [S35]; **Pipedream** (`https://remote.mcp.pipedream.net/v3`, developer client-credentials bearer + `external_user_id`, Pipedream-managed OAuth; aimed at app developers, not a Claude.ai end-user connector) [S36].
- Claude.ai remote-MCP requirements (connectors/building/authentication): OAuth 2.0 with DCR or CIMD by default, or a user-entered client ID/secret; 401 + `WWW-Authenticate: resource_metadata`; PKCE S256; callback `https://claude.ai/api/mcp/auth_callback`; no `client_credentials`; egress `160.79.104.0/21`. Could run behind OAuth for Claude.ai: Google's hosted servers (user enters their own client ID/secret), taylorwilsdon (HTTP + OAuth 2.1 mode), a-bonus (Cloud Run + OAuth 2.1), Zapier MCP and Composio/Rube (hosted); the archived reference server cannot (stdio only). [S37]

## C. Google Workspace extension points

- **Add-on types**: *Google Workspace add-ons* (one add-on across Gmail/Calendar/Drive/Docs/Sheets/Slides/Chat/Meet; Card-based UI in the right side panel; runtime = Apps Script **or HTTP endpoints, "alternate runtimes"**, in any language, deployment JSON in the Marketplace SDK; since 2025-05-27 new HTTP add-ons must support granular permissions) vs *Editor add-ons* (single editor app; HTML/CSS sidebars, dialogs, custom menu items via `DocumentApp.getUi()`; **Apps Script only**; desktop only). [S38][S39]
- **Docs capabilities**: homepage card in the side panel; `docs.onFileScopeGrantedTrigger` after the user grants per-file `drive.file` via `requestFileScopeForActiveDocument()`; then the add-on gets the doc ID and can read/write it via the Docs/Drive REST API (HTTP runtime) or `DocumentApp` (`documents.currentonly`, Apps Script services only). A menu/sidebar button "Add this doc to project X" = an Editor add-on menu item or a Workspace add-on card action calling your backend with the `drive.file`-scoped file ID. [S40][S41]
- **Drive add-on**: `drive.onItemsSelectedTrigger` shows a contextual card on selected items; the scope `drive.addons.metadata.readonly` exposes only id/title/mimeType/icon + "addonHasFileScopePermission"; content requires a `drive.file` grant per item. [S41]
- **Deployment without a public listing**: Marketplace SDK visibility **Private** (listed under "Internal Apps" for your Workspace org only, **published immediately, no Google review**, needs a Workspace account; the visibility choice is permanent) plus install setting admin-only or individual; admins can also domain-install (Claude's add-on uses exactly this "Admin install" path); Unlisted = public but hidden from search/browse (whether review still applies: unverified). A public listing requires passing OAuth verification, narrowest scopes, privacy-policy links, no bugs, a test account; review "typically several days"; a consent screen set to Internal/Testing fails public review. [S42][S43]
- **Quotas** (Apps Script, consumer / Workspace): triggers total runtime 90 min / **6 h per day**; 6 min per execution; URL Fetch 20k / **100k per day**, 50 MB response; 30 simultaneous executions per user; Docs created 250 / 1,500 per day. Add-on time-driven triggers at most once per hour. HTTP add-ons: Apps Script quotas don't apply (your own hosting). [S44][S45]
- **Apps Script as a sync engine**: time-driven installable triggers (every minute … monthly; they run as the trigger creator, i.e. the script owner's identity; Drive has no edit/change triggers, only Sheets `onEdit/onChange`, Docs only `onOpen`). Inside one Workspace org no OAuth verification is needed (owner and users in the same customer; the admin can "Trust internal apps"). Markdown export: `UrlFetchApp.fetch("https://www.googleapis.com/drive/v3/files/{id}/export?mimeType=text/markdown", {headers:{Authorization:"Bearer "+ScriptApp.getOAuthToken()}})`; images come out as base64 data URIs; 10 MB export cap; needs a `drive.readonly`/`drive.file`/`drive` scope declared in the manifest. [S46][S47][S8][S9]
- **Workspace Events API for Drive**: Pub/Sub subscriptions on files/folders/shared drives (not the shared-drive root); events `google.workspace.drive.{file,comment,reply,permission,approval,accessProposal}.v3.*` including `file.contentChanged`, `renamed`, `moved`, `trashed`; `includeDescendants`. Status: **GA since 2026-05-18** per the Google Workspace developer release notes (the Events API's own release-notes page still shows only the 2025-07-07 Developer Preview entry) **(verify)**. Subscription lifetime ≤7 days without resource data, ≤4 h with (24 h via DWD); renew via `subscriptions.patch` ttl=0; expired subscriptions are deleted. Replaces `files.watch`/`changes.watch` push channels, which need a public HTTPS webhook. [S48][S49][S50]

## Sources

- S1 https://developers.google.com/workspace/drive/api/guides/api-specific-auth
- S2 https://support.google.com/cloud/answer/13465431
- S3 https://appdefensealliance.dev/casa (2026-06-27)
- S4 https://appdefensealliance.dev/casa/casa-tiering (2026-06-27)
- S5 https://support.google.com/cloud/answer/15549945
- S6 https://developers.google.com/apps-script/guides/client-verification
- S7 https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data (2026-10-26)
- S8 https://developers.google.com/workspace/drive/api/reference/rest/v3/files/export
- S9 https://developers.google.com/workspace/drive/api/guides/ref-export-formats
- S10 https://claude.com/docs/connectors/google/drive
- S10b https://support.claude.com/en/articles/10166901-use-google-workspace-connectors
- S10c https://support.claude.com/en/articles/16951679-use-claude-in-google-docs-sheets-and-slides
- S11 https://claude.com/docs/claude-tag/admins/connections/google
- S12 https://help.openai.com/en/articles/10408842
- S13 https://help.openai.com/en/articles/10929079
- S14 https://www.notion.com/help/notion-ai-connectors-for-google-drive
- S15 https://www.notion.com/help/import-data-into-notion
- S16 https://slack.com/help/articles/205875058-Google-Drive-for-Slack
- S17 https://slack.com/help/articles/360047862573-Enable-domain-wide-authentication-for-calendar-and-file-apps-in-Slack
- S18 https://help.zapier.com/hc/en-us/articles/8495965163405 (2026-10-01)
- S19 https://apps.make.com/google-drive
- S20 https://github.com/n8n-io/n8n/blob/master/packages/nodes-base/credentials/GoogleDriveOAuth2Api.credentials.ts
- S21 https://unito.io/connectors/google-drive/ ; https://guide.unito.io/a-master-list-of-unito-product-updates
- S22 https://docs.glean.com/connectors/native/gdrive/setup (2026-09-30)
- S23 https://help.getguru.com/docs/setting-up-google-drive-google-sheets-as-a-source (2026-06-05)
- S24 https://docs.dust.tt/docs/google-drive-connection
- S25 https://docs.langdock.com/en/using-langdock/integrations/integration-directory ; https://docs.langdock.com/en/admin/manage-integrations/bring-your-own-oauth
- S26 https://help.superhuman.com/hc/en-us/articles/46210160112397-Import-Google-Docs-into-Coda (403; snippet) ; https://connect.superhuman.com/t/google-drive-principle-of-least-privilege/18062
- S27 https://support.grammarly.com/hc/en-us/articles/115000090991
- S28 https://workspace.google.com/marketplace/app/docusign_esignature_for_google_workspace/469176070494
- S29 https://developers.google.com/workspace/guides/configure-mcp-servers
- S30 https://developers.google.com/workspace/release-notes (2026-04-22, 07-13, 10-01, 10-12 entries)
- S31 https://github.com/modelcontextprotocol/servers-archived
- S32 https://github.com/taylorwilsdon/google_workspace_mcp (README, auth/scopes.py)
- S33 https://github.com/a-bonus/google-docs-mcp
- S34 https://zapier.com/mcp/google-drive
- S35 https://docs.composio.dev/kb/guide/toolkits-googledrive
- S36 https://pipedream.com/docs/connect/mcp/developers
- S37 https://claude.com/docs/connectors/building/authentication ; https://support.claude.com/en/articles/11503834
- S38 https://developers.google.com/workspace/add-ons/concepts/types
- S39 https://developers.google.com/workspace/add-ons/guides/alternate-runtimes
- S40 https://developers.google.com/workspace/add-ons/editors/gsao/building-editor-interfaces
- S41 https://developers.google.com/workspace/add-ons/concepts/workspace-scopes ; https://developers.google.com/workspace/add-ons/concepts/editor-scopes ; https://developers.google.com/workspace/add-ons/drive/building-drive-interfaces
- S42 https://developers.google.com/workspace/marketplace/how-to-publish ; https://developers.google.com/workspace/marketplace/sdk
- S43 https://developers.google.com/workspace/marketplace/about-app-review
- S44 https://developers.google.com/apps-script/guides/services/quotas
- S45 https://developers.google.com/apps-script/guides/triggers/installable
- S46 https://tanaikech.github.io/2024/07/23/convert-google-document-to-markdown-and-vice-versa-using-google-apps-script
- S47 https://workspaceupdates.googleblog.com/2024/07/import-and-export-markdown-in-google-docs.html
- S48 https://developers.google.com/workspace/events/guides/events-drive
- S49 https://developers.google.com/workspace/events/guides/update-subscription ; https://developers.google.com/workspace/events/guides/events-lifecycle
- S50 https://developers.google.com/workspace/events/docs/release-notes ; https://workspaceupdates.googleblog.com/2025/07/google-drive-events-api-now-available.html

## Addendum (2026-11-05, after review): Workspace Events API scopes for Drive

- The auth page (https://developers.google.com/workspace/events/guides/auth) says creating a subscription "With user authentication, requires a supported scope for each event type". Its "Scopes by event type" table lists for file events (`google.workspace.drive.file.v3.*`): `drive`, `drive.file`, `drive.readonly`, `drive.metadata`. Comment, reply, permission, approval and access-proposal events add `drive.metadata.readonly`. So `drive.file` is enough for content/rename/move/trash events on picked files.
- The Drive events page (https://developers.google.com/workspace/events/guides/events-drive) adds: "The user that authorizes the subscription must have permission on the file corresponding to the events that they subscribe to." Neither page carries a GA or Developer Preview label, and neither states subscription lifetimes (those are on the lifecycle page cited above).
