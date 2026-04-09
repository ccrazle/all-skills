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

HR uploads resumes and a JD together. Claude reads everything directly, scores each
applicant out of 100, and returns a ranked `.xlsx` scorecard. No external calls needed.

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

### Step 5 — Score each resume against the JD

For each resume, evaluate every requirement cluster using this scoring model:

| Category | What to assess | Weight |
|----------|---------------|--------|
| **Keyword & Skill Match** | Overlap of skills/tools between resume and JD | 46% |
| **Experience Match** | Years of experience, domain relevance | 23% |
| **Job Title / Role Alignment** | Candidate's past titles vs. the JD role | 15% |
| **Education Match** | Degree, field of study, certifications | 15% |

Apply JD tier weights (Must/Should/Nice) on top of the category scores to produce
the final weighted score out of 100.

---

### Step 6 — Build and return the `.xlsx` report

Write output to `/mnt/user-data/outputs/resume_scores.xlsx`.

**Sheet 1 — Scores** (sorted by Final Score descending):

| Rank | Applicant Name | Final Score (/100) |
|------|---------------|-------------------|

Apply conditional formatting on Final Score:
- 🟢 70–100 → Green (Strong match)
- 🟡 40–69 → Yellow (Partial match)
- 🔴 0–39 → Red (Poor match)

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
