# Rule Engine Spec — File Organizer

Purpose
This document defines the rule engine design for File Organizer: rule schema, supported matchers, evaluation order and precedence, planner behavior (dry-run), action model, conflict resolution, undo metadata, validation rules, test cases, and sample rule JSONs for initial unit/integration tests.

Goals
- Deterministic, testable rule evaluation producing an ordered Plan (PlanItems) in dry-run mode.
- Expressive filters: extensions, wildcards, regex, date ranges, size ranges, and metadata fields (EXIF, ID3, PDF metadata, Office props).
- Safe action semantics with clear undo metadata and conservative defaults.
- Extensible architecture so new matchers/actions can be added later.

Terminology
- Rule: user-defined object with matchers, actions, priority, scope, and metadata.
- Matcher: a predicate evaluated against a file's attributes/metadata.
- Action: an operation to perform on matching files (move, copy, rename, dedupe-move).
- Plan: ordered list of PlanItems produced by the engine describing exact operations to perform.
- PlanItem: { id, original_path, action, proposed_path, metadata, estimated_size }
- Run: an execution (dry-run or apply) associated with a rule and persisted to runs/run_items.

High-level design
1. Rule JSON Schema (logical)
- Fields (top-level):
  - id: string (GUID)
  - name: string
  - enabled: bool
  - priority: integer (lower number = higher priority)
  - scope: { include_paths: [string], exclude_paths: [string], max_depth: integer|null }
  - match: { combinator: "AND" | "OR" | "NOT", conditions: [Condition] }
  - action: ActionSpec
  - conflict_policy: { default: "rename" | "overwrite" | "skip" }, optional
  - options: { dry_run_default: true|false, compute_hash: true|false }
  - created_at / updated_at

2. Condition Object (matchers)
- A condition is a typed object with a `type` and type-specific properties. Examples:
  - { "type": "extension", "extensions": [".jpg",".jpeg"] }
  - { "type": "wildcard", "pattern": "IMG_*" }
  - { "type": "regex", "field": "name", "pattern": ".*invoice.*\\d{4}" }
  - { "type": "date_range", "field": "metadata.exif.DateTimeOriginal" , "from": "2020-01-01", "to": "2020-12-31" }
  - { "type": "size_range", "min_bytes": 0, "max_bytes": 104857600 }
  - { "type": "metadata_equals", "field": "metadata.author", "value": "ACME Corp" }
  - { "type": "duplicate_group", "policy": "candidate_master:first_match" }

3. ActionSpec
- Defines what to do when a file matches.
- Structure examples:
  - Move action:
    {
      "type": "move",
      "target": "C:\\Organized\\Photos\\{year}\\{month}",
      "template": { "year": "metadata.exif.DateTimeOriginal:yyyy", "month": "metadata.exif.DateTimeOriginal:MM" }
    }
  - Copy action:
    { "type": "copy", "target": "C:\\Organized\\Docs\\" }
  - Rename action:
    { "type": "rename", "pattern": "{name}_{hash:8}{ext}" }
  - Dedupe-move (special):
    { "type": "dedupe_move", "target": "C:\\Organized\\Dedupe\\" }

4. Rule Evaluation & Planner
- Steps:
  1. Scope enumeration: list files under include_paths excluding exclude_paths and respecting max_depth.
  2. For each file, lazily collect FS attributes (name, size, dates) and cached metadata if available.
  3. Evaluate match conditions according to the `match.combinator`.
     - Each Condition returns: MatchResult { matched: bool, details: {...} }
  4. If the rule's `options.compute_hash` is true and the rule or an action requires hashing (e.g., duplicate detection or rename template with {hash}), schedule hashing.
  5. For files that match, generate PlanItem with proposed action and computed tokens for templates (e.g., year, month, hash).
  6. Resolve intra-plan conflicts deterministically (see Conflict Resolution).
  7. Return ordered Plan sorted by: rule.priority (ascending), target folder path (lexicographic), original path (lexicographic). Determinism is crucial for repeatable dry-runs.

- Lazy evaluation: metadata extraction and hashing should be deferred until needed to minimize IO for simple rules.

5. Combining multiple rules
- When multiple enabled rules target overlapping files, the engine must respect rule priority.
- Process rules in ascending priority order. An earlier rule can mark a file as handled (if rule.action indicates consume=true), preventing lower-priority rules from re-processing the same file. For rules that should not consume, expose a `consume` boolean in the Rule JSON (default true for move actions, false for copy/report).

6. Conflict resolution
- Types of conflicts:
  - Target path collision: two PlanItems propose the same destination path.
  - Existing file at destination: destination path already exists on disk.
  - Cross-volume moves: moving from one volume to another may be non-atomic.

