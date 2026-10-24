# FinAI Advisor — Trustworthiness & Verifiability Layer

## Top-Level Overview

Every AI message in the prototype currently presents financial claims as plain text
with no indication of confidence, source, or recourse if wrong.
This plan adds a full trust layer on top of the existing prototype:

1. **Confidence badges** — every AI claim is auto-scanned and tagged (✅ verified /
   🟡 uncertain / 🔴 unverifiable) with a visual indicator inline
2. **Source traceability** — each claim links to the authoritative source used to
   support it (e.g. Federal Reserve data, SEC filings, academic studies, Investopedia)
3. **Challenge mechanism** — users can flag any claim; the AI self-reviews and then
   provides external links for independent verification
4. **Audit trail** — every AI response, its confidence score, sources cited, and any
   user challenges are logged and viewable in a dedicated Trust Log screen
5. **Claim validation layer** — a second AI pass (or rule-based checker) scans the
   primary response for statistically verifiable facts before the message is shown

All of this is implemented directly in `finai-prototype.html` — no backend required
for the prototype; the trust engine runs client-side in JavaScript.

---

## Architecture

```
AI Message Output
       │
       ▼
┌─────────────────────────┐
│  Auto-Scan / Claim      │  ← regex + CLAIM_DB lookup
│  Classification Engine  │
└─────────────────────────┘
       │
   ┌───┴────────────────────────┐
   ▼                            ▼                          ▼
✅ verified               🟡 uncertain              🔴 unverifiable
(green badge)            (yellow badge)             (red badge)
   │                            │                          │
   └───────────────┬────────────┘──────────────────────────┘
                   ▼
         Rendered message with
         inline claim spans + trust summary line
                   │
         User can click "🔍 Fact-check"
                   │
                   ▼
         Challenge Panel (slide-in drawer)
         ├── AI self-review (800ms simulated)
         ├── confirmed / qualified / retracted
         └── "Verify yourself →" external source link
                   │
                   ▼
         Logged in AUDIT_LOG → Trust Log screen
```

---

## Sub-Tasks

---

### Sub-Task 1 — Claim Classification Data Model

**Status:** [ ] pending

**Intent:**
Define the data structures that represent a claim, its confidence tier, its source
references, and its challenge state. Everything else builds on this foundation.

**Expected Outcomes:**
- A `CLAIM_DB` object in JS maps claim fingerprints (key phrases) to:
  - `confidence`: `'verified' | 'uncertain' | 'unverifiable'`
  - `reason`: short explanation of why that confidence was assigned
  - `sources`: array of `{ label, url }` authoritative links
  - `category`: `'statistic' | 'general-principle' | 'prediction' | 'opinion'`
- A `CLAIM_RULES` array of regex patterns used to detect claimable phrases in
  any AI message string
- A `parseClaimsFromMessage(text)` function that returns an array of matched claims
  with their positions

**Todo List:**
1. Define `CLAIM_DB` with entries covering all AI responses currently in the
   prototype (chat replies + scenario coach text + quiz feedback)
2. Write regex patterns in `CLAIM_RULES` for: percentage figures, "historically",
   "research shows", named financial instruments, named institutions (Fed, SEC),
   superlatives ("always", "never", "best", "safest")
3. Implement `parseClaimsFromMessage(text)` — returns `[{ match, start, end, claim }]`
4. Add `confidence` colour mapping: verified=green, uncertain=yellow, unverifiable=red

**Relevant Context:**
- AI replies live in `AI_REPLIES[]` array and `SIM_SCENARIOS[].{buy,hold,sell}` strings
- Claims to pre-classify include: "S&P 500 ~10%/year", "rate hikes compress P/E multiples",
  "time in the market beats timing the market", "diversification reduces risk",
  "higher volatility = higher potential reward"
- Sources to use: Federal Reserve (federalreserve.gov), SEC (sec.gov),
  Investopedia, Vanguard research, NBER, CFA Institute

