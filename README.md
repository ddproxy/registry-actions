# Registry Actions

A collection of GitHub Actions to manage a project registry.

## Actions

### `actions/add-project`
Adds a new project metadata record to a project registry repository.

### `actions/add-version`
Adds a new project version record to a project registry repository.

### `actions/trigger-dispatch`
Triggers a workflow dispatch in a registry repository. Useful for integrating from other repositories' CI/CD pipelines.

## Usage in Workflows

```yaml
- name: Add Version
  uses: ddproxy/registry-actions/actions/add-version@main
  with:
    project_name: 'my-project'
    version: 'v1.0.0'
    changelog: 'Initial release'
    asset_source: 'https://github.com/user/repo/archive/refs/tags/v1.0.0.tar.gz'
    github_token: ${{ secrets.REGISTRY_TOKEN }}
```

## Configuration Overrides

The actions are designed to be generic. You can override the registry repository and owner:

- `registry_repo`: Defaults to `project-registry`.
- `registry_owner`: Defaults to the current repository owner.
