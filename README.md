# SecretCsv

Copyright (c) 2026 edwest19

Genetated by Claude.

**Status:** pre-build / spec-first · **License:** MIT · **Platform:** Windows (WinUI 3 / .NET 10, packaged MSIX)

> This README is written as a **spec**, not just documentation. Any AI (including future Claude sessions) or human contributor should be able to read this file, look at the code in this repo, and answer a binary question for every rule below: **does the code match the spec, yes or no.** If code and README ever disagree, that's a bug — fix the code to match the README, or open an issue proposing the README change. Never silently drift.

---

## 0. One-paragraph purpose

SecretCsv is a **standalone, offline, tag-file-driven CRUD and CSV-import tool for Windows.** The owner writes a tag file describing any kind of record. 
SecretCsv either imports a matching CSV in bulk or lets records be keyed in by hand, into a store that lives entirely in the app's local AppData and never touches a network. 
The pitch is not "an offline CSV importer for one dataset." It's: **import your data once, and from that point on the app architecturally cannot make a network call, ever, for anything.** 
That claim — not any single bundled dataset — is the actual product.

---

## 1. Relationship to other Secret* apps

- **SecretList** — a separate app, separate repo, **zero shared code**. SecretList is the hand-entry-only tool: the owner types records in one at a time. SecretCsv does **not** feed SecretList, replace it, or depend on it. The two apps share only a conceptual approach (tag-file-driven, human-readable tagged-text storage) — nothing else.
- **SecretLetter** — a third, fully independent app (rough concept only, unstarted). Not a feature of SecretCsv, not integrated with it at the code level.

If any prior document (including older revisions of this README, or older revisions of any tag file) used the word "schema," described a folder-picker-driven import, or described SecretCsv as feeding SecretList or as a Contractor-only pre-populated database — that description is **superseded**. This file is the current spec.

---

## 2. Platform rules — non-negotiable

- **WinUI 3 / Windows App SDK, .NET 10, packaged MSIX.**
- **No third-party NuGet packages** — .NET BCL and Windows App SDK / COM only.
- **No network calls, ever.** No scraping, no telemetry, no sync, no update-check pings, no "phone home" of any kind. This is architectural, not a setting — there should be no code path in the app capable of opening a socket.
- **Offline-only.** Everything the app needs (tag files, records) lives on local disk.
- **No MSIX sandbox workarounds of any kind.** Read-only packaged assets via `Package.Current.InstalledLocation`; no attempts to write to the install folder; no elevation tricks.
- **No prefilled real-world data ships with the installer, and no sample data ships with the installer at all.** See §11 — the sample dataset lives as plain files in the repo, not the MSIX, and is fictional only.
- If a feature seems to need a workaround to function, the answer is to find the correct platform-supported approach — not to bypass a rule above.

---

## 3. The Import screen

There is no folder picker on Import, and no "folder is a project" model. That's the rule this section exists to enforce — it's about how files get *into* AppData **without being interpreted/parsed as new data**, not a blanket ban on folder pickers anywhere in the app. Backup and Restore (§14) both use folder pickers — Backup to choose where files get copied *out* to, Restore to choose a backup folder to copy *back in* from — but neither one runs picked contents through Import's parsing/validation; Restore just copies known files back byte-for-byte (§15). Import works from **two independent file selections**, each its own file picker:

- **CSV file** — the data to import. Any filename, any location.
- **Tag file** — the `.md` file describing the entity's tags (§4). Any filename, any location. It does not need to sit next to the CSV, and it does not need to be named after the entity — though the sample tag file (`construction.md`, kept in the repo — §11) follows that convention anyway.

**Validation runs automatically the moment both files are selected** — there's no separate "check" step. SecretCsv matches every tag against the CSV's column headers, both directions, exactly as described in §9.

**The Import button's enabled state *is* the validation result** (SecretLead binds `IsEnabled` directly to the validation result and uses WinUI's standard `AccentButtonStyle` — there's no hardcoded color, so the exact shade follows the system/app accent color, not literally green):
- Disabled, shown grayed-out via the platform's normal disabled-button styling — one or both files not yet selected, or they don't match.
- **Enabled**, shown in the accent color — both files selected and every tag matches a CSV column with nothing left over in either direction.

