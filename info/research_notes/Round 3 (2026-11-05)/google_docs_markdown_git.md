# Round 3 notes: Docs/Sheets → Markdown → git (checked 2026-11-05)

Scope: input for `google-drive-grill.md`. Legend: **(official)** Google page · **(code)** verified in repo source/tests · **(unverified)** third-party blog/vendor only · **(no evidence)** nothing found either way.

## 1. Drive `files.export` with `text/markdown`

**Official facts**
- `text/markdown` (.md) is a listed export MIME for Google Docs; Sheets has no Markdown (xlsx/ods/pdf/zip-html/csv+tsv "first sheet only"); Slides: pptx/odp/pdf/txt; Drawings: pdf/jpg/png/svg [S1].
- "Exported content is limited to 10 MB"; byte-range/partial downloads are not supported for exports. `files.download` (long-running operation) is listed as an alternative export path; the page does **not** say it lifts the 10 MB cap (the claim that it does is unverified, blog only) [S2].
- Quota: download methods (grouped; `files.export` not named separately) cost **200 quota units**; 1,000,000 units/min/project, 325,000 units/min/user/project; limits revised 2026-05-01 [S3].
- Neither the 2024-07-16 announcement nor the help pages contain an element-support chart [S4, S19].

**Rendering evidence (what the exporter actually emits)**

| Element | Finding | Evidence |
|---|---|---|
| Images | Inline `![][image1]`, definitions `[image1]: <data:image/png;base64,…>` at the **end of the file**; images are **re-compressed** (originals only via the Docs API `contentUri`) | (code) gdocs-md-cli `MarkdownPatcher.kt`, README "heavily compressed images"; md-gdoc `images.py`; google_workspace_mcp PR #181 [S10, S11, S12]; Constly: 1.1 KB doc + 1 photo → 313 KB .md [S9] |
| Headings | Carry Pandoc-style `{#heading-id}` suffixes (gdocs-md-cli strips them as "invalid header links"); Constly: wrapped in bold `# **Heading**` | (code) [S11]; (unverified) [S9] |
| TOC | Exported as a link list `[text](#anchor)` pointing at those ids | (code, inferred) [S11] |
| Tables | Real pipe tables; merged cells lost (each cell → column) | (unverified) [S9], [S13] |
| Lists | Nested lists supported per the HN launch thread; a bullet list directly followed by a numbered list has no blank line → parsers merge them | (unverified) [S14], [S9] |
| Checkboxes | GDMS README: checkboxes become bullets prefixed `o] ` / `x] `; ambiguous whether this describes Google's export or GDMS's own serialiser | (unverified) [S17] |
| Code blocks | md-gdoc e2e against the live API: fenced blocks including the language tag round-trip byte-perfect; mdisbetter claims they're dropped | (code/e2e) [S10] vs (unverified) [S13] |
| Footnotes | Conflicting: "often dropped/inlined" [S13]; md-gdoc pushes `[^1]` only with `--replace` | (unverified) |
| Links | Clean `[text](url)`; file chips → links, people chips → mailto | (unverified) [S9, S14] |
| Comments | **Dropped, no marker** (UI download). The zipped-HTML export keeps them as footnotes; the docx export keeps them as Word comments | (unverified) [S9], [S15] |
| Suggestions | Pending suggestions **excluded**; the export reflects the doc without them. mdisbetter claims the opposite | (code/e2e) md-gdoc [S10]; (unverified) [S13] |
| Tabs | UI download: **all tabs concatenated in order**, each tab's name inserted as `# **Tab name**` [S9]. API: `files.export` has no per-tab option [S16]; gogcli `--tab` resolves the tab id via the Docs API then hits the undocumented `docs.google.com/document/d/{id}/export?format=markdown&tab=…` URL | (unverified)/(code) |
| Equations, drawings, charts, page breaks, HR, header/footer | **(no evidence)** | — |
| Determinism (byte-identical re-export) | **(no evidence either way)**; nobody has published a test. Markdown is plain text (no zip timestamps as in docx), but anchor ids and image recompression are unknowns | — |

