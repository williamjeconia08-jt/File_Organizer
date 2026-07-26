# Architecture — File Organizer

Purpose
This document defines the high-level architecture for File Organizer (WinUI 3 desktop). It maps components, data flows, storage, APIs, error handling, performance strategies, and packaging. It is intended to guide implementation work in Phase 1+ and to serve as an engineering reference for testing and QA.

Goals
- Safety-first: prevent data loss via dry-run, transactional/undo metadata, and conservative defaults.
- Modularity: clear separation between UI, rule engine, file-op layer, metadata extractors, hashing/duplicate service, scheduler, and storage.
- Performance: incremental scans, batched IO, and multithreaded hashing when needed.
- Offline-first: licensing and all processing must work locally without network dependencies.

High-level components
1. UI (WinUI 3)
   - Responsibilities: navigation shell, Rule Editor (simple + advanced), Preview/Dry-run view, Run/Apply flow, History, Scheduler UI, Settings, Licensing dialog.
   - Surface: consumes core engine services via a well-defined application service layer (interfaces) and displays results.

2. Application Service Layer (App Services)
   - Responsibilities: thin orchestration layer between UI and core libraries. Exposes operations as async methods: CreateRule, ValidateRule, DryRunRule, ApplyRule, UndoRun, ScheduleRule, DetectDuplicates, ExportLogs, GetHistory.
   - Purpose: keep UI decoupled from business logic and ease unit testing.

3. Core Engine (Rule Evaluator + Planner)
   - Responsibilities:
     - Parse and evaluate rule match specifications against file metadata and file_cache.
     - Produce an ordered actionable plan (PlanItem list) describing operations (move/copy/rename) in dry-run mode.
     - Provide deterministic ordering and conflict detection.
   - Inputs: Rule definitions (JSON), file metadata (from cache or live stat), metadata fields (EXIF/ID3/PDF/Office), duplicate groups (optional).
   - Outputs: Plan (list of PlanItems) with proposed actions, estimated sizes, and risks flagged.

4. File Operation Layer (SafeOps)
   - Responsibilities:
     - Execute PlanItems with safety wrappers: support dry-run (no-op), apply (actual operations), and staged operations (quarantine/Recycle Bin support).
     - Use Windows IFileOperation/MoveFileEx/CopyFileTransacted (where applicable) and fall back to robust reimplementation when APIs unavailable.
     - Emit granular per-item status and errors to run logs.
   - Features:
     - Conflict policies (rename, overwrite, skip, prompt)
     - Integration with Recycle Bin or optional quarantine folder for safer undo
     - Per-file retries/backoff for transient errors (locks)

5. Metadata Extractor Service
   - Responsibilities:
     - Extract EXIF, ID3, PDF metadata, Office document properties, and basic FS metadata (size, mtime, ctime).
     - Normalize metadata into a common JSON schema for rule matching (e.g., metadata.cameraDate, metadata.author).
     - Provide streaming access to metadata for large scans.
   - Note: Use tested libraries (NuGet) to avoid parsing errors; run metadata extraction in worker threads with batching.

6. Hashing / Duplicate Detection Service
   - Responsibilities:
     - Compute file fingerprints (configurable: SHA-256 default; optional faster algorithms like xxHash for preliminary grouping).
     - Support staged hashing: quick-size+mtime filter → partial hash → full hash.
     - Provide grouping APIs and duplicate reports; can return candidate master based on policies (keep-newest, keep-largest, keep-path-whitelist).

7. Storage Layer (SQLite)
   - Responsibilities:
     - Persist rules, file_cache, runs, run_items, and license info.
     - Provide efficient incremental update patterns: last_seen timestamps, per-path entries.
     - Enable queries for UI (e.g., list recent runs, view run details, search history).
   - Suggested schema (examples):
     - rules(id, name, priority, match_spec_json, action_spec_json, enabled, created_at, updated_at)
     - file_cache(path, normalized_path, size, mtime, hash, metadata_json, last_seen)
     - runs(id, rule_id, start_at, end_at, mode, status, summary_json)
     - run_items(id, run_id, original_path, action, proposed_path, result, error, metadata_json)
     - licenses(key, issued_to, issue_date, expiry, signature)

8. Scheduler
   - Responsibilities:
     - Internal lightweight scheduler for saved rules (persist schedules in SQLite).
     - Optional integration with Windows Task Scheduler for runs when user opts in.
     - Ensure scheduled runs run in user context and adhere to offline constraints.

9. CLI (post-MVP)
   - Responsibilities: headless invocation of saved rules, producing JSON logs and exit codes for automation. Implemented after MVP per request.

10. Licensing & Activation
   - Responsibilities:
     - Validate signed license keys locally using public-key verification.
     - Provide trial management (14-day trial stored locally) and trial expiry UI.
     - Keep all license checks local by default; optional online activation only if user opts in.

11. Logging & Diagnostics
   - Responsibilities:
     - Per-run detailed logs (run_items) with timestamps, pre/post paths, action results, and error reasons.
     - Provide export (CSV/JSON) and in-app search/filtering.
     - Optional, opt-in diagnostic export for user to attach to support ticket (no automatic uploads).

