# Google Drive integration: the grill

Status: research note, 2026-11-05 (revision 3). Written against Google's own documentation as of this date, plus vendor help pages and open-source code where Google is silent. Full findings with page stamps are in [research_notes/Round 3 (2026-11-05)/](research_notes/Round%203%20(2026-11-05)/). Marks: **(unverified)** rests on a third party only; **(inferred)** follows from documentation that does not say it outright; **(verify)** needs a test, usually a numbered pilot step in §5. Google changes these rules often (two relevant changes in October 2026 alone), so re-check before building on any row.

What changed in revision 3: after two independent reviews. The Internal-app exemption is split into what a user token and what a service account need; the sensitive-scope middle path (`documents.readonly`) is added; the permissions answer now matches the Google Group design and has a pilot step; native `.md` and group sharing have pilot steps; Events API scopes confirmed; wrong attributions, section numbers and quota figures corrected; a list of questions not yet asked added as §7. Revision 2 added questions 9–15.

**Why this document.** The plan assumes a server of ours can read a hand-picked set of Google Docs and Sheets, keep Markdown copies, notice every change, and later do the same for other companies. The author's fear: Google will fight this at every step. This note asks the naive questions, answers each from the sources, names where Google pushes back, and gives a test that settles it in an afternoon.

**The short answer.** Google does not block third-party access to Docs. It prices it. Per-file access through `drive.file` and the Picker needs no verification beyond brand checks and is the pattern Google itself recommends. Broad access (`drive.readonly`, the Activity API) costs an annual security assessment once the app is sold outside our own Workspace; inside our own Workspace it appears to cost nothing **(verify)**. The real risks are smaller and technical: tokens bound to the person who picked a file, an export format Google never specified, and no editor identity without a restricted scope.

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

Added after a review of the first eight (revision 2):

9. Are we allowed to feed Google Docs content to an AI model at all?
10. Whose permissions win: Drive's or our registry's?
11. What about files on Drive that are not Docs or Sheets: PDFs, Word files, Slides, images?
12. How stable is all this: how often does Google change the rules, and with how much notice?
13. What does it cost in money?
14. Does writing back work: suggested edits and comments from the AI?
15. Can we avoid the API by syncing files to disk (rclone, Drive for desktop)?

---

## 2. Answers

### 2.1 Can anyone other than Google work with Google Docs effectively?

Yes, and at scale. DocuSign's Marketplace add-on has over 4 million installs. Glean, Guru and ChatGPT's admin sync **(partially unverified)** crawl whole domains with `drive.readonly` through a service account the customer's admin configures. Zapier, Make and n8n poll Drive for their customers. Anthropic's own Claude.ai connector is, per Anthropic's documentation, built and run by Google.

Three patterns exist:

| Pattern | Who uses it | Google's price |
|---|---|---|
| Per-file: `drive.file` + Picker, or an add-on inside the editor | Claude's Docs sidebar add-on, Notion's import | Brand verification only for a public app |
| Sensitive: `documents.readonly` / `spreadsheets.readonly`, reading any Doc the user can open, by ID | DocuSign (`documents` plus `drive.file`) | Brand verification plus a use-case review with a demo video; no security assessment |
| Restricted: `drive.readonly` / `drive`, via user OAuth or a service account with domain-wide delegation | Glean, Guru, ChatGPT sync, Make, n8n, Dust | Annual CASA security assessment for a public app; otherwise the customer's admin must mark the app Trusted |

**Where Google pushes back.** Only "backup/sync, productivity/education, reporting/security" app types may hold restricted Drive scopes in a public app at all. No vendor publishes its CASA status; it is inferred from the scopes they request.

**Test.** None needed. The vendor table in the Round 3 notes is the evidence.

### 2.2 What does Google offer technically?

| Route | What it gives | The catch |
|---|---|---|
| Drive REST API v3 | Metadata, export, change feed, push channels, comments, revisions, permissions | Change feed has no actor; export is Docs-only for Markdown and capped at 10 MB |
| Docs API v1 | Full document JSON including all tabs, suggestions view, comments, original image URIs; suggested edits and anchored comments via `batchUpdate` | You write the Markdown converter yourself |
| Sheets API v4 | Every tab as values, formatted or raw or formulas | No Markdown; 300 reads/min per project |
| Drive Activity API v2 | Who did what: edit, rename, move, trash, delete, restore, permission change | Restricted scope; actor is a People ID, resolving a name needs another scope |
| Workspace Events API | Pub/Sub events per file or folder: `contentChanged`, `renamed`, `moved`, `trashed`, comments, permissions; file events accept `drive.file` | GA since 2026-05-18 per the release notes, but the API's own pages carry no GA label **(verify)**; subscriptions last 7 days, 4 hours if they carry resource data |
| Push channels (`files.watch`, `changes.watch`) | A wake-up call to an HTTPS endpoint | Empty body; 1-day or 7-day channel lifetime; you still call the API |
| Google Picker | A file dialog that grants `drive.file` per picked file | The grant belongs to the user who picked |
| Google-hosted MCP servers (Drive, Docs, Sheets) | A remote MCP server Claude.ai can connect to directly | Developer Preview; needs `drive.readonly`; no file allowlist; may not be used in a public product before GA |
| Workspace add-ons (Docs, Drive) | A sidebar or menu item inside Docs, in Apps Script or as HTTP endpoints in any language | Private listing for one org needs no review; public listing needs OAuth verification and Marketplace review |
| Apps Script | A sync engine running as a Zuri user with no verification at all inside the org | 6 h/day trigger runtime, 6 min per run, no Drive change triggers |

