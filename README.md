Live URL:  https://aminbiography.github.io/CTI-Reporting-And-Decision-Support-APP/ 

---

## Description for users (security leaders, risk owners, program managers)

### What this page is

**Reporting and Decision-Support APP** is a browser-only toolkit for turning security data into **consistent decisions and executive-ready reporting**. All calculations run locally in your browser; you paste inputs and receive scored outputs, ranked lists, and copy/paste summaries.

### What it contains

The app has four tools (tabs), each designed for a specific governance workflow:

---

### 1) Risk Scoring Calculator

**Purpose:** Standardize how teams describe and compare risk using a transparent model.

**Model:**
**Risk score (0–100) = (Likelihood × Impact) scaled + context + modifiers**

* Base: Likelihood (1–5) × Impact (1–5), mapped to 0–100
* Context multiplier: **Asset criticality** and **Exposure** increase score
* Modifiers: **Exploit availability** and **Detectability**

**Inputs you provide:**

* Asset criticality (Low → Very high)
* Exposure (segmented → internet-facing)
* Likelihood (1–5) and Impact (1–5)
* Exploit availability (unknown → weaponized)
* Detectability (easy → hard)
* Optional notes (rationale)

**Outputs you get:**

* Risk score (0–100)
* Severity band (Low / Medium / High / Critical)
* Suggested SLA target (e.g., 7–14 days for High)
* A recommended action statement
* A copy/paste summary for tickets, risk register entries, or approvals

---

### 2) Vulnerability Prioritization

**Purpose:** Convert a mixed set of vulnerabilities/findings into a **ranked remediation plan**.

**How it works:**

* You paste a **CSV-like list** of findings (with or without a header).
* The tool combines:

  * **CVSS score** (if provided) and
  * **Context score** (asset criticality, exposure, exploit maturity, and age)
* You can tune weighting:

  * Weight: CVSS (default 0.55)
  * Weight: Context (default 0.45)
  * If the weights do not sum to 1, the tool normalizes them.

**Expected columns:**
`id,title,asset,exposure,cvss,exploit,age_days,owner`

**Outputs you get:**

* Ranked table: score, severity band, owner, and a “Plan” statement
* KPI counts (Critical / High / Backlog)
* Export to CSV for full ranked list (`prioritized_findings.csv`)

**Typical use cases:**

* Weekly patch and remediation meetings
* Creating sprint backlogs aligned to risk
* Aligning engineering owners and due dates

---

### 3) Security Maturity Tracker

**Purpose:** Track maturity across security domains and generate an improvement plan.

**Presets available:**

* **Basic (6 domains):** governance, identity, endpoint, logging, vulnerability management, incident response
* **NIST CSF-lite:** Identify, Protect, Detect, Respond, Recover

**What you do:**

* Rate each control on a scale:

  * 0–4 (None → Optimized) or 1–5 (Initial → Optimized)
* Set a **target maturity** (default 3)

**Outputs you get:**

* Overall maturity score (normalized to 0–4 for reporting)
* Number of domains below target
* “Top gap” domain
* Per-domain cards with a bar indicator and optional notes
* Copy/paste 90-day improvement plan
* Export posture snapshot as JSON (`maturity_posture.json`)

---

### 4) Executive Summary Generator

**Purpose:** Produce a concise, business-ready security update for leadership.

**Inputs you provide:**

* Reporting period (free text)
* Audience (Executive / IT leadership / Security leadership)
* KPIs (one per line)
* Top risks (one per line)
* Actions/decisions needed (one per line)
* Tone (neutral / urgent / optimistic)
* Length (short bullets or medium with a brief narrative)

**Outputs you get:**

* A formatted summary with:

  * Header (audience + reporting period)
  * Key metrics bullets
  * Top risks bullets
  * Actions/decisions bullets
  * Closing checkpoint line tailored to audience
* Copy-to-clipboard button (subject to browser permissions)

---

## Description for developers (implementation and extension)

### High-level architecture

* Single static HTML file with embedded CSS and JavaScript.
* Tabbed SPA-like interface driven by:

  * `setTab(active)` toggling `.hidden` panels and `aria-selected`.
