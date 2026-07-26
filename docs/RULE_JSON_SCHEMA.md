# Rule JSON Schema & Validation — File Organizer

This document contains a JSON Schema for the Rule object used by File Organizer and a validation guide with user-friendly error messages and remediation suggestions. It also documents the CLI-run option for rules (so saved rules can be executed from the command-line/automation).

Schema notes
- Uses JSON Schema Draft-07 features for compatibility.
- The schema is intentionally strict for top-level fields but allows extensibility via `additionalProperties: false` at selected places and `match.conditions` and `action` typed unions.

JSON Schema (Draft-07)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "File Organizer Rule",
  "type": "object",
  "required": ["id","name","enabled","priority","scope","match","action"],
  "additionalProperties": false,

  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "name": { "type": "string", "minLength": 1 },
    "description": { "type": "string" },
    "enabled": { "type": "boolean" },
    "priority": { "type": "integer", "minimum": 0 },

    "scope": {
      "type": "object",
      "additionalProperties": false,
      "required": ["include_paths"],
      "properties": {
        "include_paths": { "type": "array", "items": { "type": "string" }, "minItems": 1 },
        "exclude_paths": { "type": "array", "items": { "type": "string" } },
        "max_depth": { "anyOf": [ { "type": "integer", "minimum": 0 }, { "type": "null" } ] }
      }
    },

    "match": {
      "type": "object",
      "additionalProperties": false,
      "required": ["combinator","conditions"],
      "properties": {
        "combinator": { "type": "string", "enum": ["AND","OR","NOT"] },
        "conditions": {
          "type": "array",
          "minItems": 1,
          "items": { "$ref": "#/definitions/condition" }
        }
      }
    },

    "action": { "$ref": "#/definitions/action" },

    "conflict_policy": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "default": { "type": "string", "enum": ["rename","overwrite","skip"] }
      }
    },

    "options": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "dry_run_default": { "type": "boolean" },
        "compute_hash": { "type": "boolean" },
        "cli_enabled": { "type": "boolean" }
      }
    },

    "consume": { "type": "boolean", "description": "If true, this rule marks files as handled and lower-priority rules won't process them (default true for move)." }
  },

  "definitions": {
    "condition": {
      "type": "object",
      "additionalProperties": false,
      "oneOf": [
        {
          "properties": {
            "type": { "const": "extension" },
            "extensions": { "type": "array", "items": { "type": "string", "pattern": "^\\.[A-Za-z0-9]+$" }, "minItems": 1 }
          },
          "required": ["type","extensions"]
        },
        {
          "properties": {
            "type": { "const": "wildcard" },
            "pattern": { "type": "string" }
          },
          "required": ["type","pattern"]
        },
        {
          "properties": {
            "type": { "const": "regex" },
            "field": { "type": "string" },
            "pattern": { "type": "string" }
          },
          "required": ["type","field","pattern"]
        },
        {
          "properties": {
            "type": { "const": "date_range" },
            "field": { "type": "string" },
            "from": { "type": "string", "format": "date" },
            "to": { "type": "string", "format": "date" }
          },
          "required": ["type","field"]
        },
        {
          "properties": {
            "type": { "const": "size_range" },
            "min_bytes": { "type": "integer", "minimum": 0 },
            "max_bytes": { "type": "integer", "minimum": 0 }
          },
          "required": ["type"]
        },
        {
          "properties": {
            "type": { "const": "metadata_equals" },
            "field": { "type": "string" },
            "value": { "type": "string" }
          },
          "required": ["type","field","value"]
        },
        {
          "properties": {
            "type": { "const": "duplicate_group" },
            "policy": { "type": "string", "enum": ["candidate_master:first_match","candidate_master:largest","candidate_master:newest"] }
          },
          "required": ["type"]
        }
      ]
    },

    "action": {
      "type": "object",
      "additionalProperties": false,
      "oneOf": [
        {
          "properties": {
            "type": { "const": "move" },
            "target": { "type": "string" },
            "template": { "type": "object" }
          },
          "required": ["type","target"]
        },
        {
          "properties": {
            "type": { "const": "copy" },
            "target": { "type": "string" }
          },
          "required": ["type","target"]
        },
        {
          "properties": {
            "type": { "const": "rename" },
            "pattern": { "type": "string" }
          },
          "required": ["type","pattern"]
        },
        {
          "properties": {
            "type": { "const": "dedupe_move" },
            "target": { "type": "string" },
            "template": { "type": "object" }
          },
          "required": ["type","target"]
        },
        {
          "properties": {
            "type": { "const": "report" }
          },
          "required": ["type"]
        }
      ]
    }
  }
}
```

Validation guide (user-friendly messages)

Top-level validation errors
- Missing required field: `id`, `name`, `enabled`, `priority`, `scope`, `match`, or `action`.
  - Message: "Rule is missing required field '<field>'. Please provide a name, scope, match conditions, and an action."
- Unknown top-level property: (additional property present)
  - Message: "Unrecognized property '<prop>' in rule. Remove or move it under 'options' or 'description'."

Scope errors
- include_paths missing or empty
  - Message: "Please specify at least one folder to include in the rule's scope. Example: 'C:\\Users\\Alice\\Downloads'"
- max_depth invalid
  - Message: "'max_depth' must be a non-negative integer or null. Use null for unlimited depth."

Match/Condition errors
- No conditions
  - Message: "A rule must contain one or more conditions to match files."
- Invalid regex pattern
  - Message: "Regex pattern '<pattern>' could not be compiled. Check for syntax errors."
- Invalid extension format
  - Message: "Extensions must start with a dot, e.g., '.jpg' or '.pdf'."
- date_range invalid date format
  - Message: "Date must be in ISO date format (YYYY-MM-DD)."

Action errors
- Missing target for move/copy/dedupe_move
  - Message: "Provide a target folder for move/copy/dedupe actions. Example: 'C:\\Organized\\Photos\\{year}'."
- Pattern missing for rename
  - Message: "Rename action requires a 'pattern' property. Use tokens like '{name}_{hash:8}{ext}'."

Template & token errors
- Missing token field in template
  - Message: "Template references metadata field '<token>' which is not resolved. Ensure metadata extractor can provide this field or provide a fallback (e.g., file mtime)."
- Unknown token used in target
  - Message: "Unknown token '{<token>}' in target. Allowed tokens: {year,month,day,name,ext,hash,counter,metadata.<field>}."

Conflict policy errors
- Unknown policy
  - Message: "Conflict policy must be one of: rename, overwrite, skip."

Options errors
- Unknown option
  - Message: "Unknown option '<opt>' — supported options: dry_run_default, compute_hash, cli_enabled."

Duplicate condition errors
- compute_hash not enabled but duplicate_group condition present
  - Message: "Duplicate detection requires 'options.compute_hash' to be true. Enable it to detect duplicates."

CLI related validation and notes
- `options.cli_enabled` (boolean): when true the rule may be invoked from the command line or scripts by rule id (e.g., file-organizer-cli run --rule-id <id>).
  - Recommendation: Default to false for rules that should not be run headlessly; set true for automation-friendly rules.
- CLI validation errors:
  - If CLI run is requested for a rule that depends on interactive prompts (e.g., conflict_policy: prompt), validation warns: "Rule requires interactive confirmation during apply. Disable interactive prompts or set non-interactive conflict_policy (rename/overwrite/skip) to use via CLI."
- CLI safety: rules with `action.type` == `move` should be safe for CLI runs only if `consume` behavior and conflict policies are non-interactive. The validator flags rules that are enabled for CLI but require user prompts.

Examples of common fixes
- Error: "Template token metadata.exif.DateTimeOriginal missing"
  - Fix: Ensure the metadata extractor supports EXIF and the file(s) contain EXIF DateTimeOriginal. Alternatively change template to use file mtime token: `{year}` -> use fallback `file.mtime`.
- Error: "Regex pattern invalid"
  - Fix: Test the regex in a regex tester, escape backslashes properly in JSON (use `\\`), and re-save the rule.

Validation flow in the app (recommended)
1. Syntactic validation using JSON Schema (fast, immediate). Presents top-left inline errors in the rule editor.
2. Semantic validation (background): compile regex, attempt to resolve tokens against a small sample of files from scope (non-destructive). Present warnings (yellow) for tokens that might be missing in many files.
3. CLI-safety validation (when `options.cli_enabled` is true), check for interactive dependencies and warn if any.

JSON Schema tooling
- Use a standard JSON Schema validator in the UI/backend to show syntax errors immediately. For C#, popular libraries include Newtonsoft.Json.Schema (Json.NET Schema) for Draft-07 support.
- For runtime/semantic checks, create an additional validator that runs on save and provides warnings but allows saving if the user confirms.

File naming and migrations
- Keep rule schema versioned in rules: add `schema_version` to Rule objects so the app can migrate rules if the format evolves.

Next steps I will commit
- Commit this file as `docs/RULE_JSON_SCHEMA.md` to your repository.
- If you want, I can also export the JSON Schema as a machine-readable file `schemas/rule.schema.json` for direct use in the UI and CI — tell me to proceed and I will add it.

