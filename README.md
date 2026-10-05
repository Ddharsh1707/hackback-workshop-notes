# HACKBACK Workshop Notes

Notes from the HACKBACK reverse hackathon (dBug Labs), Monday 5 October 2026: what we set up, what we found in the workshop repos, the mistakes we made and fixed, and what happens next.

Playbook: https://dbuglabshackback.vercel.app/playbook

## Contents

| File | What it is |
|---|---|
| `README.md` | This log: setup, findings, mistakes, next steps |
| `accountill-notes.md` | Solo lab deliverable: 5 verified claims plus 1 correction |
| `CHATGPT_PROMPT.md` | Prompt to share with teammates who use ChatGPT as a coach |

---

## 1. What HACKBACK is

A reverse hackathon in three steps:

1. **Reverse it.** Point an AI agent (Antigravity IDE) at a real app's code and ask it 9 questions (stages 0–8). Check every answer.
2. **Spec it.** Turn the verified answers into 7 docs (stage 9).
3. **Rebuild it.** Build the core of the product in a new, empty repo from our docs only, plus 2 improvements.

**The one rule:** every claim needs evidence, in this format:

```
- <claim>
  Evidence: path/to/file:line [Confirmed]
```

- **Confirmed:** we opened the line ourselves and it proves the claim.
- **Likely:** strong signs, but not proven.
- **Guess:** no evidence; never used as fact.

To check a claim: `Ctrl+P` opens a file, `Ctrl+G` jumps to a line, and `Ctrl+Shift+F` searches the whole repo.

---

## 2. Setup

All work lives in `C:\Users\S DIVYA DHARSHAN\hackback`:

```
hackback/
├── espionage-event/          original repo (read only)
├── accountill/               original repo (read only)
├── accountill-notes.md       solo lab notes (format-correct version)
├── accountill-notes-agent.md the Antigravity agent's long-form notes
└── hackback-workshop-notes/  this repo
```

Commands used:

```bash
git clone --depth 1 https://github.com/dBug-Labs/espionage-event
git clone --depth 1 https://github.com/panshak/accountill
```

**Tools on the laptop:**
- Node v24.21.0 and npm 11.19.0
- Python 3.14.0
- Antigravity IDE (installed)
- MongoDB and Docker are not installed. They aren't needed, because the workshop only reads code and never runs it.

### The two workshop repos

| Repo | What it is | Stack |
|---|---|---|
| espionage-event | Registration and contest platform for a two-round college coding event (OTP signup, RSVP, QR attendance, MCQ round, coding round with AI grading) | Next.js 16, React 19, TypeScript, MongoDB/Mongoose 9, Nodemailer, Piston, OpenRouter |
| accountill | Invoicing app for freelancers (invoices, clients, PDF and email) | Express, Mongoose 5, MongoDB, React 17, react-scripts 4.0.3 |

---

## 3. Solo lab: accountill (12:02, 20 minutes)

Task: run stages 0, 3 (server only), 4, 7 and 8, then deliver 5 claims in OBSERVATIONS format, with at least 1 correction of the agent.

### Stage 0: Recon
- **Server:** Express, Mongoose 5, html-pdf, nodemailer, jsonwebtoken. See `server/package.json`.
- **Client:** React 17, Redux, Material UI v4, react-scripts 4.0.3. See `client/package.json`.
- **Server env variables (7):** `DB_URL`, `PORT`, `SECRET`, `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`.
- **Odd files:**
  - `server/invoice.pdf`: a generated file committed to git.
  - `client/build/`: compiled output.
  - `server/middleware/auth.js`: dead code; it is never imported.

### Stage 3: Routes (server)

| Mounted at | File | Handlers |
|---|---|---|
| `/invoices` | `server/routes/invoices.js` | 6 |
| `/clients` | `server/routes/clients.js` | 5 |
| `/profiles` | `server/routes/profile.js` | 5 (line 7 is commented out) |
| `/users` | `server/routes/userRoutes.js` | 4 |
| `/` | `server/index.js:53, 87, 97, 102` | 4 (`/send-pdf`, `/create-pdf`, `/fetch-pdf`, `/`) |
| **Total** | | **24** |