* Shared utilities:

  * `escapeHtml()` for safe rendering
  * `badgeClass(sev)` mapping to UI classes (good/warn/bad)
  * `clamp()` for numeric safety
  * `nowStr()` timestamp display

There are no external dependencies; all processing is deterministic and local.

---

### Module 1: Risk Scoring

**Core functions**

* `calcRisk()` computes score and rationale
* `riskSeverity(score)` maps numeric score to severity bands:

  * ≥80 Critical, ≥60 High, ≥35 Medium, else Low
* `riskSla(sev)` maps severity to a suggested remediation SLA window
* `riskRecommendation(score, sev)` returns a standard decision guidance string
* `renderRisk(out)` updates KPIs, bar width, recommendation, and summary textarea

**Computation details**

* Base: `(L × I)/25 × 100`
* Context multiplier: `1 + (asset-1)*0.10 + (exposure-1)*0.12`
* Modifiers: `(exploit*5) + ((detect-1)*4)`
* Final score clamped to 0–100

**Extension points**

* Replace constants with configurable thresholds (org-specific risk appetite).
* Add additional modifiers (e.g., compensating controls, data sensitivity, threat intel confidence).
* Export a JSON record of the scoring event (useful for audit trails).

---

### Module 2: Vulnerability Prioritization

**Parsing**

* `parseCsvLike(raw)` supports optional header detection (`id` and `title`).
* `splitCsvLine(line)` implements minimal CSV quoting support.

**Normalization helpers**

* `normExposure()`: local/internal/internet → 1/2/3
* `normExploit()`: unknown/none/poc/weaponized/active → 0–3
* `assetCriticalityFromName(asset)`: heuristic mapping based on keywords
* `planForSeverity(sev)`: assigns a default remediation plan string

**Scoring**

* `scoreFinding(f, wCvss, wCtx)`:

  * CVSS scaled to 0–100
  * Context score from asset criticality, exposure, exploit maturity, and age
  * Weighted combination clamped to 0–100
* Ranked descending by score
* `exportCsv(rows, filename)` writes full results

**Extension points**

* Replace asset name heuristics with explicit asset criticality input (or lookup table).
* Add “business impact tags” (regulated, PII, revenue-critical) into context score.
* Incorporate EPSS (if you later choose to allow optional offline lookup input) without live calls.

---

### Module 3: Maturity Tracker

**Preset model**

* `maturityPresets` is a simple JSON structure of domains and controls.

**Dynamic UI generation**

* `buildMaturityInputs()` rebuilds controls based on preset and scale, generating select inputs and optional notes per control.

**Computation**

* `computeMaturity()`:

  * averages per domain
  * normalizes to 0–4 when scale is 1–5
  * computes overall, below-target count, and top gap
  * generates a 90-day plan with top 3 weakest domains

**Rendering**

* `renderMaturity(out)` produces per-domain items with progress bars and optional notes.

**Export**

* `m-export` exports full posture JSON via Blob.

**Extension points**

* Add a real “radar chart” visualization (Canvas/SVG) while keeping offline behavior.
* Persist posture in localStorage (optional) for continuity across sessions.
* Add control weighting and evidence links per control.

---

### Module 4: Executive Summary Generator

**Inputs → formatted text**

* `bulletize(raw)` normalizes multi-line inputs into bullet arrays.
* `generateExecutive()`:

  * applies audience-specific header and closing line
  * applies tone-specific narrative sentence (medium length mode)
  * caps bullets (short: up to 6 per section; medium: up to 10)
* Copy uses `navigator.clipboard.writeText` (permission-dependent).

**Extension points**

* Add output templates (weekly, monthly, quarterly).
* Add a “risk acceptance register excerpt” section.
* Provide export formats: Markdown, doc-friendly plain text, JSON payload for ticketing systems.

---

## Operational intent and boundaries

* This app supports **governance, prioritization, and reporting**. It is not a scanner, and it does not connect to any systems.
* The scoring models are intentionally transparent and simple; production use typically involves aligning thresholds, SLA bands, and maturity targets to your organization’s risk appetite.

---