**A status area** holds validation messages and errors — reached via a button on the left (nav rail), not a box that's permanently visible on the Import screen. Same content rule as always: every mismatch is named explicitly (§9), never summarized.

**What happens when the enabled Import button is clicked:**

1. SecretCsv reads the entity name from the tag file's `##` line (e.g. `## Construction`).
2. The tag file itself is copied into AppData alongside the entity's records (§5) — so the app always has its own reference copy of the tag file that produced `<EntityName>.txt`, independent of wherever the original file came from or whether it still exists afterward.
3. The CSV is converted into records per the tag file's rules (§4 role/split rules, §8 normalization) and written to `<EntityName>.txt` in AppData.
4. If `<EntityName>.txt` already exists, the new records are **appended** — nothing already stored is deleted or overwritten (§6).

Nothing outside AppData is ever written. The original CSV and tag file, wherever the owner picked them from, are only ever read, never modified or moved.

---

## 4. Tag file grammar

A tag file is plain Markdown, parsed as follows:

- `#` — title, a short description of the entity, version, date (informational; not machine-enforced beyond parsing past it)
- `##` — **entity name.** This is also the name of the AppData output file: `## Contractor` → `Contractor.txt`. `## Construction` → `Construction.txt`.
- `### TAG X` — a tag name, matched **case-insensitively** against CSV column headers when importing. Any column order is fine.

**Role rule (applies to every tag file):**
- The **first** `### TAG` plays the **Category** role — top-level grouping, and the field that gets exploded into multiple records when its value is semicolon-delimited on import (one input row → one output record per semicolon-separated value, all other fields identical).
- The **second** `### TAG` plays the **Nickname** role — the searchable identifying field used for browsing.
- Tags beyond position 2 can be declared in any order. CSV columns can be in any order too, matched by name.

That's the entire grammar: a name-matcher, two positional roles, and a filename — not a schema in the type/constraint sense.

---

## 5. Storage: AppData, entity-name-keyed, global

Everything SecretCsv writes lives in the app's sandboxed AppData storage — **GUI-only**, never a location the user browses to or hand-edits directly. Each entity is a **pair** of files:

- `<EntityName>.md` — the reference copy of the tag file that produced this entity, copied in at Import time (§3).
- `<EntityName>.txt` — the actual records.

