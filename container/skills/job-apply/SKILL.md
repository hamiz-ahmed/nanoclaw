---
name: job-apply
description: Automated job application system. Part 1 — collect a job link and save to jobs.csv. Part 2 — apply to saved jobs in phases: each phase ends with user approval before continuing. Trigger on: "save job <url>", "collect job", "apply to job", "apply to saved jobs", or when user sends a job posting URL. Also handles "ok"/"approved"/"submit" replies to continue a paused application.
allowed-tools: Bash(agent-browser:*)
---

# Job Application Automation

## IMPORTANT: Phase-based approval flow

This skill runs in phases. **Each phase ends by sending a message and stopping.** The agent does NOT proceed automatically — it waits for the user to reply before the next phase starts.

Phases:
1. **Collect** — open job URL, extract info, save to CSV. Done.
2. **Cover letter** — read JD, generate letter, send to user. STOP. Wait for "ok" or feedback.
3. **Fill form** — fill all fields, send summary. STOP. Wait for "submit" or corrections.
4. **Submit** — submit the form. Done.

State is persisted to `/workspace/group/job_state.json` between phases so the agent can resume.

---

## Progress Updates — ALWAYS ON

**Send a status message before every major operation and at least every 2 minutes during processing. Never go silent for more than 2 minutes.**

Format: `⏳ <what you are about to do / currently doing>`

Examples:
- `⏳ Opening job page...`
- `⏳ Analysing job description — extracting requirements...`
- `⏳ Mapping requirements to CV experience...`
- `⏳ Writing cover letter...`
- `⏳ Opening application form...`
- `⏳ Filling section 2 of 3 — education fields...`
- `⏳ Uploading CV...`
- `⏳ Waiting for page to load after clicking Next...`

If a browser wait exceeds 30 seconds, send a message before and after: `⏳ Waiting 70s for CAPTCHA resolution...` then `✅ Page ready, continuing...`

---

## Profile & CV

Always read at the start of any phase:
```bash
cat /home/node/.claude/skills/job-apply/profile.json
```

---

## Jobs CSV

Path: `/workspace/group/jobs.csv`
Headers: `id,url,title,company,status,date_added,date_applied,notes`
Status: `pending` | `cover_review` | `email_verify` | `form_review` | `applied` | `failed` | `skipped`

```bash
[ -f /workspace/group/jobs.csv ] || echo "id,url,title,company,status,date_added,date_applied,notes" > /workspace/group/jobs.csv
```

---

## State file

Path: `/workspace/group/job_state.json`

Used to pass data between phases. Write before stopping, read at the start of each resume.

```json
{
  "phase": "cover_review",
  "job_id": 1,
  "url": "...",
  "title": "...",
  "company": "...",
  "jd_text": "...",
  "cover_letter": "...",
  "cover_letter_file": "/workspace/group/cover-letter-<company-slug>.txt",
  "account_created": false,
  "portal_password": ""
}
```

`phase` values: `cover_review` | `email_verify` | `form_review`

When `phase` is `email_verify`: user has been asked to verify their email. On "done" reply, read state, re-open the URL, log in, then continue form filling.

---

## Phase 1: Collect Job Link

**Triggered by:** user sharing a URL, "save job", "collect job", "add this job"

**1.** Send: "🔍 Fetching job details..."

**2.**
```bash
/home/node/.claude/skills/job-apply/ab open <url>
/home/node/.claude/skills/job-apply/ab wait --load networkidle
/home/node/.claude/skills/job-apply/ab snapshot -c
/home/node/.claude/skills/job-apply/ab get text @<main-content-ref>
```
Extract title, company, and save the full JD text — you'll need it for the cover letter.

**3.** Get next id (count rows in CSV). Append:
```bash
echo '<id>,"<url>","<title>","<company>",pending,<YYYY-MM-DD>,,' >> /workspace/group/jobs.csv
/home/node/.claude/skills/job-apply/ab close
```

**4.** Send: "✅ Saved *<title>* at *<company>* (ID: <id>). Say *apply to job <id>* when you're ready."

---

## Phase 2: Cover Letter Generation

**Triggered by:** "apply to job", "apply to job ID <n>", "apply to saved jobs"

**2a. Read state and profile**
```bash
cat /home/node/.claude/skills/job-apply/profile.json
cat /workspace/group/jobs.csv
```

