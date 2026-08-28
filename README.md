# AI-powered Recruiter System

AI-powered recruitment screening platform built with **Next.js**, **React 19**, **Drizzle ORM**, and **LangChain**. It combines a recruiter dashboard, candidate management, application handling, an AI chat assistant with RAG over company policy, and automated interview invitation emails.

> This repository was imported as a set of organized commits into the `AI-powered-recruiter-system` repo.

## Features

- **Authentication & user accounts** — register / login / logout with JWT sessions and protected routes (client + server).
- **Recruiter dashboard** — analytics cards, job listings, candidate grids, job detail view, and settings.
- **Job management** — create and list jobs with a seed script for demo data.
- **Applications & candidates** — apply to a job, upload resumes, store files via EdgeStore, duplicate-application detection.
- **Candidate evaluation** — parse uploaded PDF resumes, evaluate candidates against the job description, and score them.
- **AI chat assistant** — agentic chatbot with RAG over company policy using LangChain + Groq, plus AI-generated job descriptions and interview invitations.
- **Email & scheduling** — send interview invitations and schedule meetings via Nodemailer + SMTP.

## Tech Stack

- **Framework:** Next.js 16, React 19
- **Database:** PostgreSQL (Neon) via Drizzle ORM
- **AI:** LangChain, @langchain/groq, EdgeStore, unpdf
- **Auth:** bcryptjs, jsonwebtoken
- **Email:** nodemailer
- **UI:** Tailwind CSS v4, lucide-react, framer-motion, react-dropzone, react-hook-form

## Getting Started

```bash
npm install
```

Create a `.env` file with the required environment variables (database URL, JWT secret, EdgeStore token, Groq API key, SMTP credentials).

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Database

```bash
npm run drizzle:generate
npm run drizzle:migrate
npm run seed:jobs
```

## License

Private / proprietary — all rights reserved.
