# FinAI Advisor — Full-Stack Investment Guide App

## Top-Level Overview

A full-stack web application that acts as a personal financial AI assistant.  
The app walks users through four progressive stages:

1. **Learn** — Adaptive finance glossary with quiz-based mastery tracking
2. **Simulate** — AI-guided paper trading where the assistant reacts to market events
3. **Select** — Guided investment selection with AI-generated pros/cons per option
4. **Invest** — Real portfolio management (paper trading → live brokerage toggle)

Long-term goal: continuous portfolio monitoring with AI-triggered alerts and adaptive
learning that surfaces new terms contextually as the user's activity demands them.

**Stack:**
- Frontend: React (Vite), TypeScript, Tailwind CSS, shadcn/ui components
- Backend: FastAPI (Python), SQLAlchemy ORM, Alembic migrations
- Database: SQLite (dev) → PostgreSQL (prod), switchable via env
- AI: Provider-agnostic LLM client driven by `LLM_PROVIDER` + `LLM_API_KEY` env vars
- Market Data: Yahoo Finance (yfinance) for paper mode; Alpaca API for live brokerage
- Auth: JWT (access + refresh tokens), bcrypt password hashing

---

## Sub-Tasks

---

### Sub-Task 1 — Project Scaffolding & Auth Foundation

**Status:** [ ] pending

**Intent:**  
Establish the monorepo layout, dev tooling, environment config, and working
authentication so every subsequent sub-task can build on a running, logged-in app.

**Expected Outcomes:**
- `frontend/` Vite+React+TypeScript app runs on `localhost:5173`
- `backend/` FastAPI app runs on `localhost:8000`
- `POST /auth/register` and `POST /auth/login` return JWT tokens
- `GET /auth/me` returns the logged-in user (protected route)
- SQLite DB created via Alembic migration with `users` table
- `.env.example` documents all required variables

**Todo List:**
1. Create monorepo root with `frontend/` and `backend/` folders and a root `README.md`
2. Scaffold `backend/`: FastAPI app, `pyproject.toml` (or `requirements.txt`), folder structure (`routers/`, `models/`, `schemas/`, `services/`, `db/`)
3. Add SQLAlchemy `User` model (id, email, hashed_password, created_at, learning_stage)
4. Add Alembic and generate initial migration
5. Implement `/auth/register`, `/auth/login` (returns JWT), `/auth/me` router
6. Add JWT middleware (access token validation on protected routes)
7. Scaffold `frontend/`: Vite + React + TypeScript + Tailwind + shadcn/ui
8. Build Login / Register pages with form validation
9. Implement auth context (store JWT in memory + refresh token in httpOnly cookie)
10. Add protected route wrapper — redirect to login if unauthenticated
11. Create `.env.example` with all required keys

**Relevant Context:**
- `LLM_PROVIDER`, `LLM_API_KEY`, `DATABASE_URL`, `JWT_SECRET`, `ALPACA_API_KEY` are the core env vars
- Use `python-jose` for JWT, `passlib[bcrypt]` for hashing
- shadcn/ui provides accessible, composable components out of the box

---

### Sub-Task 2 — LLM Service Layer (Provider-Agnostic)

**Status:** [ ] pending

**Intent:**  
Build a single backend service that routes all AI calls through one interface,
so swapping OpenAI ↔ Anthropic ↔ Ollama requires only an env change.

**Expected Outcomes:**
- `backend/services/llm_service.py` exposes a single async `chat(messages, system_prompt)` function
- Supports `openai`, `anthropic`, `ollama` as `LLM_PROVIDER` values
- Backend `POST /chat` endpoint streams token-by-token responses to the frontend
- Frontend `<ChatWindow>` component renders streamed markdown responses

