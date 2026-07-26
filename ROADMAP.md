# File Organizer — Development Roadmap

Overview
This roadmap covers design, implementation, QA, packaging, and launch for a Windows-native file organizer (WinUI 3). It prioritizes safety, privacy (offline-only by default), and a clean modern UI. MVP includes: rule-based organizer, preview/dry-run, undo/quarantine, duplicate detection, advanced metadata rule support, scheduling, and SQLite-based metadata cache.

Phases
0. Documentation & Planning (Start Here)
- Outputs: Product spec (user stories, acceptance criteria), architecture doc, data model, rule-engine spec, wireframes, test plan.
- Tasks:
  - Define user stories for each persona (power users, sysadmins, office/home).
  - Write acceptance criteria for each story.
  - Draft architecture diagram (engine, UI, storage, API boundaries).
  - Produce wireframes for main screens.
  - Compile initial test dataset and test cases.

1. Core Engine
- Outputs: Rule engine library, op abstraction layer, metadata cache schema.
- Tasks:
  - Implement rule evaluation logic (priority/order, match types).
  - Implement safe file op abstraction (dry-run vs apply).
  - Create SQLite schema for rules, file cache, history.

2. Duplicate Detection & Metadata Rules
- Outputs: Duplicate detection module, metadata rule parser.
- Tasks:
  - Implement hashing pipeline (configurable algorithms).
  - Implement grouping and dedupe UI hooks.
  - Integrate EXIF/ID3/PDF metadata parsing and rule matching.

3. UI Shell & Rule Editor (WinUI 3)
- Outputs: App shell, rule wizard, preview pane.
- Tasks:
  - Build navigation (Rules, Scheduler, History, Settings).
  - Implement rule creation UI (simple and advanced modes).
  - Implement preview/dry-run UI with simulated tree.

4. Safety, Undo & History
- Outputs: Undo staging, Recycle Bin/quarantine, per-run logs.
- Tasks:
  - Define undo metadata format.
  - Implement undo operation flows and UI.
  - Implement exportable logs (JSON/CSV).

5. Scheduling & CLI
- Outputs: Scheduler UI + Task Scheduler integration, CLI tool for rule runs.
- Tasks:
  - Add internal scheduler; option to register Windows Task Scheduler tasks.
  - Implement simple CLI for automation.

6. Performance & Scale
- Outputs: Incremental scans, cache invalidation, plan for watch mode.
- Tasks:
  - Implement cache usage to avoid full rescans.
  - Add batch operations and async IO.
  - Document USN Journal integration plan for future.

7. Testing, QA & Accessibility
- Outputs: Automated tests, integration tests, accessibility audit.
- Tasks:
  - Unit tests for rule engine.
  - Integration tests using sandboxed folders.
  - Accessibility and keyboard navigation validation.

8. Packaging, Signing & Distribution
- Outputs: Signed MSIX, distribution plan, update-check.
- Tasks:
  - Acquire code signing cert.
  - Build MSIX packages and test installs.
  - Implement optional update-check (opt-in).

9. Beta & Launch
- Outputs: Beta feedback, final fixes, store pages, marketing checklist.
- Tasks:
  - Run closed beta, collect feedback, fix critical issues.
  - Prepare website, pricing page, license key delivery.

10. Post-launch
- Outputs: Patch schedule, user support channels.
- Tasks:
  - Respond to bug reports and user requests.
  - Plan next features (watch mode, cloud sync).

Acceptance Criteria
- Dry-run must list all planned actions without modifying files.
- Duplicate detection must correctly group duplicates in canonical test sets.
- Undo must revert last run in majority of test scenarios.
- UI must pass basic usability tests with sample users.

Risk & Mitigation
- Data loss: Dry-run default, robust undo.
- Permissions: Avoid elevation by default; handle errors gracefully.
- Performance: Use cache and incremental updates.

Tools & Stack
- WinUI 3 + .NET (C#), Visual Studio
- SQLite for storage
- Windows file APIs (IFileOperation, KnownFolders)
- NuGet metadata libraries for EXIF/ID3/PDF
- Hashing: SHA-256 or xxHash for duplicate detection (configurable to trade accuracy vs speed).
- Testing: xUnit or NUnit, PowerShell or integration harness for file-operation tests.
- Packaging: MSIX + code signing cert; optionally prepare an installer (Inno Setup or similar) for non-MSIX distribution.

Next actions
- Produce the Product Spec and Wireframes (Phase 0) then implement a minimal engine for dry-run capability.
