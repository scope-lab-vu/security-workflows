# security-workflows

Org-wide required workflow for scope-lab-vu, enforced on every repo's default branch by the `security` org ruleset.

`.github/workflows/security.yml` runs one job on each pull request with three checks:

| Check | What fails the PR |
|---|---|
| [gitleaks](https://github.com/gitleaks/gitleaks) | A secret in any commit of the PR |
| [osv-scanner](https://github.com/google/osv-scanner) | A dependency with a known vulnerability that the PR *adds* (vulnerabilities already on the base branch are ignored) |
| [zizmor](https://github.com/zizmorcore/zizmor) | A high-severity issue (e.g. code injection, dangerous triggers) in a workflow or `action.yml` file the PR changes. Lower-severity findings are shown as annotations only. Tag pins like `@v4` are accepted. |

All three always run; the final "Result" step shows which one failed.

## False positives

- **gitleaks:** add the finding's fingerprint (shown in the failed check's log) to a `.gitleaksignore` file in the affected repo.
- **osv-scanner:** upgrade the dependency, or ask an org admin if no fixed version exists.
- **zizmor:** add an inline `# zizmor: ignore[<rule>]` comment on the flagged line.

## If a real secret is found

Rotate it. The secret is already on GitHub in the PR branch; blocking the merge does not un-leak it.
