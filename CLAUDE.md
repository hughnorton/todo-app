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
  UI preferences (mode, sort, filters, optional sync URL) are in a second key
  `todo.ui.v1` and are deliberately not part of backups.
- State shape: `{ items: [], links: {...}, lastBackupAt, dirtySince }`.
  Task fields: `id, name, details, priority (1–10, 10 = highest), p10Date
  ("YYYY-MM-DD" or null), bdayDate ("MM-DD" or null), tags, done, createdAt,
  completedAt, snoozeUntil (ms timestamp or null)`.
- `normalizeItem()` / `normalizeLink()` / `normalizeLinks()` migrate any older or
  backup shape on load and on restore — keep them tolerant of missing fields.

## Three modes: Admin / Read / Watch (added 2026-09-17)

A `modeSeg` switch at the top of **Current** and **List** picks the mode (`ui.mode`,
shared between the two tabs). **Admin** is the original task app. **Read** and
**Watch** are lists of saved links (articles/tweets → Read, videos and tweets with a
video → Watch), fed by the Gmail link tracker on Hugh's PC
(`..\Gmail_Enhance.py` writes `links.json`; docs in `..\Gmail Link Tracker - README.md`).

- `state.links = { items: [link], order: { read: [ids], watch: [ids] }, catalogAt, importedAt }`.
  Link fields: `id, url, kind (Video|Article|Tweet|Other), list (read|watch), listUser,
  title, source, minutes (number|null), summary, date ("YYYY-MM-DD"), addedAt, manual,
  done, completedAt, removed, notes`.
- **`id` is derived from the URL** (`linkId()`: `yt:<videoId>`, `tw:<statusId>`, else
  `url:host/path?sorted-query` with tracking params stripped). It mirrors `link_id()` in
  `Gmail_Enhance.py` — **keep the two in sync** so a link added on the phone and the
  same link emailed later merge into one item.
- **Import** (`importCatalog`): the phone's `done / notes / order / removed / listUser`
  win; the file wins on title, source, minutes, summary. Non-manual links missing from
  the file are pruned (Hugh archived the email). `removed` links stay as hidden records
  so an import can't resurrect them. `manual` links (added on the phone) are never
  pruned; when the same link later arrives from Gmail it is upgraded in place.
  A links.json pasted/chosen in the task *Restore* box is also routed to `importCatalog`.
- Import paths: "Import file…" on the List tab (Read/Watch) or in the tools panel
  (`#linkFile` → iOS Files → OneDrive → To Do → links.json), or an optional **sync URL**
  (`ui.syncUrl`, fetched with `cache: "no-store"` on open / foreground when older than
  30 min, plus "Refresh"). No URL is set by default — hosting links.json anywhere
  public would publish Hugh's reading list, so that is Hugh's call (a secret Gist raw
  URL was the suggested option). `links.json` is in `.gitignore`.
- **Current (Read/Watch)**: sort chips (My order / Shortest / Longest / Newest),
  length-bucket chips (Read: ≤5 / 5–15 / 15+ min; Watch: ≤15 / 15–45 / 45+), Read
  also has Articles / Tweets chips. A one-line summary ("12 to watch · 11h 22m in
  total"). Cards: kind badge, source · length · date, title (tap → link sheet),
  3-line blurb (notes if any, else summary), buttons Open ↗ / ✓ Done / Later ⤓
  (Later = move to bottom of My order).
- **List (Read/Watch)**: status line + Refresh (only with a sync URL) + Import file…;
  same chips; rows with check circle, title, meta and — only in "My order" —
  ⤒ ▲ ▼ reorder buttons (`moveLink`; `ensureOrder` first materialises the current
  order for pending items so moves are well defined). Completed section + "Clear
  completed" (sets `removed`, doesn't delete).
- **Add tab**: Task / Link toggle (`addKind`), auto-picked from the mode when you open
  the tab. Link form: URL, optional title (looked up via noembed.com if blank, best
  effort), Read/Watch chips (typing a YouTube URL flips to Watch), minutes, notes.
  Duplicate URL → toast; if it was done/removed it is restored instead.
- **Link sheet** (`#linkSheet`, shares `#overlay` with the task sheet via
  `showSheet()`): Open link, title, Read/Watch (sets `listUser`), minutes, full
  summary, notes, Move to top / Move to bottom / Mark done, Save, Remove.
- Backups now include `links` (`version: 5`); "Restore (replace)" restores them too when
  present. `backupOverdue()` counts link changes as well as task changes.

## Admin mode features (unchanged from 2026-07-25)

- **Current**: shows the single top task by *effective* priority
  (`effPrio` = 10 once `p10Date` has arrived, else base priority; ties → oldest
  first). Buttons: Done, 1 Hour / 1 Day / 1 Week / 1 Month (snooze — hides the task
  from Current until then), Reduce Priority (−1; on a date-escalated task it instead
  sets priority 9 and clears `p10Date`, otherwise the date would keep it pinned at 10).
- **List**: filter chips (All / Admin / Read / Watch / Bday — these are *task tags*,
  unrelated to the Read/Watch link lists), active tasks sorted by effective priority,
  collapsible Completed section, "Clear completed", Backup & restore panel at the
  bottom (shared by all modes).
- **Add**: name, details, priority picker, optional "Priority 10 date"
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
  (`backupOverdue()`), the Current tab's Admin view replaces the top task with a
  "Backup To Do List" pseudo-task whose only button is "Email backup"; other
  tabs/modes show a banner. Email backup opens the iOS share sheet with a **JSON file
  attachment** (Hugh's explicit preference over a pre-filled "To" — the share sheet
  can't pre-fill a recipient, and he chose attachment > pre-fill; his address
  `hughnorton7@gmail.com` is `BACKUP_EMAIL`, used in the mailto fallback and toast).
  Restore accepts pasted email-body text (between `-----` fences) or a chosen file;
  "Restore (replace)" swaps the whole list, "Import & add" merges pasted tasks into
  the existing list (fresh ids on collision). Minimal import JSON works:
  `{"app":"todo","items":[{"name":"X","tags":["Bday"],"bdayDate":"08-06"}]}` —
  `normalizeItem` fills the rest.

## Deploy workflow

1. Edit files locally (this folder is the repo).
2. **Always bump `CACHE` in `sw.js`** (`todo-vN`) with any change, or phones keep
   the stale cached version. Currently `todo-v8`. The SW only handles same-origin
   GETs (cross-origin lookups / sync URL bypass it).
3. Smoke check: `run_test.py` pattern — copy `index.html` + `links.json` to a scratch
   folder, append a `<script>` that drives the functions and writes results into a
   `<pre id="TESTOUT">`, serve with `http.server`, run headless Edge
   (`msedge.exe --headless=new --virtual-time-budget=15000 --dump-dom`) and grep the
   output. No browser extension (Hugh declined it); he tests for real on his phone.
4. Commit and push to `master` (repo-local git identity already set). GitHub CLI
   needs its full path: `& "C:\Program Files\GitHub CLI\gh.exe"` — authenticated
   as `hughnorton`. Never commit `links.json` or backups (`.gitignore`).
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
- Hugh's real birthday/anniversary tasks (Dad's, Seb's, Jeremy's, Jen's, Mum's,
  Lachie's birthdays, Christmas, Emily's birthday, Valentine's Day, Mother's Day)
  were imported into his live app via "Import & add" — that's runtime data, not
  in this repo, so it won't show up in the code.