Data flow: Dry-run (typical)
1. UI triggers DryRunRule(ruleId, scopePaths).
2. App Services request file list (from file_cache or enumerated live) and metadata via Metadata Extractor.
3. Core Engine evaluates rule matchers against metadata and file attributes, producing a Plan (PlanItems) containing proposed operations.
4. Core Engine signals duplicate detection service if rule references duplicates (or UI can request DetectDuplicates first).
5. App Services return Plan to UI. No file system changes occur. UI shows preview tree, counts, and estimated sizes.

Data flow: Apply (typical)
1. User confirms Apply. UI calls AppServices.ApplyRule with Plan.
2. App Services call SafeOps to execute PlanItems in deterministic order (group by target folder to minimize conflicts).
3. SafeOps performs each operation with conflict policy and records per-item outcome in run_items table.
4. If operation succeeds, SafeOps updates file_cache for moved/copied items; if moved to quarantine, store mapping to enable undo.
5. On completion, run summary persisted and UI notified. Undo metadata saved.

Undo & rollback model
- Design: non-transactional at OS-level for cross-filesystem moves. Use an "undo journal" persisted as run_items entries capturing original_path, final_path, operation, timestamp, and optional backup location (quarantine path or Recycle Bin reference).
- Undo procedure:
  - Validate that current final_path still exists and is not modified (optional checksum compare) before moving back.
  - Attempt to move file back; if target exists, apply conflict policy (e.g., rename-with-suffix).
  - Update run_items with undo status and produce an undo report.
- Limitations:
  - If files were altered after move by external actors, restore may require user attention.
  - If original target was deleted or had permission changes, undo may fail and will be reported.

Error handling and retry strategies
- Per-item granularity: SafeOps records errors for each PlanItem.
- Retry policy for transient errors: for locked files, attempt N retries with exponential backoff (configurable), then mark skipped.
- Fatal errors (disk full, permission denied on large scale): abort run and persist partial run state for recovery. UI must present clear guidance and an option to resume or rollback.

Concurrency and performance
- Scanning strategy:
  - Initial full-scan mode: enumerate paths, collect FS metadata, and optionally compute or schedule hashing.
  - Incremental mode: refresh only paths changed since last run using file_cache.last_seen and timestamps.
- Hashing and metadata extraction are CPU/IO bound; run in worker thread pool with bounded concurrency to avoid saturating disk.
- Batching: group operations by target folder to reduce filesystem churn.
- Memory usage: stream large file lists to UI and use pagination.

APIs and module boundaries (recommended interfaces)
- IRuleService: CreateRule, UpdateRule, DeleteRule, ValidateRule, ListRules
- IRunService: DryRun(ruleId, scope), ApplyPlan(planId), ListRuns, GetRunDetails, UndoRun
- IFileOpService (SafeOps): ExecutePlanItemsAsync(IEnumerable<PlanItem>), AttemptUndo(runId)
- IMetadataService: ExtractMetadata(path), BatchExtract(paths[])
- IHashService: ComputeHash(path, algorithm), BatchCompute(paths[])
- ISchedulerService: ScheduleRule(ruleId, cronSpec), UnscheduleRule(ruleId)
- IStorage: CRUD for rules, runs, file_cache, run_items, licenses
- Note: Use dependency injection with interfaces for testability.

Security & permissions
- Avoid requiring elevation at install or runtime. If a user targets system folders, show explicit warning and only perform after per-operation elevation (if necessary).
- Respect file ACLs; do not attempt to override permissions silently.
- Protect license private keys off-device; app stores only a public key for signature verification.

Packaging & deployment
- Package format: MSIX preferred (clean install/uninstall, modern Windows integration). Also produce an offline installer (Inno Setup) if desired.
- Code signing: purchase/code-sign all binaries and installer. EV certificate recommended to reduce SmartScreen prompts.
- Update model: optional opt-in update-check (simple version JSON hosted on your site) or manual updates.

Testing & acceptance criteria (architecture-level)
- Dry-run must be deterministic and not change filesystem state.
- Applying a plan must persist run_items and update file_cache consistently.
- Undo must use run_items to revert a recent run with a success rate documented in tests.
- Duplicate detection must produce reproducible groups for canonical datasets.
- Scheduled runs must persist schedule and run reliably via internal scheduler and (optionally) via Task Scheduler.

Operational notes for developers
- Use logging levels (DEBUG, INFO, WARN, ERROR); preserve run logs locally and provide an export function.
- Keep long-running operations cancellable; provide UI cancellation and safe rollback behavior.
- Protect the SQLite database with file locks and use WAL mode for better concurrency.

Acceptance & next steps
- This file will be committed to docs/ARCHITECTURE.md and used to drive the Phase 1 implementation.
- Next tasks I will create after this commit:
  1. Detailed Rule-Engine Spec (rule evaluation order, precedence, sample rule JSON schema).
  2. Wireframes for Rule Wizard, Preview, and History/Undo screens.
  3. Sprint backlog for Phase 1 (tasks, owners, rough estimates).

