---
description: Push the current project to a GitHub repo, set up README/Pages/CI, add a screenshot, update the About section, and security-scan before anything goes out
argument-hint: <github-repo-url>
---

# Publish current project to GitHub

Target repo: `$ARGUMENTS`

If no repo link was given above, ask the user for it before doing anything else —
do not guess a repo or default to an already-attached one.

Parse the link into `owner` and `repo` (accept `https://github.com/<owner>/<repo>`,
`git@github.com:<owner>/<repo>.git`, or a bare `<owner>/<repo>`).

Work through the steps below **in order**. Each step is a checkpoint — if one
fails, stop and report it rather than pushing ahead to the next step with a
half-finished result. Give the user a one-line status update after each step
that changes repo state (push, workflow added, About updated), not after every
tool call.

## 0. Security scan — before anything is pushed

This gates every later step. Nothing in this command may push code until this
passes.

1. Run the `security-review` skill (via the Skill tool) against the project's
   working tree to catch real secrets and credential leaks, not just pattern
   matches.
2. In addition, grep the working tree yourself for the patterns a security
   skill can miss in config/data files: `.env`/`.env.*` files, `*.pem`,
   `*.key`, `id_rsa*`, AWS-style keys (`AKIA[0-9A-Z]{16}`), generic
   `api[_-]?key`, `secret`, `password`, `token` assignments with non-placeholder
   values, and any `.git-credentials`.
3. Check `git status` for anything that looks like it shouldn't be tracked
   (build artifacts, `node_modules`, `.DS_Store`, local env files) and flag it
   — don't silently `.gitignore` it without telling the user what you excluded.
4. If you find anything that looks like a real secret: **stop**, do not
   proceed to step 1, and tell the user exactly what file/line triggered it.
   Let them decide whether to remove it, rotate it, or confirm it's a
   placeholder before you continue.
5. If the scan is clean, say so in one line and continue.

## 1. Attach and connect the target repo

1. Check whether `owner/repo` is already in this session's GitHub scope. If
   not, call `add_repo` with `access: "push"`.
   - If `add_repo` fails with a cross-tier/owner-mismatch error, that means
     this session is already locked to a different repo owner and cannot
     attach this one. Tell the user plainly that a **new session** is needed
     with this repo as the initial source — don't try workarounds.
2. If a fresh clone is required, follow the clone instructions `add_repo`
   returns exactly (timeout, concurrency caveats, etc.) rather than improvising
   a different clone command.
3. Determine the repo's default branch (`git ls-remote --symref origin HEAD`
   on the clone, or the branch GitHub reports).

## 2. Push the current project's code

1. If the target repo is **empty**, commit the current project's files
   straight to its default branch.
2. If the target repo **already has content**, do not blindly overwrite it:
   - If it looks like an earlier version of this same project, work on a
     branch and open a PR rather than force-pushing over history.
   - If it looks unrelated to this project, stop and ask the user how they
     want to reconcile the two before pushing anything.
3. Never force-push, never skip hooks, never rewrite existing history on the
   remote. Use normal commits.
4. Push to the default branch (or a feature branch + PR if step 2 called for
   that), using the retry-with-backoff pattern for transient network errors.

## 3. Create or update README.md

Write (or refresh) `README.md` at the repo root covering, at minimum:
- What the project is and who it's for, in 1-3 sentences.
- How to run it locally (match the project's actual stack — don't invent a
  build step that doesn't exist; e.g. a static `index.html` just needs "open
  in a browser").
- Key features, briefly.
- Any one-time setup the project needs (e.g. FormSubmit activation, API keys
  to configure) — pull this from the project's own code/config, not from
  assumptions.

Don't overwrite an existing README's accurate content wholesale — read it
first and merge in what's missing rather than replacing a human-authored
README that already covers this.

## 4. GitHub Actions CI/CD

1. Check `.github/workflows/` for existing workflows first; update rather than
   duplicate.
2. Add a CI workflow appropriate to the project's actual stack (lint/build/test
   on push and PR). For a plain static-HTML project with no build tooling,
   this can be as light as an HTML/link-check step — don't invent a Node/test
   pipeline the project doesn't have.
