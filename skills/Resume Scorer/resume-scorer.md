---
name: Resume Scorer
description: >
  Score and rank job applicants by matching their resumes against a job description (JD).
  Use this skill whenever the user says things like "score resumes", "rank applicants",
  "evaluate candidates against a JD", "match resumes to job description", or uploads
  resumes (PDF/DOCX) alongside a JD file. Claude reads all uploaded files directly —
  no GitHub, no API calls. Scores every applicant out of 100 and returns a ranked .xlsx
  report. Trigger even when the user doesn't say "skill" — if resumes + a JD are uploaded
  and ranking/scoring is the goal, use this.
---

# Resume Scorer

HR uploads resumes and a JD together. Claude reads everything directly, applies a
two-gate eligibility check (Experience + Education), scores each qualified applicant
out of 100, and returns a ranked `.xlsx` scorecard. No external calls needed.

---

## 1. Trigger Command

The user will say something like:

```
use Resume Scorer — here are the resumes and the JD
```

or simply upload multiple resume files + one JD file and ask to score/rank them.

**Identifying which file is the JD:**
- File named with "jd", "job description", or "role" in the name → treat as JD
- If ambiguous, ask the user: "Which file is the Job Description?"
- All other `.pdf` / `.docx` files are treated as resumes

---

## 2. Workflow (Step-by-Step)

### Step 1 — Locate all uploaded files

```bash
ls /mnt/user-data/uploads/
```

Separate files into:
- **JD** — one file (see identification rules above)
- **Resumes** — all remaining `.pdf` / `.docx` files

Confirm the split with a one-liner before proceeding:
> "Found 1 JD (`jd-purchase-manager.docx`) and 5 resumes. Starting scoring..."

---

### Step 2 — Extract text from the JD

**DOCX:**
```python
from docx import Document
doc = Document("jd.docx")
jd_text = "\n".join([p.text for p in doc.paragraphs])
```

**PDF:**
```bash
pdftotext jd.pdf -
```

Read the full JD text and move to Step 3.

---

### Step 3 — Parse JD into weighted requirements

Extract every stated requirement and assign a **priority tier** based on signal words:

| Tier | Signal words | Weight |
|------|-------------|--------|
| **Must-have** | must, required, mandatory, essential, minimum, necessary | 3 |
| **Should-have** | should, preferred, expected, ideally, strongly preferred | 2 |
| **Nice-to-have** | optional, nice to have, bonus, plus, desired, advantageous | 1 |

If no signal words are found on a requirement → default to **Should-have (weight 2)**.

Group requirements into clusters:
- **Technical Skills** — tools, languages, platforms, software
- **Domain Knowledge** — industry-specific expertise
- **Experience** — years, seniority, past roles
- **Education** — degrees, certifications, licenses
- **Soft Skills** — communication, leadership, teamwork

> Why cluster? It makes scoring transparent and gives the HR team a breakdown by
> category, not just a single number.

---

### Step 4 — Extract text from each resume

**PDF resumes** (use `pdftotext`, fallback to `pypdf`):
```bash
pdftotext resume.pdf -
```

**DOCX resumes:**
```python
from docx import Document
doc = Document("resume.docx")
text = "\n".join([p.text for p in doc.paragraphs])
```

**Extracting the applicant's name:**
- Look for a name in the first 5 lines of the resume (usually the largest/first text)
- Check for a "Name:" label
- Fall back to the filename if no name can be found

If extracted text is under 100 characters → flag as `"Unable to extract — possibly scanned"`, score = N/A.

---

### Step 5 — Eligibility Gate (Mandatory Pass/Fail Check)

Before scoring, every applicant must pass **two mandatory gates**. These are not
weighted scoring categories — they are binary pass/fail checks. If an applicant fails
either gate, they are **rejected** and do not proceed to scoring.

#### Gate A — Minimum Experience (Most Important)

1. Extract the minimum experience requirement from the JD (e.g., "minimum 5 years",
   "at least 3 years of experience", "5+ years").
2. From the resume, calculate the applicant's total relevant experience by looking at
   employment dates, years mentioned in summary/objective, or explicit statements
   like "X years of experience in…".
3. **Rule:** If the applicant's experience is **less than** the JD's minimum
   requirement → **REJECTED**. Mark status as `"Rejected — Insufficient Experience"`.

> Example: JD asks for "minimum 5 years". Applicant has 3 years → Rejected.

#### Gate B — Education (Second Most Important)

