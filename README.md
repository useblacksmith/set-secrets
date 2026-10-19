<p align="center">
  <picture>
    <!-- Dark mode -->
    <source media="(prefers-color-scheme: dark)" srcset="./wordmark-white.svg" width="300">
    <!-- Light mode -->
    <source media="(prefers-color-scheme: light)" srcset="./wordmark-black.svg" width="300">
    <img alt="Blacksmith Logo" src="./wordmark-black.svg" width="300">
  </picture>
</p>

> [!WARNING]
> **Alpha.** This action and its API may change without notice. Contact Blacksmith support to enable it for your organization.

A GitHub Action that sets [code]smith and Proof sandbox env vars from a CI job. Hand them short-lived credentials your CI already mints, such as an AWS CodeArtifact token from an OIDC role, instead of storing a long-lived key in [code]smith settings.

# How it works

The action runs on a Blacksmith runner and authenticates with the token Blacksmith gives every job, so your workflow needs no secrets or extra permissions. Each value becomes a **Team** env var that is:

- **Org-wide.** [code]smith and Proof sessions in every repository see it. Give a value meant for one repository its own name, such as `API_CODEARTIFACT_TOKEN`.
- **Sensitive.** It is write-only: nobody can read it back in the dashboard or API.
- **Created or replaced on each run, with no expiry.** Refresh it more often than the credential expires.
- **Managed by CI.** [code]smith settings link it to the workflow that set it and block edits, since the next run would overwrite them. You can still delete it. [code]smith sessions see the workflow too, so they can rerun it when a value expires.

Each call is all-or-nothing: if any env var is refused, none changes. New [code]smith turns and Proof runs pick up the latest values when they start.

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

Each line is `NAME=value`, split at the first `=`, with whitespace trimmed and blank lines ignored. Values must be single-line.

Mask a value in the step that produces it, as above: GitHub logs action inputs before the action runs, so its own masking is too late. `secrets.*` values are already masked.

| Name                         | Description                                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------------- |
| `secrets` (input, required)  | 1 to 10 `NAME=value` lines. Names use letters, digits and underscores. Values up to 8KB each. |
| `created` (output)           | JSON array of the env var names this run created                                              |
| `updated` (output)           | JSON array of the env var names whose value this run replaced                                 |

Read outputs with `fromJSON`, e.g. `contains(fromJSON(steps.set.outputs.created), 'MY_TOKEN')`.

# Example: AWS CodeArtifact

CodeArtifact tokens last up to 12 hours. This workflow mints one every 6 hours with your CI's OIDC role and hands it to [code]smith and Proof.

```yaml
name: Refresh CodeArtifact token

on:
  schedule:
    - cron: "0 */6 * * *"
  workflow_dispatch:

permissions:
  id-token: write

jobs:
  refresh:
    runs-on: blacksmith-2vcpu-ubuntu-2404
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account-id>:role/<ci-role>
          aws-region: <region>
      - id: mint
        run: |
          token=$(aws codeartifact get-authorization-token --domain <domain> \
            --domain-owner <account-id> --query authorizationToken --output text)
          echo "::add-mask::$token"
          echo "token=$token" >> "$GITHUB_OUTPUT"
      - uses: useblacksmith/set-secrets@v0.1
        with:
          secrets: CODEARTIFACT_AUTH_TOKEN=${{ steps.mint.outputs.token }}
```

Run it once with **Run workflow** so the token exists before the first scheduled run. Your role's trust policy must allow the default branch, which scheduled runs use.

# Security

- Only jobs triggered by your repository's own code, such as `push`, `schedule` and `workflow_dispatch`, can set env vars. Every pull request event is refused, even from branches of the same repository, as are Dependabot jobs.
- Any such job in any of your organization's repositories can replace these env vars. Review who can push to default branches and run workflows.
- The action can't touch personal env vars or repository-scoped Team env vars.
- Values are encrypted at rest and masked in the job log. Errors name the line number, never the line.

# License

MIT. See [LICENSE](LICENSE).
