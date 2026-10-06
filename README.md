# trmnl-todo

Pushes your open iPhone Reminders to a [TRMNL](https://docs.trmnl.com)
private plugin via its webhook API — no server, no Google-style service
account. An iOS Shortcuts automation on your phone reads Reminders and POSTs
straight to TRMNL on a schedule.

```
iPhone Reminders  --(Shortcuts automation)-->  TRMNL plugin
```

Unlike [trmnlBdays](https://github.com/ricjd/trmnlBdays) and
[trmnlAsburyEvents](https://github.com/ricjd/trmnlAsburyEvents), there's no
Cloudflare Worker here. Apple doesn't expose a public REST API for iCloud
Reminders the way Google does for Calendar, but the Shortcuts app can read
Reminders locally and call the webhook directly — so the "backend" is a
Shortcut living on your phone instead of a cron job in the cloud. The
tradeoff: it only runs (and only updates the display) while your phone is
reachable and the automation actually fires — see **Notes** below.

## 1. TRMNL: create the private plugin

1. In TRMNL, go to **Plugins → Private Plugin → Add New**.
2. Set **Strategy** to **Webhook**.
3. Open the **Markup Editor**. It has one tab per layout size — paste the
   matching file into each tab you plan to use:
   - **Full** (800x480) → [`trmnl/full.liquid`](trmnl/full.liquid)
   - **Half vertical** (400x480) → [`trmnl/half_vertical.liquid`](trmnl/half_vertical.liquid)
   - **Half horizontal** (800x240) → [`trmnl/half_horizontal.liquid`](trmnl/half_horizontal.liquid)
   - **Quadrant** (400x240) → [`trmnl/quadrant.liquid`](trmnl/quadrant.liquid)

   You only need to fill in the tab(s) matching the sizes you'll actually
   use in a playlist. Use the live preview in each tab to check for
   clipping and tweak `slice: 0, N` near the top of each template to taste.
4. Copy the plugin's **webhook URL** (looks like
   `https://usetrmnl.com/api/custom_plugins/<uuid>`) — you'll need it in
   the Shortcut below. Note TRMNL's webhook limits on a free account: **12
   requests/hour** and **2KB per payload** — keep the automation's interval
   at an hour or more, and cap how many reminders it sends (12–15 fits
   comfortably).

## 2. Build the Shortcut

Open the **Shortcuts** app on your iPhone → **+** → build the following
actions in order (exact wording varies slightly by iOS version, but the
actions are all built-in, no third-party imports needed):

1. **Find Reminders** — Filter: `Completed` is `No`. Optionally add a
   condition to restrict to one or more specific lists. Sort by `Due Date`,
   ascending.
2. **Get First 12 Items from List** (or similar wording, e.g. "Get Item
   from List" → "First 12") — caps the payload before you build JSON, so it
   stays under the 2KB limit.
3. **Set Variable** `Todos` = an empty list (use the **List** action with
   no items, then **Set Variable**).
4. **Repeat with Each** item in the reminders from step 2. Inside the loop:
   - **Format Date** on the reminder's `Due Date`, style **Relative** →
     this gives you strings like "Today", "Tomorrow", "in 3 days", "3 days
     ago". Save as `Due Word`. If the reminder has no due date, this step
     is skipped — leave it blank.
   - **Format Date** on `Due Date`, style **Short** (e.g. "MMM d") → save
     as `Due Short`.
   - **If** `Due Date` is before **Current Date** → **Set Variable**
     `Overdue` to `true`, **Otherwise** `false`.
   - **Dictionary** action with these key/value pairs:
     `title` = Repeat Item's `Title`, `list` = Repeat Item's `List` →
     `Name`, `due` = `Due Short`, `due_word` = `Due Word`, `overdue` =
     `Overdue`.
   - **Add to Variable** `Todos` (appends this dictionary to the list).
5. After the loop ends, **Dictionary** action for the outer payload:
   `merge_variables` → nested **Dictionary** with `todos` = `Todos`
   variable (as an Array), `plugin_title` = `Todo`, `generated_at` =
   Current Date formatted as **ISO 8601**.
6. **Get Contents of URL**:
   - URL: paste the TRMNL webhook URL from step 1.
   - Method: `POST`.
   - Headers: `Content-Type` = `application/json`.
   - Request Body: `JSON`, set to the dictionary from step 5.
7. Name the Shortcut something like **Push Todos to TRMNL** and save it.

Run it once manually from the Shortcuts app (tap it in the list) to confirm
it completes without an error and doesn't prompt for anything unexpected.

## 3. Automate it

1. In Shortcuts, go to the **Automation** tab → **+** → **Create Personal
   Automation**.
2. Trigger: **Time of Day** → pick a start time → set it to **Repeat**
   hourly (or whatever interval you like, an hour or more to stay under
   TRMNL's rate limit).
3. Add action: **Run Shortcut** → select **Push Todos to TRMNL**.
4. Turn **off** "Ask Before Running" when prompted, so it fires silently in
   the background.

## 4. Test it

- Run the Shortcut manually and check the TRMNL plugin's live preview to
  confirm the screen renders.
- If `Get Contents of URL` fails, temporarily add a **Quick Look** or **Show
  Result** step right before it to inspect the JSON body you're about to
  send.
- Check the payload size — Shortcuts doesn't show this directly, but if
  TRMNL's preview doesn't update, trim the item count in step 2 above.

## How it works

- No code, no cloud function — `trmnl/*.liquid` is the only thing in this
  repo that TRMNL actually uses. Everything upstream of the webhook lives
  inside the Shortcut on your phone.
- Each todo dictionary sent to TRMNL has: `title`, `list` (the Reminders
  list name), `due` (short date, e.g. "Aug 28"), `due_word` (relative
  phrase, e.g. "Tomorrow" or "3 days ago"), and `overdue` (boolean). Blank
  `due`/`due_word` mean the reminder has no due date.
- The Liquid templates bold/emphasize overdue items (`item--emphasis-2`)
  and items due today (`item--emphasis-1`), same visual-weight pattern as
  trmnlBdays' upcoming-birthday highlighting.

## Notes / things to revisit

- This only updates while your phone is reachable and the automation
  actually fires. Personal Automations can be skipped if the phone is off,
  in Low Power Mode, or asleep at the trigger time with no reliable retry —
  the TRMNL display just shows stale data until the next successful run.
- If that unreliability becomes a problem, the alternative is a real
  server-side pull via iCloud's CalDAV endpoint (Reminders are stored as
  CalDAV `VTODO` items, accessible with an app-specific password) — same
  Cloudflare Worker + cron shape as trmnlBdays/trmnlAsburyEvents, just more
  code since CalDAV is XML-based rather than a clean JSON API.
- Consider restricting `Find Reminders` to specific lists if "all lists"
  turns out too noisy for a small display.
