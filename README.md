# Codika Plugins (Public)

Claude Code plugin with skills for the `codika-helper` CLI. Install this plugin to let Claude Code agents create projects, deploy use cases, and validate workflows on the [Codika](https://codika.io) platform.

## Quick Start

1. Install the CLI:
   ```bash
   npm install -g @codika-io/helper-sdk
   ```

2. Log in with your API key (get one from the Codika dashboard):
   ```bash
   codika-helper login
   ```

3. Add this plugin to your Claude Code project:
   ```bash
   claude mcp add-plugin codika-plugins-public https://github.com/codika-io/codika-plugins-public
   ```

## Skills

| Skill | Description |
|-------|-------------|
| `setup-codika` | Install the CLI and authenticate |
| `create-project` | Create a new project on the platform |
| `deploy-use-case` | Validate and deploy a use case |
| `verify-use-case` | Validate workflows without deploying |

## What is Codika?

Codika is a multi-tenant SaaS platform for building, deploying, and managing business process automations powered by n8n workflows. Users describe what they need, AI agents create the workflows, and the platform handles deployment, credential management, and execution tracking.

## License

MIT
