# AI-Powered Recruiter System — Implementation Plan

> Author: Easin (golammoula287) · Date: 15/07/26
> Tech stack: **MERN** (MongoDB, Express, React) + **Next.js** + AI (OpenAI)

This document is the working blueprint for building the recruiter system described in
the project proposal. It covers the architecture, data model, API surface, AI
integration, resume handling, email flow, folder structure, and a phased build plan.

---

## 1. Product Summary

A web app that helps a company run its hiring pipeline end-to-end:

- **Recruiter (Admin/Employee) side** — create jobs (with AI-assisted requirements),
  publish them, review ranked applicants, re-evaluate AI scores, and send
  accept / reject / interview emails.
- **Candidate side** — sign up, log in, browse published jobs, upload a resume
  (validated), review AI-extracted details, confirm, and submit an application.

The AI model is used in **two** places:
1. Generate suggested job requirements/qualifications from a title + description.
2. Parse a resume into structured data **and** score it against a job with feedback.

---

## 2. Roles & Permissions

| Role      | Capabilities |
|-----------|--------------|
| **Admin** (recruiter/employee) | Create & edit jobs, AI-generate requirements, publish/close jobs, view all applications, re-evaluate AI scores, accept/reject/shortlist/interview, send emails. |
| **Candidate** | Sign up / log in, browse public jobs, apply once per job, upload & confirm resume, see own applications. |

Permission is enforced on both the API (middleware) and the UI (route guards).

---

## 3. Architecture

Three-tier monorepo:

```
┌─────────────────────────────────────────────┐
│  Next.js Frontend (App Router, React, TS)  │
│  • Public site  • Dashboard  • Candidate   │
└───────────────────┬─────────────────────────┘
                    │  HTTP + JWT (fetch / axios)
┌───────────────────▼─────────────────────────┐
│  Node/Express REST API  (MERN backend)      │
│  • auth  • jobs  • applications  • resumes  │
└───┬──────────────┬──────────────┬───────────┘
    │              │              │
┌───▼────┐   ┌─────▼─────┐  ┌─────▼──────────┐
│ MongoDB │   │ OpenAI API│  │ SMTP (nodemailer)
│ (Mongoose)│ │ (GPT)     │  │ email services  │
└────────┘   └───────────┘  └────────────────┘
```

- **Next.js** serves the UI. Public pages can be server-rendered for SEO; the
  dashboard is a client-heavy area.
- **Express API** is the single source of truth for business logic, auth, AI calls,
  file handling, and emails.
- **MongoDB** persists all entities. Mongoose enforces schemas and the "apply once"
  uniqueness rule.
- **AI** calls go through the Express backend (never the browser) so API keys stay
  server-side.
- **SMTP** (Nodemailer) sends acceptance, rejection, and interview emails.

---

## 4. Data Model (MongoDB / Mongoose)

### 4.1 `User`
```
{ _id, name, email (unique), passwordHash, role: 'admin' | 'candidate', createdAt }
```

### 4.2 `Job`
```
{
  _id,
  title,
  description,
  requirements: [String],      // AI-generated, editable
  qualifications: [String],    // AI-generated, editable
  status: 'draft' | 'published' | 'closed',
  createdBy: ObjectId(User),
  createdAt, updatedAt
}
```

### 4.3 `Resume`
```
{
  _id,
  filename, originalName, mimeType ('application/pdf'),
  sizeBytes, pageCount,          // validated: PDF, ≤3 pages, ≤5 MB
  storagePath,                   // uploaded file location
  parsedData: {                  // AI-extracted, candidate-confirmed
    skills: [String],
    experience: [{ role, company, period, description }],
    education: [{ degree, institution, year }],
    contact: { name, email, phone, location }
  }
}
```

### 4.4 `Application`
```
{
  _id,
  candidate: ObjectId(User),
  job: ObjectId(Job),
  resume: ObjectId(Resume),
  parsedDataSnapshot: Object,    // copy of confirmed parsed resume
  aiScore: Number,               // 0–100
  aiFeedback: String,            // short explanation
  status: 'applied' | 'reviewing' | 'shortlisted' | 'interview' | 'accepted' | 'rejected',
  appliedAt
}
// Unique compound index: (candidate, job) → prevents duplicate applications
```

---

## 5. REST API Design

**Base URL:** `/api/v1` · Auth via `Authorization: Bearer <jwt>`

### 5.1 Auth
| Method | Path | Access | Description |
|--------|------|--------|-------------|
| POST | `/auth/register` | public | Create candidate account |
| POST | `/auth/login` | public | Login, return JWT + role |
| GET  | `/auth/me` | authed | Current user profile |

### 5.2 AI
| Method | Path | Access | Description |
|--------|------|--------|-------------|
| POST | `/ai/generate-requirements` | admin | Title+description → requirements/qualifications |
| POST | `/ai/parse-resume` | candidate/admin | PDF → structured resume data |
| POST | `/ai/score` | admin | Job + parsed resume → score + feedback |

