# UOB IT PMO — Kanban Board

A single-page Kanban board for an internal IT PMO demo/training tool. It's a
neutral, non-production mockup — no real UOB branding, logos, or systems are
used.

**Live demo:** https://tommy134040.github.io/test123/

## Running it locally

No build step, no install. Just open `index.html` directly in a browser
(double-click it, or `open index.html` / `xdg-open index.html`).

## Features

- Four-column Kanban board (Backlog, In Progress, Blocked, Done) with native
  HTML5 drag-and-drop between columns, plus a keyboard-accessible "Move ▸"
  control on each card for mouse-free use.
- Cards show task ID, title, project/workstream, assignee, a priority pill
  (color-coded by border), due date, and a category tag. Overdue tasks get a
  subtle warning badge.
- Inline, non-native delete confirmation ("Delete? Yes / No") on each card.
- "+ Add Task" modal with client-side validation, an auto-generated
  `UOB-ITPM-####` task ID, and optimistic UI — the card appears immediately
  while a notification email is sent in the background.
- Filter bar (project, assignee, priority) and a live summary strip (totals,
  per-status counts, overdue count) in the header.
- Seeded with 8 realistic demo tasks so the board isn't empty on first load.

## Data model and persistence

Board state lives only in an in-memory JavaScript array — there's no
`localStorage`, cookies, or backend. **Refreshing the page resets the board to
the seeded demo data.** This is intentional for a demo tool, and the UI notes
it directly under the header.

## Task notifications (FormSubmit)

Submitting the "Add Task" form sends a notification via
[FormSubmit](https://formsubmit.co)'s AJAX JSON endpoint — the only network
call this app makes, and the only backend it has.

- **Config constant:** `FORMSUBMIT_ENDPOINT` near the top of the `<script>`
  block in `index.html`. Swap in a real email address there to redirect
  notifications; it currently holds a placeholder.
- **One-time activation:** the first submission ever sent to a new address
  triggers a confirmation email from FormSubmit to that address.
  Notifications won't deliver until the link in that email is clicked.
- A failed notification never breaks the board — the card stays, and a
  non-blocking warning toast appears instead.

## Deployment

`.github/workflows/deploy-pages.yml` publishes `index.html` to GitHub Pages
on every push to `main` (or via manual dispatch). GitHub Pages must have its
source set to **GitHub Actions** once, under repo Settings → Pages, before
the first deploy can succeed.
