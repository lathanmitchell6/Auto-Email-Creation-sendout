# Orientation Welcome Letter Sender + Workspace Account Creator

A Google Apps Script tool that runs every Tuesday: for each person on that
week's orientation list it creates a Google Workspace account if they don't
have one yet, then emails them a personalized welcome letter (as a PDF)
with their account credentials filled in.

## What it does, per person on the weekly tab

1. **No Work Email yet?** Creates a real Google Workspace account for them
   via the Admin Directory API, using an email generated from their name.
2. **Account created successfully?** Writes the new email into the
   **Work Email** column.
3. **Account creation failed?** Shades that row's **Work Email** cell red
   and skips that person entirely — no letter is sent — so it can be fixed
   by hand and re-run later.
4. **Has a working email** (existing or just created): fills in the welcome
   letter template, exports it as a PDF, and emails it.

Rows that are missing a name, or already marked **Sent**, are skipped.

## Files

| File | Purpose |
|---|---|
| `Code.gs` | Everything: config, tab lookup, account creation, letter generation, email sending, the weekly trigger |
| (external) A welcome letter Google Doc template, referenced by `TEMPLATE_DOC_ID` |
| (external) A Google Sheet with one tab per orientation week, referenced by `SHEET_ID` |

## How the pieces fit together

### Weekly tab lookup
`findWeeklySheet()` looks for a tab named for "this week" — it tries
**today** and the **most recent Tuesday**, each in a few common formats
(`M/d/yyyy`, `MM/dd/yyyy`, `M/d/yy`), e.g. `8/25/2026`. If none of those tab
names exist, the run stops with an error telling you to check the tab name.

### Columns it needs (row 1 headers, exact match)
| Config key | Default header | Purpose |
|---|---|---|
| `NAME_COLUMN` | `Name` | Full name, used to generate the work email and populate the letter |
| `PERSONAL_EMAIL_COLUMN` | `Email` | Read as a placeholder source; not used to send anything |
| `WORK_EMAIL_COLUMN` | `Work Email` | Read if present; written to if the script creates an account |
| `SENT_COLUMN` | `Sent` | Auto-created if missing; timestamped once a letter is sent |

### Generating a missing work email
`generateWorkEmail()` builds `[first initial][last name]@domain`, e.g.
"Gabrielle Lewis" → `glewis@onediversity.org`. Rows where the name doesn't
split into at least a first and last name are flagged red and skipped.

### Account creation
`createWorkspaceAccount()` calls `AdminDirectory.Users.insert()` with the
generated email, name, and a shared default password
(`NEW_ACCOUNT_PASSWORD`), with `changePasswordAtNextLogin: true`. It never
throws — failures are reported back as `{ success: false, error }` so the
main loop can flag the row and move on instead of stopping the whole run.

### Letter generation
`generateWelcomeLetterPdf()`:
1. Copies `TEMPLATE_DOC_ID` into `OUTPUT_FOLDER_ID` (or Drive's root if
   unset).
2. Replaces any `{{Header}}` placeholders in the copy with that row's
   values (dates are formatted as `MMMM d, yyyy`).
3. Calls `fillCredentialBlanks()` to fill in three specific blank lines by
   exact text match:
   - `Email: @onediversity.org` → `Email: {username}@onediversity.org`
   - `Username:` (under the Therap heading) → `Username: {username}`
   - `Username: @diversity` (QuickMAR) → `Username: {username}@diversity`
   
   The **Therap** and **QuickMAR** usernames are just the work email's
   local part — computed on the fly, not stored in the sheet.
4. Exports the filled-in copy as a PDF, deletes the intermediate Doc copy,
   and returns the PDF blob.

If any of the three credential lines isn't found verbatim in the template,
a warning is logged and that line is simply left blank in the letter — the
run continues.

### Sending
`sendWelcomeEmail()` sends the PDF as an attachment, with `EMAIL_SUBJECT`
and `EMAIL_BODY` run through the same `{{Header}}` placeholder substitution
first.

## Setup

### 1. Configure `CONFIG` in `Code.gs`

