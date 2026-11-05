# Round 3 notes: write-back, disk sync tools, non-native files (checked 2026-11-05)

Scope: input for `google-drive-grill.md` questions 11, 14 and 15. No developers.google.com page fetched shows a visible "last updated" stamp; dated items are the Docs release notes, the Drive quota note (2026-05-01), the rclone forum post, and GitHub release dates (via the GitHub API today). (unverified) = third-party only; (inference) = not stated on any official page.

## Q1. Writing back to Google Docs / Sheets

**Docs API suggested edits (`documents.batchUpdate` + `writeControl.writeMode = SUGGEST`)**
- Status: GA. Release notes 2026-07-07 "Developer Preview: … create and manage comments and suggestions … Writing edits as suggestions by setting `writeControl.writeMode` to `SUGGEST`"; 2026-09-30 "Generally Available: You can now programmatically read, create, and manage comments and suggestions in Google Docs". Workspace blog 2026-09-30: "Available to all Google Workspace customers and users with personal Google accounts", gradual rollout up to 15 days; no admin control.
- Scopes on `documents.batchUpdate`: `documents`, `drive`, `drive.file` → `drive.file` suffices (no separate scope for suggest mode is documented).
- Discovery doc: `WriteControl.writeMode` enum `WRITE_MODE_UNSPECIFIED` ("Defaults to EDIT"), `EDIT`, `SUGGEST` ("Apply all updates as suggestions"). The response carries `suggestionResponses` (createdSuggestionIds, acceptedSuggestionIds, …). How-to: "All updates in the request are processed as suggestions."
- Supported requests: the how-to lists only the unsupported ones, so insertText / deleteContentRange / replaceAllText / text and paragraph styling are allowed by omission. Unsupported in suggest mode (error): `AddDocumentTab`, `CreateNamedRange`, `DeleteFooter`, `DeleteHeader`, `DeleteNamedRange`, `DeleteTab`, `UpdateDocumentTabProperties`, `UpdateTableColumnProperties`; "you cannot suggest changes to document format or header and footer settings". Footnotes and table cell content are not listed as excluded. Tabs: `Range.tabId` ("When omitted, the request applies to the first tab"), `segmentId` targets header/footer/footnote.
- Attribution (inference, not documented): the suggestion appears as the OAuth user. The official text only says reject "requires edit access … or being the author of the suggestion" and delete "requires being the author". Accept needs edit access.
- Quota (Docs API): write 600/min/project, 60/min/user/project; read 3,000 / 300.

**Anchored comments**
- Docs API (recommended): `InsertCommentRequest { content (≤ 2048 UTF-8 units), assigneeEmailAddress?, range {startIndex, endIndex, tabId, segmentId} }` in `batchUpdate`, GA 2026-09-30, same scopes (`drive.file` ok). Replies: `AddCommentReplyRequest`. Sheets/Slides got cell/element-anchored comments the same day (blog).
- Drive `comments.create` / `replies.create`: scopes `drive`, `drive.file`. `anchor` = "A region of the document represented as a JSON string" that "Developers can define their own format for". Official: "Google Workspace editor apps (such as Docs, Sheets, and Slides) don't render comments created with the Drive API anchored to content", they show as unanchored. `quotedFileContent.value` is "interpreted as plain text if set through the API". `author` is "Output only … email address and permission ID will not be populated"; (inference) displays as the OAuth user. Every comments method except delete requires `fields`.

**Sheets under `drive.file`**: `spreadsheets.values.update` and `spreadsheets.batchUpdate` list `drive`, `drive.file`, `spreadsheets` → fine. Quota: 300 read / 300 write per min per project, 60/60 per user; "no hard size limits" but a 2 MB payload is recommended, 180 s processing timeout. Limits: 20M cells / 100 MB per spreadsheet (Drive help).

## Q2. Avoiding the API by syncing to disk

