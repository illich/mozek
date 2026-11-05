# Round 3 notes: Drive/Docs change detection under drive.file (checked 2026-11-05)

Scope: input for `google-drive-grill.md`. "LU" = the page's "Last updated" footer; "disc" = Google API discovery document revision (drive v3 20261102, docs v1 20261028, driveactivity v2 20261103). (inferred) follows from field descriptions but no page states it; (unverified) = third-party only.

## 1. changes.list / getStartPageToken

| Fact | Source |
|---|---|
| A change = "an alteration made to a file's content or metadata" (content, metadata, ACL). ACL changes land in the change log of the owner, of service accounts on the ACL, and of directly affected users. First-open / star appear only in that user's log. | change-overview, LU 2026-09-03 |
| One entry per file at a time; each new change replaces the previous entry with a new ID. The prose says entries record "the user who made the change", but the `Change` resource exposes no actor field (only `fileId, file, removed, time, changeType, driveId`). "Who" is available only via `file.lastModifyingUser` (signed-in users only) and `file.trashingUser`/`trashedTime` (shared drives only). | change-overview; changes resource, disc 20261102 |
| `removed`: "whether the file or shared drive has been removed from this list of changes, for example by deletion or loss of access", i.e. permanent delete, un-share, owner deletion without transfer ⇒ `removed:true`, no `file` object; requires `includeRemoved=true`. `includeCorpusRemovals=true` returns `file` even for removed entries if still accessible. | changes.list, LU 2026-07-07 |
| Trash/untrash ⇒ ordinary entry with `file.trashed=true/false` (`explicitlyTrashed` distinguishes direct vs. via parent). Non-owners keep access to trashed files until purged. (inferred) | files resource, LU 2026-07-14 |
| `changeType`: `file` or `drive` (shared-drive membership/name changes; the `drive` object is present only while the user is still a member). | changes resource |
| Shared drives: `driveId` scopes the log to one drive (then change IDs are drive-specific); `includeItemsFromAllDrives=true` + `supportsAllDrives=true` for My Drive + shared drives in one log. `restrictToMyDrive=true` omits shared files not added to My Drive, so leave it false. | changes.list |
| drive.file is an accepted scope for changes.list/getStartPageToken/changes.watch. Visibility under drive.file = files opened with the app, picked via Picker, or created by the app; entries for other files are not returned. Whether a Picker-granted file leaving the app's visible set yields `removed:true` is not documented (unverified). | api-specific-auth, LU 2026-09-03; changes.list |
| Retention: the official text only says `startPageToken`/`nextPageToken`/`newStartPageToken` "doesn't expire". No documented change-log retention window and no documented "token too old" error. Gap. | getStartPageToken, changes.list |

## 2. Push notifications (push guide, LU 2026-09-17)

