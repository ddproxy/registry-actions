# registry-actions

GitHub/Gitea Actions for managing a [project registry](https://github.com/ddproxy/project-registry). Provides upsert semantics for both projects and versions — create on first run, patch on subsequent runs.

## Actions

### `actions/upsert-project`

Creates or updates a project metadata record in the registry.

- **Create** — builds a full record; auto-generates `displayName` from the slug if blank, defaults `license` to MIT.
- **Update** — only the fields you provide are changed; everything else is left untouched.

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
| `github_token` | Yes | Token with write access to the registry |

### `actions/upsert-version`

Creates or updates a version record in the registry. Also syncs the version index and updates `latestVersion` in the project record on create.

- **Create** — `asset_source` is required; `changelog` defaults to an empty list, `license` to MIT.
- **Update** — only the fields you provide are changed; version index is kept in sync automatically.

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
| `github_token` | Yes | Token with write access to the registry |

## Usage

### Automatic version registration on tag push

```yaml
- name: Upsert version in registry
  uses: ddproxy/registry-actions/actions/upsert-version@v0.1.0
  with:
    project_name: my-project
    version: ${{ github.ref_name }}
    changelog: |
      Fix critical bug in parser
      Improve startup performance
    asset_source: https://github.com/${{ github.repository }}/archive/refs/tags/${{ github.ref_name }}.tar.gz
    repo_tag_github: https://github.com/${{ github.repository }}/releases/tag/${{ github.ref_name }}
    github_token: ${{ secrets.REGISTRY_TOKEN }}
```

### Patching an existing record (e.g. adding a Gitea URL later)

```yaml
- name: Add Gitea URL to project
  uses: ddproxy/registry-actions/actions/upsert-project@v0.1.0
  with:
    project_name: my-project
    repo_gitea: https://git.example.com/org/my-project
    github_token: ${{ secrets.REGISTRY_TOKEN }}
```

## Secrets and variables

| Name | Where | Purpose |
|------|-------|---------|
| `REGISTRY_TOKEN` | GitHub secret | Write access to `project-registry` |
| `GITEA_REGISTRY_TOKEN` | Gitea secret | Write access to Gitea `registry` repo |
| `GITEA_BASE_URL` | GitHub org/repo variable | Gitea web base URL (e.g. `https://git.example.com`) |
| `GITEA_ORG` | GitHub org/repo variable | Gitea org slug (e.g. `com.example`) |
| `GITHUB_MIRROR_URL` | Gitea repo variable | Full GitHub repo URL for the mirror |

## Included workflows

This repository ships its own GitHub and Gitea workflow files for self-registration:

- `.github/workflows/upsert-project.yml` — manual dispatch to register or update this repo in the registry
- `.github/workflows/upsert-version.yml` — manual dispatch to patch a version record
- `.gitea/workflows/upsert-project.yml` — Gitea equivalent
- `.gitea/workflows/upsert-version.yml` — Gitea equivalent

For new repositories, see [project-template](https://github.com/ddproxy/project-template).