- Resolution strategy (in order):
  1. Detect collisions among PlanItems. If collision between items from same run: apply deterministic tie-breaker (original_path lexicographic) and apply conflict_policy: rename/overwrite/skip. Record a conflict entry in plan summary.
  2. At runtime (apply), before moving a file, check if destination exists on disk: if exists and matches byte-for-byte (or hash if available) then treat as duplicate (respect dedupe policy); otherwise apply conflict_policy.
  3. For cross-volume moves, perform copy + verified-delete approach and ensure undo metadata stores both source and backup location for recovery.

7. Dry-run semantics
- Dry-run produces Plan without making FS changes.
- PlanItems must include: id, original_path, proposed_path, action, estimated_size, reason (matcher details), tokens used for templates, any warnings (e.g., missing metadata token, locked file), and conflict flags.
- No quarantine or Recycle Bin actions executed.

8. Apply semantics
- AppServices will take a Plan (or re-run evaluation to produce one) and call SafeOps.ExecutePlanItems.
- Execution order must match Plan order.
- For each PlanItem:
  - Attempt the operation via appropriate API (IFileOperation preferred).
  - On success, insert run_item with result=success and update file_cache.
  - On failure, insert run_item with result=failed, record error, and continue or abort depending on run-level policy (configurable; default = continue and summarize failures).
- Persist undo metadata for each moved/copied item: original_path, final_path, operation, size, mtime, hash (optional), timestamp, and if quarantine used, the quarantine path.

9. Duplicate detection integration
- Duplicate detection is a separate service but the rule engine should support a `duplicate_group` condition which allows rules to match files that are detected as duplicates.
- Workflow:
  1. If any rule in the run references duplicates, engine triggers DetectDuplicates for the scope (using staged hashing strategy).
  2. The duplicate service returns groups: [[path1,path2,...], ...] with candidate_master suggestions.
  3. The engine can match files based on membership in a duplicate group and produce actions like `dedupe_move` using chosen master selection policy.
- Default behavior for duplicates: rules should not delete files automatically. `dedupe_move` moves duplicate copies (not the chosen master) to target dedupe folder.

10. Templates & tokens
- Support tokens in action targets and rename patterns. Tokens include:
  - {year}, {month}, {day} — derived from metadata or file mtime/ctime
  - {name}, {ext}
  - {hash}, {hash:8} — computed hash, truncated
  - {counter} — per-target incremental counter to avoid collisions
  - {metadata.<field>} — arbitrary metadata fields
- Token resolution should be deterministic and produce warnings if missing data (e.g., EXIF missing); fallback options should be documented (use file mtime if EXIF missing).

11. Validation & user feedback
- Validate rules on create/update:
  - Syntax: valid JSON, regex compilation
  - Template tokens: referenced tokens exist or have fallback
  - Scope: include_paths exist and are accessible (non-blocking check)
- Provide user-friendly error messages and highlight problematic condition(s).

12. Planner determinism and idempotency
- Determinism: Plan sorting rules and tie-breakers must be documented and stable across runs given same inputs.
- Idempotency: repeated dry-runs without underlying FS changes should produce identical Plan outputs.

13. Logging & observability
- For each Plan and Run, persist enough info to reproduce and undo: rule_id, scope, PlanItems (with computed tokens), hashes used, and sequence of operations.
- Keep a checksum of the Plan JSON to detect UI drift (if user edits rule after dry-run).

14. Tests and sample rule JSONs
- Unit tests: matcher evaluation, token resolution, template rendering, conflict detection.
- Integration tests: end-to-end plan generation on small sample datasets and expected PlanItems.

Sample rules (JSON)
1) Simple extension-based move (images)
{
  "id": "rule-images-001",
  "name": "Images to Photos by EXIF year/month",
  "enabled": true,
  "priority": 100,
  "scope": { "include_paths": ["C:\\Users\\Alice\\Downloads"], "exclude_paths": [] },
  "match": {
    "combinator": "AND",
    "conditions": [ { "type": "extension", "extensions": [".jpg",".jpeg",".png"] } ]
  },
  "action": {
    "type": "move",
    "target": "C:\\Organized\\Photos\\{year}\\{month}",
    "template": { "year": "metadata.exif.DateTimeOriginal:yyyy", "month": "metadata.exif.DateTimeOriginal:MM" }
  },
  "conflict_policy": { "default": "rename" },
  "options": { "compute_hash": false }
}

