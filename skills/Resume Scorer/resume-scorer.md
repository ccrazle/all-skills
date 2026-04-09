---
name: Resume Scorer
description: >
  Score and rank job applicants by matching their resumes against a job description (JD).
  Use this skill whenever the user says things like "score resumes", "rank applicants",
  "evaluate candidates against a JD", "match resumes to job description", "access JD of
  [role name]", or uploads multiple PDF/DOCX resumes alongside a JD name or reference.
  The skill fetches the JD automatically from the configured GitHub repository, reads each
  resume, scores every applicant out of 100, and returns a formatted .xlsx report. Trigger
  even when the user doesn't use the word "skill" — if resumes + a JD role name are present
  and scoring/ranking is the goal, use this.
---

# Resume Scorer

Reads a job description (JD), extracts weighted requirements, scores each uploaded resume
against those requirements, and produces a ranked `.xlsx` scorecard.

---

## 1. Trigger Command

The user will typically say something like:

```
use resume scorer skill and access jd of purchase manager
```

or

```
use Resume Scorer for all uploaded resumes against the Quality Engineer JD
```

The user only needs to mention the **role name** — no link required. The skill automatically
fetches the JD from the configured GitHub repository (see Step 1 below).

Accept any variation that mentions a role name + resumes to score.

---

## 2. Workflow (Step-by-Step)

### Step 1 — Locate inputs

**Resumes:** Check `/mnt/user-data/uploads/` for all resume files (`.pdf`, `.docx`).

```bash
ls /mnt/user-data/uploads/
```

**JD — fetch automatically from GitHub:**

The JD repository is hardcoded at:
```
JD_GITHUB_FOLDER = https://github.com/ccrazle/all-skills/tree/main/references/JDs
JD_RAW_BASE      = https://raw.githubusercontent.com/ccrazle/all-skills/main/references/JDs/
```

Follow this process every time:

1. Fetch the folder listing page:
   ```
   web_fetch("https://github.com/ccrazle/all-skills/tree/main/references/JDs")
   ```

2. Scan the returned HTML/text for filenames. Look for any `.docx` file whose name
   **fuzzy-matches** the role name the user mentioned (case-insensitive, partial match).
   Examples of matches for "purchase manager":
   - `purchase_manager_jd.docx` ✅
   - `Purchase-Manager.docx` ✅
   - `PM_JD_final.docx` ❌ (too ambiguous — ask the user)

3. Construct the raw URL:
   ```
   raw_url = JD_RAW_BASE + matched_filename
   # e.g. https://raw.githubusercontent.com/ccrazle/all-skills/main/references/JDs/purchase_manager_jd.pdf
   ```

4. Fetch the file content via `web_fetch(raw_url)`.

**If no match is found:** List the available JD filenames to the user and ask them
to confirm which one to use. Do not proceed with scoring until the JD is confirmed.

**If the user provides a direct URL or uploads the JD manually:** Use that instead
of the GitHub fetch — direct input always takes priority.

### Step 2 — Parse the JD and build a weighted requirement set

Extract every stated requirement from the JD and assign it a **priority tier**:

| Tier | Signal words | Weight |
|------|-------------|--------|
| **Must-have** | must, required, mandatory, essential, minimum | 3 |
| **Should-have** | should, preferred, strongly preferred, expected | 2 |
| **Nice-to-have** | optional, nice to have, bonus, plus, desired | 1 |

> **Why this matters:** weighting by stated priority makes the score reflect what the hiring manager actually cares about, not just keyword overlap.

Group requirements into logical clusters (e.g., Technical Skills, Domain Knowledge, Soft Skills, Certifications, Experience). This structure makes scoring transparent and debuggable.

See `references/scoring-rubric.md` for the full scoring formula and edge-case rules.

### Step 3 — Extract text from each resume and the JD

**JD (always .docx):** Download the raw file and extract with `python-docx`:

```python
import requests
from docx import Document
from io import BytesIO

response = requests.get(raw_url)
doc = Document(BytesIO(response.content))
jd_text = "\n".join([p.text for p in doc.paragraphs])
```

**Resumes — PDF** (preferred extraction with `pdftotext`, fallback to `pypdf`):

```bash
pdftotext resume.pdf -
```

**Resumes — DOCX:**

```python
from docx import Document
doc = Document("resume.docx")
text = "\n".join([p.text for p in doc.paragraphs])
```

If text extraction yields less than 100 characters for any file, flag it as "Unable to extract — possibly scanned" and assign a score of N/A.

### Step 4 — Score each resume

For each resume, check every requirement cluster:

1. Search for the requirement's keywords (and reasonable synonyms) in the extracted text.
2. Apply partial credit (see `references/scoring-rubric.md`).
3. Multiply raw cluster score by tier weight.
4. Sum weighted scores, normalize to 100.

Record a **breakdown** per applicant: how many Must/Should/Nice requirements they met. This feeds the Excel output.

### Step 5 — Build and return the `.xlsx` report

Write the output to `/mnt/user-data/outputs/resume_scores.xlsx`.

Sheet layout → see `references/output-format.md` for full column spec and formatting rules.

**Minimum columns (Sheet: Scores):**

| # | Applicant Name | Must-Have Met | Should-Have Met | Nice-to-Have Met | Final Score (/100) | Priority Rank |
|---|---------------|--------------|----------------|-----------------|-------------------|--------------|

- Sort by Final Score descending (Rank 1 = best match).
- Add a second sheet `JD Requirements` listing every extracted requirement, its tier, weight, and keyword list — so the user can audit the scoring logic.
- Use openpyxl for formatting; apply conditional formatting (green/yellow/red) on the Final Score column.

After saving, call `present_files` to return the file to the user.

---

## 3. Output Format

- **File**: `resume_scores.xlsx`
- **Sheet 1 — Scores**: Ranked applicant table (see above)
- **Sheet 2 — JD Requirements**: Auditable requirement breakdown

See `references/output-format.md` for exact column widths, header styles, and conditional formatting thresholds.

---

## 4. Edge Cases

| Situation | Action |
|-----------|--------|
| No JD filename matches the role name | List available JDs from GitHub folder; ask user to confirm |
| GitHub folder is unreachable | Ask user to upload the JD file or paste text directly |
| No priority keywords in JD | Treat all requirements as equal weight (tier 2) |
| Scanned / image-only PDF | Mark score as N/A; note in sheet |
| Applicant name not in filename | Infer from resume content (first heading / "Name:" field); fall back to filename |
| Duplicate filenames | Append `(2)`, `(3)` etc. |
| Resume in language other than English | Note language; attempt scoring; flag as "low confidence" |
| User provides a direct JD URL or uploads JD manually | Use that — skip GitHub fetch entirely |

---

## 5. Reference Files

Load these only when you need the detail they contain — don't load all at once:

- `references/scoring-rubric.md` — Full scoring formula, synonym expansion rules, partial-credit table
- `references/output-format.md` — Exact xlsx column spec, openpyxl formatting code snippets, conditional formatting thresholds

---

## 6. Test Cases

See `tests/test-cases.md` for sample prompts and expected outputs to verify the skill is working correctly.
