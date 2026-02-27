# Setup Codika

Install the `codika-helper` CLI and authenticate with the Codika platform.

## When to Use

- The user wants to set up their environment for Codika
- A deploy or project command fails with "API key is required"
- First-time setup before using any other Codika skill

## Steps

### 1. Check if already installed

```bash
codika-helper --version
```

If the command is not found, install it:

```bash
npm install -g @codika-io/helper-sdk
```

### 2. Check if already authenticated

```bash
codika-helper config show
```

If the API key shows as "not set", proceed to login.

### 3. Login

**Interactive mode** (prompts for API key with masked input):

```bash
codika-helper login
```

**Non-interactive mode** (for CI/CD or scripting):

```bash
codika-helper login --api-key <key>
```

Optionally specify a custom base URL (defaults to production):

```bash
codika-helper login --api-key <key> --base-url <url>
```

### 4. Verify

```bash
codika-helper config show
```

Should display the masked API key and base URL.

## Where to Get an API Key

API keys are created from the Codika dashboard under Organization Settings > API Keys. The user needs to create an org-level API key.

## Configuration Storage

- Config file: `~/.config/codika-helper/config.json`
- File permissions: `0o600` (owner read/write only)
- Directory permissions: `0o700` (owner only)

## Clearing Configuration

```bash
codika-helper config clear
```

## Resolution Priority

When commands need an API key or URL, they check in this order:

1. `--api-key` / `--api-url` flag
2. `CODIKA_API_KEY` / `CODIKA_BASE_URL` environment variable
3. Config file
4. Production default (base URL only)

## Error Handling

| Symptom | Action |
|---------|--------|
| `npm: command not found` | Node.js is not installed. Ask the user to install Node.js 18+. |
| `EACCES` on global install | Try `npm install -g @codika-io/helper-sdk --prefix ~/.npm-global` or suggest using `npx`. |
| Login succeeds but deploy fails | Check `codika-helper config show` — the key may not have the right permissions. |