Open the job URL to get the full JD text:
```bash
/home/node/.claude/skills/job-apply/ab open <url>
/home/node/.claude/skills/job-apply/ab wait --load networkidle
/home/node/.claude/skills/job-apply/ab get text @<main-content-ref>
/home/node/.claude/skills/job-apply/ab snapshot -c
/home/node/.claude/skills/job-apply/ab close
```

Send: "⏳ Job description loaded for *<title>* at *<company>*. Extracting requirements..."

**2b. Analyse the JD — YOU MUST OUTPUT THIS ANALYSIS BLOCK BEFORE WRITING A SINGLE WORD OF THE LETTER**

Complete all three steps below in order. Write out each step as you go — do not skip, do not summarise, do not proceed to writing until the analysis is done.

**Step A — Extract requirements from the JD you just read**

List explicitly (copy-paste the key phrases from the JD, do not paraphrase into generic terms):
- Technical requirements (tools, methods, languages, systems): list 3–4
- Role requirements (client-facing, leadership, research, communication): list 1–2
- Company domain and what they actually build/sell: one sentence

**Step B — Map every requirement to a specific experience from `work_history` in profile.json**

For EACH requirement above, find the exact matching project/outcome from `work_history`. Write the mapping out line by line:

```
Requirement: [exact phrase from JD]
Match: [company] — [specific project name or system] — [outcome or number if available]
```

Rules:
- Only include a match if it is genuinely relevant. Do not force irrelevant experience in.
- If a requirement has no match, write "No match — skip".
- The specificity rules in `cover_letter_instructions.specificity_rules` in profile.json list exact project names to use for common JD keywords (RAG, LLMs, recommendations, agents, NLP, research, etc.) — use those.

**Step C — Check the style reference**

Read `cover_letter_base` in profile.json. This is the Anthropic cover letter — use its tone, sentence rhythm, and voice as a style reference. Do NOT copy its sentences. Use it to calibrate how the letter should sound.

Send: "⏳ Requirements mapped. Writing cover letter for *<title>* at *<company>*..."

**Step D — Write the letter using ONLY the matches from Step B**

**CRITICAL: Every letter must be written from scratch. Do NOT reuse sentences, openers, or paragraphs from any previous cover letter — not from `cover_letter_base`, not from a prior application. Use `cover_letter_base` only as a voice/tone reference, never as a template to copy from. The opener, the specific experiences selected, and the company-fit paragraph must be unique to this job and this company.**

