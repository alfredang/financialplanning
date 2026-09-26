---
description: Security-scan the project, then push to GitHub, write the README and About section, deploy GitHub Pages via Actions, and link the live site on the repo
argument-hint: "[repo-name] [--private]"
allowed-tools: Bash(git:*), Bash(gh:*), Bash(gitleaks:*), Bash(grep:*), Bash(find:*), Bash(ls:*), Bash(cat:*), Bash(du:*), Bash(sleep:*), Bash(curl:*), Read, Write, Edit, Glob, Grep, mcp__playwright__browser_navigate, mcp__playwright__browser_resize, mcp__playwright__browser_wait_for, mcp__playwright__browser_evaluate, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

# Publish this project to GitHub

Arguments: `$ARGUMENTS`
- First argument (optional): the repo name. Default: the existing `origin` repo, or the current folder name if there is no remote.
- `--private`: create the repo as private (default is public). Note that GitHub Pages on a private repo needs a paid plan, so warn the user if they pass this.

Work through the steps **in order**. Step 1 is a hard gate: if it fails, stop and report. Nothing leaves this machine until it passes.

Current state, for context:
- Branch: !`git branch --show-current`
- Remote: !`git remote -v`
- Status: !`git status --short`
- gh auth: !`gh auth status 2>&1 | head -3`

---

## Step 0: Preflight

1. Confirm `git` and `gh` are available and `gh auth status` shows a logged-in account. If not, stop and tell the user to run `gh auth login`.
2. If this isn't a git repo, run `git init -b main`.
3. Work out `OWNER` (`gh api user -q .login`) and `REPO` (from the argument, the `origin` URL, or the folder name).

## Step 1: Security scan (gate, runs before anything is uploaded)

Scan **everything that would be pushed**: the working tree, staged files and the full git history.

1. **Secret scanner.** If `gitleaks` is installed, run:
   - `gitleaks detect --source . --redact --no-banner` (the history)
   - `gitleaks detect --source . --no-git --redact --no-banner` (the working tree, including untracked files)
2. **Built-in pattern scan** (always run it, even if gitleaks passed). Use Grep over the files that would be committed (tracked files plus untracked files not ignored: `git ls-files --cached --others --exclude-standard`) and look for:
   - Private keys: `-----BEGIN (RSA|EC|OPENSSH|DSA|PGP)? ?PRIVATE KEY-----`
   - AWS keys: `AKIA[0-9A-Z]{16}`, `aws_secret_access_key`
   - GitHub tokens: `gh[pousr]_[A-Za-z0-9]{36,}`, `github_pat_[A-Za-z0-9_]{80,}`
   - Anthropic/OpenAI keys: `sk-ant-[A-Za-z0-9_-]{20,}`, `sk-[A-Za-z0-9]{32,}`
   - Google API keys: `AIza[0-9A-Za-z_-]{35}`
   - Slack tokens: `xox[baprs]-[A-Za-z0-9-]{10,}`
   - Stripe keys: `sk_live_[A-Za-z0-9]{20,}`
   - Generic assignments: `(api[_-]?key|secret|token|password|passwd)\s*[:=]\s*['"][^'"]{8,}['"]` (case-insensitive)
   - Connection strings with credentials: `[a-z]+://[^:\s/]+:[^@\s]+@`
   - JWTs: `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}`
