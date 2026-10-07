# Winners Connect — Google Sheets Mapping and Webhook Setup

## Current status

Winners Connect currently runs from its application database and local date-scoped records. The Settings page displays the planned Google Sheets mapping structure, but no Google Sheets connector or webhook secret is active in the current project yet. The steps below describe the safe production setup.

## 1. Create the workbook

Create one Google Spreadsheet named `Winners Connect Operations` with these tabs:

| Tab | Primary key | Purpose |
|---|---|---|
| `Faculty` | `date + faculty_id` | Faculty roster, subject tags, attendance status and availability |
| `Courses` | `date + course_id` | Course master records, mode, status and delivery platform |
| `FacultyCourses` | `date + faculty_id + course_id` | Faculty-to-course eligibility and allowed mappings |
| `Resources` | `date + resource_code` | Hall, Studio and Online Room resources |
| `Attendance` | `date + faculty_id` | Present, Absent and Half-day status with time range |
| `Schedule` | `date + schedule_id` | Date-wise class plan, faculty, course(s), resource, operator and duration |
| `Notes` | `date + schedule_id` | PDF Upload in App, Receive PDF, review and notes status |
| `Replacements` | `date + schedule_id` | Original faculty, replacement faculty, reason and status |
| `Rules` | `rule_key` | Work window, lunch lock, lecture limit and resource rules |
| `AuditLog` | `date + event_id` | Actor, action, entity, timestamp and trace details |

Put the column headers in row 1. Keep IDs stable; do not use row numbers as IDs.

## 2. Recommended headers

### Faculty

`date, faculty_id, faculty_name, subjects, status, replacement_eligible, max_lectures, availability_from, availability_to, notes, updated_at`

### Courses

`date, course_id, course_name, mode, status, platforms, notes, updated_at`

`mode` must be `Online` or `Offline / Hybrid`. `status` should be `Upcoming` or `Live`.

### FacultyCourses

`date, faculty_id, faculty_name, course_id, course_name, allowed, notes, updated_at`

One faculty can have any number of relevant courses. Store one row per faculty-course pair.

### Resources

`date, resource_code, resource_name, type, status, notes, updated_at`

`type` must be `Hall`, `Studio` or `Online Room`.

### Attendance

`date, faculty_id, faculty_name, status, from_time, to_time, replacement_mode, notes, updated_at`

`status` must be `Present`, `Absent`, `Half-day` or `Pending`.

### Schedule

`date, schedule_id, time_from, time_to, faculty_id, faculty_name, course_ids, course_names, subject, lecture_title, mode, resource_code, resource_name, resource_type, platform, operator_name, pdf_uploaded, pdf_received, in_time, duration_minutes, out_time, review_complete, status, updated_at`

`course_ids` and `course_names` may contain multiple values separated by ` | `. The selected date is mandatory on every row.

## 3. Apps Script webhook receiver

Open **Extensions → Apps Script** in the workbook and add the following script. Replace `CHANGE_ME` with a long random secret. Do not publish the secret in the frontend or in a public document.

