# registry-actions

GitHub/Gitea Actions for managing a [project registry](https://github.com/ddproxy/project-registry). Provides upsert semantics for both projects and versions — create on first run, patch on subsequent runs.

## Actions

### `actions/trigger-dispatch`

Triggers a `workflow_dispatch` event on a registry workflow via `gh workflow run --json -`. Used by callers to submit registration requests without requiring `contents:write` on the registry — the registry runs the write with its own `GITHUB_TOKEN`.

| Input | Required | Description |
|-------|----------|-------------|
| `workflow_name` | Yes | Workflow file to trigger (e.g. `upsert-version.yml`) |
| `inputs_json` | Yes | Compact JSON object of workflow inputs |
| `registry_repo` | No | Registry repo name (default: `project-registry`) |
| `registry_owner` | No | Registry repo owner (default: current owner) |
| `github_token` | Yes | Token with `actions:write` on the registry |

### `actions/upsert-project`

Creates or updates a project metadata record directly in the registry repository. Intended for use by the **registry's own workflows** only — requires `contents:write` on the registry.

| Input | Required | Description |
|-------|----------|-------------|
| `project_name` | Yes | Slug (lowercase, hyphens only) |
| `display_name` | No | Human-readable name |
| `description` | No | Short description |
| `repo_github` | No | GitHub repository URL |
| `repo_gitea` | No | Gitea repository URL |
| `tags` | No | Comma-separated tags |
| `license` | No | License identifier (default: `MIT` on create) |
| `registry_repo` | No | Registry repo name (default: `project-registry`) |
| `registry_owner` | No | Registry repo owner (default: current owner) |
| `github_token` | Yes | Token with `contents:write` on the registry |

### `actions/upsert-version`

Creates or updates a version record directly in the registry repository. Also syncs the version index and updates `latestVersion` in the project record on create. Intended for use by the **registry's own workflows** only.

| Input | Required | Description |
|-------|----------|-------------|
| `project_name` | Yes | Project slug |
| `version` | Yes | Version string (e.g. `v1.0.0`) |
| `changelog` | No | Newline-delimited list of changes |
| `asset_source` | No* | Source tarball URL (*required on create) |
| `asset_binary` | No | Binary download URL |
| `repo_tag_github` | No | GitHub tag URL |
| `repo_tag_gitea` | No | Gitea tag URL |
| `license` | No | License identifier (default: `MIT` on create) |
| `registry_repo` | No | Registry repo name (default: `project-registry`) |
| `registry_owner` | No | Registry repo owner (default: current owner) |
| `github_token` | Yes | Token with `contents:write` on the registry |

## Usage

### Automatic version registration on tag push

Build your metadata in a `run` step, then dispatch:

```yaml
- name: Build dispatch payload
  id: payload
  run: |
    echo "inputs_json=$(jq -cn \
      --arg project_name "my-project" \
      --arg version "${{ github.ref_name }}" \
      --arg asset_source "https://github.com/${{ github.repository }}/archive/refs/tags/${{ github.ref_name }}.tar.gz" \
      --arg repo_tag_github "https://github.com/${{ github.repository }}/releases/tag/${{ github.ref_name }}" \
      '{project_name:$project_name,version:$version,asset_source:$asset_source,repo_tag_github:$repo_tag_github}')" \
      >> $GITHUB_OUTPUT

- name: Dispatch upsert-version to registry
  uses: ddproxy/registry-actions/actions/trigger-dispatch@v0.1.0
  with:
    workflow_name: upsert-version.yml
    inputs_json: ${{ steps.payload.outputs.inputs_json }}
    github_token: ${{ secrets.REGISTRY_TOKEN }}
```

## Architecture

Repositories dispatch to the registry via `trigger-dispatch` — no `contents:write` needed on the registry, no secrets passed as workflow inputs. The registry's own workflows receive the dispatch and perform the write using their own `GITHUB_TOKEN`.

```
caller repo                     ddproxy/project-registry
─────────────────────────────   ──────────────────────────────────
trigger-dispatch  ──────────►  upsert-version.yml  (GITHUB_TOKEN)
  (actions:write only)           └─ upsert-version action
                                      └─ git commit + push
```

## Secrets and variables

| Name | Where | Purpose |
|------|-------|---------|
| `REGISTRY_TOKEN` | Caller repo secret | `actions:write` on `project-registry` — triggers dispatch, never writes directly |
| `GITEA_REGISTRY_TOKEN` | Gitea secret | `actions:write` on Gitea `registry` repo |
| `GITEA_BASE_URL` | GitHub org/repo variable | Gitea web base URL (e.g. `https://git.example.com`) |
| `GITEA_ORG` | GitHub org/repo variable | Gitea org slug (e.g. `com.example`) |

## Included workflows

This repository ships its own GitHub and Gitea workflow files for self-registration:

- `.github/workflows/upsert-project.yml` — manual dispatch to register or update this repo in the registry
- `.github/workflows/upsert-version.yml` — manual dispatch to patch a version record
- `.gitea/workflows/upsert-project.yml` — Gitea equivalent
- `.gitea/workflows/upsert-version.yml` — Gitea equivalent

For new repositories, see [project-template](https://github.com/ddproxy/project-template).
