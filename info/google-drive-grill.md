# Google Drive integration: the grill

Status: research note, 2026-11-05 (revision 1). Written against Google's own documentation as of this date, plus vendor help pages and open-source code where Google is silent. Full findings with page stamps are in [research_notes/Round 3 (2026-11-05)/](research_notes/Round%203%20(2026-11-05)/). Claims that rest on a third party only are marked **(unverified)**; inferences that need a test are marked **(verify)**. Google changes these rules often (two relevant changes in October 2026 alone), so re-check before building on any row.

**Why this document.** The plan assumes a server of ours can read a hand-picked set of Google Docs and Sheets, keep Markdown copies, notice every change, and later do the same for other companies. The author's fear: Google will fight this at every step. This note asks the naive questions, answers each from the sources, names where Google pushes back, and gives a test that settles it in an afternoon.

**The short answer.** Google does not block third-party access to Docs. It prices it. Per-file access through `drive.file` and the Picker is free of verification and is the pattern Google itself recommends. Broad access (`drive.readonly`, the Activity API) costs an annual security assessment once the app is sold outside our own Workspace. Inside our own Workspace none of that applies. The real risks are smaller and technical: tokens bound to the person who picked a file, an export format Google never specified, and no editor identity without a restricted scope.

---

## 1. The questions

1. Can anyone other than Google work with Google Docs effectively?
2. What does Google actually offer: APIs, MCP, add-ons, Apps Script?
3. How do we get access to only the files we want, and keep it from a server?
4. How do we turn a Doc or a Sheet into Markdown, and how good is it?
5. Is git the right place to keep the copies and their history?
6. Does Google tell us when a document is edited, renamed, moved, deleted, or un-shared, and by whom?
7. What can a Workspace admin switch off under us?
8. Is there a shortcut: Google's own MCP servers, or Claude.ai's Drive connector?

---

## 2. Answers

### 2.1 Can anyone other than Google work with Google Docs effectively?

Yes, and at scale. DocuSign's Marketplace add-on has over 4 million installs on `documents` plus `drive.file`. Glean, Guru and ChatGPT's admin sync crawl whole domains with `drive.readonly` through a service account the customer's admin configures. Zapier, Make and n8n poll Drive for thousands of customers. Anthropic's own Claude.ai connector is, in Google's words, built and run by Google.

Two patterns exist and nothing else:

| Pattern | Who uses it | Google's price |
|---|---|---|
| Per-file: `drive.file` + Picker, or an add-on inside the editor | DocuSign, Claude's Docs sidebar add-on, Notion's import | None beyond basic brand verification for a public app |
| Broad: `drive.readonly` / `drive`, via user OAuth or a service account with domain-wide delegation | Glean, Guru, ChatGPT sync, Make, n8n, Dust | Annual CASA security assessment for a public app; otherwise the customer's admin must mark the app Trusted |

**Where Google pushes back.** Only "backup/sync, productivity/education, reporting/security" app types may hold restricted Drive scopes in a public app at all. No vendor publishes its CASA status; it is inferred from the scopes they request.

**Test.** None needed. The vendor table in the Round 3 notes is the evidence.

### 2.2 What does Google offer technically?

| Route | What it gives | The catch |
|---|---|---|
| Drive REST API v3 | Metadata, export, change feed, push channels, comments, revisions, permissions | Change feed has no actor; export is Docs-only for Markdown and capped at 10 MB |
| Docs API v1 | Full document JSON including all tabs, suggestions view, comments, original image URIs; suggested edits via `batchUpdate` | You write the Markdown converter yourself |
| Sheets API v4 | Every tab as values, formatted or raw or formulas | No Markdown; 300 reads/min per project |
| Drive Activity API v2 | Who did what: edit, rename, move, trash, delete, restore, permission change | Restricted scope; actor is a People ID, resolving a name needs another scope |
| Workspace Events API | Pub/Sub events per file or folder: `contentChanged`, `renamed`, `moved`, `trashed`, comments, permissions | GA since 2026-05-18 per the release notes, but the API's own page still says preview **(verify)**; subscriptions last 7 days |
| Push channels (`files.watch`, `changes.watch`) | A wake-up call to an HTTPS endpoint | Empty body; 1-day or 7-day channel lifetime; you still call the API |
| Google Picker | A file dialog that grants `drive.file` per picked file | The grant belongs to the user who picked |
| Google-hosted MCP servers (Drive, Docs, Sheets) | A remote MCP server Claude.ai can connect to directly | Developer Preview; needs `drive.readonly`; no file allowlist; may not be used in a public product before GA |
| Workspace add-ons (Docs, Drive) | A sidebar or menu item inside Docs, in Apps Script or as HTTP endpoints in any language | Private listing for one org needs no review; public listing needs OAuth verification and Marketplace review |
| Apps Script | A sync engine running as a Zuri user with no verification at all inside the org | 6 h/day trigger runtime, 6 min per run, no Drive change triggers |