3. **History check:** `git log --all -p | grep -nE '<same patterns>'`. A secret that was committed and later deleted is still public once pushed.
4. **Sensitive files:** flag any committed or to-be-committed `.env*` (except `.env.example`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.sqlite`/`*.db`, `credentials*.json`, `service-account*.json`, `.claude/settings.local.json`, `.playwright-mcp/`.
5. **`.gitignore`:** make sure it exists and covers the patterns in item 4. Add any that are missing.
6. **Large files:** flag anything over 50 MB (GitHub rejects files over 100 MB).
7. **Personal data:** flag real-looking email addresses, phone numbers or street addresses that aren't known placeholders (for example `*.example` domains, or content that CLAUDE.md lists as fictional).

**Result:**
- If anything is found, **stop**. Show a table with file, line, finding type and a redacted snippet (never print the full secret). Suggest a fix: remove the value, move it to an env var, add it to `.gitignore`, and if it's in history, rewrite history (`git filter-repo`) and rotate the credential. Don't continue until the user confirms it's fixed, then rerun the whole scan.
- If a finding is clearly a false positive (a placeholder or example value), list it and say why you treated it as safe.
- If nothing is found, print `Security scan: PASSED` with what was checked, and continue.

## Step 2: Commit and push to GitHub

1. `git add -A`, then review `git status` once more to make sure nothing sensitive is staged.
2. If there are changes, commit them with a clear, descriptive message.
3. If there's no `origin` remote, create the repo and push:
   `gh repo create OWNER/REPO --public --source . --remote origin --push` (use `--private` if the user passed it).
4. Otherwise run `git push -u origin <branch>`. If the push is rejected as non-fast-forward, **don't force-push**. Run `git pull --rebase` and ask the user if there are conflicts.

## Step 3: Capture a screenshot with Playwright

Use the Playwright MCP tools (`mcp__playwright__*`) to take a fresh screenshot of the site for the README. If the Playwright MCP server isn't available, keep any existing screenshot, say so in the report and move on.

1. Resize the browser to a desktop viewport: `browser_resize` to 1440 × 900.
2. Navigate to the site. Use the entry file as a `file:///` URL (for example `file:///C:/path/to/project/index.html`, with forward slashes). If `file://` is blocked, use the live Pages URL instead once it exists.
3. Wait for the page to settle: `browser_wait_for` about 3 seconds so web fonts, images and any entrance animations finish. If the page has animated counters or fade-ins, use `browser_evaluate` to jump them to their final state (for example add the `visible` class to `.fade-in` elements and set each counter to its final value) so the capture doesn't show half-finished numbers.
4. Take a viewport screenshot (not full-page) with `browser_take_screenshot`, type `png`, saved to `docs/screenshot.png` (create `docs/` if needed). Pass the absolute path as the filename so it lands in the project and not in `.playwright-mcp/`.
5. Close the browser with `browser_close`, then open the image with Read to check it shows the page properly (no blank areas, missing images or cookie banners). Retake it if not.
6. Make sure `.playwright-mcp/` is in `.gitignore` so Playwright's scratch files are never committed.

## Step 4: Create or update the README

Read the project (the entry files, CLAUDE.md and any existing README.md) so the README describes what is actually there. Then create `README.md`, or update it, keeping any sections the user wrote by hand. It should include:

- A title and a one-line tagline
- A **live demo link**: `https://OWNER.github.io/REPO/` (for a repo named `OWNER.github.io` it's `https://OWNER.github.io/`)
- The screenshot from Step 3 (`docs/screenshot.png`) with descriptive alt text
- Features: short bullets taken from the real content
- The tech stack
- Getting started: how to run it locally
- The project structure
- Deployment: GitHub Pages through GitHub Actions
- A note on placeholder or fictional content, if that applies
- A license section, only if a LICENSE file exists

Commit and push it, rerunning the Step 1 pattern scan on the changed files first.

## Step 5: GitHub Pages through GitHub Actions

1. Create or update `.github/workflows/pages.yml`:
   - Triggers: `push` to the default branch, plus `workflow_dispatch`
   - Permissions: `contents: read`, `pages: write`, `id-token: write`
   - `concurrency: { group: pages, cancel-in-progress: false }`
   - Steps: `actions/checkout@v4`, `actions/configure-pages@v5` (with `enablement: true`), `actions/upload-pages-artifact@v3`, `actions/deploy-pages@v4`
   - **Publish only the site files.** For a static site, copy the public files (for example `index.html` and any asset folders) into `_site/` in a build step and upload `_site`. Don't upload `.`, because that would also publish `CLAUDE.md`, `.claude/`, workflow files and so on. For a project that has a build step, upload its build output folder.
2. Enable Pages with the Actions build type (if the first call fails because Pages already exists, use the second):
   - `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`
   - `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`
3. Commit and push the workflow, then follow the run: `gh run list --workflow pages.yml -L 1` and `gh run watch <id> --exit-status`.
4. If the run fails, read `gh run view <id> --log-failed`, fix the problem and push again.
5. Get the live URL with `gh api repos/OWNER/REPO/pages -q .html_url` and check it with `curl -sI <url>`, which should return 200. It can take a minute to go live.

## Step 6: Update the repo About section and add the Pages link

Run all of this in one call:

```
gh repo edit OWNER/REPO \
  --description "<one-line summary, max ~120 chars>" \
  --homepage "<Pages URL from Step 5>" \
  --add-topic <topic1> --add-topic <topic2> ...
```

- Write the description from the project itself, not a generic template.
- Add 5 to 10 relevant lowercase topics (for example `html`, `css`, `javascript`, `github-pages`, `static-site`, plus topics about the domain).
- The `--homepage` flag is what puts the Pages link in the About panel. Confirm it with `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`.
- Make sure the README's live demo link matches this URL.

## Step 7: Report

Finish with a short summary:
- The security scan result (passed, or what was fixed)
- Whether the screenshot was refreshed
- The repo URL
- The live GitHub Pages URL and its HTTP status
- The About section: description, homepage and topics
- The commits that were pushed
- Anything that was skipped or needs the user to act

Rules: never force-push, never print full secrets, never pass `--no-verify`, and ask before any action that can't be undone, such as rewriting history or changing a public repo to private.
