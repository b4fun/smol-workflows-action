# smol-workflows GitHub Action

Run an [smol-workflows](https://github.com/b4fun/smol-workflows) workflow from a GitHub Actions job. The action installs a pinned `smol-wf` release and runs the selected workflow script from the caller's checked-out repository.

## Usage

```yaml
steps:
  - uses: actions/checkout@v4

  # Install and authenticate the provider CLI used by your workflow before this step.
  # For example, configure provider credentials using repository or organization secrets.

  - name: Run workflow
    id: smol-wf
    uses: b4fun/smol-workflows-action@v1
    with:
      workflow: ./workflows/review.mjs
      provider: pi
      smol-wf-version: v0.3.4
      args: |
        target: the current pull request

  - name: Display run ID
    run: echo "smol-wf run ${{ steps.smol-wf.outputs.run-id }} completed"
```

`actions/checkout` is required because the workflow script is read from the caller's workspace. The selected provider executable and its authentication must also be available on the runner. This action installs `smol-wf`; it does not install or authenticate provider CLIs. Provider-specific setup varies, so add those steps before invoking this action and pass credentials through `env` using GitHub Secrets.

The runner needs Bash, `curl`, Mike Farah's `yq`, and `jq`.

Set `smol-wf-version: latest` to install the latest stable smol-wf release. This resolves the version each time the action runs; use an explicit release tag when you want runs to stay on the same CLI version.

The default provider is `debug`, which does not call an LLM. Supported provider names are `debug`, `codex`, `claude-code`, `opencode`, and `pi`.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `workflow` | Yes | — | Workflow script path relative to `working-directory`. |
| `provider` | No | `debug` | smol-wf agent provider. |
| `smol-wf-version` | No | `v0.3.4` | Versioned smol-wf release tag to install, or `latest` to resolve the latest stable release at run time. |
| `args` | No | `{}` | Workflow arguments as a YAML mapping. Nested mappings and lists are supported. |
| `working-directory` | No | `.` | Workflow working directory relative to `GITHUB_WORKSPACE`. |
| `db-path` | No | Runner temp directory | SQLite database path. Relative paths resolve from `GITHUB_WORKSPACE`. |

## Outputs

| Output | Description |
| --- | --- |
| `result-file` | Path to the JSON run report on the runner. It contains `runID`, token usage, and workflow results. |
| `run-id` | ID of the completed smol-wf run. |
| `db-path` | Path to the SQLite database containing durable run state. |

The result and default database are under `$RUNNER_TEMP/smol-wf/` and are removed when the runner job ends. Upload them with `actions/upload-artifact` if you need to retain the report or resume the run in a later job. To resume, restore and use the same SQLite database.

```yaml
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: smol-wf-run
    path: |
      ${{ steps.smol-wf.outputs.result-file }}
      ${{ steps.smol-wf.outputs.db-path }}
    if-no-files-found: warn
```

## Release and Marketplace

The root `action.yml` is the Marketplace action manifest. Publish a GitHub release from this repository and tag compatible action versions (for example, `v1.0.0`, plus a moving `v1` major tag). Consumers can pin the full release tag or commit SHA. The `smol-wf-version` input separately selects the CLI release installed by the action.

## Tests

The `Test action` workflow runs the action with the `latest` CLI alias, a nested YAML argument mapping, and the built-in `debug` provider. It checks that the structured values survive argument parsing and appear in the JSON run report; no provider credentials or paid model calls are needed.

## License

MIT