**Where Google pushes back.** Nothing here is hidden or forbidden. The pushback is fragmentation: the thing we want (curated files, Markdown, changes with actors) spans four APIs and three scope classes.

### 2.3 How do we get access to only the files we want, and keep it from a server?

**Answer.** `drive.file` plus the Picker. The app sees only files a user picked, opened with the app, or the app created. The refresh token of the picking user should then work from a server with nobody present: the Picker guide says the token can be passed to the Drive API to "read and modify files in which the user previously granted access", and no rule beyond the token lifetimes below limits it **(verify, pilot step 3)**.

**Where Google pushes back.**

- The grant is per user. A file picked by Alice is not readable with Bob's `drive.file` token **(verify, step 7)**. If Alice leaves or revokes the app, her picks go dark; what suspension does to her grants is undocumented, one vendor says they survive **(unverified)**.
- Refresh tokens die when: unused for 6 months; the user revokes; the admin sets Drive to Restricted; a Workspace session-length policy expires them; 100 live tokens per account per client are exceeded. For an External app still in Testing status, after 7 days.
- A service account is not the escape under `drive.file`: it only covers files the service account itself opened or created, not files merely shared with it **(inferred)**. A service account that reads shared files needs `drive.readonly`.
- **The exemption the plan ignores.** Google lists "Internal use only" as a verification exemption and nowhere says restricted scopes revoke it. Two consequences, each needing its own test: a user token with a restricted scope on an Internal app needs no verification **(verify, step 6)**; a service account presents no consent screen at all, so the only question is whether the admin's API controls touch it **(verify, steps 8 and 10)**. If both hold, a service account with `drive.readonly`, shared the picked files or made a Shared Drive member **(unverified that membership is allowed)**, is legal inside Zuri. The "never restricted scopes" rule in the plan would then be a Phase 3 constraint, not a Phase 1 one.
- **The middle path.** `documents.readonly` and `spreadsheets.readonly` are sensitive, not restricted: any user's token reads any Doc or Sheet that user can open, by ID, with no Picker and no security assessment. It fixes the user-bound-token problem for Phase 3 at the price of a use-case review and a consent screen that says "see all your Google Docs". Not researched beyond the scope class **(verify)**.

**Test.** Pilot steps 2, 3, 6, 7, 8 and 10 in §5.

### 2.4 How do we turn a Doc or a Sheet into Markdown, and how good is it?

**Answer.** `files.export` with `text/markdown`, Docs only, 10 MB cap, 200 quota units per call. Google has never published what it emits. From code that depends on it:

| Element | What comes out |
|---|---|
| Headings | Pandoc-style `{#heading-id}` suffix; one vendor reports bold-wrapped titles **(unverified)** |
| Images | Re-compressed, embedded as base64 at the end of the file; originals only through the Docs API |
| Tables | Pipe tables; merged cells flattened **(unverified)** |
| Code blocks | Round-trip byte-perfect in one project's end-to-end tests |
| Comments | Dropped without a marker **(unverified, UI download)** |
| Pending suggestions | Excluded |
| Tabs | All tabs concatenated with the tab name as a heading **(unverified)**; no per-tab export in the API |
| Footnotes, equations, drawings, charts, headers and footers | No evidence either way |
| Determinism | Nobody has tested whether an unchanged doc exports to identical bytes |

Sheets have no Markdown export. Use `spreadsheets.values.batchGet` with one range per tab and render the tables ourselves; the CSV export covers only the first sheet. Slides export as plain text or pptx.

Two alternatives: build the converter on the Docs API JSON (all tabs, original images, deterministic, our rules; `documents.get` with `includeTabsContent=true`), or export docx and run pandoc (a zip, so not deterministic, and comments survive only with `--track-changes=all` **(unverified)**). `AnandChowdhary/docs-markdown` and the over-10 MB fallback in `rsheyd/google-docs-markdown-sync` do the first; `MattFisher/md-gdoc` and `mieweb/wikiGDrive` are the ones to read for image handling and mirror hygiene.

**A third option that removes the problem.** Since 2026-10-05 Docs opens and edits `.md` files natively, with sharing, comments and simultaneous editing, and the file stays `.md`. If a project's source documents were written as Markdown in Drive, the mirror would be a plain download with no conversion. The cost is a workflow change for authors; Google's help page says smart chips, HTML, colours and alignment are changed on edit, and whether raw Markdown syntax survives a save is untested **(verify, step 13)**.

**Where Google pushes back.** An undocumented format with no element chart, no determinism promise, and no per-tab export.

**Test.** Pilot steps 4 and 13 in §5.

### 2.5 Is git the right place to keep the copies and their history?

**Answer.** Yes for Phases 1–2, with four rules, and a decision to revisit in Phase 3.

