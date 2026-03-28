---
name: create-organization-key
description: Creates an organization API key for a Codika organization. Use when the user needs an API key for deployment, triggering workflows, or managing a specific organization. Requires personal key or admin key with api-keys:manage scope.
---

# Create Organization Key

Create an organization API key (`cko_`) for a Codika organization.

## When to Use

- The user needs an API key scoped to a specific organization
- Setting up deployment credentials for CI/CD or agents
- Creating keys for triggering workflows or managing integrations within an org

## Prerequisites

- `codika` CLI installed and authenticated (see `setup-codika` skill)
- Personal key (`ckp_`) or admin key (`cka_`) with `api-keys:manage` scope
- The target organization must already exist (see `create-organization` skill)

## Command

```bash
codika organization create-key --organization-id <id> --name <name> --scopes <scopes> [options]
```

### Required Options

| Flag | Description |
|------|-------------|
| `--organization-id <id>` | Organization ID to create the key for |
| `--name <name>` | Key name (for identification) |
| `--scopes <scopes>` | Comma-separated list of scopes (e.g. `deploy,trigger,integrations:manage`) |

### Optional Options

| Flag | Description |
|------|-------------|
| `--description <desc>` | Key description |
| `--expires-in-days <days>` | Number of days until the key expires (no expiry if omitted) |
| `--api-url <url>` | Override API URL |
| `--api-key <key>` | Override API key |
| `--profile <name>` | Use a specific profile instead of the active one |
| `--json` | Output result as JSON |

## Examples

**Basic key creation:**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "CI Deploy Key" \
  --scopes "deploy,trigger"
```

**With description:**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "Agent Key" \
  --scopes "deploy,trigger,integrations:manage" \
  --description "Key for autonomous agent deployments"
```

**With expiry:**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "Temp Key" \
  --scopes "trigger" \
  --expires-in-days 30
```

**JSON output (for scripting):**

```bash
codika organization create-key \
  --organization-id "xwk9CcT440Vupa8soIhY" \
  --name "Script Key" \
  --scopes "deploy,trigger" \
  --json
```

## Expected Output

**Success:**

```
✓ Organization Key Created Successfully

  Key:          cko_xxxxxxxxxxxxxxxxxxxx (shown once — save it now)
  Key Prefix:   cko_xxxxxxxx
  Name:         CI Deploy Key
  Scopes:       deploy, trigger
  Request ID:   req-789
```

> **Important:** The raw key is displayed only once at creation time. Store it securely — it cannot be retrieved later.

**JSON success:**

```json
{
  "success": true,
  "data": {
    "key": "cko_xxxxxxxxxxxxxxxxxxxx",
    "keyPrefix": "cko_xxxxxxxx",
    "name": "CI Deploy Key",
    "scopes": ["deploy", "trigger"]
  },
  "requestId": "req-789"
}
```

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| "API key is required" | Not authenticated | Run `codika login` or check `codika whoami` (see `setup-codika` skill) |
| "API key does not have api-keys:manage scope" | Key missing required scope | Use a personal or admin key with `api-keys:manage` scope |
| "Organization not found" | Invalid organization ID | Verify the `--organization-id` value with `codika whoami` or the dashboard |
| "Organization keys cannot create other keys" | Using an org key (`cko_`) to create keys | Switch to a personal key (`ckp_`) or admin key (`cka_`) |
| "Invalid scopes" | Unrecognized scope value | Check available scopes in the dashboard or documentation |
| 401 / Unauthorized | Invalid or expired API key | Re-run `codika login` with a valid key |
| 403 / Forbidden | Not a member of the target organization | Ensure you are a member or owner of the organization |

## Exit Codes

- `0` — Key created successfully
- `1` — API error (creation failed)
- `2` — CLI validation error (missing required flags)
