---
name: leave-credit
description: >
  Use this skill whenever the user says "use leave credit skill", "calculate leave credits",
  "process attendance for leave", or uploads an employee attendance Excel file and asks about
  comp offs, leave credits, or remaining leaves. This skill scans an employee attendance
  Excel/CSV file, detects company holidays and weekly offs from the Status column, counts
  leave credits earned by working on Weekly Offs or Company Holidays, deducts credits for
  leave taken on working days (verified via Working Hours), and outputs a final .xlsx report
  showing Employee Name, Employee Code, and Leave Credits Left.
  Always trigger this skill when attendance data + leave/comp-off analysis is the goal.
---

# Leave Credit Skill

Calculates remaining leave credits for employees based on their monthly attendance data.

---

## Trigger Phrase
Activate when user says: **"use leave credit skill"** or equivalent (calculate leave credits, process attendance leave, etc.)

---

## Input
User uploads an Excel (.xlsx / .xls) or CSV attendance file for a month.

### Expected Columns in Attendance File
| Column | Description |
|---|---|
| Employee Code | Unique employee ID |
| Employee Name | Full name |
| Date | Date of attendance (DD-MM-YYYY or similar) |
| In Time | Clock-in time |
| Out Time | Clock-out time |
| Working Hours | Hours worked — **this is the single source of truth for presence** |
| Status | See Status Values below |
| Totals | (ignored for credit logic) |

### Status Values and Their Meaning
| Status Value | Meaning | Leave Credit Impact |
|---|---|---|
| `WO` / `Weekly Off` / `W/O` | Paid weekly rest day | No change (unless Working Hours > 0 → **+1 credit**) |
| `CH` / `Company Holiday` / `Holiday` | Company holiday (detected from Status column) | No change (unless Working Hours > 0 → **+1 credit**) |
| `COMP OFF` / `Comp Off` / `CO` | Employee's leave day (1 comp off = 1 full day of leave) | Check Working Hours: if **0 → −1 credit**; if **> 0 → no change** |
| `A` / `Absent` | Employee took leave | Check Working Hours: if **0 → −1 credit**; if **> 0 → no change** |
| `P` / `Present` | Normal working day, present | No change |

> **Key Rule — Working Hours is the Source of Truth**: Do NOT rely solely on the Status column to determine if an employee was present or absent. **Always check the Working Hours column**: if Working Hours > 0, the employee was present; if Working Hours = 0 (or blank/missing), the employee was absent. This applies universally across all status types.

> **Comp Off Explained**: A COMP OFF is one full day of earned leave. It can be used for a full-day leave or half-day leave. When a status shows COMP OFF, it means the employee is *supposed* to be on leave — but they may have still come to work. Working Hours resolves this: 0 hours = actually on leave (−1 credit), >0 hours = present despite the status (no credit change).

---

## Company Holidays — Detected from Attendance Data

Company holidays are **NOT hardcoded**. They vary by region, year, and organization. Instead, the system detects them directly from the **Status column** of the uploaded attendance file.

A date is treated as a Company Holiday if the Status value matches:
- `CH`, `Company Holiday`, `Holiday`, or any similar pattern

This makes the skill **generalised** and usable across any company, region, or year without needing to maintain a hardcoded holiday list.

---

## Presence Detection — The Working Hours Rule

This is the **most critical rule** in the entire system. It overrides status-based assumptions.

| Working Hours | Interpretation |
|---|---|
| **> 0** (any non-zero value like 8, 9, 10.5, etc.) | Employee was **present** that day |
| **= 0** (or blank / missing / NaN) | Employee was **absent** that day |

This rule is applied **everywhere** — whether the status says Present, COMP OFF, Absent, Weekly Off, or Company Holiday. The Working Hours column is the final arbiter of physical presence.

### Why This Matters
- A status of `COMP OFF` with 9 working hours → employee came to work despite being on leave → **no credit deducted**
- A status of `COMP OFF` with 0 working hours → employee was actually on leave → **−1 credit**
- A status of `Weekly Off` with 10 working hours → employee worked on their day off → **+1 credit earned**
- A status of `Weekly Off` with 0 working hours → normal day off → **no change**

---

## Processing Logic

