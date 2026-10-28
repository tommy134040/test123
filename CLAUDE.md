# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Kanban board (`index.html`) for an internal IT PMO demo/training
tool — a neutral mockup, no real UOB branding/logos/systems. Everything
(HTML, CSS, JS) lives in that one file; there is no build step, no package
manager, and no backend other than a single outbound notification call.

`abc-it-pmo-kanban.html` is a **frozen historical snapshot** saved partway
through the project's evolution (pre-rebrand, pre-redesign) — it is not
actively maintained alongside `index.html` and shouldn't be edited when
making feature changes; update `index.html` only unless specifically asked
to touch the snapshot.

## Commands

- **Run locally:** open `index.html` directly in a browser — `open index.html`
  or `xdg-open index.html`. No install, no dev server.
- **CI (`.github/workflows/ci.yml`)**, runs on push/PR to `main`:
  - Checks `index.html` exists and has basic `<!DOCTYPE html>`/`<html>`/`</html>`
    structure.
  - Extracts the inline `<script>` block and validates it with
    `node --check` (syntax check only, no test runner/linter configured).
  To reproduce the JS syntax check locally:
  ```sh
  node -e "
    const fs = require('fs');
    const html = fs.readFileSync('index.html', 'utf8');
    const match = html.match(/<script>([\s\S]*)<\/script>/);
    fs.writeFileSync('_inline-script.js', match[1]);
  "
  node --check _inline-script.js
  ```
- **Deploy:** `.github/workflows/deploy-pages.yml` publishes `index.html` to
  GitHub Pages on every push to `main` (or manual dispatch). The repo's
  GitHub Pages source must be set to **GitHub Actions** (Settings → Pages)
  before the first deploy — this can't be done via the Actions API/token.

There are no other test/lint commands — don't invent an npm/build pipeline.

## Architecture (all in `index.html`)

The whole app is one `<script>` block operating on a single in-memory state
object — there is no framework, no modules, no router.

- **`state`** is the single source of truth: `tasks[]`, active `filters`,
  `taskIdCounter`, and `pendingDeleteId` (which card is showing the inline
  delete-confirm). Everything renders from this object.
- **No persistence whatsoever**: no `localStorage`/`sessionStorage`/cookies/
  IndexedDB. A page refresh always resets to the seeded demo data
  (`seedTasks()`) — this is intentional, not a bug, and the UI says so under
  the header.
- **Render flow**: mutate `state` → call `renderBoard()`, which calls
  `renderColumn()` per status (`STATUSES`) → `renderCard()` per task, plus
  `renderSummaryStrip()` for the header totals and `renderOverview()` for
  the status pie chart. There's no diffing/virtual DOM — renders just
  rebuild `innerHTML` from `state`.
- **Mutation functions** (`moveTask`, `deleteTask`, `addTask`, `applyFilters`,
  `clearFilters`) are the only places that change `state`; they each end by
  re-rendering. Drag-and-drop and the keyboard-accessible "Move ▸" control
  both funnel into `moveTask()`.
- **Event wiring** is centralized in `attachBoardEventListeners()`, which is
  re-run after every `renderBoard()` since cards are recreated from scratch
  each time (event delegation on rebuilt DOM, not persistent listeners).
- **Validation**: `validateForm()` does client-side validation for the
  "+ Add Task" modal before `addTask()` is called; errors are wired to their
  inputs via ARIA (`aria-invalid`/`aria-describedby`), and the modal has a
  focus trap plus return-focus-on-close (`getModalFocusable`,
  `setBackgroundInert`).
- **All user-supplied strings must go through `escapeHtml()`** before being
  interpolated into `innerHTML` — the card/column/chart renderers rely on
  this for XSS safety since there's no templating library doing it
  automatically.
- **Overview pie chart** (`renderOverview`, `pieSlicePath`,
  `polarToCartesian`): a hand-rolled inline SVG showing status totals
  (always unfiltered). Colors come from the `--chart-cat-1..4` custom
  properties — kept deliberately separate from the priority-pill colors
  (`--color-critical`/etc.) because reusing those put red (Blocked) next to
  green (Done), a colorblind-unsafe pairing caught by the dataviz skill's
  validator. Don't repurpose the priority palette for chart categories.
- **WhatsApp chatbot widget** (`chatbot-fab`/`chatbot-dialog` + the
  `renderChatbotSuggestions`/`buildWhatsAppUrl`/`useChatbotDialog`/
  `closeChatbotDialog` functions): a floating button that opens a dialog of
  suggested questions, each linking to `https://wa.me/<WHATSAPP_NUMBER>` with
  a prefilled message. `WHATSAPP_NUMBER` and `CHATBOT_SUGGESTIONS` are the
  config constants near the top of the `<script>` block. Entirely
  client-side — no backend, just outbound `wa.me` links.
- **Networking**: `notifyNewTask()` is the *only* fetch call the app makes —
  a fire-and-forget POST to `FORMSUBMIT_ENDPOINT` (a FormSubmit AJAX JSON
  endpoint) when a task is added. The UI is optimistic (card appears
  immediately regardless of the request's outcome), and a failed
  notification shows a non-blocking toast (`showToast()`) rather than
  breaking the board. `FORMSUBMIT_ENDPOINT` currently holds a placeholder
  address; a new address requires a one-time confirmation-link click from
  FormSubmit before notifications deliver. (The WhatsApp links above are
  plain `<a>` hrefs, not fetch calls.)

## Other repo contents

- **`.claude/agents/security-scanner.md`** — a project subagent that audits
  this specific repo (scoped to what it actually is: static, client-only, no
  server/auth/DB) and writes a DOCX report. Invoke it for a full security
  assessment; for a routine pre-push secrets check on a small diff, use the
  `security-review` skill instead (also what `/publish-repo` step 0 uses).
  `security-report-2026-10-06.docx` is a prior report's output, not a file
  to hand-edit.
- **`.agents/skills/` + `skills-lock.json`** — vendored third-party skills
  (`cybersecurity-analyst`, `frontend-design`, `ui-ux-pro-max`) pulled from
  external GitHub repos, each with its own large tree of reference data and
  scripts. Treat these as read-only vendored dependencies, not project code
  to modify; `skills-lock.json` pins their source and content hash.

## Repo-specific workflow

`.claude/commands/publish-repo.md` defines `/publish-repo`, a multi-step
command for pushing this project to a GitHub repo, wiring up README/CI/Pages,
capturing a screenshot, and updating the repo's About section — with a
mandatory security scan gate before anything is pushed. Use it (rather than
improvising the equivalent steps) when asked to publish or re-publish this
project.