**Who may call each route:** anyone. No route has an auth check.

### Stage 4: Data model

There are 4 Mongoose models: User, Profile, Client and Invoice.
- **Unique constraints:** only `User.email` (`server/models/userModel.js:5`) and `Profile.email` (`server/models/ProfileModel.js:5`).
- **Owner links are plain string arrays, not refs:** `creator: [String]` (`InvoiceModel.js:15`) and `userId: [String]` (`ClientModel.js:9`).
- **Client details are copied into each invoice instead of linked:** `InvoiceModel.js:17`.
- **Prices and quantities are stored as text:** `InvoiceModel.js:6`.

### Stage 7: Gaps found in accountill

| # | Type | What is wrong | Evidence | Who it hurts | Fix | Severity |
|---|---|---|---|---|---|---|
| 1 | Security | The auth middleware exists but no route uses it, so every API route is public | `server/middleware/auth.js:7`; no imports in `server/routes/*.js` | Every freelancer and their clients | Apply the middleware to all non-auth routes | High |
| 2 | Security | `GET /invoices/:id` returns any invoice by ID with no owner check | `server/controllers/invoices.js:72` | Freelancers | Check that the invoice's `creator` matches the logged-in user | High |
| 3 | Security | `GET /clients` returns every user's clients, paginated | `server/controllers/clients.js:45` | Freelancers' customers | Filter by the logged-in user | High |
| 4 | Security | The server trusts the user ID sent in the query string | `server/controllers/invoices.js:10-13` | Freelancers | Take the user ID from the verified token | High |
| 5 | Correctness | Every PDF is written to one shared `invoice.pdf`, so simultaneous requests can overwrite each other | `server/index.js:57`, `:88`, `:98` | Freelancers and their clients | Use a unique file per request, or stream the PDF from memory | High |
| 6 | Correctness | On a PDF error the handler sends a response twice | `server/index.js:74` and `:76` | Freelancers | Return after the error response | Medium |
| 7 | Correctness | Auth middleware never responds or calls `next()` when token checking fails, so the request hangs | `server/middleware/auth.js:29-31` | Everyone, once the middleware is used | Send a 401 in the catch block | Medium |
| 8 | Data | `createdAt` defaults to `new Date()`, which is evaluated once at server start, so every record gets the boot time | `server/models/InvoiceModel.js:21`, `ClientModel.js:12` | Freelancers sorting or reporting by date | Use `default: Date.now` | Medium |
| 9 | Data | Nothing prevents duplicate invoice numbers | `server/models/InvoiceModel.js:13` | Freelancers | Add a unique index on creator plus invoiceNumber | Medium |
| 10 | Security | TLS certificate checks are disabled for SMTP | `server/index.js:46`, `server/controllers/user.js:98` | Everyone receiving email | Remove `rejectUnauthorized:false` | Medium |
| 11 | Dead code | `getClient` exists but no route calls it | `server/controllers/clients.js:24` | Developers | Route it or remove it | Low |
| 12 | Data | Money values are stored as strings | `server/models/InvoiceModel.js:6` | Freelancers (totals) | Use Number or Decimal128 | Low |

### Stage 8: Deliverable

See [`accountill-notes.md`](accountill-notes.md). It has 5 Confirmed claims and 1 correction: the env variable count was first given as 6, but the code reads 7.

The Antigravity agent's own long-form notes, in `hackback/accountill-notes-agent.md`, reached the same 5 conclusions. They need reformatting into the 2-line evidence format before they can be handed in.

---

## 4. Mistakes we made, and how we fixed them

Recording these so we don't repeat them tonight.