The cache has four jobs: serve current text to the MCP server, give the summarizer diffs, keep history "for a while", and say what changed when. Git does the last three for free and adds one thing nothing else gives: a Claude.ai project can sync a GitHub repository into its knowledge, which is the plan's fallback route if the MCP connector misbehaves (zuri-context-service.md §5).

The four rules: strip images out of the Markdown into hash-named files, never commit base64. Normalise the export (anchors, headings) and commit only when the normalised hash changes, so an undetermined export cannot spam history. One commit per sync run, not per file, with the Drive `version` and `modifiedTime` in the trailer, one writer per repository. One repository per tenant from day one.

The price is all on our side: retention of "history older than 90 days" means rewriting history, safe only with a single writer and read-only consumers; and a repository on a server needs a lock, a backup and, in Phase 3, one per tenant. Postgres with a versions table (document, version, content, hash, time) does the same jobs without those costs, diffs on demand, retention as a `DELETE`, multi-tenancy as a column. It loses the GitHub sync into Claude.ai and `git log` for humans. So: git now, the store behind one interface (`put`, `get`, `history`, `diff`), and if the pilot shows the GitHub route is not needed, Postgres from the start is the simpler choice.

Google's own revision history is not a substitute: the Docs revision list "might be incomplete for files with a large revision history", and `keepForever` exists only for binary files.

**Where Google pushes back.** It doesn't. The risk is self-inflicted: base64 images and non-deterministic exports.

### 2.6 Does Google tell us when a document changes, and by whom?

**Answer.** Yes to "when", mostly no to "by whom", under `drive.file`.

| Need | How | Limit |
|---|---|---|
| Something changed | `changes.list` from a stored page token; one entry per file; `trashed`, `removed` (deleted or un-shared) | Entry has no actor; retention of the change log is undocumented |
| Content changed | `modifiedTime` and `version` from `files.get` (5 units) as a filter, then export and hash | Native Docs have no checksum field; the Docs `revisionId` is valid for 24 hours only |
| Wake me up | Push channel (empty body, renew weekly) or a Workspace Events API subscription (typed events, `drive.file` accepted, renew weekly, every 4 hours if events carry resource data) | HTTPS endpoint or Pub/Sub; both need renewal jobs |
| Who edited | `lastModifyingUser` on the file | Last editor only, signed-in users only |
| Who renamed, moved, trashed, un-shared | Drive Activity API | Restricted scope; People IDs, not names |
| Trash vs permanent delete vs un-share | Trash: entry with `trashed=true`, file still readable. Delete or un-share: `removed=true`, no file object | Delete and un-share look the same |
| Owner leaves the company | Admin transfers ownership at deletion; otherwise the account is restorable for 20 days and the untransferred files are gone after that **(inferred)** | The picking user's token dies with the account |

Nothing is documented about how fast `modifiedTime` moves while someone is typing, or whether opening or commenting bumps it **(verify, step 5)**.

**Where Google pushes back.** Actor identity sits behind a restricted scope. For the "AI complains to the maintainer" loop this is fine: the maintainer comes from our registry, not from Google. "Who changed it" in the Hub's Files view is limited to the last editor.

**Test.** Pilot steps 5, 6 and 9 in §5.

### 2.7 What can a Workspace admin switch off under us?

**Answer.** Everything, per app. Admin console → Security → Access and data control → API controls → Manage third-party app access lists every OAuth app as Trusted, Limited, Specific data, or Blocked. The default for unconfigured apps is "allow", which covers non-restricted scopes such as `drive.file`. Internal apps are not trusted automatically; there is a "Trust internal apps" checkbox. If Drive itself is set to Restricted, a sub-option still lets users grant "access to user-selected files", which is `drive.file`.

**Where Google pushes back.** For Phase 3 this means: with `drive.file` only, our app works at any customer whose admin has not blocked it. The moment we need `drive.readonly` or the Activity API, every customer's admin must mark us Trusted by client ID, after we have passed CASA.

**Test.** Pilot step 10 in §5. Also ask the Zuri admin what the "Trust internal apps" box currently says.

### 2.8 Is there a shortcut?

**Google's hosted MCP servers.** Drive, Docs and Sheets servers exist, Claude.ai is a documented client, and from 2026-11-11 they only accept OAuth clients flagged "Use this client for an AI-powered agent". This rule does not touch REST API calls, so nothing in our plan is affected by that date. The servers are still Developer Preview, require `drive.readonly`, have no file allowlist, and Google's preview terms forbid using them in a public product before GA. They are a fine personal tool, not our context service.

**Claude.ai's Drive connector.** Reads whatever the signed-in user can see, Docs as Markdown, no comments or suggestions, 10 MB cap. It cannot do curated sets, maintainers, or a mirror. It is what Phase 1 users have today, and what our MCP server has to beat.

**Open-source MCP servers.** The most used one (`taylorwilsdon/google_workspace_mcp`, MIT, 3.5k stars) cannot be limited to selected files and uses `drive.readonly`. The reference server in the `modelcontextprotocol` organisation was archived in May 2025.

### 2.9 Are we allowed to feed Google Docs content to an AI model at all?

