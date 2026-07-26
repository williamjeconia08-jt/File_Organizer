# CLI Specification — File Organizer

Purpose
This document specifies the command-line interface (CLI) for File Organizer. It is intended for sysadmins, automation, and power users who want to run saved rules or one-off operations headlessly. The CLI is designed to be safe by default: it performs dry-runs unless an explicit --apply (also aliased --yes) flag is provided to make destructive changes.

Binary name
- file-organizer-cli.exe

General safety rules
- Default behavior: dry-run. Running the CLI without --apply will generate a plan and exit with a status code indicating whether changes would have been made.
- Explicit apply required: destructive actions (move/copy/rename/dedupe_move) require the presence of --apply or --yes to execute.
- Headless rules: Rules intended for CLI execution should set options.cli_enabled = true. The CLI will refuse to run rules with cli_enabled = false when --rule-id is used.

Exit codes
- 0: Success — no errors; if dry-run, no changes required OR apply completed without errors.
- 1: Partial success — some items failed; check logs for details. (Apply mode with some failures or dry-run with non-fatal warnings.)
- 2: Validation error — rule not found, rule invalid, or CLI-incompatible rule configuration.
- 3: User refused to apply — --apply required for destructive operations (this is returned when attempt to run a destructive rule without --apply).
- 4: Internal error — unexpected exception; check diagnostic logs.

Primary commands
1) Run a saved rule by ID (dry-run by default)
- Usage:
  file-organizer-cli run --rule-id <RULE_ID> [--scope <path>] [--output <file.json>] [--verbose]
- Example (dry-run):
  file-organizer-cli run --rule-id rule-images-001 --scope "C:\\Users\\Alice\\Downloads" --output plan.json
- Example (apply):
  file-organizer-cli run --rule-id rule-images-001 --apply --yes --scope "C:\\Users\\Alice\\Downloads"
- Notes:
  - If the rule has options.cli_enabled != true, the command exits with code 2 (Validation error).
  - If rule conflict_policy == 'prompt', the CLI refuses to run in apply mode and exits with code 2 unless conflict_policy is non-interactive.
  - --scope overrides rule scope for this invocation.

2) Run an ad-hoc rule supplied as a JSON file
- Usage:
  file-organizer-cli run --rule-file <path-to-rule.json> [--apply] [--output <file.json>]
- Example:
  file-organizer-cli run --rule-file ./tests/rules/sample_rule.json --apply --yes --output ./out/plan.json
- Notes: Performs schema validation on the rule JSON before running.

3) Detect duplicates in a path
- Usage:
  file-organizer-cli detect-duplicates --path <dir> [--algorithm sha256|xxhash] [--output <file.json>]
- Example:
  file-organizer-cli detect-duplicates --path "C:\\Users\\Alice\\Pictures" --algorithm sha256 --output dupes.json
- Notes: This is non-destructive. To move duplicates after review, use a dedupe rule or a run with dedupe_move and --apply.

4) List saved rules
- Usage:
  file-organizer-cli list-rules
- Output: JSON array of rules with metadata (id, name, enabled, cli_enabled, priority).

Flags & options
- --rule-id <id> : Run the saved rule with this id (must have options.cli_enabled = true to run from CLI).
- --rule-file <file> : Run a rule provided by a JSON file (validates schema before running).
- --apply, --yes : Execute changes. Required to perform destructive operations. Without it, CLI performs a dry-run and writes the Plan to stdout or file.
- --scope <path> : Optional path to limit execution scope for this invocation (overrides rule scope.include_paths for this run).
- --output <file.json> : Write Plan (dry-run) or RunResult (apply) as JSON to file. If omitted, JSON is written to stdout.
- --verbose : More verbose logging to stderr (operational logs; JSON outputs remain machine-readable on stdout).
- --log-file <path> : Optional path to write detailed run logs. If omitted, logs are saved to the app's default log directory and path is printed in the output JSON.
- --no-color : Disable colored output for terminals that don't support it.

RunResult JSON format (when --output is provided or on apply)
- Top-level:
  {
    "run_id": "uuid",
    "rule_id": "rule-id-or-null",
    "mode": "dry-run" | "apply",
    "start_at": "ISO8601",
    "end_at": "ISO8601",
    "summary": { "planned_count": N, "applied_count": M, "failed_count": F },
    "items": [ { "original_path": "...", "proposed_path": "...", "action": "move", "result": "planned|success|failed|skipped", "error": "optional error message" } ],
    "log_path": "C:\\path\\to\\log.txt"
  }

Examples
- Dry-run and write plan to file:
  file-organizer-cli run --rule-id rule-images-001 --output C:\\temp\\plan.json

- Apply a saved rule non-interactively (requires --apply and --yes):
  file-organizer-cli run --rule-id rule-dupes-dedupe-001 --apply --yes --output C:\\temp\\run_result.json

- Detect duplicates and store report:
  file-organizer-cli detect-duplicates --path C:\\Users\\Alice\\Pictures --output C:\\temp\\dupes.json

Logging & diagnostics
- CLI writes operational logs to a default app log directory (e.g., %LOCALAPPDATA%\\FileOrganizer\\logs\\). If --log-file is provided, the CLI also writes logs there.
- For support, users can attach the RunResult JSON plus the relevant log file.

Security & permissions
- CLI runs in the current user's context. If a rule targets protected system folders, the CLI will fail unless run under an elevated shell; the CLI will not attempt elevation itself.
- The CLI obeys rule consume semantics and conflict policies.

Automation recommendations
- Use dry-run mode to validate rules before scheduling an apply in automated workflows.
- When scheduling via Task Scheduler or other orchestrators, ensure the scheduled job runs under the intended user account and has access to the target paths.
- Capture the RunResult JSON and inspect `failed_count` and non-zero exit codes to trigger alerts.

Exit behavior & examples
- Typical CI pipeline step (dry-run):
  - Step runs: file-organizer-cli run --rule-id nightly-cleanup --output plan.json
  - Pipeline inspects plan.json; if planned_count > 0, fails the pipeline or posts the plan for review.

- Production automation (apply):
  - file-organizer-cli run --rule-id nightly-cleanup --apply --yes --output run_result.json
  - If exit code != 0, pipeline fetches run_result.json and the log_path for troubleshooting.

Notes
- CLI is intentionally minimal and focused on safety. Implementors should ensure JSON outputs are machine-friendly and that logs contain sufficient detail for auditing.

