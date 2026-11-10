# security-workflows

Org-wide required workflows for scope-lab-vu, enforced via org rulesets.

- `.github/workflows/gitleaks.yml` — scans each PR's commits for secrets with [gitleaks](https://github.com/gitleaks/gitleaks). Required by the `gitleaks` org ruleset.

## False positives

Add the finding's fingerprint (shown in the failed check's log) to a `.gitleaksignore` file in the affected repo.

## If a real secret is found

Rotate it. The secret is already on GitHub in the PR branch; blocking the merge does not un-leak it.