Practical: test with two exports of an untouched doc + `sha256sum`; regardless, normalise before commit (strip `{#…}`, unwrap bold headings, move images to hash-named files) and commit only when the normalised hash changes.

## 2. Native Markdown files in Docs (2026-10-05)

- (official) .md/.markdown open in Docs "without changing the file type"; "no admin control", "no end user setting"; rollout ≤15 days from 2026-10-05; all Workspace and personal accounts [S5].
- (official help) Drive: Open with → Preview, toggle **Markdown / Preview**. Docs: double-click opens "Markdown Mode" (`.md` tag by the title); "Your file does not convert to a Google Docs file. It stays an .md file." Sharing, comments and simultaneous editing are supported. Features that change on edit: smart chips → text/links; HTML → plain text; font colours, highlights, alignment removed; "Some Google Docs features might change when you edit an .MD file"; restore via version history [S6].
- **Does Docs rewrite Markdown syntax on save?** (no evidence). Google says nothing; no blog has published a round-trip test [S18]. Treat as unknown → run a round-trip diff on representative files before adopting.
- **Reading the raw file via the API**: .md is a blob, so `files.get?alt=media` returns the bytes [S2]. `drive.file` only grants files "that you open with an app" or selected via Picker [S7]. Derived: the team must pick each .md once through the Picker, or the sync worker needs `drive.readonly`.
- Comments on .md: the Drive comments API is file-type-agnostic; anchors are app-defined opaque JSON; for editor-created comments the anchor is "internal editor data" Drive can't resolve; anchors are immutable across revisions [S8]. Whether Docs' comments on .md files expose usable anchors: (no evidence).
- UI: `Tools → Preferences → Enable Markdown` gates **Copy as Markdown / Paste from Markdown** (off by default); `File → Download → Markdown (.md)` and import are on by default [S4, S19]. UI download vs API export: (no evidence) that they differ; the `…/export?format=md` URL works for shared docs [S20], presumably the same backend (unverified).

## 3. Alternatives: own converter vs docx + pandoc

| Tool | Licence | Last push | Method |
|---|---|---|---|
| mieweb/wikiGDrive | MIT | 2026-08-18 | ODT export → own `OdtToMarkdown`; images to `.assets/`, SVG for drawings; polling `watch` + git hook |
| MattFisher/md-gdoc | MIT | 2026-09-04 | Drive `files.export text/markdown`; images → `<name>.assets/` hash-named; comments pulled to `*.comments.md`; push via Docs `batchUpdate` |
| benjaminRomano/gdocs-md-cli | MIT | 2025-07-05 | Drive export + patcher (strip `{#}` anchors, fix TOC, refetch original images) |
| openclaw/gogcli (`gog docs export --format md`) | MIT | 2026-11-05 | Drive export; `--tab` = Docs API tab lookup + undocumented `?tab=` export URL |
| rsheyd/google-docs-markdown-sync | MIT | 2026-10-31 | Drive export; falls back to its own Docs-JSON serialiser when >10 MB; Sheets → 1 CSV per tab |
| niStee/gdocs-git-sync | MIT | 2026-10-24 | 23 commits, 0 stars; pull/diff/status only, method undocumented |
| AnandChowdhary/docs-markdown (npm) | MIT | 2026-10-17 | Docs API JSON → md (paragraphs, headings, lists, links, images, tables, header/footer) |
| filipedeschamps/parse-google-docs-json | MIT | 2025-05-01 | Docs JSON → JSON/md |
| Mr0grog/google-docs-to-markdown | BSD-3 | 2026-11-01 | Clipboard HTML → md (web app), not API |
| life360-oss / chasemp mdsync | MIT | 2025-12 / 2026-03 | Drive export, point-in-time, no conflict handling |
| shanemcd/gcmd, SimonGZ/google-docs-converter, zonca/gdrivemd | none | 2026-01 / 2026-03 / 2026-05 | wrappers around the native export / gog |
| mangini/gdocs2md | Apache-2.0 | 2018-07-04 | Apps Script, dead |
| evbacher/gd2md-html (Docs to Markdown add-on) | Apache-2.0 | 2025-11-17 | Apps Script add-on |

