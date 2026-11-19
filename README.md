# auto-apply: how it works

auto-apply finds internship and co-op postings, decides which ones fit, fills out the applications, checks its own
work, and submits the clean ones. Anything it can't answer from the candidate's own facts stops and waits for him.

Diagram: [`auto-apply-architecture.excalidraw`](auto-apply-architecture.excalidraw). Open it at
[excalidraw.com](https://excalidraw.com) with File → Open, or with the Excalidraw extension in VS Code.

## Technology

| Part | Technology |
|---|---|
| Language / runtime | Python 3.11 (virtual environment in `auto-apply\.venv`) |
| Browser automation | Playwright driving Chromium; agent workers drive Chrome on ports 9231+ |
| State | SQLite (WAL mode): `auto-apply\state\apply.sqlite` (postings, attempts, gates, key-value) |
| Decisions | Claude Code CLI (`claude -p`, Sonnet) on the candidate's Claude plan: screening, deciding, auditing, essays |
| Form handlers | Greenhouse, Lever, Ashby and Workday handlers; an AI agent worker for any other form |
| Tracking | Google Sheets API (tracker sheet), Gmail API read-only (confirmation emails) |
| Review UI | Local dashboard at http://127.0.0.1:8780 |
| Scheduling | Windows Task Scheduler: `auto-apply` (lanes, sync, reports) and `auto-apply-keepalive` (watcher and dashboard), both started at logon |

## Inputs: the profile folder

Everything Claude may claim comes from here (`AUTO_APPLY_PROFILE`). Nothing outside it is treated as a fact.

| File | Role |
|---|---|
| `WORKFLOW.md` | Binding rules: buckets (Tier 1 AI/FDE, Tier 2 ML/DS, Tier 3 SWE), terms (Spring/Winter 2027 co-ops first, Summer 2027, spring part-time; no Fall 2027), skip list |
| `candidate-profile.json` | Every form answer (contact, eligibility, demographics, policies). `null` means unknown, so the form holds and asks |
| `full-experience.md` | The only work and project facts that may be used. `TODO(you)` lines are never used |
| `materials/` | Two resumes: `swe` and `ml` (AI), with SHA-256 hashes so a changed file is never uploaded unreviewed |
| `settings.json` | Queue order, lanes, submit gate (`dry_run_target`), focus switch |

## The data flow

### 1. Poll
The watcher checks ~1,100 company job boards every few minutes and drops obvious misses by title
(PhD-only, sales, non-U.S., wrong term).
```
{"polled": 1092, "new": 119, "errors": 32}
→ {company: "Varda", role: "Site Reliability Internship - Spring 2027", location: "El Segundo, CA", status: "new"}
```

### 2. Screen (Claude, titles only)
25 titles per call.
```
in:  "763. Varda | Site Reliability Internship - Spring 2027 | El Segundo, CA"
out: {id: 763, verdict: "pursue"}                  → status: queued
     {id: 381, verdict: "skip", reason: "Master's-only"} → status: skipped
```

### 3. Queue
Order: co-ops in a target role (ML, data science, forward deployed, agentic/AI) → other co-ops → target roles → the
rest. Inside each group: newest first, then best fit.

### 4. Decide (Claude, full job description)
Claude reads the job description, the form's fields and the profile folder, and returns one decision:
```
bucket: "Tier 3 SWE"   term: "Spring 2027"   resume: swe
answers: First Name=Alejandro · Sponsorship=No · On-site El Segundo=Yes · Cover letter=<facts only>
decision: apply | skip (reason) | hold (the exact question)
```
Saved as `decision.json`.

### 5. Fill
Playwright fills the form through the ATS handler, reads every field back, and takes a screenshot
(`filled.json`, `filled.png`).

### 6. Audit (a second Claude check)
Compares what's actually on the page with the profile. For example, it caught an essay that stated motivations the
candidate never gave. Result in `audit.json`: clean or not clean, with the problems.

### 7. Outcome
| Outcome | Meaning | Seen in |
|---|---|---|
| Submitted | Clean and sent (gate open) | Dashboard → Submitted, tracker sheet |
| Dry run | Clean, filled, not sent (gate closed) | Dashboard → Dry runs |
| Hold | Needs the candidate: a question, sign-in, take-home | Dashboard → Holds, email alert |
| Skipped | Not a fit, with the reason | Database |
| Manual queue | The form wouldn't advance | Dashboard |

**Submit gate:** submission stays off until `dry_run_target` (6) dry runs in a row come out clean. Then it turns on,
the clean dry runs are re-queued and sent for real, and every clean application after that is submitted on its own.

## Review, then back into the queue

```
Hold → read the question (dashboard Holds tab or email)
     → give the answer
     → the answer is saved in candidate-profile.json, so it's reused on every later form
     → .venv\Scripts\python -m apply requeue <id> --basis "<the answer>"
     → status: queued → back to step 4, refilled from scratch with the new answer
```
Not a fit after all: `.venv\Scripts\python -m apply skip <id> --owner alejandro --reason "<why>"`.

Each application's record is in `auto-apply\state\evidence\<date>\<application>\`:
`decision.json` · `filled.json` · `filled.png` · `audit.json` · `cover_letter.txt` (when a form needed one).

## Run and stop

```powershell
Start-ScheduledTask -TaskName 'auto-apply'             # lanes: screen, decide, fill, audit, submit
Start-ScheduledTask -TaskName 'auto-apply-keepalive'   # watcher + dashboard
.venv\Scripts\python -m apply pause --all              # stop everything
```
Setup check: `python setup\doctor.py`.