**Where Google pushes back.** Nothing here is hidden or forbidden. The pushback is fragmentation: the thing we want (curated files, Markdown, changes with actors) spans four APIs with four scope classes.

### 2.3 How do we get access to only the files we want, and keep it from a server?

**Answer.** `drive.file` plus the Picker. The app sees only files a user picked, opened with the app, or the app created. The refresh token of the picking user then works from a server with nobody present; the Picker guide says exactly that the token can be passed to the Drive API for files "in which the user previously granted access".

**Where Google pushes back.**

- The grant is per user. A file picked by Alice is not readable with Bob's `drive.file` token **(verify)**. If Alice leaves, is suspended, or revokes the app, her picks go dark. The plan's background sync rests on this.
- Refresh tokens die when: unused for 6 months; the user revokes; the admin sets Drive to Restricted; 100 live tokens per account per client are exceeded. For an External app still in Testing status, after 7 days.
- A service account is not the escape under `drive.file`: it only covers files the service account itself opened or created, not files merely shared with it **(inferred)**. A service account that reads shared files needs `drive.readonly`.
- **The exemption the plan ignores:** an app with the consent screen set to Internal needs no verification, including for restricted scopes. No Google page says otherwise. So inside Zuri a service account with `drive.readonly`, shared the picked files or made a Shared Drive member, is legal and needs only the admin's "Trust internal apps" box. The "never restricted scopes" rule in the plan is a Phase 3 constraint, not a Phase 1 one.

**Test.** Pilot steps 2, 3, 7 and 8 in §5.

### 2.4 How do we turn a Doc or a Sheet into Markdown, and how good is it?

**Answer.** `files.export` with `text/markdown`, Docs only, 10 MB cap, 200 quota units per call. Google has never published what it emits. From code that depends on it:

| Element | What comes out |
|---|---|
| Headings | Pandoc-style `{#heading-id}` suffix; one vendor reports bold-wrapped titles **(unverified)** |
| Images | Re-compressed, embedded as base64 at the end of the file; originals only through the Docs API |
| Tables | Pipe tables; merged cells flattened **(unverified)** |
| Code blocks | Round-trip byte-perfect in one project's end-to-end tests |
| Comments | Dropped without a marker |
| Pending suggestions | Excluded |
| Tabs | All tabs concatenated with the tab name as a heading **(unverified)**; no per-tab export in the API |
| Footnotes, equations, drawings, charts, headers and footers | No evidence either way |
| Determinism | Nobody has tested whether an unchanged doc exports to identical bytes |

Sheets have no Markdown export. Use `spreadsheets.values.batchGet` with one range per tab and render the tables ourselves; the CSV export covers only the first sheet. Slides export as plain text or pptx.

Two alternatives: build the converter on the Docs API JSON (all tabs, original images, deterministic, our rules; `documents.get` with `includeTabsContent=true`), or export docx and run pandoc (keeps comments and tracked changes but a zip is not deterministic). Several MIT projects already do the first; `MattFisher/md-gdoc` and `mieweb/wikiGDrive` are the ones to read.

**A third option that removes the problem.** Since 2026-10-05 Docs opens and edits `.md` files natively, with sharing, comments and simultaneous editing, and the file stays `.md`. If a project's source documents were written as Markdown in Drive, the mirror would be a plain download with no conversion. The cost is a workflow change for authors, and nobody has published whether Docs rewrites Markdown syntax on save **(verify)**.

**Where Google pushes back.** An undocumented format with no element chart, no determinism promise, and no per-tab export.

**Test.** Pilot step 4 in §5.

### 2.5 Is git the right place to keep the copies and their history?

**Answer.** Yes, with four rules. Strip images out of the Markdown into hash-named files, never commit base64. Normalise the export (anchors, headings) and commit only when the normalised hash changes, so an undetermined export cannot spam history. One commit per sync run with the Drive `version` and `modifiedTime` in the trailer, one writer per repository. One repository per tenant from day one, so retention, size and access stay scoped.

Google's own revision history is not a substitute: the Docs revision list "might be incomplete for files with a large revision history", and `keepForever` exists only for binary files.

