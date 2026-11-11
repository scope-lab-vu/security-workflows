# security-workflows

Org-wide required workflow for scope-lab-vu, enforced on every repo's default branch by the `security` org ruleset.

`.github/workflows/security.yml` runs one job on each pull request with three checks:

| Check | What fails the PR |
|---|---|
| [gitleaks](https://github.com/gitleaks/gitleaks) | A secret in any commit of the PR. Uses the default rules plus org-specific rules (Google/Mapbox/MongoDB/lab-service credentials, committed `.env` files) defined in the workflow. Files over 5 MB are skipped. |
| [osv-scanner](https://github.com/google/osv-scanner) | A known vulnerability that the PR *adds*. Vulnerabilities already on the base branch are ignored, and upgrading to a version that is still affected by the same advisory passes. |
| [zizmor](https://github.com/zizmorcore/zizmor) | A high-severity issue (e.g. code injection, dangerous triggers) in a workflow or `action.yml` file the PR changes. Lower-severity findings are shown as annotations only. Tag pins like `@v4` are accepted. |

All three always run; the final "Result" step shows which one failed. Tool downloads are pinned and checksum-verified.

## False positives

- **gitleaks:** add the finding's fingerprint (shown in the failed check's log) to a `.gitleaksignore` file in the affected repo. A `.env.example` with only placeholders is the common case.
- **osv-scanner:** upgrade the dependency, or ask an org admin if no fixed version exists.
- **zizmor:** add an inline `# zizmor: ignore[<rule>]` comment on the flagged line.

## If a real secret is found

**1. Rotate it first.** Revoke or regenerate it with the provider. Once it's on GitHub, assume it's been copied; rewriting history doesn't make it safe. Store the new value in a GitHub Actions secret or an untracked `.env`.

**2. Only in your PR branch (not merged)?** Squash the branch's commits into one without the secret, then force-push the branch. The check passes once no commit in the PR contains it. GitHub still keeps the old commits in the PR's history, which is why step 1 matters.

<details>
<summary>Commands</summary>

```bash
git fetch origin
git reset --soft origin/main         # undo the branch's commits, keep the changes staged
# remove the secret from the files (or git rm --cached the file and add it to .gitignore)
git commit -m "..."
git push --force-with-lease
```

</details>

**3. Merged or public? Rewrite history.** Tell collaborators first: every commit ID changes and open PRs break.

<details>
<summary>Commands</summary>

```bash
brew install git-filter-repo        # or: pipx install git-filter-repo
git clone --mirror https://github.com/scope-lab-vu/REPO.git && cd REPO.git

# Replace the value everywhere (prompted, so it stays out of shell history)
read -rs SECRET
git filter-repo --replace-text <(printf 'literal:%s==>REDACTED\n' "$SECRET") \
                --replace-message <(printf 'literal:%s==>REDACTED\n' "$SECRET")

# Or remove a whole file (private keys, credential JSON)
git filter-repo --force --invert-paths --path path/to/key.pem

gitleaks git . --redact             # verify nothing is left
```

</details>

**4. Push (needs an org admin).** Rulesets block force-pushes. Ask an admin to add the repo to the exclude list of the `security` and `require-pr` rulesets, push, and have them remove the exclusion right after.

<details>
<summary>Commands</summary>

```bash
git remote add origin https://github.com/scope-lab-vu/REPO.git
git push --force origin 'refs/heads/*:refs/heads/*' 'refs/tags/*:refs/tags/*'
```

</details>

**5. Clean up.**
- Everyone re-clones; pushing an old branch puts the secret back.
- Ask [GitHub Support](https://support.github.com/contact) to purge cached commits and PR refs ([guide](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)).
- Forks keep their own copy.

## Scanning your own repos

The PR check only scans new commits. To find secrets already in a repo's history, scan every branch, tag and PR ref. These commands use the same rules as the org check, pulled from `security.yml`.

<details>
<summary>Get the org rules (once)</summary>

```bash
brew install gitleaks gh             # gh auth login if you haven't
gh api repos/scope-lab-vu/security-workflows/contents/.github/workflows/security.yml --jq .content \
  | base64 -d | awk '/<<.TOML.$/{f=1;next} /^ *TOML$/{f=0} f' | sed 's/^ *//' > ~/gitleaks-org.toml
```

</details>

<details>
<summary>Scan one repo (full history, all refs)</summary>

```bash
git clone --mirror https://github.com/OWNER/REPO.git /tmp/scan-REPO
gitleaks git /tmp/scan-REPO --config ~/gitleaks-org.toml --log-opts="--all" --redact --verbose

# Also scan uncommitted files in your working copy
gitleaks dir /path/to/your/checkout --config ~/gitleaks-org.toml --redact --verbose
```

</details>

<details>
<summary>Scan all your repos</summary>

```bash
mkdir -p ~/leakscan && cd ~/leakscan
gh repo list YOUR_USERNAME --limit 1000 --json nameWithOwner -q '.[].nameWithOwner' | while read -r r; do
  git clone -q --mirror "https://github.com/$r.git" "${r//\//_}"
  gitleaks git "${r//\//_}" --config ~/gitleaks-org.toml --log-opts="--all" --redact \
    --report-format json --report-path "${r//\//_}.json" >/dev/null 2>&1 \
    || echo "FINDINGS: $r (see ${r//\//_}.json)"
done
```

Reports are redacted; the file, line and commit in each finding show where to look. Delete `~/leakscan` when you're done.

</details>

For each real finding, follow [If a real secret is found](#if-a-real-secret-is-found).

## Pre-commit hook (optional)

Catch secrets on your machine before they reach GitHub. This installs one global git hook that scans staged changes in every repo you commit to. It uses `~/gitleaks-org.toml` from [Get the org rules](#scanning-your-own-repos) if present, otherwise the gitleaks default rules.

<details>
<summary>Install (once per machine)</summary>

```bash
brew install gitleaks
mkdir -p ~/.git-hooks
cat > ~/.git-hooks/pre-commit <<'EOF'
#!/bin/sh
# gitleaks on staged changes; org rules if present, else gitleaks defaults
cfg="$HOME/gitleaks-org.toml"
if [ -f "$cfg" ]; then set -- --config "$cfg"; else set --; fi
if ! gitleaks git --pre-commit --staged --redact --verbose "$@"; then
  echo "gitleaks: secret in staged changes. Remove it, or use git commit --no-verify for a false positive."
  exit 1
fi
# A global hooksPath replaces the repo's own hooks; run its pre-commit too
repo_hook="$(git rev-parse --git-dir)/hooks/pre-commit"
[ -x "$repo_hook" ] && exec "$repo_hook"
exit 0
EOF
chmod +x ~/.git-hooks/pre-commit
git config --global core.hooksPath ~/.git-hooks
```

To uninstall: `git config --global --unset core.hooksPath`.

</details>

- `git commit --no-verify` skips it; the PR check still runs.
- Repos that set their own `core.hooksPath` (e.g. Husky) override the global hook.
- Re-run the "Get the org rules" command when the org rules change.

## Changing this workflow

Changes go through a PR here; the ruleset applies to this repo too. To bump a tool, update its version and SHA-256 together in `security.yml` (from the release's checksums file, or the asset digest on the release page).