**Answer.** Yes, for inference. No, for training. Google's Workspace API policy forbids "using user data to create, train, or improve a machine learning or artificial intelligence model beyond that specific user's personalized model". Sending a document to Claude to answer a question is a *transfer* of user data, and the only exception that fits is "to provide … user-facing features that are visible and prominent in the requesting application's user interface and only with the user's consent". Anthropic's commercial terms say "Anthropic may not train models on Customer Content from Services", so the no-training condition holds for API, Team and Enterprise traffic.

**Where Google pushes back.**

- These "Limited Use" rules attach to sensitive and restricted scopes, by the policy's sentence "These requirements apply to data derived from both Sensitive and Restricted scopes", and "If your app is only used by users within your own domain, then these requirements do not apply." Under `drive.file` alone they do not apply either **(verify)**. So they bite in Phase 3, and only if we move beyond `drive.file`.
- When they bite, Google wants a disclosure of data use inside the app itself, "not only in a privacy policy", and a privacy policy stating Limited Use compliance and no training of non-personalised models.
- Two rules apply to every app regardless of scope: users must "confirm the MCP, tool, skill, or other agentic behavior invocation", and the app must protect against prompt injection. Whether the own-domain exemption covers these is not stated **(verify)**.
- **A trap on our side, independent of Google's policy:** Claude.ai Free, Pro and Max accounts may be used for training when the user's setting is on. A customer's documents would then train a third-party model. Employees reaching our MCP server must be on Team or Enterprise, or have training off. Zuri is on Team, so fine today; a Phase 3 customer on Pro is not.

**Test.** None for the pilot. For Phase 3: write the disclosure screen and the two privacy-policy sentences before the first external customer.

### 2.10 Whose permissions win: Drive's or our registry's?

**Answer.** Ours, and nothing in the plan says so. Once a document is in the mirror, the registry's roles decide who reads it. A project reader gets the content of a Doc they could not open in Drive. Google neither prevents nor notices this; it is the normal consequence of a server-side copy, and every indexing product faces the same choice. Glean and Guru appear to answer it by mapping Drive permissions per user, which is why they ask for `drive.readonly` plus directory and audit scopes **(inferred from their setup pages)**.

Three options:

1. **Registry only.** Registering a document declares it readable by the project. Simple and explicit, but a reader may see content they cannot open in Drive, and the link in the AI's answer will not open for them.
2. **Drive only.** Check the asking user's Drive ACL at query time. Not possible under `drive.file`: Bob's token cannot read a file he never picked. It needs `drive.readonly` per user (restricted) or the sensitive `documents.readonly` path, so it is a Phase-3-with-verification option only.
3. **Registry drives Drive.** One Google Group per project. When an admin registers a document, the service shares it with that group using the picking user's token (`permissions.create`, type `group`, allowed under `drive.file` for picked files **(verify, step 14)**). Project membership in the registry mirrors group membership. Then whoever is in the project opens the document in Drive too, a document in several projects is shared with several groups, and links in answers work. On every sync the worker calls `permissions.list` and flags a document whose group is gone as `access mismatch` to the maintainer. If the picking user cannot share, registration stops at `not shared` rather than silently readable.

Option 3 is the recommendation. Its cost: someone creates and maintains the groups. In Phase 1 the Zuri admin does it by hand; in Phase 3 the customer's admin does, or we do through the Admin SDK, which is another scope and another admin consent. One hole no option closes: a reader removed from the group but left in the registry reads the mirror until the next sync, so removal from a project must be one action that does both.

**Where Google pushes back.** It doesn't. Whether Zuri's sharing settings allow group shares and whether the Picker grant includes sharing rights is untested.

**Test.** Pilot step 14 in §5. Decision for [zuri-context-service.md](zuri-context-service.md) §3.1 and §4.

### 2.11 What about files on Drive that are not Docs or Sheets?

**Answer.** The Picker selects them (there are views for PDFs, images, presentations, drawings, forms), `drive.file` covers them once picked, and `files.get?alt=media` downloads them up to Drive's 5 TB limit. Drive extracts no text for us; the indexable-text field is write-only, and OCR runs only when a file is converted into a Doc on upload. So conversion is ours: markitdown (MIT, 194k stars), docling (MIT, table recognition and OCR) or pandoc for docx. Slides come as plain text or pptx, or as JSON from the Slides API including speaker notes. A Word file can also be uploaded with the Docs MIME type, which creates a Google Doc copy the app can export as Markdown; the copy belongs to the uploading user and drifts from the original `.docx`, so the registry must say which one it tracks.

**Where Google pushes back.** Only by offering nothing. The 10 MB cap applies to exports, not to downloads.

**Test.** Pilot step 12 in §5.

### 2.12 How stable is all this?

**Answer.** The APIs are stable; the rules around them move every few months. Google's API terms let it "discontinue the APIs … for any reason and at any time" with changes effective "no sooner than 30 days after they are posted". The Cloud terms' 12-month notice for backwards-incompatible changes covers Workspace core services, and whether that includes the developer APIs is not established. For the coming billing change Google has promised 90 days.

What changed since 2019, from the release notes: restricted-scope verification (2020), OAuth out-of-band flow removed (2023), service accounts lost Drive storage (2025), granular consent for HTTP add-ons (2025), the quota-unit model (May 2026), Events API GA (May 2026 **(verify)**), native Markdown (October 2026), the MCP client flag (November 2026), billing above a threshold "later in 2026". None of them changed `drive.file`, the Picker, `files.export` or the change feed. The pattern: Google tightens auth and adds billing, and leaves the data APIs alone.