**rclone (rclone.org/drive)**
- `--drive-export-formats` default `docx,xlsx,pptx,svg`; the table includes `md` = `text/markdown`, plus docx, odt, html, txt, rtf, epub, pdf, csv/tsv/xlsx/ods, pptx/odp, svg/png/jpg and link files. `--drive-import-formats`: no default. "rclone adds the extension to the google doc".
- `--drive-scope`: `drive` (default), `drive.readonly`, `drive.file` ("access only to files and folders rclone creates. Files uploaded through the web interface are not visible"), `drive.appfolder`, `drive.metadata.readonly`. Consequence: rclone cannot operate under the app's Picker/`drive.file` model; it needs `drive.readonly`/`drive` (restricted) or a service account with domain-wide delegation (`--drive-impersonate`) or folders shared with the SA.
- Shared client ID: "This shared client_id is being retired and will stop working during 2026." Forum (rclone maintainer, 2026-07-06): Google "will start charging for API requests on rclone's built in default client-ID", "Later in 2026, following 90 days of notice"; update 2026-09-19 "Google have gone silent". Own client: "In most cases verification is not actually required"; personal use "click through the 'unverified app' warning screen"; Testing-mode grants expire after a week; "default Google quota is 10 transactions per second".
- `--drive-shared-with-me`, shared drives via `--drive-team-drive`.
- Native Docs: "appear as size -1 in rclone ls", size 0 in VFS; sync/copy work because "rclone knows to ignore the size"; mount may yield a 0-byte file. `--drive-skip-gdocs` makes them invisible. With no usable size/hash for exports, change detection falls to modtime (not documented). The 10 MB `files.export` limit applies to rclone exports too.
- Change watching: no `rclone watch` (issue #4381 is a 2020 proposal). `backend/drive/drive.go` implements `ChangeNotify` via StartPageToken/changes polling, used only by `rclone mount`/`serve` `--poll-interval` (default 1m); `rclone sync` on cron does a full listing.

**Drive for desktop**: Docs/Sheets/Slides "open in your web browser" (answer/10838124, /7329379); offline Docs only via Drive on the web with the "Google Docs Offline Chrome extension" (answer/2375012). So no local docx/md content; local `.gdoc` pointer files (unverified, third-party).

**Takeout**: official: "Automatically create an archive of your selected data every 2 months for one year"; no Takeout API. Format choices DOCX/PDF/ODT, no Markdown (unverified). The Data Portability API lists Chrome, Maps, Play, Search, Shopping, YouTube, no Drive.

**gog CLI (openclaw/gogcli, MIT, v0.47.1 2026-10-27)**: requires your own Desktop OAuth client; scopes for Drive/Docs: `drive` (full) + `documents`, i.e. restricted, not `drive.file`. `gog docs export <id> --format pdf|docx|txt|md|html`, `--tab` experimental (undocumented per-tab endpoint). Change watching: `gog drive changes poll --state-file … --interval` and `gog drive changes serve` (push, needs public HTTPS), `gog docs comments poll`.

**Other Google-supported sync for native Docs**: none found; every path (rclone, gog, Apps Script, Colab) exports through `files.export` or the Docs API.

## Q3. Non-native files

- Picker ViewIds: `DOCS` (all types), `DOCUMENTS`, `SPREADSHEETS`, `PRESENTATIONS`, `PDFS`, `DOCS_IMAGES`, `DOCS_VIDEOS`, `DRAWINGS`, `FORMS`, `FOLDERS`. Desktop/mobile: "only `drive.file` is permitted".
- Download: `files.get?alt=media` with a read-capable scope (`drive.file` ok for picked files); `Range` partial downloads; no numeric API limit, Drive file limit "Up to 5 TB". `files.export` "limited to 10 MB", no partial export. Text extraction: none. `contentHints.indexableText` is "never populated in responses" (write-only, 128 KB); OCR only on import ("When you convert an image to a Docs, Drive uses OCR", `ocrLanguage` hint; JPEG/PNG/GIF/BMP/PDF → Docs with the image embedded).
- Convert .docx → Google Doc: `files.create` with a Google `mimeType` on upload is the documented path (Word/ODT/HTML/RTF/txt/Markdown → Docs; check `about.importFormats`). `files.copy` scopes include `drive.file`; v2→v3 table: `convert` → "Provide the target mimeType in the resource body", but community reports say copy + mimeType may not convert (unverified). Either way the new file is created by the app, so visible under `drive.file`, then exportable as `text/markdown`.
- Slides: `presentations.get` scopes include `drive.file`; text in `pageElements[].shape.text.textElements[].textRun`, notes via `slideProperties.notesPage`; export pptx/odp/pdf/txt. Drawings: pdf/jpg/png/svg. Forms: `forms.responses.list` scopes `drive`, `drive.file`, `forms.responses.readonly`.
- Converters (GitHub, 2026-11-05):
  - microsoft/markitdown: MIT, v0.1.9 (2026-10-21), 194k stars; pdf/docx/pptx/xlsx/images; tables kept; images via OCR/EXIF, LLM descriptions optional.
  - docling: MIT, v2.147.0 (2026-11-04); pdf/docx/pptx/xlsx/html/images; layout and table structure recognition, OCR, `export_to_markdown()`.
  - PyMuPDF 2.1 (AGPL-3.0, 2026-11-02); the pymupdf4llm repo is now a placeholder, code lives in pymupdf 2.x; PDF-focused Markdown with tables/images.
  - unstructured: Apache-2.0, 0.27.36 (2026-10-30); pdf/docx/pptx/images (Tesseract); table quality lower than the hosted API (vendor benchmark); telemetry on by default.
  - pandoc: GPL-2.0, 3.13 (2026-10-26); reads docx/odt (not pdf/pptx/xlsx); tables and `--extract-media` for images.

## Sources
- Docs batchUpdate: https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/batchUpdate
- Docs suggestions/comments how-to: https://developers.google.com/workspace/docs/api/how-tos/suggestions
- Docs release notes: https://developers.google.com/workspace/docs/release-notes
- Workspace blog 2026-09-30: https://workspaceupdates.googleblog.com/2026/09/programmatic-comment-and-suggestion.html
- Docs requests: https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/request
- Docs discovery: https://docs.googleapis.com/$discovery/rest?version=v1
- Docs tabs: https://developers.google.com/workspace/docs/api/how-tos/tabs ; limits: https://developers.google.com/workspace/docs/api/limits
- Drive comments guide: https://developers.google.com/workspace/drive/api/guides/manage-comments ; create: https://developers.google.com/workspace/drive/api/reference/rest/v3/comments/create ; resource: https://developers.google.com/workspace/drive/api/reference/rest/v3/comments
- Drive scopes: https://developers.google.com/workspace/drive/api/guides/api-specific-auth ; limits: https://developers.google.com/workspace/drive/api/guides/limits
- Sheets scopes: https://developers.google.com/workspace/sheets/api/scopes ; values.update: https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/update ; batchUpdate: https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets/batchUpdate ; limits: https://developers.google.com/workspace/sheets/api/limits
- Drive export formats: https://developers.google.com/workspace/drive/api/guides/ref-export-formats ; downloads: https://developers.google.com/workspace/drive/api/guides/manage-downloads ; uploads/convert: https://developers.google.com/workspace/drive/api/guides/manage-uploads ; files.copy: https://developers.google.com/workspace/drive/api/reference/rest/v3/files/copy ; files resource: https://developers.google.com/workspace/drive/api/reference/rest/v3/files ; v2→v3: https://developers.google.com/workspace/drive/api/guides/v2-to-v3-reference
- Picker: https://developers.google.com/workspace/drive/picker/guides/overview ; ViewId: https://developers.google.com/workspace/drive/picker/reference/picker.viewid
- Slides: https://developers.google.com/workspace/slides/api/reference/rest/v1/presentations/get ; https://developers.google.com/workspace/slides/api/reference/rest/v1/presentations.pages
- Forms: https://developers.google.com/workspace/forms/api/reference/rest/v1/forms.responses/list
- Drive file limits: https://support.google.com/drive/answer/37603 ; Drive for desktop: https://support.google.com/drive/answer/10838124 , https://support.google.com/drive/answer/7329379 ; offline: https://support.google.com/drive/answer/2375012
- Takeout: https://support.google.com/accounts/answer/3024190 ; Data Portability: https://developers.google.com/data-portability/user-guide/overview
- rclone drive: https://rclone.org/drive/ ; mount: https://rclone.org/commands/rclone_mount/ ; forum: https://forum.rclone.org/t/google-drive-and-google-photos-users-action-required/54005/ ; issue: https://github.com/rclone/rclone/issues/4381 ; source: https://raw.githubusercontent.com/rclone/rclone/master/backend/drive/drive.go
- gogcli: https://github.com/openclaw/gogcli
- Converters: https://github.com/microsoft/markitdown , https://github.com/docling-project/docling , https://github.com/pymupdf/pymupdf4llm , https://github.com/pymupdf/PyMuPDF , https://github.com/Unstructured-IO/unstructured , https://github.com/jgm/pandoc/releases