Structure (4–5 paragraphs, no more):
- **Para 1:** Warm, direct opener. Name the role and company. Include one specific hook — something concrete about the company or role that connects to Hamiz's actual work. Never use "I am writing to express my strong interest" verbatim. Vary the opener each time.
- **Para 2:** The 2 most relevant experiences from Step B. Must name a specific project. Must include at least one outcome or number.
- **Para 3:** A second angle — consulting if the role needs client-facing/agent work, Fraunhofer if research-heavy, Regenold if healthcare/NLP. Choose based on the JD. Skip entirely if nothing genuinely matches.
- **Para 4:** Why this specific company. Reference their actual product, domain, or a real aspect of the role. Not a generic mission statement.
- **Closing:** Brief. Thank them. Mention Medium (https://medium.com/@hamizahmed). Sign off as Muhammad Hamiz Ahmed.

Voice rules from `cover_letter_instructions.voice_rules` in profile.json apply. Banned phrases from `cover_letter_instructions.things_to_avoid` apply.

**Self-check before saving: the letter must contain at least 2 specific project names and at least 1 number/outcome. If it doesn't, rewrite it.**

**2c. Save cover letter to a named file, save state, and STOP**

Create a company-specific cover letter file. Use the company name as the filename — lowercase, spaces replaced with hyphens:
```bash
# Example: company "BestSecret GmbH" → cover-letter-bestsecret-gmbh.txt
cat > /workspace/group/cover-letter-<company-slug>.txt << 'EOF'
<full letter text>
EOF
```

Save state with the file path:
```bash
cat > /workspace/group/job_state.json << 'EOF'
{
  "phase": "cover_review",
  "job_id": <id>,
  "url": "<url>",
  "title": "<title>",
  "company": "<company>",
  "cover_letter": "<full letter text — escape quotes>",
  "cover_letter_file": "/workspace/group/cover-letter-<company-slug>.txt"
}
EOF
```

Update CSV status to `cover_review`:
```bash
# Use sed or awk to update the status column for this job id
```

Send the analysis + cover letter preview to the user and STOP:

```
📋 *Analysis — <role> at <company>*

*JD requirements:*
• [requirement 1]
• [requirement 2]
• [requirement 3]
...

*CV matches:*
• [requirement 1] → [project/outcome]
• [requirement 2] → [project/outcome]
...

---

📄 *Cover letter preview:*

---
<full letter text>
---

Saved as: `cover-letter-<company-slug>.txt`

Reply *ok* to proceed to form filling, or give me feedback to revise.
```

**DO NOT open the browser. DO NOT fill any forms. STOP here.**

---

## Phase 3: Cover Letter Revision (if feedback given)

**Triggered by:** user replies with feedback on the cover letter (not "ok")

**3a.** Read state:
```bash
cat /workspace/group/job_state.json
cat /home/node/.claude/skills/job-apply/profile.json
```

**3b.** Revise the letter based on the feedback. The CV matches from the original analysis still apply — only adjust the writing, not the underlying experience mapping (unless the feedback specifically asks to change what's included).

Apply the same specificity and voice rules from `cover_letter_instructions` in profile.json.

**3c.** Save the revised letter to state file. Send it again:

```
📄 *Revised cover letter:*

---
<revised letter>
---

Reply *ok* to proceed, or give more feedback.
```

**STOP. Do not fill forms.**

---

## Phase 4: Form Filling

**Triggered by:** user replies "ok", "looks good", "approved", "proceed", "yes" after a cover letter — OR user replies "done" after email verification (check `job_state.json` phase to distinguish)

**If phase is `email_verify`:** skip cover letter — go straight to logging in and continuing the form.

**4a.** Read state:
```bash
cat /workspace/group/job_state.json
cat /home/node/.claude/skills/job-apply/profile.json
```

Send the approved cover letter as a reminder, then the status:

```
📄 *Cover letter being submitted for <role> at <company>:*

---
<cover_letter from job_state.json>
---

⏳ Opening application form...
```

**4b.** Navigate to application:
```bash
/home/node/.claude/skills/job-apply/ab open <url>
/home/node/.claude/skills/job-apply/ab wait --load networkidle
/home/node/.claude/skills/job-apply/ab snapshot -i
```

Send: "⏳ Page loaded. Locating Apply button..."

Find and click Apply button.

**If the page shows a login or signup wall (sign in required, create account, etc.):**

Check if an existing account might exist — try clicking "Sign in" first and enter `personal.email` from profile.json. If login fails (wrong password, no account found), proceed to account creation below.

**Account creation flow:**

1. Click "Create account" / "Sign up" / "Register"
2. Fill the signup form using profile.json:
   - Email: `personal.email`
   - First name: `personal.first_name`
   - Last name: `personal.last_name`
   - Password: generate a strong password in format `Ham!z<CompanyName><Year>` (e.g. `Ham!zAshby2026`) and save it to state
   - Any other required fields from profile.json
3. Submit the signup form
4. Save state with `phase: "email_verify"` and the generated password, then send:

```
📧 *Account created at <company> portal.*

I've registered with *hamizahmed93@gmail.com*. Please check your inbox and click the verification link.

Reply *done* once you've verified your email.
```

**STOP. Do not continue until user replies "done".**

5. When user replies "done" — read state, re-open the job URL, log in with the registered email + saved password, re-snapshot, then continue to fill the application form as normal.

State file additions for account creation:
```json
{
  "account_created": true,
  "portal_password": "<generated password>"
}
```

**4c.** Before starting to fill, send: "⏳ Application form open. Starting to fill fields..."

Fill every field. After each field, send:
```
✍️ *<Field Label>* → <value>
```

Field mappings from profile.json:

| Field | Value |
|---|---|
| First name | `personal.first_name` |
| Last name | `personal.last_name` |
| Full name | `personal.full_name` |
| Email | `personal.email` |
| Phone | `personal.phone` |
| Location / City | `personal.location` |
| LinkedIn | `personal.linkedin` |
| GitHub | `personal.github` |
| Current company | `current_role.company` |
| Current title | `current_role.title` |
| Years of experience | `experience_years` |
| Work authorization | `personal.work_authorization` |
| Salary expectation | `salary_expectation.amount` |
| Cover letter (text field) | The approved text from `job_state.json` → `cover_letter` |
| Cover letter (file upload) | The file at `job_state.json` → `cover_letter_file` |

For unknown required fields: ask the user, wait for reply, then fill.

Re-snapshot every 3–4 fields. For multi-page forms: send "⏳ Moving to next page...", click Next, re-snapshot, send "⏳ Page <n> — continuing to fill...", continue filling.

For CV upload (path comes from `$CV_PATH` env var):
```bash
/home/node/.claude/skills/job-apply/ab upload @<ref> "/workspace/project/$CV_PATH"
```
Send: "📎 *Resume* → Uploaded"

For cover letter file upload:
```bash
/home/node/.claude/skills/job-apply/ab upload @<ref> "<cover_letter_file from job_state.json>"
```
Send: "📎 *Cover Letter* → Uploaded (Cover Letter - <Company>)"

**4d.** When all fields are filled — DO NOT SUBMIT. Save state and stop.

Update state:
```bash
# Update phase to "form_review" in job_state.json
```

Send a full summary and STOP:

```
✅ *All fields filled for <title> at <company>.*

Here's what I entered:
• First Name: <First Name you entered>
• Last Name: <Last Name you entered>
• Email: <Email you entered>
• Phone: <Phone you entered>
• Location: <Location of the user>
• Current Role: <Current working role of the user>
• Years of Experience: <Work experience based on CV>
• Salary: <Expected salary for the role>
• Work Authorization: <Whether user has work authorization or not>
• Cover Letter: [as approved]
• Resume: Uploaded

Reply *submit* to submit the application, or tell me what to change.
```

**DO NOT click Submit. STOP here.**

---

## Phase 5: Submit

**Triggered by:** user replies "submit" after the form review summary

**5a.** Read state:
```bash
cat /workspace/group/job_state.json
```

Send: "🚀 Submitting application for *<title>* at *<company>*..."

**5b.**
```bash
/home/node/.claude/skills/job-apply/ab find role button click --name "Submit"
```
Send: "⏳ Submit clicked. Waiting for confirmation..."
```bash
/home/node/.claude/skills/job-apply/ab wait --load networkidle
/home/node/.claude/skills/job-apply/ab snapshot -c
```

Check for confirmation text ("Application submitted", "Thank you", "We received").

**5c.** Take a screenshot of the confirmation page and send it:
```bash
/home/node/.claude/skills/job-apply/ab screenshot /workspace/group/confirmation-<company-slug>.png
```

Then send via `mcp__nanoclaw__send_message` with:
- `image_path`: `/workspace/group/confirmation-<company-slug>.png`
- `text`: `📸 Confirmation page — <title> at <company>`

**5d.** Update CSV and send result:
- ✅ `status=applied`, `date_applied=today` → "✅ Applied to *<title>* at *<company>*!"
- ❌ `status=failed`, `notes=<reason>` → "❌ Failed: <reason>"

```bash
/home/node/.claude/skills/job-apply/ab close
```

Delete state file:
```bash
rm -f /workspace/group/job_state.json
```

---

## Viewing saved jobs

```bash
cat /workspace/group/jobs.csv
```

---

## Browser and CAPTCHA

All browser commands use the `ab` wrapper which runs agent-browser with the CapSolver extension loaded automatically. CAPTCHAs are solved without user intervention.

**After navigating to a page that may have a CAPTCHA, wait before interacting:**

- reCAPTCHA v2 (checkbox / image challenge): wait 70 seconds after page load
- reCAPTCHA v3 / Cloudflare Turnstile / all others: wait 35 seconds after page load

Use `/home/node/.claude/skills/job-apply/ab wait --timeout <ms>` for the pause.

**Do NOT mention CAPTCHAs or CapSolver to the user. Wait silently, then continue.**

---

## Error handling

| Situation | Action |
|---|---|
| CAPTCHA (any type) | Wait silently using the times above — CapSolver handles it |
| Login wall (existing account) | Try signing in with `personal.email`; if fails, create account |
| Login wall (no account) | Run account creation flow; stop for email verification |
| Email verification pending | Stop, send message, wait for "done" reply |
| Unknown field | Ask user. Stop. Resume on reply. |
| Page error | Screenshot, describe, mark failed. |