### 5.3 Jobs
| Method | Path | Access | Description |
|--------|------|--------|-------------|
| GET  | `/jobs` | public | List published jobs |
| GET  | `/jobs/:id` | public | Job detail |
| GET  | `/admin/jobs` | admin | All jobs (any status) |
| POST | `/admin/jobs` | admin | Create job (draft) |
| PATCH| `/admin/jobs/:id` | admin | Edit requirements/qualifications |
| PATCH| `/admin/jobs/:id/publish` | admin | Publish job |
| PATCH| `/admin/jobs/:id/close` | admin | Close job |

### 5.4 Applications — Candidate
| Method | Path | Access | Description |
|--------|------|--------|-------------|
| POST | `/jobs/:id/applications` | candidate | Submit application (validate + parse + confirm) |
| GET  | `/my/applications` | candidate | Candidate's own applications |

### 5.5 Applications — Admin
| Method | Path | Access | Description |
|--------|------|--------|-------------|
| GET  | `/admin/jobs/:id/applications` | admin | Ranked list (by aiScore desc) |
| GET  | `/admin/applications/:id` | admin | Full app + resume + parsed details |
| POST | `/admin/applications/:id/re-evaluate` | admin | Re-run AI scoring |
| PATCH| `/admin/applications/:id/decision` | admin | accept / reject / shortlist / interview + send email |

---

## 6. AI Integration (OpenAI)

A dedicated service module (`services/aiService.js`) wraps all model calls. Each
function is deterministic in input/output shape so it's easy to test and swap.

| Function | Input | Output | Prompt strategy |
|----------|-------|--------|-----------------|
| `generateRequirements(title, description)` | job title + description | `{ requirements: [], qualifications: [] }` | Instruct model to derive bullet lists, return strict JSON. |
| `parseResume(pdfText)` | extracted PDF text | `{ skills, experience, education, contact }` | Extract structured JSON from resume text. |
| `scoreResume(job, parsedResume)` | job + parsed resume | `{ score: 0–100, feedback: string }` | Compare resume against requirements, return numeric score + short rationale. |

- Requests ask for **strict JSON** output to keep parsing reliable.
- Errors are caught and surfaced as clear messages ("AI service unavailable").
- Raw PDF text is extracted with `pdf-parse` before being sent to the model.

---

## 7. Resume Validation & Parsing

**Validation (before any processing):**
1. `mimeType` must be `application/pdf` → else reject with clear message.
2. File size must be `≤ 5 MB` → else reject.
3. Page count must be `≤ 3` pages (measured with `pdf-parse`) → else reject.

Uploads handled with **multer** (memory/disk storage). Any rule violation returns a
`400` with a specific error message the candidate sees immediately.

**AI extraction flow:** after validation passes, PDF text is extracted → sent to
`parseResume` → candidate is shown the extracted skills/experience/education/contact
→ they **confirm or correct** → the confirmed snapshot is stored with the application.

---

## 8. Email Notifications (SMTP)

Nodemailer sends branded emails from a template service (`services/emailService.js`).

| Event | Recipient | Template |
|-------|-----------|----------|
| Job accepted | candidate | acceptance email |
| Job rejected | candidate | rejection email |
| Interview invite | candidate | interview invitation (date/link) |

- SMTP host/port/user/pass come from environment variables.
- Sending failures are logged and surfaced (but never block the decision action).

---

## 9. Auth & Security

- Passwords hashed with **bcrypt**.
- **JWT** (signed, short expiry) for sessions; role embedded in the token.
- Middleware: `requireAuth` (any logged-in user) and `requireAdmin` (admin only).
- API keys for OpenAI live only in `.env` / server env.
- Route guards on the frontend redirect unauthenticated users to `/login`.
- File uploads validated by type/size/page-count; stored outside public web root.

---

## 10. Frontend Pages (Next.js App Router)

### Public (candidate-facing)
| Route | Purpose |
|-------|---------|
| `/` | Landing page |
| `/jobs` | Public job listing (all published jobs) |
| `/jobs/[id]` | Job detail page with Apply button |
| `/signup` | Candidate registration |
| `/login` | Candidate / admin login |
| `/apply/[jobId]` | Apply flow: upload → validate → AI parse → confirm → submit |
| `/my-applications` | Candidate's own applications & status |

### Dashboard (admin-only)
| Route | Purpose |
|-------|---------|
| `/dashboard` | Overview |
| `/dashboard/jobs` | All jobs |
| `/dashboard/jobs/new` | Create job + AI-assisted requirements |
| `/dashboard/jobs/[id]` | Edit, publish/close |
| `/dashboard/jobs/[id]/applications` | Ranked applicant list with scores + AI feedback |
| `/dashboard/applications/[id]` | Full review: resume + parsed details, re-evaluate, decision buttons |

**Apply flow state machine (frontend):**
`upload → validated → parsing → review(confirm/correct) → submitting → done`
Each step is a distinct UI state so errors (invalid file, AI failure) are clear.

---

## 11. Project Structure

