# To-Do App (Hugh's personal iPhone PWA)

Single-user to-do app for Hugh's iPhone only — no app store, no accounts, no server.
Live at **https://hughnorton.github.io/todo-app/** (GitHub Pages, public repo
`hughnorton/todo-app`, branch `master`, GitHub user `hughnorton`). Hugh uses it as a
home-screen web app installed via Safari's "Add to Home Screen".

## Architecture

- **Everything is in `index.html`** — inline CSS and vanilla JS, no build step, no
  dependencies. `sw.js` (service worker) + `manifest.webmanifest` + `icon-*.png`
  make it an installable, offline-capable PWA.
- **All data lives in `localStorage`** on the phone (key `todo.v1`) — by explicit
  choice, nothing leaves the device. Loss protection is manual email backups.
- State shape: `{ items: [], lastBackupAt, dirtySince }`. Item fields: `id, name,
  details, priority (1–10, 10 = highest), p10Date ("YYYY-MM-DD" or null),
  bdayDate ("MM-DD" or null), tags, done, createdAt, completedAt,
  snoozeUntil (ms timestamp or null)`.
- `normalizeItem()` migrates any older/backup item shape on load and on restore —
  keep it tolerant of missing fields when adding new ones.

## Features (as of 2026-07-22)

- **Three tabs** (bottom bar): **Current**, **List**, **Add Task**.
- **Current**: shows the single top task by *effective* priority
  (`effPrio` = 10 once `p10Date` has arrived, else base priority; ties → oldest
  first). Buttons: Done, 1 Hour / 1 Day / 1 Week / 1 Month (snooze — hides the task
  from Current until then), Reduce Priority (−1; on a date-escalated task it instead
  sets priority 9 and clears `p10Date`, otherwise the date would keep it pinned at 10).
- **List**: filter chips (All / Admin / Read / Watch / Bday), active tasks sorted by
  effective priority, collapsible Completed section, "Clear completed",
  Backup & restore panel at the bottom.
- **Add Task**: name, details, priority picker, optional "Priority 10 date"
  (task auto-escalates to P10 on that date), tag chips.
- **Detail sheet** (tap any task name): edit everything, un-snooze, Mark Uncomplete
  (for completed tasks), delete.
- **Bday tasks** (recurring birthdays/events): selecting the `Bday` tag swaps the
  priority + P10-date controls for date pickers. `bdayDate` is either "MM-DD"
  (fixed date) or "RMM-N-W" (Nth weekday rule: month MM, N 1–4 or 5=last,
  W 0=Sun..6=Sat — e.g. Mother's Day = 2nd Sunday of May = "R05-2-0").
  Priority is fully automatic (`bdayPrio`): 1 when >91 days out, linear up to 10
  at ≤14 days.
  "Done" / the check circle never completes them — `rollBday()` snoozes to the day
  after the date, so they recur yearly. Lists show "🎂 1 Feb · Nd" countdown.
  Priority badges show the bare number (no "P" prefix); tags/snooze/countdown share
  one `metaRow` line to keep list rows compact.
- **Backup**: when last backup >48h old AND there are unbacked changes
  (`backupOverdue()`), the Current tab replaces the top task with a "Backup To Do
  List" pseudo-task whose only button is "Email backup"; other tabs show a banner.
  Email backup opens the iOS share sheet with a **JSON file attachment**
  (Hugh's explicit preference over a pre-filled "To" — the share sheet can't
  pre-fill a recipient, and he chose attachment > pre-fill; his address
  `hughnorton7@gmail.com` is `BACKUP_EMAIL`, used in the mailto fallback and toast).
  Restore accepts pasted email-body text (between `-----` fences) or a chosen file;
  "Restore (replace)" swaps the whole list, "Import & add" merges pasted items into
  the existing list (fresh ids on collision). Minimal import JSON works:
  `{"app":"todo","items":[{"name":"X","tags":["Bday"],"bdayDate":"08-06"}]}` —
  `normalizeItem` fills the rest.

## Deploy workflow

1. Edit files locally (this folder is the repo).
2. **Always bump `CACHE` in `sw.js`** (`todo-vN`) with any change, or phones keep
   the stale cached version. Currently `todo-v7`.
3. Smoke check: `python -m http.server 8642 --directory .` then fetch
   `index.html` and grep for the new element IDs (no browser automation available —
   Hugh declined the Chrome extension; he tests on his phone).
4. Commit and push to `master` (repo-local git identity already set). GitHub CLI
   needs its full path: `& "C:\Program Files\GitHub CLI\gh.exe"` — authenticated
   as `hughnorton`.
5. Verify the deploy by polling `https://hughnorton.github.io/todo-app/` until the
   response contains a marker string from the new version (~20–60 s).

## Conventions & gotchas

- iOS PWA quirks matter: no `height: 100%` on body (it broke scrolling —
  content rubber-banded back behind the tab bar); keep generous bottom padding for
  the fixed tab bar; `env(safe-area-inset-*)` everywhere; date inputs need
  `-webkit-appearance: none`.
- Light/dark theme via CSS variables + `prefers-color-scheme` — style both.
- Tags are the hardcoded `TAGS` array (Admin / Read / Watch / Bday); `TAG_RENAMES`
  in `normalizeItem` migrates old stored/backup tags (Reading→Read, Watching→Watch).
  A tag-management UI may be requested later.
- Hugh described himself as non-developer-ish: explain trade-offs in plain terms,
  flag anything that risks his data, and let him test on the phone before piling on
  more changes.
