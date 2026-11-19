---
description: Publish this project to a GitHub repo (push, README, About, Pages, CI/CD) after a sensitive-data scan
argument-hint: <github repo URL or owner/repo>
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
---

Publish this project to GitHub. Repo info from the user: `$ARGUMENTS`

If `$ARGUMENTS` is empty, ask the user for the repo (`owner/repo` or URL) and stop until they answer.

Follow the steps in order. **Do not push anything until Step 2 passes and the user has confirmed.**

## Step 0 - Preflight
1. Normalise the repo to `owner/repo` (accept `https://github.com/o/r(.git)`, `git@github.com:o/r.git`, or `o/r`).
2. Check `gh --version` and `gh auth status`. If `gh` is missing or not logged in, tell the user to run `gh auth login` and stop.
3. Check `gh repo view <owner/repo>`. If it doesn't exist, ask whether to create it (and public or private) before running `gh repo create`. GitHub Pages on a free account needs a **public** repo.
4. If this folder isn't a git repo, `git init -b main`. Read `CLAUDE.md` and the project files so later steps match what the project really is.

## Step 1 - Sensitive-data scan (everything that would be uploaded)
Scan the **working tree and full git history** (`git log -p --all`) for:
- Credentials: API keys, tokens (`ghp_`, `github_pat_`, `AKIA`, `sk-`, `xox[bap]-`), passwords, private keys (`-----BEGIN ... PRIVATE KEY-----`), connection strings, `Authorization`/`Bearer` values.
- Files that must never be committed: `.env*`, `*.pem`, `*.key`, `*.p12`, `id_rsa*`, `credentials*`, `secrets*`, `*.sqlite`, `*.db`, `.claude/settings.local.json`, `node_modules/`, OS/editor junk.
- Personal data: real email addresses, phone numbers, home paths like `C:\Users\<name>`, internal hostnames/IPs, real customer or employee data.
- Project-specific rules from `CLAUDE.md` (e.g. an email may appear only in one config constant; no real trademarks/logos). Check that those rules still hold.

Use Grep/`git grep`/`git log -p -S` and report each hit as `file:line` with the value **masked**.

Handling:
- Hits in the working tree: fix them (move to config/placeholder/`.gitignore`) or ask the user.
- Hits in git history: **stop and tell the user.** Cleaning requires rewriting history; never force-push without explicit approval. A leaked secret must be rotated regardless.
- An email used as a public form endpoint (e.g. FormSubmit) is exposed by design: flag it and let the user decide whether to keep a placeholder.
- Ensure a `.gitignore` exists covering the files above (create/extend it).

Finish with a short report: PASS / FAIL, and what was checked. Ask the user to confirm before pushing. On FAIL, do not continue.

## Step 2 - Push the code
1. Review `git status`; stage only intended files (no `git add -A` blindly), commit with a clear message ending with the attribution line required by the session.
2. `git remote` - add `origin https://github.com/<owner/repo>.git` or update it if different. Don't overwrite a different existing remote without asking.
3. `git push -u origin main`. Never use `--force`. If the remote has unrelated history, report it and ask.

## Step 3 - README.md
Create or edit (preserve any good existing content) so it contains: title and one-line description, live demo link (filled in at Step 6), features, how to run locally, tech stack/constraints, project structure, deployment (Pages + CI), and any configuration the user must do (e.g. replacing a placeholder email). Keep it accurate to the code; don't invent features. Contain no secrets or personal data.

## Step 4 - GitHub Pages
- Enable Pages with the Actions source: `gh api -X POST repos/<owner/repo>/pages -f build_type=workflow` (if it already exists, use `-X PUT`).
- Ensure the workflow publishes only the files the site needs (for this project, `index.html`) - never the whole repo, to avoid publishing stray files.

## Step 5 - CI/CD with GitHub Actions
In `.github/workflows/`:
- `pages.yml`: deploy to Pages on push to `main` (+ `workflow_dispatch`) using `actions/configure-pages`, `upload-pages-artifact`, `deploy-pages`, with minimal permissions. Keep it if it already exists and is correct.
- `ci.yml`: on push and pull_request, run checks suited to the stack. For a no-build static site: verify `index.html` exists, no external resource URLs (CDN/fonts), no `localStorage`/`sessionStorage`/`alert(`/`confirm(` if the project forbids them, and a basic secret-pattern grep. Fail the build on violations.
- Pin actions to major versions; set `permissions: contents: read` unless more is needed.
Commit and push the workflow changes, then watch the run: `gh run watch` / `gh run list`. If it fails, read `gh run view --log-failed`, fix, and re-push.

## Step 6 - Repo About + Pages link
After the first successful deploy, get the URL: `gh api repos/<owner/repo>/pages --jq .html_url`.
Set the About section:
`gh repo edit <owner/repo> --description "<short accurate description>" --homepage "<pages url>" --add-topic <topic> ...`
Choose 3-6 relevant topics. Description must contain no secrets or personal data. Then put the same URL in the README's live-demo link, commit and push.

## Step 6b - Live-site screenshot (Playwright MCP)
Once the Pages URL returns 200:
1. With the Playwright MCP tools, `browser_navigate` to the live Pages URL, `browser_resize` to 1280x720, then `browser_take_screenshot` (`type: png`, `fullPage: true`) saving to `docs/screenshot.png` (absolute path). Read the image to confirm it shows the real, fully rendered site.
2. Embed it in the README under the live-demo link: `![Screenshot of the live site](docs/screenshot.png)` (add `docs/screenshot.png` to the project-structure tree if listed).
3. If the project has an assessment/report `.docx` with an empty "Paste your screenshot" box, insert the image there (e.g. `python-docx`: `cell.paragraphs[0].add_run().add_picture("docs/screenshot.png", width=Inches(5.9))`). Never overwrite other content in the file.
4. Commit `docs/screenshot.png` and the README change and push. Keep the Pages workflow publishing only `index.html`; the screenshot is for the README only.
If Playwright MCP is unavailable, report that plainly and skip rather than faking an image.

## Step 7 - Final verification and report
- Open the live URL check: `curl -sI <pages url>` returns 200.
- Re-run a quick scan on what is now on GitHub (`git ls-files`, plus a grep of the tracked files) to confirm nothing sensitive was published.
- Report: repo URL, Pages URL, Actions status, scan result, and any follow-ups the user must do manually (e.g. rotate a secret, activate FormSubmit email).

## Rules
- Never print secret values; mask them.
- Never force-push, delete branches/repos, or change repo visibility without explicit approval.
- Do not put the user's personal email in README, About, commits of new files, or workflow files unless it is already the sanctioned config constant.
- If any step fails, report it plainly rather than skipping it.
