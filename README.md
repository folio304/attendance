# Attendance & Payroll Register — Batra Deepak & Associates

An installable web app on GitHub Pages that reads the daily Secureye biometric
export, pushes the punches to one Google Sheet, shows a date-wise register,
emails late and absent staff, and works out accrued salary for the month.

| File | Where it goes |
|---|---|
| `index.html` | Repository root |
| `manifest.webmanifest` | Repository root |
| `sw.js` | Repository root |
| `icons/` | Repository root, as a folder |
| `Code.gs` | Pasted into the Sheet's Apps Script editor |

---

## Updating an existing install

Do the script **before** the repository. The page calls actions the old script
doesn't have, so the reverse order gives errors for a few minutes.

1. Extensions → Apps Script → select all → paste the new `Code.gs` → save.
2. Run **`setup`**. It creates the new `Exceptions` tab, adds `InTime` and
   `Salary` columns to the `Employees` tab without disturbing existing rows, and
   converts your old 10:06 cut-off into a 10:00 in-time with 6 minutes' grace.
3. **Deploy → Manage deployments → pencil → Version: New version → Deploy.**
   Use *Manage deployments*, not *New deployment* — a new deployment mints a
   different `/exec` URL and the page would stop connecting.
4. Push the repository files.
5. On the Employees tab, fill in each person's in-time and monthly salary, then
   **Save employee list**.

`CACHE_VERSION` in `sw.js` is already bumped to `v2` for this release.

---

## First-time setup

<details>
<summary>Google Sheet and Apps Script</summary>

1. Create a Google Sheet.
2. Extensions → Apps Script, delete the placeholder, paste `Code.gs`, save.
3. Run **`setup`** and approve the permissions prompt. Five tabs appear:
   `Attendance`, `Employees`, `EmailLog`, `Holidays`, `Exceptions`.
4. **Deploy → New deployment → Web app**, Execute as **Me**, access **Anyone**.
   If the URL differs from the one in `index.html`, paste the new one into
   `API_URL` near the top of the `<script>` block.

"Anyone" is required because the page calls it without a Google login. The
access key is what protects it — and since both the key and the URL sit in
`index.html`, **keep the repository private**.
</details>

<details>
<summary>GitHub Pages</summary>

1. Create a repository, upload the four items from the table above.
2. Settings → Pages → Deploy from a branch, `main`, `/ (root)`.
3. Live at `https://<username>.github.io/<repo>/` after a minute.
</details>

<details>
<summary>Installing it as an app</summary>

- **Android / Chrome:** menu → *Add to Home screen*.
- **iPhone / Safari:** Share → *Add to Home Screen*. Must be Safari.
- **Desktop Chrome / Edge:** install icon at the right of the address bar.

Installed, it opens without browser chrome and keeps working offline for
*viewing* the last loaded register. Upload, marking and email need a connection —
an amber bar appears when there isn't one.

**Whenever you change `index.html`, bump `CACHE_VERSION` in `sw.js`.** Installed
copies then show a "newer version is ready" bar. Skip it and people can sit on
the old page for days.
</details>

---

## The page

**Day register** opens on today's date. **Payroll** and **Employees** are the
other two tabs. **Upload file** and **Settings** are the buttons at the top
right.

### Reporting time

Each employee has their own in-time on the Employees tab. Everyone gets the same
grace period, set once under Settings (6 minutes). Late means the in-punch is
later than *that person's* in-time plus the grace — so an 08:30 employee
punching at 08:37 is late, while a 10:00 employee punching at 10:05 is not.

Leave an employee's in-time blank to use the office default.

### Marking leave and outdoor duty

Every row in the register — and in the upload preview — has a **Mark** dropdown:
*On leave* or *Outdoor duty*. It saves the moment you choose it, independently
of the attendance rows, so you can mark someone before or after uploading the
day's file.

A marking overrides whatever the machine recorded. Marking someone on leave when
they did punch in will show them as on leave and pay them nothing for the day, so
use it deliberately.

### Status rules

| Status | Meaning | Paid? |
|---|---|---|
| **Present** | Punched in within their in-time plus grace. No out-punch needed — you export mid-day. | Yes |
| **Late** | Punched in after the allowance. Minutes over are shown and quoted in the email. | Yes, in full |
| **Outdoor duty** | Marked by hand. | Yes |
| **On leave** | Marked by hand. | No |
| **Absent** | Active employee, no punch, no marking. | No |
| **Off day** | Sunday or a listed holiday. | Yes — paid without being checked |

### Emails

The **Send emails** button sends, for the chosen date:

- **Late** — the in-punch, their reporting time, the allowance, minutes over, and
  a note that lateness carries no deduction.
- **Absent** — asks them to report leave or outdoor duty, and warns that
  unexplained absence is unpaid.
- **On leave** — confirms the marking and states it is unpaid.
- **Outdoor duty** — confirms the marking and states it counts as a full day.

Every manager gets one summary listing all four groups. Sends are logged in
`EmailLog`, so pressing the button twice won't double-mail anyone.

---

## Payroll

```
per-day rate = monthly salary ÷ calendar days in the month
accrued      = per-day rate × (days counted − leave − absent)
```

Every calendar day is paid unless it is leave or unexplained absence. Sundays
and holidays are therefore paid outright, and lateness costs nothing.

**Days counted** run from the 1st to the last date you uploaded a file for, so
mid-month the figure is a genuine accrual rather than the whole month's salary.
Once the month is complete it equals the calendar days.

**Days checked** — the subset used to decide who was present or absent — are the
dates that are not Sundays, not holidays, and have an uploaded file. A weekday
with no upload can't be checked, so nobody is marked absent on it and everyone
is paid for it, exactly like a holiday.

Your worked example, replayed through the code:

> ₹25,000 · 30-day month · 4 Sundays · 1 holiday · 5 days' leave
> Per day = 25,000 ÷ 30 = **₹833.33**. Paid days = 30 − 5 = **25**.
> Accrued = 833.33 × 25 = **₹20,833.33**.

The 4 Sundays and the holiday are inside those 25 paid days; only the 5 leave
days come off. There's a CSV download for the month.

## How the export is read

Your machine produces a Secureye ONtime *Date wise Daily Attendance Report
(Detailed)*, which is awkward: every field is merged across many columns, a blank
row sits between records, and the punch timestamps use a different date order to
the report header.

| Thing | How it's handled |
|---|---|
| Report date | From the `On Dated : 01/09/2026` header (dd/mm/yyyy). Falls back to the filename. |
| Punch timestamps | The stamp reads `09/01/2026 10:09:00` — mm/dd/yyyy. Only the clock is used, so date order never matters. |
| Merged columns | Header column positions become anchors; each data cell snaps to its nearest anchor. |
| Blank spacer rows | Rows with fewer than four filled cells are skipped. |
| Footer lines | `Generated By`, `Printed On`, `Page 1 of 1` filtered out. |
| Same person twice | Merged: earliest in-punch, latest out-punch. |

Only employee code, name, in time and out time leave your browser. Card number,
shift hours and OT are dropped.

---

## Staying inside the free-Gmail limits

A free `@gmail.com` allows **100 email recipients per day** through Apps Script.
With 13 staff and 3 managers the worst day uses 16.

- Page load is **one** request: employees, dates, holidays, settings and today's
  register together.
- One upload is **one** request carrying the whole day.
- Writes use a single `setValues()` rather than `appendRow()` in a loop.
- The employee list is cached for 15 minutes.
- The three manager summaries go out in one `sendEmail` call.
- Re-uploading a date replaces those rows instead of duplicating them.

The quota remaining is shown at the top right. It resets around 12:30 PM IST.

---

## Things worth knowing

- **Employee code can't be edited** in the page. Changing it would orphan that
  person's attendance history. Remove and re-add if a code genuinely changes.
- **Changing the in-time or grace re-classifies history.** The register is
  computed on demand, never stored, so old dates follow the new rule.
- **Removing an employee** only takes them off the master list. Past attendance
  rows stay in the Sheet, and past payroll for that month will change because
  they're no longer counted.
- **Redeploying `Code.gs`** always needs *Manage deployments → New version*.

---

## If something goes wrong

| Symptom | Cause |
|---|---|
| "Unknown action: getPayroll" | Script pasted but not redeployed as a new version. |
| "Access key rejected" | Run `setup` again, then redeploy. |
| "Could not reach the Apps Script" | Deployment isn't set to *Anyone*, or `API_URL` is stale. |
| "Could not find the report date" | Wrong export type — you need *Date wise Daily Attendance Report (Detailed)*. |
| Everyone shows Absent | Employee codes don't match the machine's EMP Code. |
| Payroll shows nothing accrued | No files uploaded for that month yet. |
| Payroll looks like a part month | Correct — it counts to your last uploaded file. Upload the rest. |
| Someone paid for a day they were absent | That day had no uploaded file, so absence couldn't be detected. Upload it and reload. |
| Old version keeps loading | `CACHE_VERSION` in `sw.js` wasn't bumped. |
