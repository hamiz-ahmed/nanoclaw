# Job Apply Skill

Automated job application workflow for NanoClaw. Saves job links, generates tailored cover letters, fills application forms via browser automation, and submits — all through Telegram chat with human approval at every gate.

---

## Prerequisites

You need a working **NanoClaw** install with **Telegram** as the channel. If you have not done that yet:

1. Install [Claude Code](https://claude.com/product/claude-code) (the `claude` CLI).
2. Fork and clone this repo:

```bash
gh repo fork qwibitai/nanoclaw --clone
cd nanoclaw
```

3. Run `claude`, then inside the Claude Code prompt:

```text
/setup
```

`/setup` walks you through Node, dependencies, container runtime, and service startup. When it asks which channel to add, choose **Telegram** (or run `/add-telegram` after setup). You will need a bot token from [@BotFather](https://t.me/BotFather). `/setup` also registers your Telegram chat so NanoClaw knows where to listen — that registered chat is where `jobs.csv` and all job state lives.

Once you can message your bot in Telegram and get a reply, NanoClaw is ready.

---

## Configuration (`.env`)

All job-apply inputs go in the project root `.env` file. Add these lines:

```bash
# --- Job Apply Skill ---

# Path to your resume PDF (relative to project root)
CV_PATH=cv/your-name-resume.pdf

# CapSolver API key for automatic CAPTCHA solving
CAPSOLVER_API_KEY=CAP-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# Cover letter model: "claude" (default) or "kimi"
JOB_APPLY_MODEL=claude

# Only needed if JOB_APPLY_MODEL=kimi
KIMI_API_KEY=
```

These are read by `src/container-runner.ts` at startup and:

- **`CV_PATH`**, **`JOB_APPLY_MODEL`**, **`KIMI_API_KEY`** are passed as environment variables into each agent container. The agent and the `kimi` helper script read them directly from `$CV_PATH`, `$JOB_APPLY_MODEL`, `$KIMI_API_KEY`.
- **`CAPSOLVER_API_KEY`** is auto-injected into `CapSolver/assets/config.js` before each container run. You do not need to manually edit `config.js` — just set the key in `.env`.

---

## Setting up your CV

1. Create a `cv/` folder at the project root.
2. Place your resume PDF inside it.
3. Set `CV_PATH` in `.env` to match (e.g. `CV_PATH=cv/Jane_Doe_Resume.pdf`).

The `cv/` folder is mounted read-only into containers at `/workspace/project/cv/`.

---

## Setting up your profile

The skill reads your candidate data from `container/skills/job-apply/profile.json`. This file drives cover letter personalization and form field mappings.

**Generate it from your CV using Claude Code.** Run `claude` from the repo root, then:

```text
Read my resume at cv/your-name-resume.pdf and create container/skills/job-apply/profile.json with this structure:

{
  "personal": {
    "first_name": "",
    "last_name": "",
    "full_name": "",
    "email": "",
    "phone": "",
    "location": "",
    "linkedin": "",
    "github": "",
    "medium": "",
    "work_authorization": ""
  },
  "current_role": {
    "title": "",
    "company": ""
  },
  "experience_years": 0,
  "salary_expectation": {
    "amount": 0,
    "currency": "EUR"
  },
  "work_history": [
    {
      "company": "",
      "title": "",
      "dates": "",
      "projects": [
        {
          "name": "",
          "description": "",
          "outcome": "",
          "technologies": []
        }
      ]
    }
  ],
  "education": [],
  "skills": [],
  "certifications": []
}

Fill every field from my CV. For work_history, include specific project names, measurable outcomes (numbers/percentages), and technologies used. These details are critical for cover letter quality.
```

Review the generated file and fix anything the extraction missed.

---

## Customizing SKILL.md for your profile

The cover letter prompts and form field mappings in `SKILL.md` need to match **your** data. Run `claude` from the repo root, then:

```text
Read container/skills/job-apply/SKILL.md and container/skills/job-apply/profile.json. Make these changes in SKILL.md:

1. Update the "Specificity rules" mapping in the cover letter Step 2 to match my actual projects from profile.json
2. Update the field mappings table in Phase 4 with my real personal data from profile.json (name, email, phone, location, LinkedIn, GitHub, current company, title, years of experience, work authorization, salary)
3. Update the CV upload path to: /workspace/project/<my CV_PATH from .env>

Keep all other instructions, prompts, review criteria, and phases unchanged.
```

---

## Setting up CapSolver (CAPTCHA solving)

Most job portals have CAPTCHAs. Without a solver, form filling will fail on protected sites.

### Step 1 — Get a CapSolver API key

1. Go to [capsolver.com](https://www.capsolver.com/) and create an account.
2. Open the **Dashboard** and copy your **API key** (labeled "API Key" on the main dashboard page, starts with `CAP-`).
3. Add balance to your account — solving consumes credits per solve. Even a few dollars is enough to start.

### Step 2 — Add the key to `.env`

```bash
CAPSOLVER_API_KEY=CAP-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

That's it. NanoClaw reads this at startup and auto-injects it into the CapSolver browser extension config (`CapSolver/assets/config.js`) before each container run. You never need to edit `config.js` manually.

### Step 3 — Install the extension folder

1. Download the latest release ZIP from [capsolver-browser-extension releases](https://github.com/capsolver/capsolver-browser-extension/releases).
2. Unzip it.
3. Place the unzipped folder at **`CapSolver/`** in the NanoClaw repo root (sibling of `src/`, `container/`, etc.).

You should have `CapSolver/manifest.json` at minimum. The `assets/config.js` file is gitignored — NanoClaw generates it from your `.env` key.

### Step 4 — Rebuild the container

```bash
./container/build.sh
```

### Verifying it works

After a real run, check the CapSolver dashboard — balance/usage should move when solves happen. If solves fail: check the key in `.env`, check your balance, confirm `CapSolver/manifest.json` exists at repo root.

---

## How it works

### Trigger phrases (send in your registered Telegram chat)

| What you want | What to send |
| --- | --- |
| Save a job for later | `save job <url>` or just paste a job URL |
| Start applying | `apply to job <id>` or `apply to saved jobs` |
| Approve cover letter | `ok`, `approved`, `yes` |
| Give revision feedback | Any message that isn't an approval keyword |
| Confirm email verification | `done` |
| Submit the application | `submit` |
| Cancel current run | `abort`, `stop`, `cancel` |

### Example flow

```
You: save job https://company.com/jobs/123
Bot: ✅ Saved "Role" at "Company" (ID: 1)

You: apply to job 1
Bot: [analysis + cover letter draft]
Bot: Reply ok to proceed, or give feedback

You: ok
Bot: ⏳ Opening application form...
Bot: ✍️ First name → ...
Bot: ✍️ Email → ...
Bot: ✅ All fields filled. Reply submit or tell me what to change.

You: submit
Bot: 🚀 Submitting...
Bot: ✅ Applied!
```

### State files

All state is written per registered Telegram chat (under `groups/<folder>/` on host, `/workspace/group/` in container):

- **`jobs.csv`** — queue of saved jobs with status tracking
- **`job_state.json`** — current in-progress application (deleted after submit/abort)
- **`cover-letter-<company>.txt`** — generated cover letter file
- **`confirmation-<company>.png`** — screenshot evidence after submission

---

## Architecture

```
Orchestrator (reads SKILL.md)
├── Job Researcher    — browser: scrape and extract JD
├── Cover Letter      — write + self-review loop
├── Form Filler       — browser: fill every field
└── Submitter         — browser: click submit + screenshot
```

Files in this skill folder:

| File | Purpose |
| --- | --- |
| `SKILL.md` | Full execution protocol — prompts, phases, field mappings |
| `ab` | Wrapper script that calls `agent-browser` |
| `kimi` | CLI helper for Moonshot Kimi API (optional model path) |
| `profile.json` | Your candidate data (you create this) |

---

## Full setup checklist

1. NanoClaw running with Telegram (`claude` → `/setup` → `/add-telegram`).
2. Resume PDF in `cv/`.
3. `.env` updated with `CV_PATH`, `CAPSOLVER_API_KEY`, and optionally `JOB_APPLY_MODEL` / `KIMI_API_KEY`.
4. `profile.json` generated from your CV (Claude Code command above).
5. `SKILL.md` field mappings and CV path customized (Claude Code command above).
6. `CapSolver/` folder at repo root (download from GitHub releases).
7. Container rebuilt: `./container/build.sh`.
8. Send `save job <url>` in Telegram and confirm a row appears.

---

## Troubleshooting

| Problem | Check |
| --- | --- |
| Bot does not respond | NanoClaw service running? Telegram chat registered? `logs/nanoclaw.log` |
| CAPTCHA not solved | `CAPSOLVER_API_KEY` set in `.env`? Balance on CapSolver dashboard? Container rebuilt? |
| Cover letter uses wrong details | `profile.json` populated? `SKILL.md` specificity rules updated? |
| CV upload fails | `cv/` folder has your PDF? `CV_PATH` in `.env` matches? Path in `SKILL.md` updated? |
| Form fills wrong values | Field mapping table in `SKILL.md` still has original author's data? Run the customization command. |
| `kimi` helper fails | `KIMI_API_KEY` set in `.env`? `JOB_APPLY_MODEL=kimi`? |