**Where Google pushes back.** 30 days is the only contractual promise. A product sold to others needs a watch on the Workspace release notes and the Workspace Updates blog, and an architecture where the Google adapter is one module.

**Test.** None. Subscribe to the two feeds.

### 2.13 What does it cost in money?

| Item | Today | Later |
|---|---|---|
| Drive, Docs, Sheets APIs | Free, "no additional cost" | Charges above 400 million units/day "planned later in 2026", 90 days' notice, no price list yet. The plan's 5-minute `files.get` polling of 500 files is about 0.7 million units/day, plus 200 units per changed file |
| Google Cloud project | No billing account needed to enable the APIs **(inferred)** | Quota increases will require billing |
| Pub/Sub for the Events API | First 10 GiB of throughput per billing account free | $40 per TiB after |
| OAuth verification | Free | Free |
| CASA assessment (restricted scopes, Phase 3 only) | Not needed | Google charges nothing; the lab does: about $540 per year for Tier 2 **(unverified)**, once a year |
| Marketplace listing | Free per the current overview; an older developer-terms page mentions a listing fee **(unverified which governs)** | Same |
| Developer Preview programme (hosted MCP) | Free | Not usable in a public product |

**Where Google pushes back.** The one unknown is the API price list, which does not exist yet. At our volume it should not matter; at Phase 3 volume it is a per-tenant cost line we cannot quote today.

### 2.14 Does writing back work?

**Answer.** Yes, since 30 September 2026, and under `drive.file`. The Docs API's `batchUpdate` with `writeMode: SUGGEST` applies every request as a suggestion the maintainer can accept or reject; anchored comments go through the Docs API's `InsertCommentRequest` with a text range and an optional assignee, and Sheets and Slides got anchored comments the same day. Drive's older `comments.create` still exists but Google says the editors "don't render comments created with the Drive API anchored to content". Both appear under the OAuth user's name, so the AI's comment is signed by whoever's token the service uses **(verify, step 11)**.

**Where Google pushes back.** Suggest mode cannot touch document format, headers, footers, tabs or named ranges. Docs API writes are limited to 60 per minute per user. Rejecting a suggestion needs edit access or authorship, so the service account or user that wrote it should keep access.

**Test.** Pilot step 11 in §5.

### 2.15 Can we avoid the API by syncing files to disk?

**Answer.** No. Drive for desktop keeps native Docs as links that open in the browser; there is no local content. Takeout runs at most every two months with no API. rclone can export Docs to Markdown (`md` is in its export-format list), but it cannot work under our per-file model: its `drive.file` mode sees "only files and folders rclone creates", so it needs `drive.readonly` or `drive`, or a service account with domain-wide delegation. Its shared client ID "is being retired and will stop working during 2026". The gog CLI is the same story with `drive` plus `documents`. Every one of these tools ends up calling `files.export` and the change feed, which is what our sync worker does without the detour.

**Where Google pushes back.** By making the per-file scope incompatible with generic sync tools. That is the point of `drive.file`.

**Test.** None.

---

## 3. Where Google pushes back: Zuri now vs. Phase 3

| Barrier | Internal app, Zuri's own Workspace (Phases 1–2) | Public product (Phase 3) |
|---|---|---|
| OAuth app verification | Not required | Brand verification for `drive.file`; use-case review for sensitive scopes |
| Restricted scopes (`drive.readonly`, `drive`, Activity API) | Allowed with no assessment **(verify)**; admin ticks "Trust internal apps" | Annual CASA assessment by an authorised lab, about $540 per year **(unverified)**; app type must qualify |
| 100-user cap for unverified apps | Does not apply | Does not apply while we hold only `drive.file`; applies if we add a sensitive or restricted scope before verification |
| 7-day token expiry in Testing status | Does not apply | Applies until published |
| Customer admin app controls | Our own admin | Every customer's admin can block us; restricted scopes need an explicit Trusted entry |
| Token lifetime rules (6 months unused, 100 per client, revocation, session policies, departed users) | Apply | Apply |
| `drive.file` grant bound to the picking user | Applies | Applies, unless the sensitive `documents.readonly` path is taken |
| Markdown export: Docs only, 10 MB, undocumented, no per-tab export | Applies | Applies |
| Change feed without actor; actor needs a restricted scope | Workable (internal scope allowed) | Actor unavailable unless CASA |
| Push channels payload-less, 7-day renewal; Events API 7-day renewal | Applies | Applies |
| Quotas (1M units/min per project; export 200 units) | Fine | Per-project quota shared by all tenants; charges "planned later in 2026" |
| Limited Use policy: in-app disclosure, privacy-policy statements, no training | Does not apply (own domain) | Applies once we hold a sensitive or restricted scope; `drive.file` alone is exempt **(verify)** |
| Agentic rules: user confirms tool calls, prompt-injection protection | Apply to every app; own-domain exemption unclear **(verify)** | Apply |
| Claude.ai consumer accounts may train on inputs | Zuri is on Team | Customers must be on Team/Enterprise or have training off |
| API billing above 400 million units/day | Far below | A cost line with no price list yet |
| Generic sync tools (rclone, gog) incompatible with `drive.file` | Applies | Applies |
| Google-hosted MCP servers in Developer Preview | Usable personally | May not be used in a public product before GA |
| Marketplace listing | Private listing, no review | Public listing needs OAuth verification and Marketplace review |
| OAuth client "AI-powered agent" flag from 2026-11-11 | Only if we call Google's MCP servers | Same |