---

### Sub-Task 2 — Message Rendering with Inline Confidence Badges

**Status:** [ ] pending

**Intent:**
Replace plain-text AI message rendering with annotated HTML that wraps detected
claims in spans with colour-coded confidence badges the user can interact with.

**Expected Outcomes:**
- AI chat messages render with claim spans: flagged text is underlined with a
  coloured dot (🟢/🟡/🔴) and a superscript badge number linking to the source
- Hovering or tapping a badged claim shows a tooltip: confidence level + one-line
  reason + "See sources" link
- The overall message shows a trust summary line: e.g.
  "3 claims · 2 verified · 1 uncertain · 0 unverifiable"
- Scenario coach responses and quiz feedback are rendered the same way

**Todo List:**
1. Write `renderMessageWithClaims(text)` — wraps matched claim spans in
   `<span class="claim claim-{confidence}">` with a `<sup class="claim-badge">`
2. Add `.claim-verified`, `.claim-uncertain`, `.claim-unverifiable` CSS styles
   (underline colour, badge colour)
3. Add `.claim-tooltip` CSS — positioned tooltip card that appears on hover/tap
4. Write `buildTooltip(claim)` — generates tooltip HTML with reason + source links
5. Replace the `a.textContent = AI_REPLIES[...]` plain assignment with
   `a.innerHTML = renderMessageWithClaims(AI_REPLIES[...])`
6. Apply same renderer to simulator coach text when it appears
7. Add the per-message trust summary line beneath each AI message

**Relevant Context:**
- Chat messages are created as `<div class="msg ai">` elements in `sendChat()`
- Simulator coach text is set via `document.getElementById('sim-coach-text').textContent`
  — this must change to `.innerHTML`
- Tooltips must not overflow the chat panel; use `position:fixed` for the tooltip
  when inside the narrow chat bubble

---

### Sub-Task 3 — Challenge Button & AI Self-Review Flow

**Status:** [ ] pending

**Intent:**
Give users a visible, low-friction way to challenge any AI claim. The challenge
triggers a simulated AI self-review that re-examines the claim and either
confirms it with stronger evidence or retracts/qualifies it.

**Expected Outcomes:**
- Each AI message has a "🔍 Fact-check this" button beneath it
- Clicking any claim badge OR the fact-check button opens the Challenge Panel:
  a slide-in drawer listing all claims in the message with a "Challenge" button
  per claim
- Clicking "Challenge" on a claim triggers the AI self-review response (pre-written
  per claim in `CLAIM_DB`) + surfaces the source links for independent verification
- The challenged claim badge updates to show a "reviewed" state with the outcome
  (✅ confirmed / ⚠️ qualified / ❌ retracted)
- A "Verify yourself →" link opens the primary source URL in a new tab

**Todo List:**
1. Add "🔍 Fact-check" button rendered beneath every AI message
2. Build `<div id="challenge-panel">` — slide-in drawer (right side, 320px wide)
   with: message excerpt, list of claims, per-claim Challenge button
3. Write `SELF_REVIEW_DB` — maps claim fingerprints to:
   `{ outcome: 'confirmed|qualified|retracted', reviewText, primarySourceUrl }`
4. Wire "Challenge" button: show spinner 800ms (simulating AI re-evaluation),
   then render `reviewText` + source link, update badge state
5. Add CSS: `.claim-confirmed`, `.claim-qualified`, `.claim-retracted` post-review
   states; `.challenge-panel` slide-in animation
6. Ensure the panel is accessible from the floating AI bubble chat AND from the
   simulator coach area