- Docs API JSON route: `documents.get` with `includeTabsContent=true` returns all tabs (default: first tab only) [S21]; quota 300 reads/min/user, 3,000/min/project [S22]. Gives you the suggestions view (`suggestionsViewMode`), original image `contentUri`, and deterministic output you control.
- docx + pandoc: `pandoc doc.docx -t gfm --extract-media=./media`; Constly test: 1 KB .md + 217 KB image file, comments dropped unless `--track-changes=all` [S9, S23]. The docx export keeps comments/suggestions as Word comments/tracked changes (unverified) [S15]. docx is a zip → byte-level nondeterminism likely (unverified).

## 4. Sheets and Slides

- Sheets API: `spreadsheets.get` → tab list/order/gid; `values.batchGet` with `ranges=['Tab name']` (whole tab), `valueRenderOption` = FORMATTED_VALUE (sheet locale, "$1.23") / UNFORMATTED_VALUE (1.23) / FORMULA ("=A1") [S24, S25]. Two passes (formatted + formula) if formulas matter.
- CSV via `https://docs.google.com/spreadsheets/d/ID/export?format=csv&gid=GID` with `Authorization: Bearer <token>` works for private sheets per Apps Script how-tos (unverified, undocumented endpoint) [S26, S27]. The official API `text/csv` = first sheet only [S1].
- xlsx export → openpyxl/markitdown: carries merged cells and number formats that `values.batchGet` can't; costs zip nondeterminism and a second parser. Only worth it if merges/formatting matter.
- Sensible 20-tab rendering: one `.md` per tab (`<sheet>/<tab>.md`) or one file with `## <Tab>` sections, header row = row 1, escape `|` and newlines, pad ragged rows, skip trailing empty rows/cols, optional `formulas.md`. Only GDMS does per-tab output (CSV, not md tables) [S17]; no maintained md-tables tool found.
- Slides: API export is `text/plain` or pptx [S1]; Slides API `presentations.get` JSON → own md; or pptx → `markitdown` (MIT, pushed 2026-10-28; speaker notes appended after slide text) [S28].

## 5. Git as the mirror store

**For**: text deltas compress well; diff/blame/review for free; any tool reads it. **Against**: binary images, noisy commits, unbounded history.
- Images: never commit base64; extract to `assets/<sha256>.<ext>` (md-gdoc, wikiGDrive pattern) or LFS; GitHub: keep the repo <1 GB ideally, <5 GB strongly; >100 MiB files are blocked [S29].
- Noisy commits: commit only when the normalised content hash changes (export determinism unknown → normalise). 500 docs × hourly = up to 12k exports/day = 2.4M quota units/day, fine vs limits [S3].
- Growth: with change-gated commits, growth ≈ real edits; worst case a monthly squash. Rewriting history (`git filter-repo`) is safe only if the mirror has a single writer and every consumer reclones [S29]; cleaner: a periodic snapshot onto an orphan branch with the old branch archived, or `git replace` grafts. Shallow clones for consumers.
- Concurrency: one sync worker per repo with a lock; batch one commit per sync run (not per doc) with doc id + Drive `modifiedTime`/`version` in a trailer. Per-tenant repos keep rewrites, ACLs and size scoped.
- Postgres (content + diff): simpler ops, transactional, queryable; MediaWiki stores every revision's full text with gzip bundling of several versions [S30]. Git wins on tooling/diff UI, Postgres on multi-tenant isolation and retention policies. Hybrid: Postgres as the system of record, git export as a view.
- Google revision history: the Docs revision list "might be incomplete for files with a large revision history"; `keepForever` and deletion only for blob files; max 200 keep-forever revisions [S31] → not a retention store you control.
- Lessons: wikiGDrive — `modifiedTime` change detection, `quota.json` throttling, title-rename redirects, `.assets/` per doc, a single `google_files.json` didn't scale [S32]; md-gdoc — hash-named images for stable pulls, comment anchors orphan when text changes, images not pushable; gdocs-md-cli — the export recompresses images, refetch originals via the Docs API [S10, S11]; GDMS — formatting-only remote changes shouldn't rewrite files, the Drive export fails >10 MB so keep a JSON fallback [S17].