**Todo List:**
1. Create `llm_service.py` with a provider-dispatch pattern (match on `LLM_PROVIDER`)
2. Implement OpenAI adapter (openai SDK, `gpt-4o` default model, model overridable via `LLM_MODEL` env)
3. Implement Anthropic adapter (anthropic SDK, `claude-3-5-sonnet` default)
4. Implement Ollama adapter (httpx streaming against `localhost:11434`)
5. Add `POST /chat` FastAPI route using `StreamingResponse`
6. Build `<ChatWindow>` React component: message history, streamed typing animation, markdown rendering (`react-markdown`)
7. Build `<ChatInput>` with send-on-Enter, loading state, disable-during-stream

**Relevant Context:**
- System prompts per app section (Learn / Simulate / Select / Invest) will be injected by each feature module, not hardcoded here
- Streaming prevents UI blocking on long AI responses (critical for investment explanations)

---

### Sub-Task 3 — Finance Glossary & Adaptive Learning Module

**Status:** [ ] pending

**Intent:**  
Implement the adaptive learning layer: a curated term bank, per-user mastery tracking,
AI-explained definitions, and quiz-based reinforcement that unlocks new terms.

**Expected Outcomes:**
- DB tables: `terms` (id, slug, title, difficulty 1–5, prerequisites[], category), `user_term_progress` (user_id, term_id, mastery_score 0–100, last_seen, times_reviewed)
- `GET /learn/next-term` returns the most relevant unmastered term for the user
- `POST /learn/quiz-answer` scores the answer and updates mastery
- AI explains any term in context of what the user is currently doing in the app
- Frontend `/learn` page: term card, AI explanation panel, quiz question, progress bar

**Todo List:**
1. Create `terms` and `user_term_progress` DB models + Alembic migration
2. Seed initial term bank (20–30 foundational terms: asset, equity, bond, ETF, diversification, market cap, P/E ratio, dividend, bull/bear market, index fund, volatility, liquidity, portfolio, compound interest, inflation, yield, short selling, margin, order types, risk tolerance)
3. Implement mastery scoring logic: correct quiz answer +20, wrong −10, decay over time if not reviewed
4. Build `GET /learn/next-term` — scores terms by: low mastery + prerequisite terms mastered + relevant to user's current activity
5. Build `POST /learn/quiz` — multiple choice question generated by LLM for that term
6. Build `POST /learn/quiz-answer` — evaluate answer, update mastery, return AI feedback
7. Build `GET /learn/glossary` — all terms with user's mastery percentage
8. Frontend `/learn` route: `<TermCard>`, `<QuizPanel>`, `<GlossaryDrawer>`, `<MasteryProgressBar>`
9. Hook AI context: when user is in Simulate/Select/Invest and a relevant term appears in chat, surface a "Learn this term?" tooltip that opens the Learn panel

**Relevant Context:**
- Prerequisites are stored as a JSON array of term slugs on the `terms` table
- Mastery score of 80+ means "mastered"; terms below 50 are surfaced as review candidates
- The AI system prompt for this section: "You are a patient finance tutor. Explain terms simply, use analogies, avoid jargon in explanations."

---

### Sub-Task 4 — Investment Simulation Module

**Status:** [ ] pending

**Intent:**  
Give users a zero-risk sandbox where the AI narrates market scenarios and guides
them through decisions (buy / hold / sell), building intuition before real money.

**Expected Outcomes:**
- DB tables: `simulations` (id, user_id, scenario_name, started_at, ended_at), `simulation_positions` (sim_id, symbol, qty, avg_cost, current_price)
- Real market price data fetched via `yfinance` for realism
- AI generates scenario events ("Tech stocks drop 8% — what's your move?") and reacts to user decisions with coaching
- Frontend `/simulate` page: portfolio view, AI chat panel, action buttons (Buy / Sell / Hold)

**Todo List:**
1. Create `simulations` and `simulation_positions` DB models + Alembic migration
2. Add `yfinance` wrapper service: fetch current price, OHLCV history, basic stats
3. Implement `POST /simulate/start` — create simulation with $10k virtual cash
4. Implement `GET /simulate/{id}/state` — current holdings, cash, P&L
5. Implement `POST /simulate/{id}/trade` — buy/sell with virtual cash
6. Build AI scenario engine: LLM generates a market event, evaluates user's trade decision, provides coaching message ("Good — you bought the dip. Here's why that worked…")
7. Implement `POST /simulate/{id}/next-event` — generates next market scenario from LLM
8. Frontend `/simulate`: `<SimPortfolio>` table, `<ScenarioCard>` showing AI event, `<TradePanel>`, chat coach panel
9. Track simulation history so users can review past decisions