### Step 1 — Load the Attendance File
- Use `pandas` to read the uploaded Excel/CSV file.
- Auto-detect header row (may not always be row 1 — look for rows containing "Employee Code", "Date", "Status").
- Normalize column names: strip whitespace, lowercase for matching.
- Parse the `Date` column to `datetime.date`.
- **Parse the Working Hours column** to a numeric value (handle HH:MM:SS format by converting to decimal hours; treat blanks/NaN as 0).

### Step 2 — Identify Day Type (Status + Date Verification)
For each row, classify the day using **both** the Status column and the **actual day-of-week from the Date column**:

**From Status column:**
- Status matches `CH` / `Company Holiday` / `Holiday` patterns → mark as `company_holiday`
- Status matches `WO` / `Weekly Off` / `W/O` patterns → mark as `weekly_off`
- Status matches `COMP OFF` / `Comp Off` / `CO` patterns → mark as `comp_off_day`
- Status matches `A` / `Absent` patterns → mark as `absent_day`
- Otherwise → mark as `working_day`

**From Date column (independent verification):**
- Parse the date and compute the **day of the week** (Monday=0 … Sunday=6).
- First, do a pre-scan of the data to detect which day-of-week is the weekly off for each employee. Look at all rows where Status = `Weekly Off` / `WO` / `W/O` and find the most common day-of-week — this is the employee's weekly off day (typically Sunday, i.e. day 6).
- Store this as `weekly_off_day` per employee (default to Sunday/6 if not enough data).

### Step 2.5 — Verify Every COMP OFF Against the Calendar
**This is critical.** The Status column sometimes says `COMP OFF` on a date that is actually the employee's weekly off day (e.g., Sunday). The system must catch this every time:

1. For **every** row where Status = `COMP OFF`:
   - Get the day-of-week from the **Date column** (e.g., `date.weekday()` → 6 = Sunday).
   - Compare it to the employee's detected `weekly_off_day`.
   - **If the date's day-of-week matches the weekly off day → override: treat this as a Weekly Off, NOT a COMP OFF.**
   - This means: no credit deduction even if Working Hours = 0 (it's just a normal day off).

2. Also check if the date is a Company Holiday (by checking if other employees on the same date have `CH` / `Company Holiday` status):
   - **If the date is a Company Holiday → override: treat this as a Company Holiday, NOT a COMP OFF.**

> **Why this matters:** Without this check, an employee marked `COMP OFF` on a Sunday would lose −1 credit for simply staying home on their normal day off. The calendar-based verification prevents this.

### Step 3 — Detect "Worked on Off Day" (+1 Credit)
A credit is **earned** if:
- Day is classified as `weekly_off` OR `company_holiday`, AND
- **Working Hours > 0** (employee actually worked — this is the only reliable check)

> Do NOT use Status = "Present" as the check. Use Working Hours > 0 exclusively.

### Step 4 — Detect "Leave Taken on Working Day" (−1 Credit)
A credit is **deducted** only if ALL of the following are true:
- Status matches `COMP OFF` or `Absent`, AND
- The date's day-of-week is **NOT** the employee's weekly off day (verified from Date column), AND
- The date is NOT a `company_holiday` (verified from Status column of all employees on that date), AND
- **Working Hours = 0** (employee was actually absent — confirmed by hours)

> If the status says COMP OFF but the date is actually a Sunday (weekly off day), **no credit is deducted** — even if hours = 0. The calendar check catches this.

> If the status says COMP OFF but Working Hours > 0, the employee came to work. **No credit is deducted.**

### Step 4.5 — Summary: COMP OFF Decision Tree
For every row where Status = `COMP OFF`:
```
1. Is the date a Weekly Off day? (check day-of-week from Date column)
   → YES → No deduction. It's a normal off day.
   → NO  → continue...

2. Is the date a Company Holiday? (check Status of other employees on same date)
   → YES → No deduction. It's a holiday.
   → NO  → continue...

3. Is Working Hours > 0?
   → YES → No deduction. Employee was present despite COMP OFF status.
   → NO  → **−1 credit. Employee was on leave on a working day.**
```

### Step 5 — Aggregate Per Employee
Group by `Employee Code` + `Employee Name`:
```
leave_credits = (days worked on WO or CH) − (leaves taken on working days)
```
Credits can go negative if the employee has overdrawn their leave entitlement.

### Step 6 — Output Excel Report

---

## Output

Generate a professionally formatted `.xlsx` file with:

### Sheet: "Leave Credits Summary"
| Column | Description |
|---|---|
| Employee Code | From attendance data |
| Employee Name | From attendance data |
| Working Days on WO/Holiday (Credits Earned) | Count of +1 events |
| Leaves Taken on Working Days (Credits Used) | Count of −1 events |
| **Leave Credits Left** | Earned − Used |

### Sheet: "Attendance Detail" (optional but recommended)
Row-by-row breakdown showing each date, status, working hours, day type, and credit impact (+1 / −1 / 0) per employee.

### Formatting
- Use Arial 11pt font throughout
- Bold header row with light blue fill (`BDD7EE`)
- Auto-fit column widths
- Highlight "Leave Credits Left" column in light green if > 0, light red if < 0
- Add title row: "Leave Credits Report — [Month Year]"

---

## Python Implementation Template

```python
import pandas as pd
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment
from openpyxl.utils import get_column_letter
from datetime import date
import re

# --- NO hardcoded company holidays ---
# Company holidays are detected from the Status column in the attendance file.

WEEKLY_OFF_PATTERNS = re.compile(r'\bw/?o\b|weekly[\s_-]?off', re.IGNORECASE)
COMP_OFF_PATTERNS   = re.compile(r'\bco\b|comp[\s_-]?off', re.IGNORECASE)
PRESENT_PATTERNS    = re.compile(r'\bpresent\b|\bp\b', re.IGNORECASE)
HOLIDAY_PATTERNS    = re.compile(r'\bch\b|company[\s_-]?holiday|\bholiday\b', re.IGNORECASE)
ABSENT_PATTERNS     = re.compile(r'\babsent\b|^\s*a\s*$', re.IGNORECASE)

def is_weekly_off(status):
    return bool(WEEKLY_OFF_PATTERNS.search(str(status)))

def is_comp_off_status(status):
    """Checks if status label says COMP OFF (does NOT check working hours)."""
    return bool(COMP_OFF_PATTERNS.search(str(status)))

def is_absent_status(status):
    """Checks if status label says Absent (does NOT check working hours)."""
    return bool(ABSENT_PATTERNS.search(str(status).strip()))

def is_present(status):
    return bool(PRESENT_PATTERNS.search(str(status)))

def is_holiday_status(status):
    """Detects company holidays from the Status column — no hardcoded list."""
    return bool(HOLIDAY_PATTERNS.search(str(status)))

def is_leave_status(status):
    """Status says COMP OFF or Absent — but actual presence is confirmed by hours."""
    return is_comp_off_status(status) or is_absent_status(status)

def parse_working_hours(val):
    """
    Parse working hours from various formats to decimal hours.
    Handles: HH:MM:SS, HH:MM, decimal numbers, blanks/NaN.
    Returns 0.0 if not parseable or missing.
    """
    if pd.isna(val) or str(val).strip() in ('', 'nan', 'NaN', 'NaT'):
        return 0.0
    s = str(val).strip()
    # Try HH:MM:SS or HH:MM format
    parts = s.split(':')
    if len(parts) >= 2:
        try:
            hours = float(parts[0])
            minutes = float(parts[1])
            seconds = float(parts[2]) if len(parts) >= 3 else 0.0
            return hours + minutes / 60.0 + seconds / 3600.0
        except (ValueError, IndexError):
            pass
    # Try plain numeric
    try:
        return float(s)
    except ValueError:
        return 0.0

def was_actually_present(hours):
    """
    THE source-of-truth check for physical presence.
    Working Hours > 0 → present. Working Hours = 0 → absent.
    """
    return hours > 0

def process_attendance(filepath):
    df = pd.read_excel(filepath, header=None)
    
    # Find header row
    header_row = None
    for i, row in df.iterrows():
        if any('employee' in str(v).lower() or 'date' in str(v).lower() for v in row):
            header_row = i
            break
    
    df = pd.read_excel(filepath, header=header_row)
    df.columns = [str(c).strip() for c in df.columns]
    
    # Map columns flexibly
    col_map = {}
    for col in df.columns:
        cl = col.lower()
        if 'employee code' in cl or 'emp code' in cl or 'empcode' in cl:
            col_map['emp_code'] = col
        elif 'employee name' in cl or 'emp name' in cl or 'name' in cl:
            col_map['emp_name'] = col
        elif 'date' in cl:
            col_map['date'] = col
        elif 'status' in cl:
            col_map['status'] = col
        elif 'working hour' in cl or 'work hour' in cl:
            col_map['hours'] = col
    
    df['_date'] = pd.to_datetime(df[col_map['date']], errors='coerce', dayfirst=True).dt.date
    df['_status'] = df[col_map['status']].fillna('').astype(str)
    # Parse working hours using the robust parser
    df['_hours'] = df[col_map.get('hours', '_none')].apply(parse_working_hours) if 'hours' in col_map else 0.0
    df['_emp_code'] = df[col_map['emp_code']].astype(str).str.strip()
    df['_emp_name'] = df[col_map['emp_name']].astype(str).str.strip()
    
    # Remove total/summary rows
    df = df[df['_date'].notna()]
    df = df[~df['_emp_code'].str.lower().isin(['nan', 'total', 'totals', ''])]
    
    # ──────────────────────────────────────────────────────────
    # PRE-SCAN: Detect each employee's weekly off day-of-week
    # ──────────────────────────────────────────────────────────
    # Look at all rows where Status = "Weekly Off" / "WO" / "W/O"
    # and find the most common day-of-week. This tells us which
    # day (e.g., Sunday=6) is the employee's weekly off.
    from collections import Counter
    
    employee_weekly_off_day = {}  # key: (emp_code, emp_name) → int (0=Mon..6=Sun)
    
    for (code, name), grp in df.groupby(['_emp_code', '_emp_name']):
        wo_rows = grp[grp['_status'].apply(is_weekly_off)]
        if len(wo_rows) > 0:
            day_counts = Counter(d.weekday() for d in wo_rows['_date'] if d is not None)
            most_common_day = day_counts.most_common(1)[0][0]
            employee_weekly_off_day[(code, name)] = most_common_day
        else:
            # Default to Sunday (6) if no Weekly Off rows found
            employee_weekly_off_day[(code, name)] = 6
    
    # ──────────────────────────────────────────────────────────
    # PRE-SCAN: Detect company holiday dates from Status column
    # ──────────────────────────────────────────────────────────
    # A date is a Company Holiday if ANY employee has CH/Holiday
    # status on that date. This catches cases where one employee
    # is marked COMP OFF on a date that is actually a holiday
    # (visible from other employees' statuses).
    company_holiday_dates = set()
    for _, row in df.iterrows():
        if is_holiday_status(row['_status']) and row['_date'] is not None:
            company_holiday_dates.add(row['_date'])
    
    # ──────────────────────────────────────────────────────────
    # MAIN LOOP: Process each row with calendar verification
    # ──────────────────────────────────────────────────────────
    results = {}
    
    for _, row in df.iterrows():
        key = (row['_emp_code'], row['_emp_name'])
        if key not in results:
            results[key] = {'earned': 0, 'used': 0, 'detail': []}
        
        d = row['_date']
        status = row['_status']
        hours = row['_hours']
        actually_present = was_actually_present(hours)
        
        # --- Classify day type from Status column ---
        is_ch = is_holiday_status(status)      # Status says Company Holiday
        is_wo_status = is_weekly_off(status)    # Status says Weekly Off
        is_co = is_comp_off_status(status)      # Status says COMP OFF
        is_ab = is_absent_status(status)        # Status says Absent
        
        # --- CALENDAR VERIFICATION (the key change) ---
        # Check the actual day-of-week from the Date column
        emp_wo_day = employee_weekly_off_day.get(key, 6)  # default Sunday
        date_is_weekly_off = (d.weekday() == emp_wo_day) if d else False
        
        # Check if this date is a company holiday (from pre-scan)
        date_is_holiday = (d in company_holiday_dates) if d else False
        
        # Final off-day determination: use BOTH status and calendar
        # A day is an off day if:
        #   - Status says Weekly Off, OR
        #   - Status says Company Holiday, OR
        #   - The date's day-of-week is the employee's weekly off day, OR
        #   - The date is in the company holiday set (from other employees)
        is_off_day = is_wo_status or is_ch or date_is_weekly_off or date_is_holiday
        
        # --- COMP OFF VERIFICATION ---
        # If status says COMP OFF, check if the date is actually an off day
        comp_off_is_actually_off_day = False
        if is_co and is_off_day:
            comp_off_is_actually_off_day = True
            # This COMP OFF falls on a weekly off or holiday — not a real leave day
        
        credit_change = 0
        reason = ''
        day_of_week_name = d.strftime('%A') if d else ''
        
        # RULE 1: Worked on an off day → +1 credit
        if is_off_day and actually_present:
            results[key]['earned'] += 1
            credit_change = +1
            reason = f'Worked on off day ({day_of_week_name}, Hours > 0)'
        
        # RULE 2: COMP OFF / Absent on off day → no deduction
        # The date is a weekly off or holiday — no credit lost
        elif is_leave_status(status) and is_off_day and not actually_present:
            credit_change = 0
            reason = f'Leave status on off day ({day_of_week_name}) — no deduction'
        
        # RULE 3: Leave on a working day, hours = 0 → −1 credit
        elif is_leave_status(status) and not is_off_day and not actually_present:
            results[key]['used'] += 1
            credit_change = -1
            reason = f'Leave taken on working day ({day_of_week_name}, Hours = 0)'
        
        # RULE 4: Status says leave but hours > 0 on working day → no change
        elif is_leave_status(status) and not is_off_day and actually_present:
            credit_change = 0
            reason = f'Status says leave but present ({day_of_week_name}, Hours > 0) — no change'
        
        results[key]['detail'].append({
            'Date': str(d), 'Day': day_of_week_name,
            'Status': status, 'Working Hours': round(hours, 2),
            'Actually Present': actually_present,
            'Is Holiday': is_ch or date_is_holiday,
            'Is Weekly Off': is_wo_status or date_is_weekly_off,
            'Credit Change': credit_change, 'Reason': reason
        })
    
    return results

def write_output(results, output_path, month_label=''):
    wb = Workbook()
    ws = wb.active
    ws.title = 'Leave Credits Summary'
    
    header_fill = PatternFill('solid', fgColor='BDD7EE')
    bold = Font(bold=True, name='Arial', size=11)
    normal = Font(name='Arial', size=11)
    
    # Title
    ws.merge_cells('A1:E1')
    ws['A1'] = f'Leave Credits Report — {month_label}'
    ws['A1'].font = Font(bold=True, name='Arial', size=13)
    ws['A1'].alignment = Alignment(horizontal='center')
    
    headers = ['Employee Code', 'Employee Name',
               'Credits Earned (WO/Holiday Work)',
               'Credits Used (Leave Taken)',
               'Leave Credits Left']
    
    for col, h in enumerate(headers, 1):
        cell = ws.cell(row=2, column=col, value=h)
        cell.font = bold
        cell.fill = header_fill
        cell.alignment = Alignment(horizontal='center', wrap_text=True)
    
    green_fill = PatternFill('solid', fgColor='C6EFCE')
    red_fill   = PatternFill('solid', fgColor='FFC7CE')
    
    for row_idx, ((code, name), data) in enumerate(results.items(), start=3):
        left = data['earned'] - data['used']
        row_data = [code, name, data['earned'], data['used'], left]
        for col, val in enumerate(row_data, 1):
            cell = ws.cell(row=row_idx, column=col, value=val)
            cell.font = normal
            if col == 5:
                cell.fill = green_fill if left >= 0 else red_fill
    
    for col in range(1, 6):
        ws.column_dimensions[get_column_letter(col)].width = 28
    ws.row_dimensions[2].height = 35
    
    # Detail sheet
    ws2 = wb.create_sheet('Attendance Detail')
    detail_headers = ['Employee Code', 'Employee Name', 'Date', 'Day', 'Status',
                      'Working Hours', 'Actually Present',
                      'Is Holiday', 'Is Weekly Off', 'Credit Change', 'Reason']
    for col, h in enumerate(detail_headers, 1):
        cell = ws2.cell(row=1, column=col, value=h)
        cell.font = bold
        cell.fill = header_fill
    
    r = 2
    for (code, name), data in results.items():
        for entry in data['detail']:
            ws2.cell(row=r, column=1, value=code)
            ws2.cell(row=r, column=2, value=name)
            for col_idx, key in enumerate(
                ['Date', 'Day', 'Status', 'Working Hours', 'Actually Present',
                 'Is Holiday', 'Is Weekly Off', 'Credit Change', 'Reason'], 3
            ):
                ws2.cell(row=r, column=col_idx, value=entry[key])
            r += 1
    
    for col in range(1, 12):
        ws2.column_dimensions[get_column_letter(col)].width = 22
    
    wb.save(output_path)
    print(f'Saved: {output_path}')
```

---

## Execution Steps

1. **Read the SKILL.md** (this file) ✓
2. **Identify the uploaded file** from `/mnt/user-data/uploads/`
3. **Run the processing logic** using the Python template above via `bash_tool`
4. **Save output** to `/mnt/user-data/outputs/leave_credits_report.xlsx`
5. **Present the file** using `present_files` tool
6. **Summarize** in 2–3 lines: number of employees processed, month covered, any anomalies (e.g., negative credits)

---

## Edge Cases

| Scenario | Handling |
|---|---|
| `COMP OFF` with Working Hours > 0 | Employee came to work → **no deduction** (hours override status) |
| `COMP OFF` with Working Hours = 0 on a working day | Employee was on leave → **−1 credit** |
| `COMP OFF` on a Sunday (weekly off day) | Calendar check detects it's a weekly off → **no deduction** regardless of hours |
| `COMP OFF` on a date that is another employee's `Company Holiday` | Holiday pre-scan catches it → **no deduction** regardless of hours |
| `COMP OFF` on a date that is both a weekly off AND a holiday | Off day takes priority → **no deduction** |
| `Absent` with Working Hours = 0 on a working day | Same as COMP OFF → **−1 credit** |
| `Absent` with Working Hours > 0 | Employee was present despite status → **no deduction** |
| `Absent` on a Weekly Off (by day-of-week) | No deduction — calendar confirms it's a day off |
| `Absent` on a Company Holiday | No deduction — it's already a holiday |
| `Weekly Off` with Working Hours > 0 | Employee worked on their off day → **+1 credit** |
| `Company Holiday` with Working Hours > 0 | Employee worked on holiday → **+1 credit** |
| Company holiday + weekly off on same day | Both flags set; still counts as off day; +1 credit only if hours > 0 |
| Status says `COMP OFF` but day-of-week is Sunday | **Calendar override kicks in** — treated as Weekly Off, not leave |
| Date not parseable | Skip row, warn user |
| Missing Employee Code or Name | Skip row or group under "UNKNOWN" |
| Multi-sheet attendance file | Process all sheets, deduplicate by emp code + date |
| Attendance spans multiple months | Process all; group output by month if needed |
| Working Hours column missing entirely | Fall back to Status-based detection; warn user that results may be less accurate |
| Employee has non-standard weekly off (e.g., Saturday) | Pre-scan detects it from the pattern of `Weekly Off` statuses in that employee's data |

---

## Notes for the User
- **No hardcoded holidays.** Company holidays are detected automatically from the Status column (`CH`, `Company Holiday`, `Holiday`). This works for any region, any year.
- **Working Hours is king.** The single most reliable way to know if someone was present or absent is the Working Hours column. All credit decisions are ultimately gated by this value.
- **Calendar verification catches hidden weekly offs.** The system pre-scans all `Weekly Off` statuses to learn which day-of-week is each employee's off day. Then for every `COMP OFF` row, it checks the Date column to see if that date falls on the weekly off day. If it does, no credit is deducted — even if hours are 0.
- **Day-of-week is logged.** The Attendance Detail sheet includes a "Day" column (Monday, Tuesday, etc.) so you can visually verify that COMP OFF entries on Sundays (or other weekly off days) were correctly identified and not penalized.
- Weekly offs are detected from the `Status` column (e.g., "WO", "W/O", "Weekly Off") to learn the pattern, and then **independently verified** against the Date column's day-of-week. This dual check prevents false deductions.
- COMP OFF is a type of leave. It represents 1 full day of earned leave. The status label in the sheet (`COMP OFF`, `CO`, `Comp Off`) just means the employee was scheduled to be on leave — Working Hours confirms whether they actually took it, and the calendar confirms whether it was a real working day.
