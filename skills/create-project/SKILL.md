# Create Project

Create a new project on the Codika platform.

## When to Use

- The user wants to create a new project for deploying use cases
- Before deploying a use case if no project exists yet

## Prerequisites

- `codika-helper` CLI installed and authenticated (see `setup-codika` skill)

## Command

```bash
codika-helper project create --name "My Project" [options]
```

### Required Options

| Flag | Description |
|------|-------------|
| `--name <name>` | Project name |

### Optional Options

| Flag | Description |
|------|-------------|
| `--description <desc>` | Project description |
| `--template-id <id>` | Template ID (defaults to platform default) |
| `--organization-id <id>` | Organization ID (required for admin keys, derived from org API keys) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--json` | Output result as JSON |

## Examples

**Basic project creation:**

```bash
codika-helper project create --name "Email Automations"
```

**With description and JSON output:**

```bash
codika-helper project create \
  --name "Sales Pipeline" \
  --description "Automated sales workflow processes" \
  --json
```

## Expected Output

**Success:**

```
✓ Project Created Successfully

  Project ID: abc123-def456
  Request ID: req-789
```

**JSON success:**

```json
{
  "success": true,
  "data": {
    "projectId": "abc123-def456"
  },
  "requestId": "req-789"
}
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika-helper login` (see `setup-codika` skill) |
| "Organization ID is required" | Using an admin key without specifying org | Add `--organization-id <id>` |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika-helper login` with a valid key |
| 403 / Forbidden | Key doesn't have project creation permission | Check API key permissions in the dashboard |

## Exit Codes

- `0` — Project created successfully
- `1` — API error (deployment failed)
- `2` — CLI validation error (missing required flags)