---

## 4. What this means for the plan

No document has been edited. These are inputs for [zuri-context-service.md](zuri-context-service.md).

1. **The scope rule splits by phase.** "Never restricted scopes" (§3.2, §7.1, §7.4 rule 5) stays right for Phase 3. For Phases 1–2 the Internal exemption, if the pilot confirms it **(verify)**, makes a service account with `drive.readonly` legal.
2. **The written fallback is wrong.** §3.2 and the §5 checklist give "share picked files with a service account" under `drive.file` as the fallback. A service account under `drive.file` sees only files it opened or created **(inferred)**; the fallback has to be the `drive.readonly` variant, and the registry must run in both access modes.
3. **The middle path is missing.** `documents.readonly` / `spreadsheets.readonly` (sensitive, no CASA) belongs in §8 as the Phase 3 answer to departed pickers.
4. **Change feed.** §3.3 plans 5-minute polling. The Workspace Events API gives typed events under `drive.file`; test it in the pilot and keep polling as the fallback.
5. **Mirror hygiene.** §3.3 and §3.4 should state: images out of Markdown into hash-named files, normalise, commit on hash change only, one commit per run (not per file as §3.3 says now), `version` in the trailer, store behind one interface so Postgres can replace git in Phase 3.
6. **Converter ownership.** Replace the docx+pandoc fallback in §3.3 with a Docs-JSON converter; the native export alone is an unspecified dependency.
7. **Actor identity.** The maintainer loop needs no Google actor. Say so in §3.6, and limit the Files view to "last modified by".
8. **Native `.md` in Docs** is a new option for projects whose authors accept writing Markdown. Worth one paragraph in §8.
9. **The 2026-11-11 rule** needs no action unless we use Google's MCP servers.
10. **Permissions model.** State in §3.1 and §4 that the registry decides who reads a mirrored document and that the service shares each registered document with a per-project Google Group, checks the ACL on every sync, and makes removal from a project one action.
11. **Phase 3 compliance list** for §7.1, adopted regardless of scope because it is cheap: in-app data-use disclosure, privacy policy with the Limited Use and no-training sentences, user confirmation of MCP tool calls, prompt-injection mitigation, and a requirement that customers' Claude.ai users are on Team or Enterprise.
12. **Non-native files.** §3.3 should name the converter for PDF, docx and pptx (markitdown or docling) and treat Slides as JSON from the Slides API.
13. **Write-back** (§6.5 step 2.4) is confirmed GA under `drive.file` with the Docs API, including anchored comments; drop the Drive `comments.create` route.

---

## 5. One-afternoon pilot: prove it or kill it

Do this before any other Phase 1 work. It costs one Google Cloud project, one test Doc, one test Sheet and a laptop. Record the result of every step; the rows that fail become open questions. The pilot checklist in zuri-context-service.md §5 covers mostly the Claude side; its two Google rows are superseded by steps 4, 7 and 8 here.

1. **Google Cloud project.** Create a project, enable the Drive API, Docs API, Sheets API, Drive Activity API and Workspace Events API. Set the OAuth consent screen user type to **Internal**. Create one Web application OAuth client (redirect to `http://localhost`) and one API key restricted to the Picker API and `https://docs.google.com/*`. Note what Google asked for and what it refused.
2. **Picker.** Run Google's Picker web sample with the `drive.file` scope. Pick one Doc and one Sheet that sit in a colleague's My Drive, then one Doc in a Shared Drive. Store the refresh token.
3. **Read from the server.** With the stored refresh token and no browser, call `files.get` on all three files, then `files.export` (`text/markdown`) for the Doc and `spreadsheets.values.batchGet` for every tab of the Sheet. Commit the output to a scratch git repository.
4. **Fidelity.** Put a table, a nested list, a checkbox list, an image, a footnote, a second tab and one pending suggestion into the test Doc. Export again and diff. Export a third time without changing anything and compare the hashes. Download the same Doc as Markdown from the Docs UI and diff it against the API export.
5. **Change feed.** Call `changes.getStartPageToken`, then edit the Doc, rename it, move it to another folder, trash it, restore it, un-share it from the picking user, and share it again. After each action call `changes.list` and `files.get` and write down exactly what came back (`removed`, `trashed`, `modifiedTime`, `version`, `lastModifyingUser`). Time how long after a keystroke `modifiedTime` moves.
6. **Who did it, and the Internal exemption.** Call the Drive Activity API for the Doc with the `drive.file` token and expect a scope error. Then add `drive.activity.readonly` to the Internal app, consent as a user, and record whether Google shows any verification or warning screen and whether the call now returns actors.
7. **Second token.** Sign in as a second user who did not pick the file and try `files.get` with that user's token. Then revoke the first user's token at myaccount.google.com and repeat step 3. If the admin agrees, suspend a test account that picked a file and repeat step 3 with its stored token.
8. **Service account.** Share the Doc with a service account and try `files.get` with `drive.file`, then with `drive.readonly`. Try to add the service account to the Shared Drive as a member and repeat for the Shared Drive Doc.
9. **Events.** Create a Workspace Events API subscription on the Doc with a Pub/Sub topic under `drive.file`; then, in parallel, a `changes.watch` channel on an `ngrok` HTTPS endpoint. Edit the Doc and record what each delivers, and when each expires.
10. **Admin side.** In the Workspace admin console open Security → Access and data control → API controls → Manage third-party app access. Record whether the Internal app appears, its status, the state of "Trust internal apps", whether the service account from step 8 is listed anywhere, and what the admin could switch off.
11. **Write back.** With the `drive.file` token call the Docs API `batchUpdate` with `writeMode: SUGGEST` to replace one sentence, and `InsertCommentRequest` to anchor a comment on it. Open the Doc and record how both appear and under whose name.
12. **Non-native files.** Pick a PDF and a Word file through the Picker, download both with `files.get?alt=media`, run markitdown on them, and judge the Markdown. Upload the Word file with the Google Docs MIME type and export the result as Markdown.
13. **Native Markdown.** Rewrite the test Doc as an `.md` file in Drive. Have two people edit it in Docs at the same time and add a comment. Download it with `files.get?alt=media` and diff the bytes against what was written.
14. **Group sharing.** Create a Google Group for the test project. With the picking user's token call `permissions.create` (type `group`) on the picked Doc, then `permissions.list`. Open the Doc as a group member who was not on the file before. Remove the group in the Drive UI and check that the next `permissions.list` shows it.

