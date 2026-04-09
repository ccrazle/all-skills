---
name: leave-credit
description: >
  Use this skill whenever the user says "use leave credit skill", "calculate leave credits",
  "process attendance for leave", or uploads an employee attendance Excel file and asks about
  comp offs, leave credits, or remaining leaves. This skill scans an employee attendance
  Excel/CSV file, applies company holiday rules, counts leave credits earned by working on
  Weekly Offs or Company Holidays, deducts credits for COMP OFF taken on working days, and
  outputs a final .xlsx report showing Employee Name, Employee Code, and Leave Credits Left.
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
| Working Hours | Hours worked |
| Status | See Status Values below |
| Totals | (ignored for credit logic) |

### Status Values and Their Meaning
| Status Value | Meaning | Leave Credit Impact |
|---|---|---|
| `WO` / `Weekly Off` / `W/O` | Paid weekly rest day | No change (unless employee worked) |
| `CH` / `Company Holiday` / `Holiday` | Company holiday | No change (unless employee worked) |
| `COMP OFF` / `Comp Off` / `CO` | Employee used a comp off | **−1 credit** (only if it's a working day) |
| `A` / `Absent` | **Treated as Comp Off** — employee took the day off | **−1 credit** (only if it's a working day) |
| `P` / `Present` | Normal working day, present | No change |
| Present on `WO` or `CH` | Employee worked on day off/holiday | **+1 credit** |

> **Key Rule**: Both `Absent` and `COMP OFF` statuses are treated identically — they represent the employee taking a comp off day. A deduction of −1 credit only applies if the day is a **working day** (not a Weekly Off or Company Holiday).

---

## Company Holidays (2026) — HEAD OFFICE / ROHTAK / BHIWADI

Hardcoded from the official company holiday list:

```
1.  New Year              → 01-01-2026
2.  Republic Day          → 26-01-2026
3.  Holi                  → 04-03-2026
4.  Mahavir Jayanti       → 31-03-2026
5.  Independence Day      → 15-08-2026
6.  Raksha Bandhan        → 28-08-2026
7.  Mahatma Gandhi Jayanti→ 02-10-2026
8.  Dussehra              → 20-10-2026
9.  Diwali                → 08-11-2026
10. Govardhan Pooja/Vishkarma Day → 09-11-2026
```

Store these as a Python set of `datetime.date` objects for comparison.

---

## Processing Logic

### Step 1 — Load the Attendance File
- Use `pandas` to read the uploaded Excel/CSV file.
- Auto-detect header row (may not always be row 1 — look for rows containing "Employee Code", "Date", "Status").
- Normalize column names: strip whitespace, lowercase for matching.
- Parse the `Date` column to `datetime.date`.

### Step 2 — Identify Working Days vs. Off Days
For each row:
- Check if the date is in **Company Holidays** list → mark as `company_holiday`
- Check if `Status` matches weekly off patterns → mark as `weekly_off`
- Otherwise → mark as `working_day`

### Step 3 — Detect "Worked on Off Day" (+1 Credit)
A credit is **earned** if:
- Date is a `weekly_off` OR `company_holiday`, AND
- Employee has a non-zero Working Hours OR Status contains "Present" / "P"

### Step 4 — Detect "COMP OFF / Absent on Working Day" (−1 Credit)
A credit is **deducted** if:
- Status matches `COMP OFF` / `Comp Off` / `CO` **OR** `A` / `Absent`, AND
- Date is NOT a weekly_off AND NOT a company_holiday (i.e., it is a working day)

> `Absent` = `COMP OFF` — both mean the employee took a comp off day.

### Step 5 — Aggregate Per Employee
Group by `Employee Code` + `Employee Name`:
```
leave_credits = (days worked on WO or CH) - (COMP OFFs taken on working days)
```
Credits cannot go below 0 unless the business logic explicitly allows negative.

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
| Comp Offs Taken on Working Days (Credits Used) | Count of −1 events |
| **Leave Credits Left** | Earned − Used |

### Sheet: "Attendance Detail" (optional but recommended)
Row-by-row breakdown showing each date, status, and credit impact (+1 / −1 / 0) per employee.

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
from openpyxl.styles import Font, PatternFill, Alignment, PatternFill
from openpyxl.utils import get_column_letter
from datetime import date
import re

# --- Company Holidays 2026 ---
COMPANY_HOLIDAYS = {
    date(2026, 1, 1), date(2026, 1, 26), date(2026, 3, 4),
    date(2026, 3, 31), date(2026, 8, 15), date(2026, 8, 28),
    date(2026, 10, 2), date(2026, 10, 20), date(2026, 11, 8),
    date(2026, 11, 9)
}

WEEKLY_OFF_PATTERNS = re.compile(r'\bw/?o\b|weekly[\s_-]?off', re.IGNORECASE)
COMP_OFF_PATTERNS   = re.compile(r'\bco\b|comp[\s_-]?off|\babsent\b|\b^a$\b', re.IGNORECASE)
PRESENT_PATTERNS    = re.compile(r'\bpresent\b|\bp\b', re.IGNORECASE)
HOLIDAY_PATTERNS    = re.compile(r'\bch\b|company[\s_-]?holiday|\bholiday\b', re.IGNORECASE)
ABSENT_PATTERNS     = re.compile(r'\babsent\b|^\s*a\s*$', re.IGNORECASE)

def is_weekly_off(status):
    return bool(WEEKLY_OFF_PATTERNS.search(str(status)))

def is_comp_off(status):
    return bool(COMP_OFF_PATTERNS.search(str(status)))

def is_absent(status):
    return bool(ABSENT_PATTERNS.search(str(status).strip()))

def is_present(status):
    return bool(PRESENT_PATTERNS.search(str(status)))

def is_holiday_status(status):
    return bool(HOLIDAY_PATTERNS.search(str(status)))

def is_credit_deduction(status):
    """Absent and COMP OFF both count as taking a comp off day."""
    return is_comp_off(status) or is_absent(status)

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
    df['_hours'] = pd.to_numeric(df.get(col_map.get('hours', '_none'), 0), errors='coerce').fillna(0)
    df['_emp_code'] = df[col_map['emp_code']].astype(str).str.strip()
    df['_emp_name'] = df[col_map['emp_name']].astype(str).str.strip()
    
    # Remove total/summary rows
    df = df[df['_date'].notna()]
    df = df[~df['_emp_code'].str.lower().isin(['nan', 'total', 'totals', ''])]
    
    results = {}
    
    for _, row in df.iterrows():
        key = (row['_emp_code'], row['_emp_name'])
        if key not in results:
            results[key] = {'earned': 0, 'used': 0, 'detail': []}
        
        d = row['_date']
        status = row['_status']
        hours = row['_hours']
        
        is_ch = (d in COMPANY_HOLIDAYS) or is_holiday_status(status)
        is_wo = is_weekly_off(status)
        is_off_day = is_ch or is_wo
        
        credit_change = 0
        reason = ''
        
        if is_off_day and (is_present(status) or hours > 0):
            # Worked on a day off → earn credit
            results[key]['earned'] += 1
            credit_change = +1
            reason = 'Worked on WO/Holiday'
        elif is_credit_deduction(status) and not is_off_day:
            # Absent or Comp Off on a working day → use credit
            results[key]['used'] += 1
            credit_change = -1
            reason = 'Absent/Comp Off on Working Day'
        
        results[key]['detail'].append({
            'Date': str(d), 'Status': status,
            'Is Holiday': is_ch, 'Is Weekly Off': is_wo,
            'Credit Change': credit_change, 'Reason': reason
        })
    
    return results

def write_output(results, output_path, month_label=''):
    wb = Workbook()
    ws = wb.active
    ws.title = 'Leave Credits Summary'
    
    header_fill = PatternFill('solid', start_color='BDD7EE')
    bold = Font(bold=True, name='Arial', size=11)
    normal = Font(name='Arial', size=11)
    
    # Title
    ws.merge_cells('A1:E1')
    ws['A1'] = f'Leave Credits Report — {month_label}'
    ws['A1'].font = Font(bold=True, name='Arial', size=13)
    ws['A1'].alignment = Alignment(horizontal='center')
    
    headers = ['Employee Code', 'Employee Name',
               'Credits Earned (WO/Holiday Work)',
               'Credits Used (Comp Off)',
               'Leave Credits Left']
    
    for col, h in enumerate(headers, 1):
        cell = ws.cell(row=2, column=col, value=h)
        cell.font = bold
        cell.fill = header_fill
        cell.alignment = Alignment(horizontal='center', wrap_text=True)
    
    green_fill = PatternFill('solid', start_color='C6EFCE')
    red_fill   = PatternFill('solid', start_color='FFC7CE')
    
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
    detail_headers = ['Employee Code', 'Employee Name', 'Date', 'Status',
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
            for col, key in enumerate(
                ['Date', 'Status', 'Is Holiday', 'Is Weekly Off', 'Credit Change', 'Reason'], 3
            ):
                ws2.cell(row=r, column=col, value=entry[key])
            r += 1
    
    for col in range(1, 9):
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
| `Absent` on a working day | Treated as COMP OFF → **−1 credit** |
| `Absent` on a Weekly Off | No deduction — it's already a day off |
| `Absent` on a Company Holiday | No deduction — it's already a holiday |
| Employee works AND is marked absent same day | Status wins; flag as anomaly |
| Date not parseable | Skip row, warn user |
| Company holiday + weekly off on same day | Both flags set; still counts as off day |
| Missing Employee Code or Name | Skip row or group under "UNKNOWN" |
| Multi-sheet attendance file | Process all sheets, deduplicate by emp code + date |
| Attendance spans multiple months | Process all; group output by month if needed |
| COMP OFF on a Company Holiday | **No deduction** — it's already a holiday |
| COMP OFF on a Weekly Off | **No deduction** — it's already a day off |

---

## Notes for the User
- The company holiday list is hardcoded for **2026** (HEAD OFFICE / ROHTAK / BHIWADI).
- If holidays change year-over-year, update the `COMPANY_HOLIDAYS` set in the script.
- Weekly offs are detected from the `Status` column (e.g., "WO", "W/O", "Weekly Off"). Ensure your attendance system uses consistent labeling.