```javascript
const WEBHOOK_SECRET = 'CHANGE_ME';
const ALLOWED_TABS = new Set([
  'Faculty', 'Courses', 'FacultyCourses', 'Resources', 'Attendance',
  'Schedule', 'Notes', 'Replacements', 'Rules', 'AuditLog'
]);

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents || '{}');
    if (body.secret !== WEBHOOK_SECRET) return json({ ok: false, error: 'Unauthorized' });
    if (!body.tab || !ALLOWED_TABS.has(body.tab)) return json({ ok: false, error: 'Invalid tab' });
    if (!body.date || !/^\\d{4}-\\d{2}-\\d{2}$/.test(body.date)) {
      return json({ ok: false, error: 'A valid selected date is required' });
    }

    const rows = Array.isArray(body.rows) ? body.rows : [];
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(body.tab);
    if (!sheet) return json({ ok: false, error: 'Sheet not found' });

    const values = sheet.getDataRange().getValues();
    const headers = values.length ? values[0].map(String) : [];
    const dateIndex = headers.indexOf('date');
    if (dateIndex < 0) return json({ ok: false, error: 'Sheet requires a date column' });

    // Replace only the selected date for this tab; preserve all other dates.
    const kept = values.slice(1).filter(row => String(row[dateIndex]) !== body.date);
    const incoming = rows.map(row => headers.map(header => row[header] ?? ''));
    const next = [headers, ...kept, ...incoming];
    sheet.clearContents();
    sheet.getRange(1, 1, next.length, headers.length).setValues(next);
    return json({ ok: true, tab: body.tab, date: body.date, rows_written: incoming.length });
  } catch (err) {
    return json({ ok: false, error: String(err) });
  }
}

function json(value) {
  return ContentService
    .createTextOutput(JSON.stringify(value))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## 4. Deploy the webhook

1. In Apps Script, choose **Deploy → New deployment**.
2. Select **Web app**.
3. Execute as the workbook owner.
4. Set access to the organization or users who are permitted to operate Winners Connect. Avoid “Anyone” unless a strong secret, rate limiting and a gateway are in place.
5. Copy the generated web app URL.
6. Store the URL and secret in a protected server-side connector configuration, never in client-side React code.

## 5. Synchronization payload

For each changed tab and selected date, send a request like this from the server-side connector:

```json
{
  "secret": "server-side-secret",
  "tab": "Schedule",
  "date": "2026-10-10",
  "rows": [
    {
      "date": "2026-10-10",
      "schedule_id": "SCH-101",
      "time_from": "08:00",
      "time_to": "09:15",
      "faculty_id": "FAC-001",
      "faculty_name": "Example Faculty",
      "course_ids": "C-01 | C-02",
      "course_names": "Course One | Course Two",
      "subject": "Biology",
      "lecture_title": "Cell Structure",
      "mode": "Offline",
      "resource_code": "HALL-A",
      "resource_name": "Hall A",
      "resource_type": "Hall",
      "platform": "Offline",
      "operator_name": "Operator Name",
      "pdf_uploaded": "No",
      "pdf_received": "No",
      "in_time": "08:03",
      "duration_minutes": 72,
      "out_time": "09:15",
      "review_complete": "Yes",
      "status": "Done",
      "updated_at": "2026-10-10T09:20:00+05:30"
    }
  ]
}
```

The server should POST to the Apps Script web-app URL with `Content-Type: application/json`, retry transient 5xx responses with exponential backoff, and record each attempt in `AuditLog`.

## 6. Date-wise safety rules

- Always send the selected header date in the payload.
- Replace or upsert rows only for that date; never clear the entire workbook.
- Use stable IDs and upsert by the tab’s primary key.
- Master Data changes should be written before Schedule rows are synchronized.
- A historical date must be read-only for Admin, Teacher / Operator and Analysis roles.
- A schedule row may contain multiple course IDs, but it must still have one time range, one faculty, one mode and one resource.
- Online rows require `Studio` or `Online Room`; Offline rows require `Hall`.

## 7. Testing checklist

1. Select a future date in the Winners Connect header.
2. Create or edit a faculty, course mapping and schedule row.
3. Confirm the application reads only that selected date.
4. Send one payload for `Schedule` and confirm only that date’s rows change in Sheets.
5. Return to the previous date and confirm its rows are unchanged.
6. Send an invalid secret and confirm a JSON `Unauthorized` response.
7. Send an invalid tab or missing date and confirm the request is rejected.
8. Compare application row counts with the corresponding Sheet tab.
9. Verify every webhook attempt appears in AuditLog.

## 8. Current project limitation

The current Winners Connect Preview is **Sheets-ready but not connected**. To activate synchronization, the project still needs an authorized connector or server-side secret configuration for the Apps Script URL. Until that is enabled, use the in-app database and Export controls as the source of truth.