**Relevant Context:**
- LLM system prompt: "You are a simulation coach. Generate realistic market scenarios, evaluate user trades pedagogically, never just tell them the answer — ask Socratic questions first."
- yfinance is free and requires no API key; use it for all data in simulation mode
- Positions use last fetched price for P&L, refreshed on each `GET /simulate/{id}/state` call

---

### Sub-Task 5 — Investment Selection Guide Module

**Status:** [ ] pending

**Intent:**  
Help users choose where to invest by presenting asset categories with AI-generated
pros/cons tailored to their profile and risk tolerance.

**Expected Outcomes:**
- `GET /select/options` returns curated investment categories (Stocks, ETFs, Bonds, REITs, Crypto, Index Funds, Mutual Funds)
- AI generates a personalized pros/cons analysis for each option based on user's profile (risk tolerance, learning stage, time horizon)
- User can compare up to 3 options side-by-side
- Frontend `/select` page: option cards, AI comparison panel, "Choose This" button that carries selection into the Invest module

**Todo List:**
1. Add `user_profile` DB model fields: risk_tolerance (1–5), investment_horizon (short/medium/long), initial_capital_range
2. Build `POST /profile/setup` onboarding flow (3-question wizard run once after registration)
3. Build `GET /select/options` — static category list + dynamic AI analysis injected per category
4. Build `POST /select/compare` — takes up to 3 category slugs, returns AI side-by-side comparison
5. Build `POST /select/explain-risk` — AI explains a specific risk factor in the context of the user's profile
6. Frontend `/select`: `<InvestmentOptionCard>` (icon, name, summary, risk badge), `<CompareDrawer>`, AI chat panel, "Start Investing →" CTA
7. Carry selected asset class + risk profile into `/invest` initial state

**Relevant Context:**
- AI system prompt: "You are an objective financial advisor. Present balanced pros and cons. Always mention risks prominently. Never recommend a specific stock — only asset categories."
- Risk tolerance 1–2 = conservative, 3 = moderate, 4–5 = aggressive (drives AI tone and emphasis)

---

### Sub-Task 6 — Real Investment Module (Paper → Live Toggle)

**Status:** [ ] pending

**Intent:**  
The core investment interface: user makes real decisions (what, how much), AI assists
but never auto-executes. Paper trading (Alpaca paper) by default; one toggle flips to live.

**Expected Outcomes:**
- DB tables: `portfolios` (user_id, mode paper/live, broker_account_id), `portfolio_positions` (portfolio_id, symbol, qty, avg_cost, current_value)
- Alpaca paper trading API fully wired: search ticker, get quote, place market/limit order
- Toggle between paper and live mode requires explicit user confirmation modal
- AI suggests actions but user must confirm every trade
- Frontend `/invest` page: portfolio dashboard, ticker search, order panel, AI assistant panel

**Todo List:**
1. Create `portfolios` and `portfolio_positions` DB models + Alembic migration
2. Build Alpaca service wrapper (`alpaca-trade-api`): paper base URL vs live base URL controlled by portfolio mode
3. Implement `GET /invest/portfolio` — sync positions from Alpaca + store in local DB
4. Implement `POST /invest/search` — ticker search via Alpaca asset list + yfinance fundamentals
5. Implement `POST /invest/order` — place order via Alpaca (paper or live), confirm with user before execution
6. Implement `POST /invest/toggle-mode` — switch paper/live, require re-confirmation, log mode change
7. Build AI order assistant: user says "I want to invest $500 in tech" → AI suggests 2–3 tickers with rationale, user picks one, order panel pre-fills
8. Frontend `/invest`: `<PortfolioDashboard>` (total value, gain/loss, allocation pie), `<TickerSearch>`, `<OrderPanel>` with confirmation step, `<AIAssistant>` chat
9. Add paper/live mode badge and confirmation modal for toggle

