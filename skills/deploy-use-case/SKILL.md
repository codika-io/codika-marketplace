# Deploy Use Case

Validate and deploy a use case to the Codika platform.

## When to Use

- The user wants to deploy a use case (n8n workflows + config) to Codika
- After creating or modifying workflow files
- After a successful `verify-use-case` check

## Prerequisites

- `codika-helper` CLI installed and authenticated (see `setup-codika` skill)
- A use case folder with `config.ts` and `workflows/` directory

## Recommended Flow

Always validate before deploying:

```bash
# Step 1: Validate
codika-helper verify use-case <path>

# Step 2: Deploy (only if validation passes)
codika-helper deploy use-case <path>
```

## Use Case Folder Structure

```
my-use-case/
  config.ts           # Exports PROJECT_ID, getConfiguration(), optionally WORKFLOW_FILES
  workflows/
    main-workflow.json
    sub-workflow.json
  version.json        # Optional — semantic version tracking
```

## Deploy Command

```bash
codika-helper deploy use-case <path> [options]
```

### Options

| Flag | Description |
|------|-------------|
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--version-strategy <strategy>` | `minor_bump` (default), `major_bump`, or `explicit` |
| `--explicit-version <version>` | Required when strategy is `explicit` |
| `--additional-file <abs:rel>` | Add extra file (repeatable). Format: `absolutePath:relativePath` |
| `--json` | Output result as JSON |

## Examples

**Standard deployment:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case
```

**Major version bump:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case --version-strategy major_bump
```

**With additional metadata file:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case \
  --additional-file "/path/to/prd.md:docs/prd.md"
```

**JSON output for scripting:**

```bash
codika-helper deploy use-case ./use-cases/marketplace/my-use-case --json
```

## Expected Output

**Success:**

```
✓ Deployment Successful

  Template ID:  tmpl-abc123
  API Version:  2.1.0
  Process ID:   proc-def456
  Status:       active

  Workflows:
    - main-workflow: deployed (n8n: 42)
    - sub-workflow: deployed (n8n: 43)
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` (see `setup-codika` skill) |
| "Use case path does not exist" | Wrong path | Check the path points to a folder with `config.ts` |
| Validation errors in response | Workflow issues | Run `codika-helper verify use-case <path> --fix` first (see `verify-use-case` skill) |
| "config.ts must export PROJECT_ID" | Missing export | Ensure `config.ts` exports `PROJECT_ID` as a string constant |
| "config.ts must export getConfiguration" | Missing export | Ensure `config.ts` exports a `getConfiguration()` function |
| 401 / Unauthorized | Invalid API key | Re-run `codika-helper login` |

## Exit Codes

- `0` — Deployment successful
- `1` — Deployment failed (API error or workflow-level failure)
- `2` — CLI validation error (missing path, invalid flags)