Exit criterion: step 4 passes, or the Docs-JSON fallback is accepted; and either steps 3 and 5 pass under `drive.file` alone, or step 8 passes with a service account on `drive.readonly`. Either result lets Phase 1 start. If the access half fails both ways, the context service needs a different access model before any more design work.

---

## 6. What we found

**Google is not the enemy of this plan.** It does not block third-party access to Docs; it prices it, and the price is close to zero for the access pattern we chose. Per-file access through `drive.file` and the Picker needs no verification beyond brand checks, no security assessment, no Limited Use paperwork, and works at any customer whose admin has not blocked us. Every expensive rule (CASA, disclosure screens, admin trust entries, approved app types) starts the moment we ask for `drive.readonly` or the Activity API in a product sold outside our own Workspace. Inside Zuri none of it appears to apply **(verify)**.

**The traps are smaller and technical, in this order:**

1. A `drive.file` grant belongs to the person who picked the file. Their departure, revocation or six months of inactivity switches the document off. The fallback written in the plan, a service account under `drive.file`, does not work **(inferred)**; a service account needs `drive.readonly`, which appears legal inside Zuri **(verify)** and is a CASA cost in Phase 3. The sensitive `documents.readonly` scope is the untested middle path.
2. The Markdown export is an unspecified format: base64 images at the end, comments and suggestions dropped, tabs concatenated **(unverified)**, determinism untested. The mirror has to normalise, commit on content hash, and keep a Docs-JSON converter as the fallback.
3. "Who changed it" sits behind a restricted scope. Under `drive.file` we get "something changed" and the last editor; delete and un-share look alike. The maintainer loop does not need more.
4. Phases 1–2 and Phase 3 run under different rules. Anything built on a restricted scope now is rework later. The registry must run in both access modes.
5. The registry overrides Drive permissions and the plan does not say so. The resolution: the registry is the source of truth, and the service makes Drive follow it by sharing each registered document with a per-project Google Group, checked on every sync **(verify)**.
6. Claude.ai consumer accounts can train on inputs. Phase 3 customers must be on Team or Enterprise.
7. Sheets have no Markdown and PDFs have no text. All conversion beyond Docs is our code.
8. Google promises 30 days' notice. It has changed auth and quotas repeatedly and left the data APIs alone. The Google adapter is one module.

**Source of truth.** For content it is Drive; the mirror is a cache stamped with `version` and export time, in git for now and behind an interface. Google Docs stay the source in Phase 1. Native `.md` files in Docs, editable since October 2026, are the cheapest way out of trap 2 and are in the pilot as step 13. Whether the Hub itself becomes the editor one day, so that Markdown in our store is the source and Drive only the import, is a Phase 2+ product decision, not a Google constraint.

**What settles it.** The pilot in §5 and its exit criterion.

---

## 7. Not yet asked

Questions the reviews raised that no section above answers. Each is one afternoon of reading or one pilot step.

1. Can `files.export` fail on a picked file whose owner disabled download and copy for viewers, or under an admin DLP rule, and what does the mirror show then?
2. What does picking a folder grant under `drive.file`: the folder only, or its descendants? How many files can one Picker session select?
3. Where do 30 users' refresh tokens live, how is `invalid_grant` surfaced as `status=error`, and which admin policies kill tokens silently?
4. Does a Picker grant on a Shared Drive file outlive the picking user's departure, since the organisation owns the file?
5. For a Word file converted into a Google Doc, who re-syncs the copy when the original changes, and does the registry point at the copy or the original?