- **The entity name is the only key.** There is no per-source, per-CSV, or per-import-session isolation. If two different imports both declare `## Contractor`, they read and write the **same** `Contractor.txt` (and update the same `Contractor.md`).
- Keeping the tag file's own copy in AppData means an entity stays fully self-describing even if the original tag file the owner picked gets renamed, moved, or deleted afterward.
- **"Sandboxed" is now literal, not aspirational (§14, §16).** `EntityStore` resolves this folder via `Windows.Storage.ApplicationData.Current.LocalFolder` — real package-scoped storage, not a plain hardcoded path like SecretLead used (SecretLead couldn't use this; it ships unpackaged, and this API throws without package identity). Decided by edwest19: uninstalling SecretCsv should leave nothing behind, and package-scoped storage is what Windows actually removes on uninstall. **Confirmed working via a real run (2026-08-18, chat history)** — edwest19 ran the app via `dotnet run`, imported the sample dataset (§11), and it wrote to and read back from `ApplicationData.Current.LocalFolder` successfully: 4 records landed correctly, category navigation and search worked. Not yet separately confirmed running as the actual installed signed MSIX (a related but distinct code path) — see §14.

---

## 6. Existence-of-file is the mechanic

SecretCsv's home/browse screen lists whatever entities currently exist in AppData — discovered by the `<EntityName>.txt` files present there, not by any folder the owner has open.

| State | Behavior |
|---|---|
| First run, AppData has no entities yet | **AppData starts empty. Nothing is auto-seeded — this is a deliberate decision, not a gap.** The browse screen shows its normal empty state until the owner runs Import at least once. The Construction sample dataset (§11) exists so a first-time user has something real to try Import against, not to pre-populate their data store before they've chosen to — but it lives in the repo, not the app, so trying it is an explicit, separate step (download or clone, then Import). |
| Imported tag file names an entity with **no** existing `<EntityName>.txt` | Import creates `<EntityName>.md` and `<EntityName>.txt` fresh. |
| Imported tag file names an entity that **already exists** | Import appends the new records into the existing `<EntityName>.txt` and updates the stored `<EntityName>.md` copy. Existing records are never deleted or overwritten. |

**Still open, not decided (see §14):** whether SecretCsv should compare the incoming tag file's tag set against the entity's already-stored `<EntityName>.md` before appending — flagging a mismatch as a possible different dataset reusing the same entity name — or continue relying purely on entity-name-as-key with no structural check.

---

## 7. CRUD + Import, together

Against whatever entity is selected, SecretCsv supports both:

- **Manual CRUD** — Add / Edit / Delete records by hand, reached from the browse/list screen.
- **CSV Import** — bulk-convert a matching CSV into records, reached from the Import screen (§3) at any time — not gated behind any prior folder or file selection.

Same list/detail/print interaction model either way. This combination — any tag file, plus both hand-entry and bulk import, against one generic AppData store — is the actual generalization. It is not just "CSV import extended to more entity types."

---

## 8. Field normalization

Inferred purely from **tag name substring** (case-insensitive), no separate datatype field:

| Tag name contains | Rule |
|---|---|
| `date` | Reformat to `mm/dd/yyyy` regardless of source format |
| `email` | No validation or transform — stored as-is |
| `phone` | Strip to digits, then reformat by digit count: 10 digits → `(631) 555-1212`; 11 digits → `+1 (631) 555-1212`; 7 digits → `555-1212`; anything else → left as-is |

**Tag-file-specific exception:** Business Name falling back to `Last Name, First Name` when blank is specific to the sample **Construction** tag file (§11) — it only makes sense where those three tags exist, and is **not** a generalized rule other tag files inherit automatically.

---

## 9. Import validation

Hard stop, both directions, no silent skipping — this is what the Import button's enabled/disabled state (§3) reflects:

- Any tag with no matching CSV column → blocks import (button stays disabled).
- Any CSV column with no matching tag → blocks import (button stays disabled).
- Both cases are named explicitly, by name, in the status area (§3) — reached via the left nav button, not a screen the owner has to scroll past.

This logic is fully generalized — it never depended on Contractor/Construction specifically and needs no per-entity change.

---

## 10. Print layout

A4 portrait, two records per page, duplex, folded into an A5 handout. No requirement for front/back alignment between the two records on a sheet. This is a structural layout mechanic — whatever tags a tag file declares get printed — not something tied to contractor data specifically.

---

## 11. Sample dataset: the Construction demo

SecretCsv doesn't bundle any sample data in the installer or the MSIX package — nothing about this dataset ships with the app at all. Instead, `construction.md` (the tag file) and `construction.csv` (matching fictional data) live as plain files in this repo, so anyone can grab them, point Import's two file pickers (§3) at them, and see the whole flow work without owning real data of their own yet.

- `construction.md` — the tag file. Entity name `## Construction` (12 tags: Category, Business Name, License ID, Last Name, First Name, Address, Phone, Email, Status, Date Issued, Date Expire, Website). Exists today in SecretLead's `Assets\` folder; needs to move into this repo's own sample-data location once that's set up (§14).
- `construction.csv` — **fictional sample data**, matching `construction.md`'s tags exactly (real CSV headers, not internal storage format — see note below). Reserved `555` phone numbers, `example.com` email/website domain. No real business names, addresses, phone numbers, or license numbers, ever. **Status: does not exist yet and needs to be created — see §14.**

**Note on the format:** this is a genuine `.csv` with column headers, not SecretCsv's internal `:tag.TagName.Value` storage format — it goes through the real Import screen and gets validated like any other import, the same as a real owner's data would.

The real, hand-verified `contractor.csv` and the source `.xlsx` county exports it's built from are **strictly development-time references** — used to ground this spec in real data quirks, never bundled, never shipped, never even added to this repo's sample-data location.

The intent of this demo is explicit: show Import working end-to-end against real-shaped construction-trade data, then show that running Import with a different tag file adds a completely different, independent entity alongside it. SecretCsv is a strict superset of what a Contractor-only tool would do — everything that design did is still possible here, it's just no longer the *only* thing possible.

---

## 12. Verification checklist (pass/fail)

An AI or human auditing this repo should be able to check each line below against the actual code:

- [ ] App makes zero network calls under any code path — no `HttpClient`, no sockets, nothing reachable from any menu or background timer.
- [ ] Zero third-party NuGet package references in any `.csproj`.
- [ ] Import screen has two independent file pickers (CSV, tag file) — no folder picker on Import.
- [ ] The two files can be selected from different locations with unrelated filenames and import still works.
- [ ] Import button's enabled/color state updates automatically the instant both files are selected — no separate "validate" action required.
- [ ] Status/error area is reached via a left-side button, not a permanently visible box.
- [ ] Successful import copies both `<EntityName>.md` and `<EntityName>.txt` into AppData — never anywhere else.
- [ ] Original CSV and tag file, wherever picked from, are left untouched — read-only access.
- [ ] Record store and tag-file-copy files live only under the app's sandboxed AppData path.
- [ ] `<EntityName>.txt` / `<EntityName>.md` filenames exactly match the `##` entity name from the imported tag file.
- [ ] Importing into an entity that already exists appends records rather than overwriting.
- [ ] First run with empty AppData shows the normal empty state — nothing is auto-seeded. The bundled Construction sample (§11) can be found and imported by the owner through the normal Import screen without any special-cased seeding code path.
- [ ] Import validates tags vs. CSV columns in both directions; mismatches are named explicitly in the status area.
- [ ] Semicolon-delimited values in the Category-role tag explode into one record per value on import.
- [ ] `date`-containing tags normalize to `mm/dd/yyyy`; `phone`-containing tags normalize by digit count per §8; `email`-containing tags are untouched.
- [ ] Business Name blank-fallback logic exists only in code paths specific to the Construction tag file, not as a generic rule.
- [ ] Print output is A4 portrait, duplex, two records per page.
- [ ] No installer asset contains real contractor names, addresses, phone numbers, emails, or license numbers.
- [ ] No MSIX sandbox workaround exists anywhere (no install-folder writes, no elevation).
- [ ] No use of the word "schema" anywhere in code, comments, or docs — "tag file" and "description" only.

Any unchecked item is either a bug to fix or a signal this README is stale and needs updating — treat divergence as a defect in one document or the other, never as acceptable drift.

---

## 13. Instructions for future AI sessions

If you are an AI (Claude or otherwise) opening this repo cold:

1. **This README is authoritative.** Where code and README disagree, that is a bug. Prefer fixing the code; if the README itself is what's out of date, say so explicitly and propose the edit rather than silently working around it.
2. **Never use the word "schema."** Call the file a **tag file** and call its contents a **description**. If you find "schema" anywhere in this repo, it's leftover wording from before the rename — fix it, don't propagate it.
3. **Don't reintroduce the folder-as-project model.** There is no folder picker on Import — it uses two independent file pickers (§3); that supersedes any earlier design where a single picked folder held both files. This is not a blanket ban on folder pickers anywhere in the app, and it's not even a blanket ban on reading a picked folder's contents back in — Restore (§14) does exactly that. The actual rule is narrower: never run a picked folder's contents through Import's *interpretation* (parsing a CSV/tag-file pair as new data) without the owner explicitly picking each file via Import's two pickers. Restore doesn't interpret anything — it copies the app's own previously-exported files back byte-for-byte, so it doesn't reintroduce the risk this rule guards against.
4. **Don't reintroduce the old Contractor-only, pre-populated-database design.** That model is superseded (§0, §1). SecretCsv does not ship real contractor data, does not vet licenses at runtime, and is not scoped to contractors at all — Construction is a sample demo entity, with its files living in this repo rather than the installer, not the product.
5. **Don't add network calls, sync, telemetry, or NuGet dependencies** to "make something easier." If a task seems to need one, stop and flag it instead of implementing a workaround.
6. **Treat §12 as a literal checklist** before considering any implementation task complete.
7. **Flag open decisions honestly** rather than resolving them silently — see §14.

---

## 14. Open items — tracked honestly

- [x] **Decided: no first-run auto-seeding into AppData (§6, §11).** AppData starts empty; the owner runs the bundled Construction sample through the normal Import screen themselves if they want it. This reverses the earlier SecretLead-derived draft of §6/§11, which described automatic seeding — that behavior was never actually built in SecretLead anyway, and is now explicitly not wanted.
- [x] **Decided: sample dataset lives in the repo, not the app (§11).** `construction.md`/`construction.csv` sit as plain files in this repo (a `SampleData\` folder — see the open item below on `SampleData/README.md`), not bundled in the MSIX and not attached to GitHub Releases. No in-app delivery code, no `Package.Current.InstalledLocation` reads, no changes to `build-and-release.yml`. Anyone who wants to try Import grabs the files from the repo directly.
- [x] `construction.csv` created in `SampleData\` — real CSV with headers matching `construction.md`'s tags (§11), fictional data only. Three rows, chosen to exercise the Category-explode rule (§4), the Business Name blank-fallback (§8), and all three phone-normalization branches (§8).
- [x] `construction.md` added to `SampleData\` (ported from SecretLead's `Assets\construction.md`, header prose rewritten to match the no-auto-seed decision — §11).
- [x] **Decided: real packaged `ApplicationData.Current`, not a plain hardcoded folder (§5, §16).** `EntityStore` uses `Windows.Storage.ApplicationData.Current.LocalFolder`, not SecretLead's plain `%LOCALAPPDATA%\SecretLead\`-style path. Decision from edwest19: uninstalling SecretCsv should leave nothing behind — no leftover folder, no manual cleanup — and package-scoped storage is what Windows actually removes on uninstall; a plain hardcoded folder wouldn't be. Whether package-scoped AppData also participates in Windows Backup / "keep apps and data" during a PC reset or transfer is a separate, still-open question — not promised anywhere in this README, and not something to claim without checking.
- [x] **Confirmed against a real run, via `dotnet run`.** edwest19 ran the app, imported the sample Construction dataset (§11) through the real Import screen, and it round-tripped correctly through `ApplicationData.Current.LocalFolder`: 4 records (3 CSV rows, one exploded by the semicolon-delimited Category rule — §4), correct entity name, working category navigation and search.
- [ ] **Not yet separately confirmed running as the actual installed, signed MSIX** (as opposed to `dotnet run`'s debug package identity) — a related but distinct code path. Worth a quick check once you package and install it, since that's the real end-state anyway.
- [x] `scripts\Reset-SecretCsvAppData.ps1` written for real (§16). Reads the Identity Name out of `Package.appxmanifest` at runtime, finds the matching installed package via `Get-AppxPackage`, and resolves `%LOCALAPPDATA%\Packages\<PackageFamilyName>\LocalState\` from that — no hardcoded path, no DisplayName-matching. Deletes only `.md`/`.txt` entity pairs (narrower than Backup's full-folder copy — that's precisely what "first-run state" means per §6's existence-of-file mechanic). Works against both the installed signed MSIX and `dotnet run`'s debug package identity, since both register under the same Identity Name.
- [ ] Whether SecretCsv should validate an incoming tag file's tag set against an existing entity's stored `<EntityName>.md` before appending records, or continue relying purely on entity-name-as-key (§6).
- [x] **Backup out of AppData, and Restore back in — the full cycle.** Built: the Backup button (`MainPageViewModel.BackupAsync`) opens a folder picker and copies **everything** in AppData, recursively, into the chosen folder — not just `<EntityName>.txt`/`.md` pairs, so a backup can't silently miss something added to AppData later. The Restore button (`MainPageViewModel.RestoreAsync`) does the inverse: picks a folder and copies its entire contents back into AppData, overwriting on conflict, additive only (never deletes anything not in the source — same philosophy as records never being deleted by Import, §6). Together they support: back up, uninstall, reinstall, restore — "the whole kit and kaboodle," per edwest19. This is a deliberate, explicit amendment to the original "Backup copies out, never back in" rule (§3, §13.3, §15) — see those sections for why Restore doesn't reintroduce the folder-as-project risk that rule existed to prevent. Precedent for having Backup at all: SecretList's `records.txt` was lost with no recoverable trace after a Windows feature update.
- [x] **Nav rail reorganized: Backup/Restore moved to the bottom, Status/Exit moved up (owner's explicit layout call).** The main Actions group is now Import, Browse, Status, Exit, in that order — normal day-to-day use. Backup and Restore are pinned to the bottom of the rail as a separate "Data" group, since they're infrequent/administrative rather than normal operation.
- [ ] **Restore's glyph (`&#xE895;`) hasn't been visually double-checked against Segoe Fluent Icons' actual mapping** — picked as a reasonable "download/bring back" icon but not verified with certainty. Worth a glance once you run the app; swap it if it looks wrong.
- [ ] **Atomic write-then-swap on every save.** Still not built for either app — the one-click backup above is a manual safety net, not protection against a crash mid-write corrupting the live `.txt`/`.md` files.
- [ ] Splitting the bundled `Construction` demo tag file into separate real-usage tag files (`Construction`/general Home Improvement, `Electrician`, `Plumber`) once real data replaces the demo — future intent, not yet spec'd.
- [ ] SecretLetter's actual design — input format, output format, whether it reads another Secret-family app's `.txt` file or is fully self-contained.
- [ ] Whether any disclaimer mechanism is needed for arbitrary user-defined tag files, or only ever for a licensing-specific demo/dataset.
- [ ] `SampleData/README.md` — walkthrough of exporting source `.xlsx` data and converting it to CSV, for anyone building their own real dataset against the Construction tag file.
- [ ] Legal review of any disclaimer language, before any public sale.

---

## 15. Non-negotiables (summary)

- No third-party NuGet packages — .NET BCL and Windows App SDK / COM only
- No network calls, no scraping, no automated querying of any site, ever
- Offline-only, no cloud/sync/telemetry
- No MSIX sandbox workarounds — read-only packaged assets, no install-folder writes, no elevation tricks
- No real-world personal/business data ships with the installer
- A human reviews imported records before trusting them for anything
- No use of the word "schema" — tag file / description only
- No folder picker on Import — it uses two independent file pickers (§3). Backup and Restore (§14) are the two exceptions: Backup opens a folder picker to choose where AppData's contents get copied *out* to; Restore opens one to choose a backup folder to copy contents back *in* from. Restore is allowed to read a picked folder's contents back in — unlike Import, it never interprets/parses what it copies, it just restores the app's own previously-exported files byte-for-byte, so it doesn't reintroduce the risk the original folder-as-project rule (§3, §13.3) was written to prevent.

---

## 16. Developer scripts

Not part of the shipped app itself — tooling for maintainers and end users to run manually from a terminal. Doesn't touch or relax any rule in §2; the app binary still makes zero network calls and has zero NuGet dependencies. These scripts do.

- `scripts\Reset-SecretCsvAppData.ps1` — deletes every stored entity (`<EntityName>.md` / `<EntityName>.txt`) from AppData, resetting SecretCsv to first-run state — existence-of-file is the whole mechanism (§6), so this is a complete, clean reset for what "first-run" actually means, without touching anything else that might ever live in AppData (Backup, §14, is the tool for a full-folder copy). **Folder-resolution mechanism:** matches `EntityStore` exactly — reads the Identity Name out of `Package.appxmanifest` at runtime, finds the matching installed package via `Get-AppxPackage`, and resolves `%LOCALAPPDATA%\Packages\<PackageFamilyName>\LocalState\` from that. No hardcoded path, no DisplayName-matching (an earlier draft of this README incorrectly described DisplayName-matching; that was never built and never accurate). Works against both the installed signed MSIX and `dotnet run`'s debug-registered package identity, since `Microsoft.Windows.SDK.BuildTools.WinApp` registers the debug identity under the same Identity Name. Prompts for confirmation by default (supports `-WhatIf`/`-Confirm:$false`); pass `-Force` to skip the prompt. If Windows blocks it as a downloaded script ("running scripts is disabled on this system"), right-click the file → Properties → Unblock, or run `powershell -ExecutionPolicy Bypass -File .\scripts\Reset-SecretCsvAppData.ps1`.