Postgres with content plus diff per version is the alternative. It is simpler to operate and to isolate per tenant; git wins on diff, blame and tooling. A hybrid, Postgres as the record and git as a view, is possible but doubles the work. Wait for the pilot's commit noise before choosing.

**Where Google pushes back.** It doesn't. The risk is self-inflicted: base64 images and non-deterministic exports.

### 2.6 Does Google tell us when a document changes, and by whom?

**Answer.** Yes to "when", mostly no to "by whom", under `drive.file`.

| Need | How | Limit |
|---|---|---|
| Something changed | `changes.list` from a stored page token; one entry per file; `trashed`, `removed` (deleted or un-shared) | Entry has no actor; retention of the change log is undocumented |
| Content changed | `modifiedTime` and `version` from `files.get` (5 units) as a filter, then export and hash | Native Docs have no checksum field; the Docs `revisionId` is valid for 24 hours only |
| Wake me up | Push channel (empty body, renew weekly) or a Workspace Events API subscription (typed events, renew weekly) | HTTPS endpoint or Pub/Sub; both need renewal jobs |
| Who edited | `lastModifyingUser` on the file | Last editor only, signed-in users only |
| Who renamed, moved, trashed, un-shared | Drive Activity API | Restricted scope; People IDs, not names |
| Trash vs permanent delete vs un-share | Trash: entry with `trashed=true`, file still readable. Delete or un-share: `removed=true`, no file object | Delete and un-share look the same |
| Owner leaves the company | Admin transfers ownership at deletion, or the file is gone after 20 days | The picking user's token dies with the account |

Nothing is documented about how fast `modifiedTime` moves while someone is typing, or whether opening or commenting bumps it **(verify)**.

**Where Google pushes back.** Actor identity sits behind a restricted scope. For the "AI complains to the maintainer" loop this is fine: the maintainer comes from our registry, not from Google. "Who changed it" in the Hub's Files view is limited to the last editor.

**Test.** Pilot steps 5, 6 and 9 in §5.

### 2.7 What can a Workspace admin switch off under us?

**Answer.** Everything, per app. Admin console → Security → API controls → App access control lists every OAuth app as Trusted, Limited, Specific data, or Blocked. The default for unconfigured apps is "allow", which covers non-restricted scopes such as `drive.file`. Internal apps are not trusted automatically; there is a "Trust internal apps" checkbox. If Drive itself is set to Restricted, a sub-option still lets users grant "access to user-selected files", which is `drive.file`.

**Where Google pushes back.** For Phase 3 this means: with `drive.file` only, our app works at any customer whose admin has not blocked it. The moment we need `drive.readonly` or the Activity API, every customer's admin must mark us Trusted by client ID, after we have passed CASA.

**Test.** Pilot step 10 in §5. Also ask the Zuri admin what the "Trust internal apps" box currently says.

### 2.8 Is there a shortcut?

**Google's hosted MCP servers.** Drive, Docs and Sheets servers exist, Claude.ai is a documented client, and from 2026-11-11 they only accept OAuth clients flagged "Use this client for an AI-powered agent". This rule does not touch REST API calls, so nothing in our plan is affected by that date. The servers are still Developer Preview, require `drive.readonly`, have no file allowlist, and Google's preview terms forbid using them in a public product before GA. They are a fine personal tool, not our context service.

**Claude.ai's Drive connector.** Reads whatever the signed-in user can see, Docs as Markdown, no comments or suggestions, 10 MB cap. It cannot do curated sets, maintainers, or a mirror. It is what Phase 1 users have today, and what our MCP server has to beat.

**Open-source MCP servers.** The most used one (`taylorwilsdon/google_workspace_mcp`, MIT, 3.5k stars) cannot be limited to selected files and uses `drive.readonly`. The reference server from Anthropic's organisation was archived in May 2025.

---

## 3. Where Google pushes back: Zuri now vs. Phase 3

