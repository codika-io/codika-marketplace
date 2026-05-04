# GitHub Integration Guide

GitHub is **not a built-in Codika integration** — there is no `github` integration UID, no `n8n-nodes-base.github` shorthand we expose. Use cases that need GitHub define their own custom integration with `cstm_github` (or any `cstm_*` ID) and call the GitHub REST API via standard `n8n-nodes-base.httpRequest` nodes authenticated with `httpHeaderAuth`.

This guide is the worked example for that pattern. The reference implementation is `agency/projects/iconic-group/iconic-group-cms/processes/iconic-group-cms/` — a use case that opens / updates / archives GitHub PRs on behalf of a CMS dashboard.

> **See also:** [use-case-guide.md § 15 Custom Integrations](../use-case-guide.md#15-custom-integrations) for the generic `customIntegrations` schema.

---

## 1. Why custom and not built-in

Built-in n8n GitHub nodes exist (`n8n-nodes-base.github`, `n8n-nodes-base.githubTrigger`) but are not exposed as a Codika built-in integration today. Going through `httpHeaderAuth` + `httpRequest` instead has three concrete benefits:

1. **Per-instance credentials.** Every process instance can hold its own PAT scoped to a single repo, so the dashboard's user-specific install pattern stays clean. (Built-in nodes typically need an org-level OAuth app.)
2. **Full API surface.** You can hit any endpoint — `/repos/.../git/refs`, `/repos/.../contents/...`, `/search/issues`, `/repos/.../pulls/.../labels` — without waiting for n8n to add operations.
3. **Same n8n credential per repo.** The credential is a single header value, easy to swap when rotating tokens.

Trade-off: you write the API calls yourself. For a CMS pipeline that's perfectly fine; for trivial "just create an issue" use cases, the built-in node may be lighter.

---

## 2. Token strategy

### Recommended: fine-grained PAT scoped to one repo

Generate at **GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens**.

| Setting | Value |
|---|---|
| **Token name** | `<bot-handle>-<repo>` (e.g. `iconic-cms-bot-iconic-group-website`) |
| **Resource owner** | The org that owns the target repo |
| **Repository access** | "Only select repositories" → pick the one repo |
| **Repository permissions (CMS-style PR pipeline)** | `Contents` R/W, `Pull requests` R/W, `Issues` R/W |
| **Expiration** | 90 or 365 days; rotate via `codika integration set --force` |

Permissions vary by what your workflow actually does. A read-only data-pull use case might only need `Contents: Read` and `Metadata: Read`.

### Avoid: classic PATs and OAuth tokens for production

- **Classic PATs (`ghp_`)** grant org-wide access; over-scoped for a single use case.
- **OAuth tokens (`gho_`)** from `gh auth login` work but inherit YOUR user permissions — not portable, not auditable, can disappear when your gh CLI re-authenticates. Acceptable for local testing only.

### Bot user vs your user

For PR-creation pipelines, generate the PAT under a dedicated GitHub user (e.g. `<product>-cms-bot`) so commits and PRs are clearly attributable to the bot. The bot user needs `read:org` to be added to the org and write access to the target repo.

---

## 3. The `cstm_github` schema

Add this to `customIntegrations` in your `config.ts`. Lift wholesale; the only fields you'll typically tweak are `description`, `id` (if you want a more specific name like `cstm_github_iconic`), and `contextType` (almost always `process_instance`).

```typescript
import type { CustomIntegrationSchema } from 'codika';

const githubIntegration: CustomIntegrationSchema = {
  id: 'cstm_github',
  name: 'GitHub',
  description:
    'Fine-grained PAT scoped to the target repo. Used by the workflow to ' +
    'open PRs, commit files, manage labels.',
  contextType: 'process_instance',
  n8nCredentialType: 'httpHeaderAuth',
  n8nCredentialMapping: {
    GITHUB_TOKEN: 'value',
    HEADER_NAME: 'name',
  },
  secretFields: [
    {
      key: 'GITHUB_TOKEN',
      label: 'GitHub Token (with Bearer prefix)',
      type: 'password',
      description:
        'Full Authorization header value, INCLUDING the "Bearer " prefix ' +
        '(e.g. "Bearer github_pat_..."). The httpHeaderAuth credential sends ' +
        'this value verbatim, so the prefix must be present here.',
      placeholder: 'Bearer github_pat_...',
      required: true,
    },
    {
      key: 'HEADER_NAME',
      label: 'Header Name',
      type: 'string',
      description: 'HTTP header name. GitHub expects "Authorization".',
      placeholder: 'Authorization',
      required: true,
    },
  ],
  icon: 'GitPullRequest',
  color: '#181717',
};

// In getConfiguration():
return {
  // ...
  customIntegrations: [githubIntegration],
  integrationUids: ['cstm_github', /* others */],
  // ...
};
```

### ⚠️ The "Bearer prefix" gotcha

`n8nCredentialType: 'httpHeaderAuth'` sends the value field **verbatim** as the named header. There is no automatic `Bearer ` prefixing anywhere in the platform or in the workflow. So:

- ❌ Setting `GITHUB_TOKEN=github_pat_...` produces `Authorization: github_pat_...` → GitHub returns **HTTP 422 Validation Failed** ("The listed users and repositories cannot be searched either because the resources do not exist or you do not have permission to view them").
- ✅ Setting `GITHUB_TOKEN=Bearer github_pat_...` produces `Authorization: Bearer github_pat_...` → GitHub accepts.

The `secretFields[].description` and `placeholder` MUST make this explicit, or every fresh install will fail the same way and the error message gives no hint that the prefix is missing.

The `Bearer` form works for fine-grained PATs, classic PATs, and OAuth tokens. The older `token <pat>` form also works for classic PATs but not for fine-grained — use `Bearer` everywhere for consistency.

---

## 4. Wiring credentials into HTTP nodes

Every workflow HTTP node that hits the GitHub API references the credential by `INSTCRED` placeholder (because `contextType: 'process_instance'`):

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "GET",
    "url": "=https://api.github.com/...",
    "options": {
      "response": { "response": { "responseFormat": "json" } }
    }
  },
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

