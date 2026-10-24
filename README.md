# CareerTwin — AI Resume & Interview Coach

> Don't just fix your resume. Build your **Career Digital Twin** — an AI profile that evolves with you and tells you exactly what to do next.

## Quick start

**Prerequisites:** Node.js 20.9 or newer (required by Next.js 16) and npm.

```bash
git clone https://github.com/vishwasrao-lgtm/GirlGeeks.git
cd GirlGeeks
npm install
cp .env.example .env.local   # optional, see below
npm run dev                  # http://localhost:3000
```

On Windows Command Prompt, use `copy .env.example .env.local` instead of `cp`. PowerShell accepts `cp`.

### API keys (optional)

Add `GEMINI_API_KEY` or `OPENAI_API_KEY` to `.env.local` to enable AI-written wording. Groq and other OpenAI-compatible providers also work; see `.env.example` for the variable names.

**No API key? Everything still works.** A deterministic rule engine powers every feature offline, and each result is tagged **AI-generated** or **Rule engine** so you can see which path ran. Click **"Try with demo profile"** on the home page to load a sample student.

### Production build (recommended for demos)

```bash
npm run build && npm start
```

This is faster and has no dev overlay.

### Browser support

Use **Chrome or Edge** for voice answers (Web Speech API). Webcam analysis runs fully on-device with MediaPipe (model files are bundled in `public/mediapipe`). Camera and mic access need `localhost` or HTTPS.

## Tech stack

| Layer | Choice |
|---|---|
| App | Next.js 16 (App Router, Route Handlers), React 19, TypeScript 5 |
| UI | Tailwind CSS 4, Lucide icons |
| Client state | Zustand, persisted in `localStorage` |
| Resume parsing | unpdf (PDF.js), on the Next.js server |
| Skill / role engine | TypeScript in `src/lib` — 87 skills, 10 role models, interview state machine |
| Face presence | MediaPipe Tasks Vision (`FaceLandmarker`), WASM + model served from `public/mediapipe`, runs in the browser |
| Speech | Web Speech API (transcript) and Web Audio (pace, pauses). Spoken questions use the browser’s speech synthesis |
| Optional language model | Google Gemini (`gemini-2.5-flash` by default) **or** any OpenAI-compatible API (OpenAI `gpt-4o-mini`, Groq, etc.) |
| External data | GitHub REST API for a public profile (not a model) |

No database. The twin, gap report, interview history, and roadmap live in the browser.

## What is offloaded to a cloud AI

Only wording goes to a cloud model, and only when `GEMINI_API_KEY` or `OPENAI_API_KEY` is set. The call is made from the Next.js server (`src/lib/llm.ts`). Gemini is used if its key is present; otherwise the OpenAI-compatible endpoint is used. The sidebar shows which one is active.

Scores, matches, and the decision to raise or lower interview difficulty are computed locally and are not asked of the model.

| Feature | Sent to the cloud model | Kept on our side |
|---|---|---|
| Digital Twin | Headline, summary, archetype, and the wording of strengths and weak areas | Skill detection, evidence scores, role-fit percentages, projects, experience, education |
| Ask your twin | The chat reply | The profile it is allowed to talk about |
| JD Gap Detector | The 30-day plan text and the one-line verdict | Match %, per-skill ✓ / ◌ / ✗, which skills are missing |
| Adaptive interview | The question text, and a second opinion on the answer | The 44-question bank, the next-difficulty decision, the fallback grade |
| Hire-me pitch | The spoken pitch | Which facts it may use (proved skills, projects, interview topics) |
| Truth Checker | The rewritten bullet | Which lines are hype, the credibility score, the evidence questions |

If the key is missing, the request errors, or the model returns bad JSON, that cell falls back to the local engine. The card is tagged **AI-generated** or **Rule engine**.

Two other network calls are not this language model:

- **GitHub.** Fetching a public profile (`/api/github`) hits `api.github.com`. That response is languages, repos, and stars. Skill names are extracted locally.
- **Speech-to-text.** Chrome’s Web Speech API transcribes the microphone in the browser’s own speech service (Google, in Chrome). Our server never receives the audio. Pause detection, filler counts, and the webcam all stay on the device. Speech synthesis for “read the question aloud” is local to the browser.

## Features