```
ai-powered-recruiter-system/
├─ client/                    # Next.js app
│  ├─ app/
│  │  ├─ page.tsx             # landing
│  │  ├─ jobs/                # listing + detail
│  │  ├─ apply/[jobId]/       # apply flow
│  │  ├─ signup/ login/ my-applications/
│  │  └─ dashboard/           # admin area
│  ├─ components/
│  ├─ lib/                    # api client, auth context
│  └─ ...
├─ server/                    # Express API
│  ├─ src/
│  │  ├─ config/              # env, db connect
│  │  ├─ models/              # User, Job, Resume, Application
│  │  ├─ routes/              # auth, ai, jobs, applications
│  │  ├─ controllers/
│  │  ├─ middleware/          # auth, admin, upload, error
│  │  ├─ services/            # aiService, emailService, pdfService
│  │  └─ utils/
│  ├─ uploads/                # resume files (gitignored)
│  └─ server.js
├─ .env.example
├─ PLAN.md
└─ README.md
```

---

## 12. Implementation Milestones

### Phase 1 — Foundation (M0)
- [ ] Monorepo scaffold: `client` (Next.js + TS + Tailwind) and `server` (Express).
- [ ] MongoDB connection + Mongoose models (`User`, `Job`, `Resume`, `Application`).
- [ ] `.env.example` with all config keys; gitignore for uploads and env.

### Phase 2 — Auth & Roles (M1)
- [ ] Register / login endpoints (bcrypt + JWT).
- [ ] `requireAuth` / `requireAdmin` middleware.
- [ ] Frontend signup, login, and session/role context.

### Phase 3 — Jobs & AI Requirements (M2)
- [ ] Admin job create/edit + publish/close endpoints.
- [ ] `aiService.generateRequirements` + `/ai/generate-requirements`.
- [ ] Dashboard job creation with editable AI-suggested requirements.
- [ ] Public job listing + detail pages.

### Phase 4 — Apply Flow & Resume Handling (M3)
- [ ] Resume upload with multer + validation (PDF, ≤3 pages, ≤5 MB).
- [ ] PDF text extraction (`pdf-parse`).
- [ ] `aiService.parseResume` + `/ai/parse-resume`.
- [ ] Apply UI: upload → validate → review/confirm extracted details → submit.
- [ ] Unique `(candidate, job)` index blocks double applications.

### Phase 5 — Scoring & Ranking (M4)
- [ ] `aiService.scoreResume` + `/ai/score`.
- [ ] Auto-score on application submission.
- [ ] Dashboard ranked applicant list (score desc) + AI feedback.
- [ ] Full application review page (resume + parsed details).
- [ ] Re-evaluate endpoint + button.

### Phase 6 — Decisions & Emails (M5)
- [ ] Decision endpoint (accept/reject/shortlist/interview).
- [ ] `emailService` + SMTP config.
- [ ] Acceptance, rejection, and interview templates.
- [ ] Candidate "my applications" status view.

### Phase 7 — Polish & QA (M6)
- [ ] Form validation + friendly error states throughout.
- [ ] Loading states for AI calls (spinners, disabled buttons).
- [ ] Unit tests for validation rules, scoring prompts, email rendering.
- [ ] Seed script for a demo admin + sample jobs.
- [ ] README with setup + run instructions.

---

## 13. Environment Variables

```env
# Server
PORT=5000
MONGODB_URI=mongodb://localhost:27017/recruiter
JWT_SECRET=change_me
JWT_EXPIRES_IN=7d

# AI
OPENAI_API_KEY=sk-...

# SMTP
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=...
SMTP_PASS=...
EMAIL_FROM="Recruiter System <no-reply@example.com>"

# Uploads
UPLOAD_MAX_SIZE_MB=5
UPLOAD_MAX_PAGES=3

# Client
NEXT_PUBLIC_API_URL=http://localhost:5000/api/v1
```

---

## 14. Out of Scope (for now)
- Resume parsing to file-stored images / OCR — only text-based PDF parsing.
- Bulk email campaigns.
- Multi-company tenancy (single employer assumed).
- Third-party resume databases / job boards.

---

## 15. Acceptance Checklist (maps to proposal steps 1–18)
- [ ] Admin can log in, enter title+description, and get editable AI requirements.
- [ ] Admin can publish a job; it appears on the public job listing page.
- [ ] Candidate can sign up / log in and browse published jobs.
- [ ] Candidate can view a job detail page and apply.
- [ ] Resume upload enforces PDF / ≤3 pages / ≤5 MB with clear error messages.
- [ ] AI extracts skills, experience, education, and contact; candidate confirms/corrects.
- [ ] Application links candidate + resume + job; duplicates prevented.
- [ ] AI scores each resume vs. job and gives feedback.
- [ ] Applicants are ranked by score on the dashboard.
- [ ] Admin can re-evaluate a score.
- [ ] Admin can open any application to see the full resume and parsed details.
- [ ] Admin can accept / reject / shortlist / invite to interview.
- [ ] Acceptance, rejection, and interview emails are sent via SMTP.