For other context types, swap the placeholder per [use-case-guide.md § 15](../use-case-guide.md#15-custom-integrations):

| `contextType` | Placeholder pair |
|---|---|
| `process_instance` | `INSTCRED_CSTM_GITHUB_{ID,NAME}_DERCTSNI` |
| `organization` | `ORGCRED_CSTM_GITHUB_{ID,NAME}_DERCGRO` |
| `member` | `USERCRED_CSTM_GITHUB_{ID,NAME}_DERCRESU` |

---

## 5. Setting the credential at install time

After deploying the use case (`codika deploy use-case .`), set the per-instance credential via CLI. **Note the `Bearer` prefix in the value:**

```bash
codika integration set cstm_github \
  --context-type process_instance \
  --process-instance-id <PROCESS_INSTANCE_ID> \
  --environment <dev|prod> \
  --secret 'GITHUB_TOKEN=Bearer github_pat_...' \
  --secret HEADER_NAME=Authorization \
  --force
```

Then redeploy so the workflows bind to the new n8n credential ID:

```bash
codika redeploy --process-instance-id <PROCESS_INSTANCE_ID> --environment <dev|prod> --force
```

To rotate the PAT later, re-run the same `integration set` (the `--force` flag deletes + recreates the credential), then redeploy.

---

## 6. Common GitHub API operations

These are the building blocks for a typical PR-pipeline. Each is a standalone HTTP node — use n8n branching / merging to assemble flows.

### 6.1 Search for an existing PR by article ID

Useful for idempotency: skip if a PR already exists for this article.

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "GET",
    "url": "={{ $json.searchUrl }}",
    "options": {}
  },
  "name": "Search Existing Bot PR",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

Build `searchUrl` upstream:

```javascript
// In a Code node before the HTTP request
const q = [
  `repo:${owner}/${repo}`,
  'is:pr',
  'state:open',
  `label:${botLabel}`,
  `"Article-Id: ${articleId}" in:body`,
].join(' ');
return [{ json: { searchUrl: `https://api.github.com/search/issues?q=${encodeURIComponent(q)}` } }];
```

### 6.2 Get the SHA of a base branch

Needed to create a new branch off it.

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "GET",
    "url": "=https://api.github.com/repos/{{ $('Build Context').first().json.owner }}/{{ $('Build Context').first().json.repo }}/git/ref/heads/{{ $('Build Context').first().json.baseBranch }}",
    "options": {}
  },
  "name": "Get Base SHA",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

### 6.3 Create a branch

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "POST",
    "url": "=https://api.github.com/repos/{{ $('Build Context').first().json.owner }}/{{ $('Build Context').first().json.repo }}/git/refs",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"ref\": \"refs/heads/{{ $('Build Context').first().json.branchName }}\",\n  \"sha\": \"{{ $json.object.sha }}\"\n}",
    "options": {}
  },
  "name": "Create Branch",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

### 6.4 Commit a file (create or update via Contents API)

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "PUT",
    "url": "=https://api.github.com/repos/{{ $('Build Context').first().json.owner }}/{{ $('Build Context').first().json.repo }}/contents/{{ $('Build Context').first().json.markdownPath }}",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"message\": \"chore(cms): publish {{ $('Build Context').first().json.slug }}\",\n  \"content\": \"{{ $json.contentBase64 }}\",\n  \"branch\": \"{{ $('Build Context').first().json.branchName }}\",\n  \"author\": {\n    \"name\": \"Iconic CMS Bot\",\n    \"email\": \"{{ $('Build Context').first().json.commitAuthorEmail }}\"\n  },\n  \"committer\": {\n    \"name\": \"Iconic CMS Bot\",\n    \"email\": \"{{ $('Build Context').first().json.commitAuthorEmail }}\"\n  }\n}",
    "options": {}
  },
  "name": "Commit Markdown",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

When **updating** an existing file, you must include the file's current `sha` in the body — fetch it first via `GET /repos/.../contents/<path>?ref=<branch>`, then pass the returned `sha`.

### 6.5 Open a PR

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "POST",
    "url": "=https://api.github.com/repos/{{ $('Build Context').first().json.owner }}/{{ $('Build Context').first().json.repo }}/pulls",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"title\": \"chore(cms): publish {{ $('Build Context').first().json.title }}\",\n  \"head\": \"{{ $('Build Context').first().json.branchName }}\",\n  \"base\": \"{{ $('Build Context').first().json.baseBranch }}\",\n  \"body\": \"Article-Id: {{ $('Build Context').first().json.articleId }}\\n\\nAuto-generated by the CMS bot.\"\n}",
    "options": {}
  },
  "name": "Open PR",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

### 6.6 Apply a label

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "POST",
    "url": "=https://api.github.com/repos/{{ $('Build Context').first().json.owner }}/{{ $('Build Context').first().json.repo }}/issues/{{ $json.number }}/labels",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"labels\": [\"{{ $('Build Context').first().json.botLabel }}\"]\n}",
    "options": {}
  },
  "name": "Label PR",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

### 6.7 Delete a file (archive flow)

```json
{
  "parameters": {
    "authentication": "predefinedCredentialType",
    "nodeCredentialType": "httpHeaderAuth",
    "method": "DELETE",
    "url": "=https://api.github.com/repos/{{ $('Build Context').first().json.owner }}/{{ $('Build Context').first().json.repo }}/contents/{{ $('Build Context').first().json.markdownPath }}",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={\n  \"message\": \"chore(cms): archive {{ $('Build Context').first().json.slug }}\",\n  \"sha\": \"{{ $json.sha }}\",\n  \"branch\": \"{{ $('Build Context').first().json.branchName }}\"\n}",
    "options": {}
  },
  "name": "Delete File",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "credentials": {
    "httpHeaderAuth": {
      "id": "{{INSTCRED_CSTM_GITHUB_ID_DERCTSNI}}",
      "name": "{{INSTCRED_CSTM_GITHUB_NAME_DERCTSNI}}"
    }
  }
}
```

---

## 7. PR-pipeline flow assembly

A typical "publish article" sub-workflow in the CMS reference uses these nodes in sequence:

```
Execute Workflow Trigger
  → Build Context (Code: builds owner/repo/branchName/markdownPath/etc.)
  → Search Existing Bot PR (HTTP: skip if already open)
    └─ Has Stale PR? (IF) → Submit Result (existing PR)
  → Get Base SHA (HTTP)
  → Create Branch (HTTP)
  → Download Image (HTTP)
  → Hash Image → Commit Image (HTTP)
  → Build Markdown (Code) → Commit Markdown (HTTP)
  → Open PR (HTTP) → Label PR (HTTP)
  → Update Article State (Postgres)
  → Format Result → Codika Submit Result
```

See `agency/projects/iconic-group/iconic-group-cms/processes/iconic-group-cms/workflows/sub-publish-article.json` for the full reference.

The sister sub-workflows (`sub-update-article`, `sub-archive-article`) follow the same pattern with different operations: update fetches the existing file SHA before re-committing; archive deletes via DELETE without committing.

---

## 8. Companion: GitHub Actions for two-way callback

The CMS reference uses a follow-up callback pattern: after the bot's PR is opened, GitHub Actions auto-merges it and POSTs back to a Codika webhook so the dashboard can flip the article's `publish_status` to `live`. This is documented for the iconic-group-cms case in:

- `agency/projects/iconic-group/iconic-group-website/.github/workflows/cms-auto-merge.yml`
- `agency/projects/iconic-group/iconic-group-website/.github/workflows/cms-callback.yml`

Two GitHub-side gotchas worth lifting if you build something similar:

1. **GITHUB_TOKEN-driven pushes don't trigger workflows.** When `cms-auto-merge.yml` runs `gh pr merge --squash` with `secrets.GITHUB_TOKEN`, the resulting push to the base branch does NOT fire `pull_request: closed` or `push` events on other workflows. Use `workflow_run` triggers on the merging workflow if you need a follow-up.
2. **`gh pr merge --squash` drops the PR body** by default. Title becomes the commit message; body becomes the list of source commits. If you need the PR body (e.g. to extract `Article-Id:` from a callback), look it up via `gh pr list --json body` instead of parsing the squash commit message.

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| HTTP 422 "Validation Failed" with `"The listed users and repositories cannot be searched either because the resources do not exist or you do not have permission to view them"` | Token sent without `Bearer ` prefix → request hits GitHub unauthenticated → search on private repo returns this error | Re-set credential: `--secret 'GITHUB_TOKEN=Bearer github_pat_...'`, then `codika redeploy --force` |
| HTTP 401 "Bad credentials" | Token revoked / expired / wrong | Generate fresh PAT, re-set credential, redeploy |
| HTTP 404 on a private repo | Token doesn't have access to the repo (fine-grained PAT scoped to wrong repos) | Regenerate PAT with the correct repository selection |
| HTTP 403 "Resource not accessible by personal access token" | Missing permission on the fine-grained PAT (e.g. `Contents` is Read but you're trying to write) | Edit PAT permissions in GitHub settings, no need to regenerate |
| HTTP 422 "Reference already exists" on Create Branch | Branch with that name already exists from a prior run | Use a unique branch name (e.g. include a timestamp suffix) or delete + retry |
| HTTP 422 "Update is not a fast forward" on Commit | Another commit landed on the branch since your last fetch | Re-fetch the file's `sha` and retry |

To verify the credential value is being applied correctly, look at the failed execution's node output — the URL is shown but headers usually aren't. Compare with a successful execution from a working instance side-by-side: same node, different credential, one works, one doesn't → it's the credential.

---

## 10. Requirements checklist

- [ ] Custom integration ID is `cstm_*` (e.g. `cstm_github`)
- [ ] `n8nCredentialType: 'httpHeaderAuth'`, mapping `GITHUB_TOKEN: 'value'` and `HEADER_NAME: 'name'`
- [ ] `secretFields[].description` and `placeholder` for the token field explicitly mention the `Bearer ` prefix
- [ ] Fine-grained PAT scoped to the single target repo with the minimum permissions needed
- [ ] Set the credential via `codika integration set ... --secret 'GITHUB_TOKEN=Bearer github_pat_...'`
- [ ] Redeploy after setting the credential so the workflow's HTTP nodes bind to the new n8n credential ID
- [ ] Workflow HTTP nodes use `authentication: 'predefinedCredentialType'`, `nodeCredentialType: 'httpHeaderAuth'`, and the matching `INSTCRED` / `ORGCRED` / `USERCRED` placeholder pair
- [ ] Idempotency check (search for existing PR / file) before each write, so retries don't duplicate