| # | Feature | What makes it different |
|---|---------|-------------------------|
| 1 | **Career Digital Twin** (`/twin`) | Resume PDF + GitHub API + LinkedIn + certs → profile tree (skills, soft skills, projects, experience, education, strengths, weak areas, target career). Every skill has an **evidence trail** (listed → project → work → GitHub → cert). "Ask your twin" chat. |
| 2 | **JD → Gap Detector** (`/gap`) | Paste a title or full JD → weighted match %, ✓ / ◌ / ✗ per requirement, understands "Python, Java **or** C++", company emphasis (Google → DSA + System Design). Generates a **30-day plan that reuses your existing projects**. |
| 3 | **Adaptive AI Interviewer** (`/interview`) | Scores **conceptual vs practical** knowledge separately. Easy → Medium → Hard → Scenario, with targeted follow-ups. Shows the interviewer's reasoning live ("Conceptual: Strong · Practical: Weak → asking a practical follow-up"). |
| 4 | **Speech & Presence Coach** | WPM, filler words, long pauses (WebAudio), hedging, clarity, delivery confidence. Webcam: face visible %, looking away %, movement, posture shifts. **Consent-gated, local-only, no emotion/personality inference.** |
| 5 | **"Why hire you?" Pitch** (`/pitch`) | Resume + JD + interview strengths → 30-second pitch. "Compare 4 companies" shows Google / Microsoft / Amazon / Startup variants side by side. Text-to-speech playback + practise-by-voice timer. |
| 6 | **Resume Truth Checker** (`/truth`) | Flags hype without evidence ("highly scalable", "robust", "significantly"), asks evidence questions, rewrites using **only facts you provide**. Per-bullet credibility score. |
| 7 | **ATS Simulator** (`/ats`) | Same resume scored against 10 job families + your JD. Splits keywords into **Proven / Listed-but-unproven / Missing — learn first, don't claim**. ATS parse-ability checklist. |
| 8 | **Career Roadmap** (`/roadmap`) | Role skill tree (Programming / ML / Systems / GenAI …) → Projects → Internship → Job, auto-marked ✓ Learned / 🟡 Learning / 🔴 Missing from evidence. Click to update. |

**The twin evolves.** Mock-interview results adjust skill strengths and role readiness, and every change is plotted on the dashboard timeline.

## Architecture

```
Browser (Next.js App Router, React 19, Tailwind 4)
 ├─ Zustand store (localStorage)  ← twin, gap report, interview history, roadmap
 ├─ Web Speech API + WebAudio     ← transcript, WPM, pauses, fillers
 └─ MediaPipe FaceLandmarker WASM ← presence signals, on-device only
        │ JSON
Next.js Route Handlers (/api/*)
 ├─ parse     PDF → text (unpdf)
 ├─ github    GitHub REST API → languages, repos, skills
 ├─ twin      rule engine builds twin → LLM enriches narrative
 ├─ ask / gap / interview / pitch / rewrite
 └─ lib/llm   Gemini | OpenAI-compatible (Groq) | none → automatic fallback
Core engine (src/lib, pure TypeScript, unit-testable)
 ├─ skills.ts     87 skills with 150+ aliases, implication graph (MySQL ⇒ SQL)
 ├─ roles.ts      10 role models (weighted skills, pillars), company flavours, learning guides
 ├─ engine.ts     parsing, evidence scoring, role fit, JD gap, 30-day plan, ATS, truth check, pitch
 └─ interview.ts  44-question bank × 4 levels, evaluator, adaptive state machine
```

**Hybrid design.** Scores are computed by the deterministic engine, so they're explainable and stable. The LLM adds language: narratives, plans, dynamic questions, rewrites and pitches. Prompts tell the LLM never to invent skills or numbers.


## Rubric mapping

- **Technical implementation (30):** real PDF parsing, live GitHub API, an LLM provider abstraction with graceful fallback, an adaptive state machine, WebAudio signal processing, on-device computer vision (MediaPipe WASM), typed end to end.
- **Impact & problem-solution fit (20):** tells students *what to do next* (plans, roadmaps, rewrites) instead of just giving a score.
- **Innovation (15):** an evolving digital twin, evidence-weighted skills, the truth checker, conceptual-vs-practical adaptive interviewing, the anti-keyword-stuffing ATS.
- **Scalability & feasibility (15):** stateless API routes (serverless-ready on Vercel), webcam/speech processing on the client (zero server cost), works without paid APIs, LLM-agnostic.
- **UX/UI (10):** a single coherent dark UI, explainable scores, source tags on every result.

## Responsible AI

- No emotion, honesty or personality inference from faces. Only observable signals, with explicit consent.
- Video never leaves the browser. Resumes stay in the user's own `localStorage` and are not stored on our server.
- Our server never receives microphone audio. Chrome’s speech-to-text uses the browser’s speech service; everything else about delivery (pace, pauses, fillers) is computed locally.
- Never recommends claiming skills the user lacks. Rewrites use only user-provided evidence.

## Testing

```bash
node scripts/smoke.mjs   # exercises every API route against a running server
```