**Relevant Context:**
- Self-review responses must be honest — if a claim is a general principle rather
  than a verified statistic, the review should qualify it ("This is a historical
  average, not a guarantee") rather than falsely confirm it
- "10% S&P annual return" → confirmed (cite Vanguard, Shiller data) with caveat
  about past performance
- "Time in market beats timing" → qualified (true on average, but depends on entry
  point and horizon)
- "Rate hikes compress P/E" → confirmed (standard DCF theory, cite CFA Institute)

---

### Sub-Task 4 — Audit Trail / Trust Log Screen

**Status:** [ ] pending

**Intent:**
Build a Trust Log screen where users can review every AI response ever shown,
its confidence breakdown, any challenges made, and the outcomes — creating a
full accountability record.

**Expected Outcomes:**
- New "🛡️ Trust Log" nav item added to the sidebar
- Each entry shows: timestamp, which screen/module the message came from,
  the message excerpt, claim count by confidence tier, challenge count, outcome
- Clicking an entry expands it to show the full annotated message and all
  source links
- Filter bar: "All | Challenged | Unverifiable | Verified"
- A summary banner at the top: "X messages · Y claims verified · Z challenged"

**Todo List:**
1. Add `AUDIT_LOG` array to state; each entry: `{ id, ts, module, text, claims[], challenges[] }`
2. Every call to `renderMessageWithClaims()` appends an entry to `AUDIT_LOG`
3. Add `screen-trustlog` HTML and nav item `data-screen="trustlog"` in sidebar
4. Build `renderTrustLog()` — generates the log entry list from `AUDIT_LOG`
5. Add expandable entry rows with full annotated message on click
6. Add filter bar that filters `AUDIT_LOG` by state
7. Add summary banner that recalculates on every render

**Relevant Context:**
- `SCREENS` array and `SCREEN_META` object must include `'trustlog'`
- Log is in-memory for the prototype; in the full app this persists to the DB
- The sidebar should show a count badge on Trust Log if any unverifiable claims
  exist (like the red badge on Alerts)

---

### Sub-Task 5 — Global Confidence Indicator & Risk Disclaimer

**Status:** [ ] pending

**Intent:**
Surface a persistent, app-level signal of AI trustworthiness — so users always
know the overall reliability of the guidance they're receiving, not just per-message.

**Expected Outcomes:**
- A "Trust Score" chip in the topbar (e.g. "AI Confidence: 87%") that updates
  as more messages are scanned and challenged
- Clicking the chip navigates to the Trust Log
- A dismissible risk disclaimer banner on first load:
  "AI responses are educational, not financial advice. Claims are auto-verified
   where possible — always check sources before investing."
- The banner can be re-shown from the Trust Log screen
- The Learn module term definitions show a "Source: Investopedia" attribution line

**Todo List:**
1. Add `.trust-chip` to topbar HTML and CSS — pill showing emoji + score
2. Write `recalcTrustScore()` — `(verifiedCount / totalClaimCount) * 100`, rounded
3. Call `recalcTrustScore()` after every message render and after every challenge
   resolution; update chip text and colour (green >80%, yellow 60–80%, red <60%)
4. Build dismissible disclaimer banner HTML + CSS (`#disclaimer-banner`) — shown
   on app entry, stored-dismissed in `sessionStorage`
5. Add "Source:" attribution line to every term definition in the Learn screen
   (Investopedia for all current terms)
6. Wire Trust Log link from chip click

**Relevant Context:**
- Trust chip sits in `.topbar-actions` alongside the alerts bell
- Score starts at 100% and degrades as unverifiable/challenged claims accumulate
- Disclaimer uses the standard SEC/FINRA educational-use language pattern

---

## Design Principles for This Layer

- **Additive, not disruptive** — all changes layer on top of existing UI; no
  screens are removed or restructured
- **Honest qualification > false confidence** — self-reviews must qualify uncertain
  claims; never auto-confirm a general principle as a verified statistic
- **User agency** — the user can always reach the primary source URL and verify
  independently; the app never asks them to just "trust us"
- **Proportional friction** — low-confidence claims are visible at a glance;
  deep investigation is available but not forced on every interaction