**Relevant Context:**
- Alpaca paper endpoint: `https://paper-api.alpaca.markets`
- Alpaca live endpoint: `https://api.alpaca.markets`
- ALPACA_API_KEY and ALPACA_SECRET_KEY in env
- AI must never call the order endpoint directly — it only pre-fills the order panel

---

### Sub-Task 7 — Portfolio Monitoring & AI Alerts

**Status:** [ ] pending

**Intent:**  
Long-term value feature: continuous market monitoring that triggers AI-generated
guidance when significant portfolio events occur.

**Expected Outcomes:**
- Background scheduler checks portfolio positions every 15 minutes during market hours
- Alert conditions: position down >5%, position up >15%, earnings report due, macro news spike
- AI generates a contextual action recommendation per alert ("AAPL is down 7% — here's what to consider")
- Alerts stored in DB and shown as notification badges in the UI
- New relevant finance terms are automatically queued for the user based on alert context

**Todo List:**
1. Add `alerts` DB model (user_id, portfolio_id, symbol, alert_type, message, ai_recommendation, seen, created_at)
2. Add APScheduler job in FastAPI lifespan: runs `check_portfolio_alerts()` every 15 min on weekdays 9:30–16:00 ET
3. Build alert detection logic: compare current price vs avg cost and yesterday close; flag earnings dates from yfinance calendar
4. Build `POST /alerts/generate-recommendation` — LLM generates recommendation for a given alert
5. Implement `GET /alerts/unseen` and `POST /alerts/{id}/mark-seen`
6. Add alert-driven term surfacing: map alert types to relevant term slugs, queue them in `user_term_progress` as "needs review"
7. Frontend: `<NotificationBell>` with badge count, `<AlertsDrawer>` with AI recommendation per alert, "Learn More" link opens term in Learn module

**Relevant Context:**
- APScheduler's `AsyncIOScheduler` integrates cleanly with FastAPI's async event loop
- Market hours check should use `pytz` to handle ET timezone correctly
- AI system prompt for alerts: "You are a portfolio monitor. Be concise. State the event, the risk, and 1–2 specific actions the user could consider. Always end with a relevant learning suggestion."

---

### Sub-Task 8 — Navigation Shell & UX Polish

**Status:** [ ] pending

**Intent:**  
Wire all modules together into a cohesive single-page app with clear progression,
persistent layout, and a smooth user experience.

**Expected Outcomes:**
- Persistent left sidebar with: Learn, Simulate, Select, Invest, Alerts, Profile
- Progress indicator showing user's stage across the four main phases
- Dark/light mode toggle
- Mobile-responsive layout
- Global AI chat bubble accessible from any page

**Todo List:**
1. Build `<AppShell>` layout component: sidebar, top bar with user avatar + alerts bell, main content area
2. Implement React Router v6 routes: `/learn`, `/simulate`, `/select`, `/invest`, `/alerts`, `/profile`
3. Add stage progress bar in sidebar (Learn → Simulate → Select → Invest) with completion percentages
4. Add dark/light mode (Tailwind `dark:` classes + `localStorage` preference)
5. Build `<GlobalAIChat>` floating button → slide-up panel, context-aware (knows which page user is on)
6. Make layout responsive (sidebar collapses to bottom nav on mobile)
7. Add loading skeletons for all data-fetching pages
8. Write root `README.md` with setup instructions (clone → env setup → `docker-compose up`)

**Relevant Context:**
- shadcn/ui `Sheet` component for sidebar on mobile, `Sidebar` layout for desktop
- The global AI chat should send the current route as context to the backend so the correct system prompt is chosen
- Docker Compose should wire frontend (nginx) + backend + postgres for production-like local dev

---

## Architecture Diagram (reference only — do not place in plan)

See chat for Mermaid diagram.
