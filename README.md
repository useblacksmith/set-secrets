<p align="center">
  <picture>
    <!-- Dark mode -->
    <source media="(prefers-color-scheme: dark)" srcset="./wordmark-white.svg" width="300">
    <!-- Light mode -->
    <source media="(prefers-color-scheme: light)" srcset="./wordmark-black.svg" width="300">
    <img alt="Blacksmith Logo" src="./wordmark-black.svg" width="300">
  </picture>
</p>

A GitHub Action that sets [code]smith sandbox env vars from a CI job. Use it to hand [code]smith short-lived credentials your CI already mints, such as an AWS CodeArtifact token from an OIDC role, instead of storing a long-lived key in [code]smith settings.

> [!NOTE]
> Preview (v0.1): this action must be enabled for your organization. Contact Blacksmith support to turn it on.

# How it works

The action runs on a Blacksmith runner and authenticates with the token Blacksmith gives every job. No secrets or extra permissions are needed in your workflow.

Blacksmith looks up which repository the job runs in. It then writes each value to a **Team** env var that is:

- **scoped to that repository only.** [code]smith sessions in other repositories never see it.
- **sensitive.** The value is write-only. Nobody can read it back in the dashboard or API.
- **created on first use and replaced on later runs.** The env var has no expiry. Refresh it on a schedule shorter than the credential's lifetime.

All env vars in one call are written together: if any is refused, none is changed. New [code]smith turns pick up the latest values when they start.

# Usage

```yaml
- id: mint
  run: |
    token=$(mint-my-token)
    echo "::add-mask::${token//%/%25}"
    echo "token=$token" >> "$GITHUB_OUTPUT"
- uses: useblacksmith/set-secrets@v0.1
  with:
    secrets: |
      MY_TOKEN=${{ steps.mint.outputs.token }}
      MY_API_KEY=${{ secrets.MY_API_KEY }}
```

Each line is `NAME=value`, split at the first `=`. Blank lines are ignored and whitespace around names and values is trimmed. Values must be single-line.

Mask a value in the step that produces it, as above. GitHub logs action inputs before this action runs, so its own masking comes too late to hide an unmasked step output. Values from `secrets.*` are already masked.

## Inputs

| Input     | Description                                                                                  | Required |
| --------- | -------------------------------------------------------------------------------------------- | -------- |
| `secrets` | 1 to 10 `NAME=value` lines. Names use letters, digits and underscores. Values up to 8KB each. | Yes      |

## Outputs

| Output    | Description                                                 |
| --------- | ----------------------------------------------------------- |
| `created` | JSON array of the env var names this run created            |
| `updated` | JSON array of the env var names whose value this run replaced |

Read them with `fromJSON`, e.g. `contains(fromJSON(steps.set.outputs.created), 'MY_TOKEN')`.

# Example: AWS CodeArtifact

CodeArtifact tokens last at most 12 hours. This workflow mints one every 6 hours with the same OIDC role your CI already uses, then hands it to [code]smith. Sandboxes in this repository can then install private packages.

```yaml
name: Refresh [code]smith CodeArtifact token

on:
  schedule:
    - cron: "0 */6 * * *"
  workflow_dispatch:

permissions:
  id-token: write # for AWS OIDC
  contents: read

concurrency: codesmith-codeartifact-token

jobs:
  refresh:
    runs-on: blacksmith-2vcpu-ubuntu-2404
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account-id>:role/<ci-role>
          aws-region: <region>

      - id: codeartifact
        run: |
          token=$(aws codeartifact get-authorization-token \
            --domain <domain> --domain-owner <account-id> \
            --duration-seconds 43200 \
            --query authorizationToken --output text)
          echo "::add-mask::$token"
          echo "token=$token" >> "$GITHUB_OUTPUT"

      # uv reads its index credentials from a separate variable.
      - uses: useblacksmith/set-secrets@v0.1
        with:
          secrets: |
            CODEARTIFACT_AUTH_TOKEN=${{ steps.codeartifact.outputs.token }}
            UV_INDEX_CODEARTIFACT_PASSWORD=${{ steps.codeartifact.outputs.token }}
```

Run it once with **Run workflow** so the env vars exist before the first schedule fires. Values that never change, such as `UV_INDEX_CODEARTIFACT_USERNAME=aws`, can be added once in [code]smith settings.

If your IAM role's trust policy only allows certain branches or events, it must allow this workflow. Scheduled runs use the default branch, so their OIDC subject is `repo:<owner>/<repo>:ref:refs/heads/<default-branch>`.

# Security

- Any job on a Blacksmith runner in a repository can replace that repository's env vars. Call this action only from trusted triggers such as `schedule`, `workflow_dispatch` or `push` to your default branch. Do not call it from `pull_request` workflows. In public repositories, require approval for workflows from fork pull requests.
- The action can't touch other repositories' env vars, personal env vars, or a Team env var shared by several repositories.
- Values are encrypted at rest. The action masks every value in the job log, and its errors name the line number, never the line.

# Errors

Any error leaves every env var unchanged.

| HTTP status | Meaning                                                                                                                                  |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 401         | The job's Blacksmith token is missing or invalid.                                                                                        |
| 403         | The job has ended, its repository could not be resolved, or the runner's token cannot be tied to a job. Contact support for the last.    |
| 404         | The action is not enabled for your organization.                                                                                         |
| 409         | A Team env var with one of these names is shared by several repositories. Scope it to this repository alone in [code]smith settings.     |
| 422         | An invalid name or value, an existing env var that is not marked sensitive, or the Team env var limit would be exceeded.                 |
| 503         | Blacksmith could not resolve the repository's owner. Retry later.                                                                        |

# License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
