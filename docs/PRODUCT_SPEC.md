# Product Spec — File Organizer (Phase 0)

Overview
- Product: File Organizer — a privacy-first, Windows-native (WinUI 3) desktop app that helps users organize files by rules with strong safety features (dry-run, undo), duplicate detection, and advanced metadata rules.
- Target users: power users, office workers, sysadmins, home users.
- Scope: single-user desktop app (no cloud sync in MVP).
- Target Windows versions: Windows 10 (21H2+) and Windows 11.
- Monetization: offline-friendly one-time Pro purchase; 14-day trial.

Primary goals
- Give users a safe, predictable, and fast way to reorganize files via rules.
- Prevent data loss with preview, undo, and conservative defaults.
- Provide powerful features for power users (advanced rules, duplicate detection, metadata rules) while retaining a clean modern UI for non-technical users.

MVP (must-have) — prioritized
1. Rule engine (core)
   - Match types: extension, filename wildcard, regex, created/modified date ranges, file size ranges, metadata (EXIF, ID3, PDF metadata, Office doc properties).
   - Actions: move, copy, rename, group into folders (e.g., Photos/YYYY/MM).
   - Conflict handling: rename-with-suffix, skip, overwrite, prompt.
2. Preview / Dry-run
   - Simulated actions and simulated target tree; show counts and estimated disk usage.
3. Undo / Safe rollback
   - Recycle Bin integration or local quarantine; metadata to revert last N operations.
4. Operation logging & history
   - Per-run logs, exportable (CSV/JSON), searchable.
5. Duplicate detection (added)
   - Hash-based duplicate grouping with configurable algorithm (SHA-256 default; faster option like xxHash).
   - Default behavior: report-only; on user command, move duplicates to user-specified dedupe folder.
6. Advanced metadata rules (added)
   - EXIF (images), ID3 (audio), PDF metadata, Office document properties (title, author, tags).
7. Scheduling & manual run
   - Manual rule runs and simple scheduler (internal scheduler + optional Task Scheduler registration).
8. Performance & scanning
   - SQLite metadata cache for incremental scans, include/exclude folders, scan depth control.
9. Rule test tooling
   - Run test against sample files and show matched examples in UI.
10. UI
   - Clean, modern, friendly WinUI 3 UI with dark/light theme, sidebar (Rules, Scheduler, History, Settings), simple and advanced rule editors.

Nice-to-have (post-MVP)
- Watch mode (USN Journal or FileSystemWatcher)
- Explorer context-menu integration
- Tagging/virtual overlay
- CLI for automation (useful for sysadmins) — consider adding to MVP if you want
- ML-assisted rule suggestions
- Cloud integrations

User personas (summary)
- Power user: wants fast bulk operations, complex rules, keyboard shortcuts, and CLI. Values speed and precision.
- Office worker: needs easy templates (invoice/client folder), safe defaults, undo, and scheduling.
- Sysadmin: needs CLI/scripting, predictable non-interactive runs, logging for audits.
- Home user: wants simple wizards, safe preview, and one-click rules for Photos/Documents.

User stories + acceptance criteria (selected, prioritized)

1) As a user, I can create a simple rule that moves files by extension into a target folder.
   - Acceptance:
     - Rule UI accepts extension filter (e.g., .jpg) and target folder.
     - Dry-run shows matched files and proposed move actions.
     - Apply moves files; history shows before/after entries.

2) As a user, I can create an advanced rule using regex and metadata (EXIF date).
   - Acceptance:
     - Rule editor validates regex and metadata field selection.
     - Dry-run shows matches based on EXIF date (e.g., images from 2020).
     - Apply moves files into YYYY/MM folders based on EXIF.

3) As a user, I can preview all proposed changes without altering files.
   - Acceptance:
     - Dry-run lists every file that would be changed with proposed destination.
     - No file system changes occur during preview.

4) As a user, I can undo the last apply operation.
   - Acceptance:
     - Undo restores files to original paths in 95% of sandbox tests (document exceptions).
     - Undo uses metadata logs (pre/post path, timestamps) and integrates with Recycle Bin/quarantine.

5) As a user, I can detect duplicates and choose to move duplicates to a dedupe folder.
   - Acceptance:
     - Duplicate detection groups identical files by hash.
     - Default detect operation reports groups.
     - “Move duplicates” action moves duplicates (user-chosen) to a folder; original master can be kept or chosen by user.

6) As a user, I can schedule a rule to run daily at a given time.
   - Acceptance:
     - Scheduler persists rules and next-run times.
     - Scheduled run can be registered with Task Scheduler (user opt-in).
     - Scheduled run logs created with same undo metadata.

7) As a user, the app respects file locks and fails gracefully.
   - Acceptance:
     - If a file is locked, the run logs the item as skipped with reason; user can retry.