| Barrier | Internal app, Zuri's own Workspace (Phases 1–2) | Public product (Phase 3) |
|---|---|---|
| OAuth app verification | Not required | Brand verification for `drive.file`; use-case review for sensitive scopes |
| Restricted scopes (`drive.readonly`, `drive`, Activity API) | Allowed, no assessment; admin ticks "Trust internal apps" | Annual CASA assessment by an authorised lab, about $540 per year **(unverified)**; app type must qualify |
| 100-user cap for unverified apps | Does not apply | Applies until verified |
| 7-day token expiry in Testing status | Does not apply | Applies until published |
| Customer admin app controls | Our own admin | Every customer's admin can block us; restricted scopes need an explicit Trusted entry |
| Token lifetime rules (6 months unused, 100 per client, revocation, suspended users) | Apply | Apply |
| `drive.file` grant bound to the picking user | Applies | Applies |
| Markdown export: Docs only, 10 MB, undocumented, no per-tab export | Applies | Applies |
| Change feed without actor; actor needs a restricted scope | Workable (internal scope allowed) | Actor unavailable unless CASA |
| Push channels payload-less, 7-day renewal; Events API 7-day renewal | Applies | Applies |
| Quotas (1M units/min per project; export 200 units) | Fine for 500 files hourly | Per-project quota shared by all tenants; charges "planned later in 2026" |
| Google-hosted MCP servers in Developer Preview | Usable personally | May not be used in a public product before GA |
| Marketplace listing | Private listing, no review | Public listing needs OAuth verification and Marketplace review |
| OAuth client "AI-powered agent" flag from 2026-11-11 | Only if we call Google's MCP servers | Same |

---

## 4. What this means for the plan

No document has been edited. These are inputs for [zuri-context-service.md](zuri-context-service.md).

1. **The scope rule splits by phase.** "Never restricted scopes" (§2, §4, §7.4 rule 5) stays right for Phase 3. For Phases 1–2 the Internal exemption makes a service account with `drive.readonly` legal, which fixes the user-bound-token problem for background sync (§3.2, §8 Q11). Decide in the pilot, and keep the registry able to run in both modes.
2. **Change feed.** §3.3 plans polling plus push channels. The Workspace Events API gives typed events and may be GA; test it in the pilot and keep polling as the fallback.
3. **Mirror hygiene.** §3.3 and §3.4 should state: images out of Markdown into hash-named files, normalise, commit on hash change only, one commit per run, `version` in the trailer.
4. **Converter ownership.** Plan for a Docs-JSON converter as the fallback to the native export; the native export alone is an unspecified dependency.
5. **Actor identity.** The maintainer loop needs no Google actor. Say so in §3.6, and limit the Files view to "last modified by".
6. **Native `.md` in Docs** is a new option for projects whose authors accept writing Markdown. Worth one paragraph in §8.
7. **The 2026-11-11 rule** needs no action unless we use Google's MCP servers.

---

## 5. One-afternoon pilot: prove it or kill it

Do this before any other Phase 1 work. It costs one Google Cloud project, one test Doc, one test Sheet and a laptop. Record the result of every step; the rows that fail become open questions. The existing pilot checklist in zuri-context-service.md §5 covers the Claude side; this one covers Google.

1. **Google Cloud project.** Create a project, enable the Drive API, Docs API, Sheets API, Drive Activity API and Workspace Events API. Set the OAuth consent screen user type to **Internal**. Create one Web application OAuth client (redirect to `http://localhost`) and one API key restricted to the Picker API and `https://docs.google.com/*`. Note what Google asked for and what it refused.
2. **Picker.** Run Google's Picker web sample with the `drive.file` scope. Pick one Doc and one Sheet that sit in a colleague's My Drive, then one Doc in a Shared Drive. Store the refresh token.
3. **Read from the server.** With the stored refresh token and no browser, call `files.get` on all three files, then `files.export` (`text/markdown`) for the Doc and `spreadsheets.values.batchGet` for every tab of the Sheet. Commit the output to a scratch git repository.
4. **Fidelity.** Put a table, a nested list, a checkbox list, an image, a footnote, a second tab and one pending suggestion into the test Doc. Export again and diff. Export a third time without changing anything and compare the hashes. Download the same Doc as Markdown from the Docs UI and diff it against the API export.
5. **Change feed.** Call `changes.getStartPageToken`, then edit the Doc, rename it, move it to another folder, trash it, restore it, un-share it from the picking user, and share it again. After each action call `changes.list` and `files.get` and write down exactly what came back (`removed`, `trashed`, `modifiedTime`, `version`, `lastModifyingUser`). Time how long after a keystroke `modifiedTime` moves.
6. **Who did it.** Call the Drive Activity API for the Doc with the same `drive.file` token. Expect a scope error; then add `drive.activity.readonly` to the Internal app and repeat.
7. **Second token.** Sign in as a second user who did not pick the file and try `files.get` with that user's token. Then revoke the first user's token at myaccount.google.com and repeat step 3.
8. **Service account.** Share the Doc with a service account and try `files.get` with `drive.file`, then with `drive.readonly`. Add the service account to the Shared Drive as a member and repeat for the Shared Drive Doc.
9. **Events.** Create a Workspace Events API subscription on the Doc with a Pub/Sub topic, and in parallel a `changes.watch` channel on an `ngrok` HTTPS endpoint. Edit the Doc and record what each delivers, and when each expires.
10. **Admin side.** In the Workspace admin console open Security → API controls → App access control. Record whether the Internal app appears, its status, the state of "Trust internal apps", and what the admin could switch off.