| # | What went wrong | Why it mattered | Fix | Lesson |
|---|---|---|---|---|
| 1 | We ran `npm install` in both repos and created `.env` files, then started to install MongoDB | The playbook says "read it, don't run it". The install changed accountill's `client/package-lock.json` and `server/package-lock.json` | Reverted the lockfiles with `git checkout` and deleted `node_modules` and the `.env` files | Read the playbook before setting anything up. Workshop repos are read-only |
| 2 | Our walkthrough said "6 env variables" but listed 7 | A wrong count would have gone into the notes as fact | Recounted from the code: `server/index.js:39-43`, `:106-107`, `server/middleware/auth.js:5` | Even the coach is wrong sometimes. Count from the code |
| 3 | The Antigravity agent saved `accountill-notes.md` inside the accountill repo, and it was staged for commit | It broke the "never change the original repo" rule | Unstaged it and moved it to `hackback/accountill-notes-agent.md` | Start every chat with "Do not create, edit or save any files" |
| 4 | `server/models/InvoiceModel.js` was modified (spacing only, from auto-format on save) and staged | It changed the original repo | `git restore --staged --worktree server/models/InvoiceModel.js` | Turn off auto-save and format-on-save |
| 5 | A `COMMIT_EDITMSG` tab was open, meaning a commit had been started in the original repo | A commit would have permanently changed the original's history | Closed the tab without committing; closed unsaved tabs with "Don't Save" | Never click Commit in an original repo. Source Control should show 0 changes |
| 6 | The agent's notes used "Status: VERIFIED" and code blocks, and ran to 244 lines | The lab requires exactly 5 claims in the 2-line evidence format | Told the agent to rewrite in the exact format; a format-correct version is also kept | Check the deliverable format before handing in |
| 7 | Common agent error to watch for: "routes are protected by the auth middleware" | False in accountill; the middleware is never imported | Proved with `Ctrl+Shift+F` for `middleware/auth` (no imports) | A file existing does not mean it is used |

### Side issue: C++ practice file (not workshop)
- **Error:** `gcc.exe: error: practice: No such file or directory` and `.c: No such file or directory`.
- **Cause:** the file was named `practice .c`, with a space, and C++ code was saved as a `.c` file and compiled with `gcc`.
- **Fix:** rename it to a name with no spaces ending in `.cpp` (for example `employee.cpp`), then compile with `g++ employee.cpp -o employee`.

---

## 5. Safe-working checklist for tonight

- [ ] The first message in every agent chat is: "Do not create, edit or save any files. Do not install or run anything. Only read and answer."
- [ ] Auto-save is off in Antigravity.
- [ ] Source Control shows 0 changes in the original repo before leaving it.
- [ ] Notes are saved outside the original repo.
- [ ] Every claim has a `path:line`, and at least 3 per stage are opened by hand.

---

## 6. Next steps

| When | What |
|---|---|
| 3 PM Mon | Card Drop: get the original repo, Rebuild Brief and 3 Killer Tests. Clone the repo into `hackback`. |
| 3–4 PM | Run stages 0–8 on the card's repo in one chat. Stage 5 traces the Killer Test flow. |
| 4 PM | Create a new, empty, public GitHub repo. No commits before 4 PM; never fork or copy the original. |
| 4–10 PM | Run Stage 9 to write the 7 docs into our repo's `docs/`, then read and fix them. Push `docs/` before any code. |
| 10 PM Mon | Checkpoint: repo link submitted and first docs pushed (otherwise −1 Visa). |
| Overnight | Rebuild from docs only: make the 3 Killer Tests pass, then add 2 improvements from GAPS.md. |
| 8:30 AM Tue | Docs freeze. |
| 12:30 PM Tue | Code freeze. |

**Required in our rebuild repo:**
- `README.md`
- `SUBMISSION.md`
- `deck.pdf`
- `.env.example`
- `docs/OBSERVATIONS.md`
- `docs/PRD.md`
- `docs/ARCHITECTURE.md`
- `docs/DATA_MODEL.md`
- `docs/API.md`
- `docs/GAPS.md`
- `docs/AGENT_LOG.md`

**Clean-room rules:** empty repo after 4 PM, docs before code, no original code or packages, no secrets in git. Breaking these is Game Over.