2) Regex + metadata rule (invoices)
{
  "id": "rule-invoices-001",
  "name": "Invoices to Client Folders",
  "enabled": true,
  "priority": 200,
  "scope": { "include_paths": ["C:\\Users\\Alice\\Documents\\Inbox"], "exclude_paths": [] },
  "match": {
    "combinator": "AND",
    "conditions": [
      { "type": "regex", "field": "name", "pattern": ".*(invoice|inv)[-_]?(\\d{4})" },
      { "type": "metadata_equals", "field": "metadata.office.Author", "value": "ACME Corp" }
    ]
  },
  "action": {
    "type": "move",
    "target": "C:\\Organized\\Invoices\\ACME\\{metadata.office.Year}",
    "template": { "metadata.office.Year": "metadata.office.Created:yyyy" }
  },
  "conflict_policy": { "default": "overwrite" },
  "options": { "compute_hash": false }
}

3) Duplicate detection rule (report-only)
{
  "id": "rule-dupes-report-001",
  "name": "Find duplicates in Pictures",
  "enabled": true,
  "priority": 50,
  "scope": { "include_paths": ["C:\\Users\\Alice\\Pictures"], "exclude_paths": [] },
  "match": {
    "combinator": "AND",
    "conditions": [ { "type": "duplicate_group" } ]
  },
  "action": {
    "type": "report",
    "target": "",
    "template": {}
  },
  "options": { "compute_hash": true }
}

4) Move duplicates on user command (dedupe_move)
- This is a sample action used only after review (UI triggers apply with this action).
{
  "id": "rule-dupes-dedupe-001",
  "name": "Move duplicate copies to Dedupe Folder",
  "enabled": false,
  "priority": 50,
  "scope": { "include_paths": ["C:\\Users\\Alice\\Pictures"], "exclude_paths": [] },
  "match": {
    "combinator": "AND",
    "conditions": [ { "type": "duplicate_group" } ]
  },
  "action": {
    "type": "dedupe_move",
    "target": "C:\\Organized\\Dedupe\\{year}_{month}",
    "template": { "year": "metadata.exif.DateTimeOriginal:yyyy", "month": "metadata.exif.DateTimeOriginal:MM" }
  },
  "conflict_policy": { "default": "rename" },
  "options": { "compute_hash": true }
}

Expected Planner Output Example (PlanItem)
{
  "plan_id": "plan-20260726-0001",
  "rule_id": "rule-images-001",
  "items": [
    {
      "id": "item-0001",
      "original_path": "C:\\Users\\Alice\\Downloads\\IMG_001.JPG",
      "action": "move",
      "proposed_path": "C:\\Organized\\Photos\\2021\\08\\IMG_001.JPG",
      "estimated_size": 3456789,
      "tokens": { "year": "2021", "month": "08" },
      "warnings": []
    }
  ]
}

15. Validation & unit test cases (days estimates)
- Spec writing: 2 days (this document + examples) — DONE
- Unit tests for matchers (each matcher): 1 day per matcher (estimate):
  - extension: 0.25 days
  - wildcard: 0.5 days
  - regex: 0.5 days
  - date_range: 0.5 days
  - size_range: 0.25 days
  - metadata_equals: 0.5 days
  - duplicate_group (integration test with hash service): 1 day
- Token rendering tests: 1 day
- Planner determinism tests (idempotency/repeatability): 1 day
- Integration tests (small dataset dry-run): 2 days

16. Implementation backlog (Phase 1 tasks & days)
- Implement Rule JSON model + validation: 2 days
- Implement matcher library (each matcher implemented as modular evaluators): 6 days total (distributed as above)
- Implement token resolver & template engine: 3 days
- Implement planner that produces deterministic PlanItems (dry-run): 5 days
- Add unit tests and CI for the rule engine library: 3 days
- Integrate with storage (persist rules and plan snapshots): 2 days

Total Phase 1 (rule engine core) estimate: 21 days (3–4 weeks solo) — includes tests and basic storage integration.

17. Extensibility notes
- Matcher and action implementations should follow interface contracts so new types can be added without changing planner core.
- Consider a plugin system (post-MVP) to support custom matchers or external metadata sources.

18. Edge cases & constraints
- Long paths: normalize using Windows long-path handling where supported. Warn user if paths exceed 260-char and recommend enabling long path support.
- Symbolic links/junctions: treat as files by default; add optional rule to follow or ignore symlinks.
- Partial metadata: fallback strategies must be explicit (use file mtime if EXIF missing).

19. Security and safety
- All file operations must respect ACLs; never attempt to bypass via elevation without user consent.
- In dry-run, do not create temporary files except in a true apply run.

20. Next steps
- Commit this doc to docs/RULE_ENGINE_SPEC.md (done by the assistant).
- Start implementation of the Rule Engine library: create a C# class library project (RuleEngine.Core) with interfaces for matchers, actions, planner, and test projects.
- Begin with unit tests for basic matchers and token resolver.