Exit criterion: steps 3, 4 and 5 pass with `drive.file` alone, or step 8 passes with a service account on `drive.readonly`. Either result lets Phase 1 start. If both fail, the context service needs a different access model before any more design work.

---

## Sources

Full source lists with page stamps are in the four Round 3 notes: [google_oauth_admin_tokens.md](research_notes/Round%203%20(2026-11-05)/google_oauth_admin_tokens.md), [google_drive_change_detection.md](research_notes/Round%203%20(2026-11-05)/google_drive_change_detection.md), [google_docs_markdown_git.md](research_notes/Round%203%20(2026-11-05)/google_docs_markdown_git.md), [google_integrators_mcp_addons.md](research_notes/Round%203%20(2026-11-05)/google_integrators_mcp_addons.md). The pages this note leans on most:

- Scopes and verification: https://developers.google.com/workspace/drive/api/guides/api-specific-auth , https://developers.google.com/identity/protocols/oauth2/production-readiness/overview , https://support.google.com/cloud/answer/13464321 , https://support.google.com/cloud/answer/13465431 , https://support.google.com/cloud/answer/15549945 , https://appdefensealliance.dev/casa/casa-tiering
- Tokens and admin controls: https://developers.google.com/identity/protocols/oauth2 , https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data , https://knowledge.workspace.google.com/admin/apps/control-api-access-with-domain-wide-delegation
- Picker and service accounts: https://developers.google.com/workspace/drive/picker/guides/web-picker , https://developers.google.com/workspace/drive/picker/guides/desktop-mobile-picker , https://developers.google.com/workspace/drive/api/guides/about-shareddrives
- Changes, push, activity, revisions: https://developers.google.com/workspace/drive/api/guides/manage-changes , https://developers.google.com/workspace/drive/api/guides/push , https://developers.google.com/workspace/drive/activity , https://developers.google.com/workspace/drive/api/guides/manage-revisions , https://developers.google.com/workspace/events/guides/events-drive
- Export and Docs API: https://developers.google.com/workspace/drive/api/guides/ref-export-formats , https://developers.google.com/workspace/drive/api/reference/rest/v3/files/export , https://developers.google.com/workspace/docs/api/how-tos/tabs , https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/get , https://workspaceupdates.googleblog.com/2026/10/preview-edit-and-collaborate-on-Markdown-files-natively-across-Drive-and-Docs.html , https://support.google.com/docs?p=markdown_mode
- Quotas: https://developers.google.com/workspace/drive/api/guides/limits , https://developers.google.com/workspace/docs/api/limits , https://developers.google.com/workspace/sheets/api/limits
- MCP and connectors: https://developers.google.com/workspace/guides/configure-mcp-servers , https://developers.google.com/workspace/release-notes , https://developers.google.com/workspace/preview , https://claude.com/docs/connectors/google/drive , https://support.claude.com/en/articles/10166901-use-google-workspace-connectors , https://github.com/taylorwilsdon/google_workspace_mcp
- Add-ons and Apps Script: https://developers.google.com/workspace/add-ons/concepts/types , https://developers.google.com/workspace/add-ons/guides/alternate-runtimes , https://developers.google.com/workspace/marketplace/how-to-publish , https://developers.google.com/apps-script/guides/services/quotas
- Converters and git: https://github.com/MattFisher/md-gdoc , https://github.com/mieweb/wikiGDrive , https://github.com/benjaminRomano/gdocs-md-cli , https://github.com/rsheyd/google-docs-markdown-sync , https://constly.com/blog/google-docs-to-markdown/ (unverified)
- Vendors: https://workspace.google.com/marketplace/app/docusign_esignature_for_google_workspace/469176070494 , https://docs.glean.com/connectors/native/gdrive/setup , https://help.getguru.com/docs/setting-up-google-drive-google-sheets-as-a-source , https://help.openai.com/en/articles/10929079 , https://www.notion.com/help/notion-ai-connectors-for-google-drive
