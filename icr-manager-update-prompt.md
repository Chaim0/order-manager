# ICR Manager — feature update prompt (v1.8.2 → v1.9.0)

Paste this whole file as your first message to Claude Code (or reference it with
`@icr-manager-update-prompt.md`). It assumes the repo already contains the
project's `CLAUDE.md` (the "ICR Manager — Project Context" doc) — read that
first for the data model, paste format, sync protocol, and commit/versioning
conventions. Everything below builds on top of it.

The single HTML file may be named `order-manager.html` or `index.html`
depending on the repo — find it, don't assume.

## Ground rules

- Stays a single self-contained HTML file. No backend, no build step, no new
  external dependencies beyond what's already CDN-loaded (pako, qrcodejs).
- Don't break the existing schema, migration path (`migrateOrder`), or sync
  export/import format (`ICRv1:N/T:...`) — these changes are additive.
- Work through the list below autonomously. Where a design decision is needed,
  a decision is already made for you inline — implement it as specified
  rather than pausing to ask. Where you genuinely can't tell what's feasible
  (mainly the clipboard-read item), pick the most feasible approach and note
  the decision in your PR description / commit message instead of stopping.
- Commit per the project's existing conventions: one PR-sized body of work on
  the `dev` branch (or a few small logical PRs if that's cleaner — your call),
  one commit per distinct concern, `type: present-tense description` commit
  messages, version bump as its own final commit. Bump to `v1.9.0` (new
  features, no breaking data-format changes).

---

## 1. Click-to-open instead of click-to-copy, for ICR and CSA/CNC

Currently clicking an order ID (`copyOrderId`) or a CNC/CSA tag (`copyTag`)
copies the value to the clipboard. Change both to open a URL in a new tab
instead, built from a per-type URL template the operator configures (see
§2). Leave SKU tags as copy-to-clipboard — this change is only for the
order ID cell and the CNC/CSA tags.

- Each template is a plain string containing the literal token `{id}`. To
  build the URL, replace `{id}` with the order ID (for the ID column) or the
  CNC/CSA reference (for that column). If a template has no `{id}` token,
  just append the id to the end of the string.
- Open with `window.open(url, '_blank')` so the operator doesn't lose their
  place in the table.
- If the relevant template isn't configured yet, fall back to today's
  copy-to-clipboard behavior and show a notice pointing them at Settings
  (e.g. "Set an ICR URL in Settings to enable click-through").
- Keep the existing "highlight the row until the next click" affordance
  (`copiedOrderId`) for the ID column even though the click now navigates —
  it's still a useful "last one I opened" marker.

## 2. Settings panel

Add a consolidated Settings area (a `<details>` panel like the existing sync
panel, or a small modal opened from a gear/"Settings" button — your call on
which fits the existing visual style better). It should contain:

- Two text inputs: **ICR URL template** and **CSA/CNC URL template**, each
  saved to `localStorage` on change (new key, e.g. `order_manager_config_v1`,
  `{ icrUrlTemplate, cncUrlTemplate }`). Show the `{id}` placeholder in the
  input's placeholder text so the format is self-explanatory.
- The entire contents of the existing "Sync state between machines" panel
  (export/import/QR), moved in here rather than living at the top level.
- The **Clear all orders** button, moved out of the search row and into
  here (same confirmation modal as today).

The import-hint bar and the (reworked, see §6) Import button stay where they
are — those are everyday actions, not configuration.

## 3. Checkbox archive instead of confirm-and-remove

Replace the ✕ "archive" button on active rows with a checkbox:

- Unchecked = not archived. No confirmation modal on click (this is the
  point of the change — remove the modal for this action specifically; the
  delete-from-history ✕ and Clear all/Clear history modals stay as they are).
- Clicking it calls the existing `archiveOrder(id, 'manual')` immediately —
  the order really does move from `orders` to `history_orders` in
  `localStorage` right away, same as today's confirmed archive.
- But the row should **stay visible in the active table**, dimmed (reuse the
  `.archived-row` style) with the checkbox now checked, until the page is
  reloaded. On reload, `load()` reads from `localStorage` as normal and the
  order only shows up in the archived/history view from then on.
- Implementation: track a session-only `Set` of ids archived this session
  (like the existing `expandedSkuRows` pattern — not persisted). When
  building the active-row list for `renderTable()`, include any
  `history_orders` entries whose id is in that set alongside the real
  `orders` entries, marked as checked/dimmed. Don't let old archive events
  from a previous session resurrect a row.
- These "pending" rows can be treated as read-only after checking (no other
  interactions besides seeing the checked box) — don't worry about
  supporting unchecking/undo unless it falls out naturally.

## 4. Shown vs. total counters, filters as buttons

Add a counter next to the (moved) filter controls: the number of active
rows currently shown after search + column filters are applied. When search
or a filter narrows the list, show it as `shown (total)` — the shown count
at normal size, the total active-order count (excluding archived/history) in
a smaller font in parentheses right after. When nothing is filtering the
view (shown === total), just show the single number.

Move the Qty/SKU/Age filter controls off the table headers and onto buttons
on this same row, next to the counter. Reuse the existing `cycleFilter`
logic and badge labels (`QTY_BADGES` etc.) — just change the trigger element
from `<th onclick>` to a button.

Since the counter row now shows the active order count, drop the redundant
"Active orders" stat card from the stats row — keep "With customer orders"
and "History" as they are.

## 5. Clicking a table header sorts instead of filtering

Now that filtering lives in its own button row (§4), repurpose header
clicks for sorting:

- Sortable columns: Order ID (string), Lines/qty (numeric), Order /
  hasCustomerOrders (boolean), Date (via `earliestDate`, nulls sort last),
  SKUs (by line count), CNC/CSA (by tag count). In the history view, the
  Archived column sorts by `archivedAt`.
- Click a header once to sort ascending by that column; click the same
  header again to reverse; click a different header to switch columns
  (default to ascending on the new column). Show a small ▲/▼ indicator on
  the active sort column.
- Default/initial sort stays Order ID ascending, matching current behavior.
- Sort state is session-only (like `expandedSkuRows`), not persisted.
- Sorting and the column filters/search apply together — filter and search
  first, then sort the remaining rows.

## 6. Import revamp: one button, read the clipboard directly

Rename "Commit import" to just **Import**. Preferred behavior:

- On click, attempt `navigator.clipboard.readText()` (this is allowed
  without a separate permission prompt because it's triggered by a user
  gesture — the click). If it returns a valid ICR export JSON payload
  (same shape the current paste handler checks for) that isn't already
  fully in the buffer, parse it via the existing `parseJsonImport` and merge
  it into the buffer — then immediately run the existing commit logic
  (diff against active orders, auto-archive missing ones, etc.) in the same
  click. This collapses "paste each partial view, then click Commit" into
  "copy a view, click Import" repeated per view, since the buffer keeps
  accumulating across clicks in a session rather than clearing after each
  commit (only "Clear buffer" or you explicitly resetting it clears it) —
  that's what keeps the disappeared-order detection accurate across
  multiple partial views.
- Give the button a visible state change — color/border, a small pending
  badge, brief flash, your call — both when the clipboard currently holds an
  unimported valid payload and right after a successful import, so the
  operator gets feedback without a modal.
- If clipboard read isn't available (permissions denied, insecure context,
  clipboard doesn't contain JSON), fall back to today's flow: keep the
  global paste listener filling the buffer, and let the (renamed) Import
  button just commit what's in the buffer, same as "Commit import" does
  today. Note in your PR description which path you ended up shipping and
  why — this genuinely depends on what the browser allows here, don't spend
  time asking about it.

## 7. Fix wrong CSA/CNC linking

There have been bugs where an ICR line links to the wrong CSA/CNC order.
Look at `parseJsonImport` / `normalizeDocRef`: it strips the CSA/CNC prefix
and all non-digit characters from both `sourceDocumentNo` and the CSA
order's `no`, then matches on the remaining digits alone. That throws away
the prefix, so a CNC reference and a CSA reference that happen to share the
same digit run would incorrectly match each other.

Rework this to match on the full reference — prefix plus digits,
case-insensitive, ignoring whitespace and (for CSA numbers) an optional
trailing letter — so a CNC reference can only match a CNC-type order and a
CSA reference can only match a CSA-type order. Every ICR line already
carries its own CSA/CNC tag via `sourceDocumentNo`; don't add SKU-based
matching as a fallback or signal anywhere in this logic, it's unnecessary
and out of scope. If you find a sample of the real bookmarklet JSON payload
in the repo, use it to confirm the exact reference format instead of
guessing further from the current field names.

Keep the `matchStats` reporting (`matched` / `sourceLines` / `customerOrders`
/ `icrs`) working the same way so the post-commit notice still shows
accurate auto-link counts.

---

## Before you finish

- Smoke-test by hand: open the file in a browser (a quick
  `python3 -m http.server` in the repo dir works fine, or just open it
  directly), and click through each changed area — redirect fallback with
  no template set, settings save/reload, checkbox-archive-then-refresh,
  counters with an active filter, header sort both directions, import with
  and without clipboard access, a couple of CSA/CNC linking cases.
- Bump the version string (title tag + `const VERSION`) to `v1.9.0` as the
  final commit.