```js
const CONFIG = {
  SHEET_ID: '...',               // the orientation tracking sheet
  TEMPLATE_DOC_ID: '...',        // the welcome letter template doc
  OUTPUT_FOLDER_ID: '',          // optional Drive folder for generated letters

  NAME_COLUMN: 'Name',
  PERSONAL_EMAIL_COLUMN: 'Email',
  WORK_EMAIL_COLUMN: 'Work Email',
  SENT_COLUMN: 'Sent',

  WORK_EMAIL_DOMAIN: 'onediversity.org',
  ORG_UNIT_PATH: '/',
  NEW_ACCOUNT_PASSWORD: 'Diversity',

  EMAIL_SUBJECT: 'Welcome to the Team, {{Name}}!',
  EMAIL_BODY: `...`,

  ERROR_COLOR: '#f4cccc',
};
```

- Each week's tab must be named for that week's date, e.g. `8/25/2026`.
- The welcome letter template should contain the three exact credential
  lines listed above wherever credentials belong, and may also use
  `{{Header}}` placeholders anywhere else (e.g. `{{Name}}`) matching any
  column header in the sheet.

### 2. Enable the Admin SDK service

In the Apps Script editor: **Services (+) > Admin SDK API** (this exposes
the `AdminDirectory` object the script calls).

### 3. Run as a super admin

The script must be run by a Google Workspace super admin, or an admin role
with the **Create users** privilege — `AdminDirectory.Users.insert()` will
fail otherwise.

### 4. Authorize

Run `sendOrientationLetters` once manually from the editor and approve the
requested scopes (Admin Directory, Sheets, Docs, Drive, Gmail).

### 5. (Optional) Install the weekly trigger

Run `createWeeklyTrigger()` once. It removes any existing trigger for
`sendOrientationLetters` first (safe to re-run), then installs:

| Function | Schedule |
|---|---|
| `sendOrientationLetters` | Every Tuesday, 8:00 AM |

## Behavior to know about

- **Re-running is safe**: rows already marked **Sent** are skipped, and
  rows flagged red from a previous failed account creation are retried
  automatically (since their Work Email cell is still empty).
- **A failed account creation blocks that person's letter entirely** —
  nothing is sent until the row is fixed (manually, or by the error being
  resolved) and the script re-run.
- **The "Sent" column is created automatically** if the sheet doesn't have
  one yet, appended as the next empty column.
- **The template copy is temporary**: the Doc copy used to build the PDF is
  trashed after export, so only the final PDF (attached to the email, and
  optionally saved to `OUTPUT_FOLDER_ID`) persists.

## Security note

`NEW_ACCOUNT_PASSWORD` is a single shared password used for every newly
created account. `changePasswordAtNextLogin: true` forces a change on
first login, which limits the exposure, but the initial password is also
printed in plain text in this file and in the welcome letter template
convention it assumes — treat `Code.gs` and the template doc as sensitive,
and consider rotating or randomizing this value if that fits your
organization's security requirements.

## Troubleshooting

| Problem | Likely cause |
|---|---|
| "Could not find a tab named for this week's date" | This week's tab doesn't exist yet, or isn't named exactly like `8/25/2026` |
| Column not found error | A header in row 1 doesn't exactly match `NAME_COLUMN` / `PERSONAL_EMAIL_COLUMN` / `WORK_EMAIL_COLUMN` |
| Work Email cell stays red | Account creation failed — check **Executions** for the Admin Directory error (e.g. duplicate email, insufficient privileges, invalid org unit) |
| Credential line not filled in | The exact text (`Email: @onediversity.org`, `Username:`, `Username: @diversity`) isn't present verbatim in the template — check the warning in **Executions** |
| `AdminDirectory is not defined` | The Admin SDK API service wasn't added under **Services** |
| Account creation always fails | The account running the script isn't a super admin / lacks the "Create users" privilege |
| No letters sent even though accounts were created | Check **Executions** for a per-row "Failed to send letter" log line |

Logs for every run are in the Apps Script **Executions** view.