1. Extract the education requirement from the JD (e.g., "B.Tech required",
   "MBA mandatory", "Bachelor's degree in Computer Science").
2. From the resume, extract the applicant's highest qualification, field of study,
   and any relevant certifications.
3. **Rule:** If the applicant does **not** hold the minimum required degree or
   qualification stated in the JD → **REJECTED**. Mark status as
   `"Rejected — Education Criteria Not Met"`.

> Example: JD requires "MBA". Applicant holds only a Bachelor's → Rejected.

**Important notes on the gates:**
- If the JD does not explicitly state a minimum experience or education requirement,
  treat that gate as **auto-pass** for all applicants.
- If an applicant fails **both** gates, mark as `"Rejected — Insufficient Experience & Education"`.
- Rejected applicants still appear in the final report (with their rejection reason)
  but receive **no score** — the Final Score column shows `"REJECTED"` instead of a number.

---

### Step 6 — Score each qualified resume against the JD

**Only applicants who passed both gates proceed to scoring.**

For each qualified resume, evaluate using this scoring model:

| Category | What to assess | Weight |
|----------|---------------|--------|
| **Skills Match** | Overlap of technical skills, tools, platforms, software, domain knowledge, and soft skills between resume and JD. Includes keyword matching and contextual skill assessment. | **60%** |
| **Job Title / Role Alignment** | How closely the applicant's past job titles, roles, and responsibilities align with the JD's role. Checks whether the candidate has actually worked in a similar capacity. | **25%** |
| **Experience Depth** | Beyond the minimum gate, rewards additional relevant experience. An applicant with 8 years when JD asks for 5 scores higher than one with exactly 5. Also considers domain relevance and progression. | **10%** |
| **Education Strength** | Beyond the minimum gate, rewards higher qualifications, relevant certifications, prestigious institutions, or closely matched fields of study. | **5%** |

Apply JD tier weights (Must/Should/Nice) on top of the category scores to produce
the final weighted score out of 100.

> **Why this distribution?** Experience and Education already serve as hard gates —
> applicants who don't meet those minimums are already eliminated. Among qualified
> candidates, **Skills** is the strongest differentiator (60%), followed by
> **Role Alignment** (25%) to verify the candidate actually fits the position.
> The remaining 15% rewards depth beyond the minimum thresholds.

---

### Step 7 — Build and return the `.xlsx` report

Write output to `/mnt/user-data/outputs/resume_scores.xlsx`.

**Sheet 1 — Scores** (sorted by: Qualified applicants first by Final Score descending,
then Rejected applicants alphabetically):

| Rank | Applicant Name | Status | Experience Gate | Education Gate | Skills (/60) | Role Alignment (/25) | Experience Depth (/10) | Education Strength (/5) | Final Score (/100) | Rejection Reason |
|------|---------------|--------|----------------|---------------|-------------|---------------------|----------------------|------------------------|-------------------|-----------------|

- **Rank** — numbered only for qualified applicants; rejected applicants show `"—"`
- **Status** — `"Qualified"` or `"Rejected"`
- **Gate columns** — `"Pass"` or `"Fail"`
- **Score columns** — filled for qualified applicants; `"—"` for rejected
- **Final Score** — numeric for qualified; `"REJECTED"` for rejected
- **Rejection Reason** — blank for qualified; specific reason for rejected

Apply conditional formatting on Final Score:
- 🟢 70–100 → Green (Strong match)
- 🟡 40–69 → Yellow (Partial match)
- 🔴 0–39 → Red (Poor match)
- ⚫ REJECTED → Grey background

After saving, call `present_files` to return the file to the user.

---

## 3. Edge Cases

| Situation | Action |
|-----------|--------|
| Can't identify which file is the JD | Ask the user before proceeding |
| Scanned / image-only PDF resume | Score = N/A |
| Applicant name not found | Use filename as name |
| No priority keywords in JD | Default all requirements to Should-have (weight 2) |
| Duplicate filenames | Append `(2)`, `(3)` |
| Resume in non-English language | Attempt scoring, flag as "Low confidence" in score cell |
| Only 1 file uploaded | Ask if it's a resume or JD; request the missing file |
| JD has no minimum experience stated | Auto-pass all applicants on Experience Gate |
| JD has no education requirement stated | Auto-pass all applicants on Education Gate |
| Applicant's experience is ambiguous/undated | Estimate conservatively; flag as "Experience unclear" in notes |
| Applicant has equivalent qualification (e.g., PGDM vs MBA) | Treat widely recognized equivalents as passing; flag for HR review |