**Biggest open items to test ourselves (no published evidence exists):** (a) byte-determinism of the `text/markdown` export on an unchanged doc; (b) whether Docs Markdown Mode rewrites .md syntax on save; (c) whether the UI "Download as Markdown" equals the API `files.export` output; (d) export behaviour for equations, drawings, charts, page breaks, HR, header/footer.

## Sources
- [S1] https://developers.google.com/workspace/drive/api/guides/ref-export-formats
- [S2] https://developers.google.com/workspace/drive/api/guides/manage-downloads
- [S3] https://developers.google.com/workspace/drive/api/guides/limits
- [S4] https://workspaceupdates.googleblog.com/2024/07/import-and-export-markdown-in-google-docs.html (2024-07-16)
- [S5] https://workspaceupdates.googleblog.com/2026/10/preview-edit-and-collaborate-on-Markdown-files-natively-across-Drive-and-Docs.html (2026-10-05)
- [S6] https://support.google.com/docs?p=markdown_mode
- [S7] https://developers.google.com/workspace/drive/api/guides/api-specific-auth
- [S8] https://developers.google.com/workspace/drive/api/guides/manage-comments
- [S9] https://constly.com/blog/google-docs-to-markdown/ (2026-09-08, updated 2026-10-02; vendor)
- [S10] https://github.com/MattFisher/md-gdoc
- [S11] https://github.com/benjaminRomano/gdocs-md-cli
- [S12] https://github.com/taylorwilsdon/google_workspace_mcp/pull/181
- [S13] https://mdisbetter.com/blog/google-docs-export-sucks (2026-05-10; vendor, no test method)
- [S14] https://news.ycombinator.com/item?id=40982118
- [S15] https://discuss.google.dev/t/feature-request-native-option-to-export-or-print-google-docs-to-pdf-with-side-margin-comments/398495 ; https://www.wps.com/office-quick-solution/document-file-issues/how-to-fix-google-docs-suggested-edits-missing-in-a-word-file/
- [S16] https://dev.to/googleworkspace/exporting-individual-tabs-from-google-docs-as-pdfs-2903
- [S17] https://github.com/rsheyd/google-docs-markdown-sync
- [S18] https://www.animaapp.com/blog/ai-en/google-docs-markdown/ ; https://9to5google.com/2026/10/05/google-docs-drive-markdown-support/
- [S19] https://support.google.com/docs/answer/12014036
- [S20] https://mareksuppa.com/til/google-docs-markdown-export/
- [S21] https://developers.google.com/workspace/docs/api/how-tos/tabs
- [S22] https://developers.google.com/workspace/docs/api/limits
- [S23] https://pandoc.org/MANUAL.html
- [S24] https://developers.google.com/workspace/sheets/api/reference/rest/v4/ValueRenderOption
- [S25] https://developers.google.com/workspace/sheets/api/reference/rest/v4/spreadsheets.values/batchGet
- [S26] https://spreadsheet.dev/comprehensive-guide-export-google-sheets-to-pdf-excel-csv-apps-script
- [S27] https://gkit.mreshank.com/blog/how-to-save-google-sheet-as-csv
- [S28] https://github.com/microsoft/markitdown
- [S29] https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github
- [S30] https://www.mediawiki.org/wiki/Manual:Text_table
- [S31] https://developers.google.com/workspace/drive/api/guides/manage-revisions
- [S32] https://github.com/mieweb/wikiGDrive
- Repo licence/last-push data: GitHub API queried 2026-11-05; source-level claims verified in shallow clones of md-gdoc, gdocs-md-cli, gogcli, google-docs-to-markdown.