- Only `files.watch` (one file) and `changes.watch` (the user's whole change log, needs `pageToken`) exist. The body is always empty (`Content-Length: 0`); no payload option; you must call `files.get` / `changes.list`.
- Lifetime: max 86,400 s (1 day) for `files`, 604,800 s (1 week) for `changes`; default 3,600 s if `expiration` is omitted. No auto-renew: call `watch` again with a new channel `id` before expiry (channels overlap). `channels.stop` to end early.
- Endpoint: HTTPS with a valid, trusted certificate (self-signed/mismatched rejected); sender UA `APIs-Google`, honours robots.txt. The current page has **no** domain-verification (Search Console / Cloud Console) requirement.
- `X-Goog-Resource-State`: `sync` (channel created, message #1, may arrive before the watch response), `add` (created/shared), `remove` (deleted/unshared), `update` (metadata), `trash`, `untrash`, all for `files`; `change` for `changes` (the example shows `changed`; the page is inconsistent). `X-Goog-Changed` on `update` only: `content, properties, parents, children, permissions`. So files.watch tells you the event type; changes.watch only says "something changed".
- Delivery: return 200/201/202/204/102; retries with backoff only for 500/502/503/504. No delivery guarantee stated. Watch/stop calls count against quota; deliveries don't. `token` (≤256 chars) is echoed in `X-Goog-Channel-Token` for routing/anti-spoofing.

## 3. Polling via files.get (disc 20261102; files resource LU 2026-07-14)

- `modifiedTime`: "last time the file was modified by anyone". `lastModifyingUser`: only if a signed-in user made the last change.
- `version`: monotonic, "reflects every change made to the file on the server, even those not visible to the user"; may bump on non-content events; not a pure content signal.
- `md5Checksum`, `sha1/sha256Checksum`, `headRevisionId`: binary files only; **not populated for Docs Editors files**. Content-change detection for native Docs: Docs API `revisionId` (§5) or export-and-hash.
- Debounce while typing, the effect of opening or commenting on `modifiedTime`: no official statement found; third-party results also silent. Gap: test empirically.

## 4. Drive Activity API v2 (query LU 2026-02-24; disc 20261103)

- Action types: create, edit, move (`addedParents/removedParents`), rename (`oldTitle/newTitle`), delete (`type`: TRASH, PERMANENT_DELETE), restore (`type`: UNTRASH), permissionChange (`added/removedPermissions`), comment, dlpChange, reference, settingsChange, appliedLabelChange. `Edit` is "an empty message", no diff.
- Actors: `user.knownUser{personName: "people/ACCOUNT_ID", isCurrentUser}` (name/email needs the People API and its scope), `user.deletedUser`, `user.unknownUser`, `anonymous`, `impersonation` (admin acting as user; admin identity not available), `system` (USER_DELETION, TRASH_AUTO_PURGE), `administrator`. So yes, it names the editor, as a People ID.
- Visibility: only activity visible to the authenticated user per the item's sharing; a user who lost access sees only the period they had access (Drive Activity intro page).
- Scopes: only `drive.activity` / `drive.activity.readonly`, both **restricted** (restricted-scope verification; security assessment if data is stored server-side). drive.file does not authorize this API; adding the Activity scope forfeits the "non-sensitive only" posture (api-specific-auth, LU 2026-09-03).
- Quotas: /activity/v2/limits returns 404; no public figure found. Check Cloud Console per-project quotas. Gap.

## 5. Revisions (manage-revisions LU 2026-09-03; revisions resource disc 20261102)

- Docs/Sheets/Slides revisions "may be merged"; `revisions.list` may omit older ones for frequently edited files; not every save is a revision. Docs revisions can't be deleted via the API. Listing needs the owner/organizer/fileOrganizer/writer role.
- `keepForever` (max 200, 30-day purge otherwise) is binary-only. `headRevisionId` binary-only.
- Past-revision export: `revisions.get?fields=exportLinks` (Docs Editors files), or `files.download` with `revisionId` + `mimeType` (long-running operation; `revisionId` allowed for blobs, Docs, Sheets only; operations valid 24 h). `revisions.get?alt=media` is blob-only. Whether `text/markdown` works for a past revision: unverified.
- Docs API `revisionId`: opaque, edit access only, "only guaranteed valid for 24 hours", not shareable across users; unchanged ⇒ document unchanged; changed "usually" means edited but can be format churn (documents resource, disc 20261028).

## 6. Docs-specific

- `files.export` mimeType `text/markdown` is official (ref-export-formats LU 2026-09-03); 10 MB cap, no Range. The official docs say nothing about suggestions/comments/tabs in the Markdown export; third-party MCP/skill pages claim comments and suggestions are excluded (unverified); tab coverage unknown.
- `documents.get` (LU 2026-09-30): `includeTabsContent=true` populates `tabs[]` (recursive `childTabs`); the default returns the first tab only. `suggestionsViewMode`: DEFAULT_FOR_CURRENT_ACCESS (view-only ⇒ PREVIEW_WITHOUT_SUGGESTIONS, else SUGGESTIONS_INLINE), SUGGESTIONS_INLINE, PREVIEW_SUGGESTIONS_ACCEPTED, PREVIEW_WITHOUT_SUGGESTIONS. `commentsViewMode=COMMENTS_VIEW_MODE_INCLUDED` populates `comments[]`/`suggestions[]`/`commentAnchors`, requires `includeTabsContent=true` and `suggestionsViewMode=SUGGESTIONS_INLINE`. Scopes include drive.file. This is sufficient to build your own Markdown (body, headers/footers, footnotes, lists, inlineObjects, namedRanges).
- No official Docs-JSON→Markdown sample. Closest official-org project: `googleworkspace/google-docs-hast` (JSON→HTML AST). Community converters exist (unverified quality).

## 7. Deletion specifics and quota costs

| Case | What the app sees |
|---|---|
| Trash (owner/organizer/fileOrganizer only; 30-day auto-purge; shared-drive calls need `supportsAllDrives`) | change entry, `trashed=true`, file still readable/exportable (delete guide LU 2026-09-03) |
| Permanent delete (`files.delete`, `emptyTrash`, purge) | `removed:true`, no `file`; all sharees lose access |
| Un-share | `removed:true` (loss of access); files.watch `remove` |
| Workspace user deleted | The admin may transfer ownership at deletion; transfer doesn't change sharing; Trash is never transferred; the previous owner's access ends. The account is restorable for up to 20 days, then "the data is gone". Untransferred non-trash files: the KB page doesn't spell it out (knowledge.workspace.google.com, LU 2026-10-26). The Activity API reports `system.USER_DELETION`. |
| Service account | drive.file covers only files opened/picked/created by the app, not files merely shared to the SA ⇒ an SA that watches shared files needs `drive.readonly`/`drive` (restricted) (inferred). An SA on the ACL receives ACL changes in its own change log (change-overview). |

Quota units (limits page LU 2026-10-30; the new model applies to projects created on or after 2026-05-01, older projects keep prior quotas): 1,000,000 units/min/project, 325,000 units/min/user; 400 M units/day billing threshold (charges planned later in 2026, ≥90 days notice). Costs: `files.get` metadata 5; `changes.getStartPageToken` 5; `changes.list` 100; `files.list` 100; `changes.watch`/`files.watch` 100; `files.export` 200 (also `files.get?alt=media`, `files.download`); `files.update` 50. Errors 403 "User rate limit exceeded" / 429 ⇒ exponential backoff.

## Practical reading

- Under drive.file alone: `changes.watch` (1-week channel) → `changes.list` with `includeRemoved=true, includeItemsFromAllDrives=true, supportsAllDrives=true` gives edit/rename/move/trash/untrash/removal with timestamps; attribution only via `lastModifyingUser`. Rename/move/un-share actors and the trash-vs-delete distinction need the restricted Activity API.
- For native Docs content change: compare the Docs `revisionId` (editor access) or hash the `documents.get`/`files.export` output; `version` and `modifiedTime` are the cheap pre-filters.

## Sources

- https://developers.google.com/workspace/drive/api/guides/manage-changes (LU 2026-09-03)
- https://developers.google.com/workspace/drive/api/guides/change-overview (LU 2026-09-03)
- https://developers.google.com/workspace/drive/api/reference/rest/v3/changes/list (LU 2026-07-07)
- https://developers.google.com/workspace/drive/api/reference/rest/v3/changes ; .../changes/getStartPageToken
- https://developers.google.com/workspace/drive/api/guides/push (LU 2026-09-17)
- https://developers.google.com/workspace/drive/api/reference/rest/v3/files (LU 2026-07-14)
- https://developers.google.com/workspace/drive/api/reference/rest/v3/revisions ; .../guides/manage-revisions (LU 2026-09-03)
- https://developers.google.com/workspace/drive/api/guides/manage-downloads (LU 2026-09-03) ; .../guides/ref-export-formats (LU 2026-09-03)
- https://developers.google.com/workspace/drive/api/guides/delete (LU 2026-09-03)
- https://developers.google.com/workspace/drive/api/guides/limits (LU 2026-10-30)
- https://developers.google.com/workspace/drive/api/guides/api-specific-auth (LU 2026-09-03) ; https://developers.google.com/identity/protocols/oauth2/scopes
- https://developers.google.com/workspace/drive/activity ; .../activity/v2/reference/rest/v2/activity/query (LU 2026-02-24) ; .../activity/delete
- https://developers.google.com/workspace/docs/api/reference/rest/v1/documents/get (LU 2026-09-30) ; .../rest/v1/documents ; https://developers.google.com/workspace/docs/api/how-tos/tabs (LU 2026-09-03)
- Discovery docs: https://www.googleapis.com/discovery/v1/apis/drive/v3/rest (rev 20261102), https://docs.googleapis.com/$discovery/rest?version=v1 (rev 20261028), https://driveactivity.googleapis.com/$discovery/rest?version=v2 (rev 20261103)
- https://knowledge.workspace.google.com/admin/drive/transfer-drive-files-to-a-new-owner-as-an-admin (LU 2026-10-26) ; https://knowledge.workspace.google.com/admin/users/restore-a-recently-deleted-user (LU 2026-10-26)
- https://workspaceupdates.googleblog.com/2024/07/import-and-export-markdown-in-google-docs.html (2024-07-16)
- Third-party (unverified): https://glama.ai/mcp/servers/A1-x-Tech/mcp-google-docs ; https://github.com/googleworkspace/google-docs-hast