3. If the project is a static site suitable for GitHub Pages, add a
   `deploy-pages.yml` using `actions/configure-pages`,
   `actions/upload-pages-artifact`, and `actions/deploy-pages`, triggered on
   push to the default branch plus `workflow_dispatch`. Do **not** pass
   `enablement: true` to `configure-pages` — the workflow's `GITHUB_TOKEN`
   cannot create a Pages site via the API (confirmed: it 403s with "Resource
   not accessible by integration"); creating the site requires the one-time
   manual step in step 5 below.

## 5. Enable and verify GitHub Pages

1. Trigger the deploy workflow (push, or `workflow_dispatch` via the Actions
   API) and check its conclusion.
2. If it fails because Pages isn't enabled yet (`Get Pages site failed` /
   `Not Found`), tell the user the exact manual step required and why it
   can't be automated:
   - Repo → **Settings → Pages → Source → GitHub Actions**.
3. After the user confirms they've done that, re-trigger the workflow and
   confirm it succeeds before reporting a live URL. Never hand back a Pages
   URL you haven't confirmed resolves via a successful deploy run.
4. The resulting URL is `https://<owner>.github.io/<repo>/` (or the custom
   domain if one is configured) — confirm which applies.

## 6. Capture a screenshot and add it to README.md

1. Look for a connected Playwright MCP tool (search if not already loaded)
   and use it if available.
2. If no Playwright MCP tool is connected — check first rather than assuming
   — fall back to the Playwright *npm package* driving the pre-installed
   Chromium directly via Bash/Node (install with
   `npm install playwright --no-save --prefix <scratch-dir>` if not already
   present; the browser binary is already at
   `/opt/pw-browsers/chromium-*/chrome-linux/chrome` — do not
   `playwright install`).
3. Try navigating to the live Pages URL confirmed in step 5. If the session's
   egress proxy denies that host (403/`connect_rejected` — check
   `$HTTPS_PROXY/__agentproxy/status` if unsure), do not retry or route
   around it. Fall back to rendering the local file directly via a
   `file://` URL on the project's entry point (e.g. `index.html`) — same
   content, no network dependency, and more reliable for a static site
   anyway.
4. Save the screenshot into the repo (e.g. `screenshot.png` at the repo
   root) and reference it near the top of `README.md` with a Markdown image
   tag, close to the live-demo link.
5. Commit and push both files together.

## 7. Update the repository "About" section

1. Look for a GitHub MCP tool that updates repo metadata (description,
   homepage, topics) — search for it if not already loaded (e.g. tool names
   containing "update_repository" or "repository update"). Use it to set:
   - **Description**: a short, accurate one-liner for the project.
   - **Topics**: a few relevant tags (language/stack, project type).
2. If no such tool is available or it's denied, fall back to the GitHub REST
   API via `gh api` (`PATCH /repos/{owner}/{repo}` for description/homepage,
   `PUT /repos/{owner}/{repo}/topics` for topics). If that path is also
   blocked by the session's proxy ("Repository settings writes are not
   permitted through this proxy" — confirmed behavior, don't retry it), tell
   the user which fields need setting and give them the exact manual steps
   (gear icon next to "About" on the repo page) rather than silently
   skipping it.

## 8. Add the Pages link to the About section

Once step 5 has confirmed a working Pages URL, set the repo's `homepage`
field to that URL using the same mechanism as step 7 (don't guess the URL —
use the one confirmed live).

## Final report

End with a short summary (not a wall of logs):
- Repo pushed to, and branch/PR if applicable.
- Confirmation the security scan passed (or what was flagged and how it was
  resolved).
- CI workflow status.
- Live Pages URL (only if confirmed working).
- Whether the screenshot came from the live URL or a local fallback render.
- What got set in the About section.
- Anything still waiting on the user (e.g. a manual Settings step).