---

## Sources

Full source lists with page stamps are in the six Round 3 notes: [google_oauth_admin_tokens.md](research_notes/Round%203%20(2026-11-05)/google_oauth_admin_tokens.md), [google_drive_change_detection.md](research_notes/Round%203%20(2026-11-05)/google_drive_change_detection.md), [google_docs_markdown_git.md](research_notes/Round%203%20(2026-11-05)/google_docs_markdown_git.md), [google_integrators_mcp_addons.md](research_notes/Round%203%20(2026-11-05)/google_integrators_mcp_addons.md), [google_data_policy_stability_cost.md](research_notes/Round%203%20(2026-11-05)/google_data_policy_stability_cost.md), [google_writeback_sync_tools_nonnative.md](research_notes/Round%203%20(2026-11-05)/google_writeback_sync_tools_nonnative.md). The pages this note leans on most:

- Scopes and verification: https://developers.google.com/workspace/drive/api/guides/api-specific-auth , https://developers.google.com/identity/protocols/oauth2/production-readiness/overview , https://support.google.com/cloud/answer/13464321 , https://support.google.com/cloud/answer/13465431 , https://support.google.com/cloud/answer/15549945 , https://appdefensealliance.dev/casa/casa-tiering
- Tokens and admin controls: https://developers.google.com/identity/protocols/oauth2 , https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data , https://knowledge.workspace.google.com/admin/apps/control-api-access-with-domain-wide-delegation
- Picker and service accounts: https://developers.google.com/workspace/drive/picker/guides/web-picker , https://developers.google.com/workspace/drive/picker/guides/desktop-mobile-picker , https://developers.google.com/workspace/drive/api/guides/about-shareddrives
- Changes, push, activity, revisions, events: https://developers.google.com/workspace/drive/api/guides/manage-changes , https://developers.google.com/workspace/drive/api/guides/push , https://developers.google.com/workspace/drive/activity , https://developers.google.com/workspace/drive/api/guides/manage-revisions , https://developers.google.com/workspace/events/guides/events-drive , https://developers.google.com/workspace/events/guides/auth
- Export and Docs API: https://developers.google.com/workspace/drive/api/guides/ref-export-formats , https://developers.google.com/workspace/drive/api/reference/rest/v3/files/export , https://developers.google.com/workspace/docs/api/how-tos/tabs , https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/get , https://workspaceupdates.googleblog.com/2026/10/preview-edit-and-collaborate-on-Markdown-files-natively-across-Drive-and-Docs.html , https://support.google.com/docs?p=markdown_mode
- Quotas: https://developers.google.com/workspace/drive/api/guides/limits , https://developers.google.com/workspace/docs/api/limits , https://developers.google.com/workspace/sheets/api/limits
- MCP and connectors: https://developers.google.com/workspace/guides/configure-mcp-servers , https://developers.google.com/workspace/release-notes , https://developers.google.com/workspace/preview , https://claude.com/docs/connectors/google/drive , https://support.claude.com/en/articles/10166901-use-google-workspace-connectors , https://github.com/taylorwilsdon/google_workspace_mcp
- Add-ons and Apps Script: https://developers.google.com/workspace/add-ons/concepts/types , https://developers.google.com/workspace/add-ons/guides/alternate-runtimes , https://developers.google.com/workspace/marketplace/how-to-publish , https://developers.google.com/apps-script/guides/services/quotas
- Data policy and terms: https://developers.google.com/terms/api-services-user-data-policy , https://developers.google.com/workspace/workspace-api-user-data-developer-policy , https://support.google.com/cloud/answer/13463817 , https://developers.google.com/terms , https://developers.google.com/workspace/tools-safety , https://www.anthropic.com/legal/commercial-terms , https://privacy.claude.com/en/articles/7996868-is-my-data-used-for-model-training
- Write-back: https://developers.google.com/workspace/docs/api/how-tos/suggestions , https://workspaceupdates.googleblog.com/2026/09/programmatic-comment-and-suggestion.html , https://developers.google.com/workspace/drive/api/guides/manage-comments
- Sync tools and converters for non-native files: https://rclone.org/drive/ , https://forum.rclone.org/t/google-drive-and-google-photos-users-action-required/54005/ , https://github.com/openclaw/gogcli , https://support.google.com/drive/answer/10838124 , https://github.com/microsoft/markitdown , https://github.com/docling-project/docling
- Converters and git: https://github.com/MattFisher/md-gdoc , https://github.com/mieweb/wikiGDrive , https://github.com/AnandChowdhary/docs-markdown , https://github.com/rsheyd/google-docs-markdown-sync , https://constly.com/blog/google-docs-to-markdown/ (unverified)
- Cost: https://cloud.google.com/pubsub/pricing , https://developers.google.com/workspace/marketplace/overview
- Vendors: https://workspace.google.com/marketplace/app/docusign_esignature_for_google_workspace/469176070494 , https://docs.glean.com/connectors/native/gdrive/setup , https://help.getguru.com/docs/setting-up-google-drive-google-sheets-as-a-source , https://help.openai.com/en/articles/10929079 , https://www.notion.com/help/notion-ai-connectors-for-google-drive