8) As an admin/sysadmin, I can run a saved rule from a CLI script (optional in MVP).
   - Acceptance:
     - CLI invocation triggers a saved rule and returns an exit code indicating success/failure and generates logs.

Non-functional requirements
- Privacy: No telemetry or automatic uploads by default.
- Offline capability: License validation must work offline (signed keys).
- Performance: On a 100k-file corpus, initial scan + dry-run completes within the target you set (example target: <10 minutes) on typical SSD-equipped systems; incremental scans significantly faster.
- Reliability: Undo should work for the vast majority of operations; logs must be complete.
- Compatibility: Windows 10 (21H2+) and Windows 11.
- Accessibility: Keyboard navigation and high-contrast support.
- Local storage: Use SQLite for rules, history, and cached metadata.

Success metrics (for MVP)
- Functional: Dry-run accurately predicts >99% of apply actions in tests (exceptions documented).
- Safety: No user data loss incidents in beta group of N users (target N=10–50).
- Performance: Initial scan target met on baseline test machine (document baseline).
- UX: Beta users score the Rule Wizard ≥4/5 for clarity (usability test).
- Licensing: Trial flow activates and expires as expected for 14-day trial in tests.

Example test datasets & cases
- Small sample set (for quick iterative tests):
  - 200 images with EXIF dates spanning 2018–2023, some with same content but different filenames.
  - 100 audio files where 10 pairs are exact binary duplicates (same hash), some with different ID3 tags.
  - 50 PDFs with and without metadata fields populated; some PDFs have same content but different metadata.
  - Office docs (Word/Excel) with custom Document Properties (author, client).
- Edge cases:
  - Files with identical content but different names and metadata (duplicates).
  - Very large files (>4GB), symbolic links, and junctions — ensure skip or special handling.
  - Locked/open files (simulate with an app locking a file).
  - Files with special characters in filenames (Unicode, long paths >260 if supported).
  - Read-only files and files requiring elevation.

Data model (high level)
- SQLite tables (examples):
  - rules: id, name, priority, match_spec (json), action_spec (json), enabled, created_at, updated_at
  - file_cache: path, normalized_path, size, mtime, hash (nullable), metadata_json, last_seen
  - runs: id, rule_id, start_at, end_at, mode (dry-run/apply), status, summary_json
  - run_items: run_id, original_path, action, proposed_path, result (success/failure/skipped), error_message
  - licenses: license_key, issued_to, issue_date, expiry (nullable), signature

Safety & privacy design notes
- Default to dry-run on first run; require explicit “Apply” confirmation.
- Use Recycle Bin or a local quarantine folder for destructive actions (default Recycle Bin).
- Keep all logs and metadata local. No telemetry by default; if optional telemetry is added later, make it opt-in and minimal.

Duplicate detection policy (explicit)
- Default policy: Report-only — produce groups and a clear UI showing candidates and suggested master selection rules (keep newest, keep largest, keep path whitelist).
- “Move duplicates” on user command: move all duplicates (except the chosen master) into a dedupe folder specified by the user. Record undo metadata so this action can be reversed.

Trial policy & licensing
- Trial length: 14 days.
- Trial mechanics: trial starts on first run; store trial start in local obfuscated storage; no server dependence required. Provide clear UI for trial expiration and purchase path.
- Offline license validation: signed license keys validated locally (public-key signature).

Risk & mitigations
- Data loss: Dry-run by default, easy undo, clear confirmations, and thorough test datasets.
- Permission/UAC problems: avoid elevation unless necessary; surface clear guidance for system folders and offer per-operation elevation only when needed.
- Performance: metadata caching, incremental scans, option to skip hashing on initial run.
- Explorer shell complexity: postpone until after MVP.

Phase 0 deliverables (what I will produce or you should approve next)
1. Product Spec (this doc) — confirm or request changes.
2. Architecture doc — component diagram + data flow (engine, UI, SQLite, file op layer, hashing service, scheduler).
3. Rule-engine spec — evaluation order, precedence, sample rule syntax, short-circuit rules, conflict resolution details.
4. Wireframes — Rule Wizard, Preview/Dry-run, History/Undo (sketch-level).
5. Test plan & initial datasets — concrete tests to validate safety and correctness.

Suggested immediate tasks (next iteration)
- Review this Product Spec and confirm or request edits.
- If approved, I’ll commit this as docs/PRODUCT_SPEC.md to your repository.
- After commit, I will produce the Architecture doc (recommended next artifact) or the wireframes — tell me which one you prefer.

Would you like me to:
- A) Commit this Product Spec to docs/PRODUCT_SPEC.md now, or
- B) Update the spec first per any edits you want, or
- C) Skip commit and produce the Architecture doc or Wireframes next?

If you choose A, I’ll commit the file to the repo and then start the Architecture doc.
