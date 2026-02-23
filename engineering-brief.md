# Engineering Brief — Dynamic BRD Generation

## Current State

The tool has a working 4-step form with validation, a results step (Step 5), and a PDF generation pipeline. However, all outputs are simulated — no real AI calls are made anywhere.

Two things use fake data and need to be made dynamic:

1. **Step 5 preview** — `generateBRD()` calls `buildSampleResponse()` after a hardcoded 2.2s delay. The response is constructed from the user's input using template strings — no AI involved. The same logic runs for every user.
2. **PDF download** — `submitEmail()` calls `downloadPDF()` which always downloads the static file `demo-brd.pdf`. The `generatePDF()` function exists and is wired to populate the `#pdf-template` HTML — but it is never called.

The infrastructure is further along than it looks:
- `renderResults()` already maps a JSON response to the Step 5 DOM
- `generatePDF()` already populates the `#pdf-template` and triggers `html2pdf.js`
- The PDF HTML template (`#pdf-template`) is already built with all 8 sections
- Loading state, error state, and retry logic are already in the HTML

What's missing is: replacing `buildSampleResponse()` with a real API call, and calling `generatePDF()` instead of `downloadPDF()` after email submit.

---

## What Needs to Be Built

### 1. Replace `buildSampleResponse()` with a real API call

When the user reaches Step 5 (`goToStep(5)` → `generateBRD()`), instead of calling `buildSampleResponse()`, make a POST request to your backend with all form data.

**Inputs to send** (already collected in `state.formData`):

| Variable | Key in state.formData | Source |
|---|---|---|
| `{{problem_statement}}` | `problemStatement` | Step 1, textarea #1 |
| `{{problem_scenario}}` | `problemScenario` | Step 1, textarea #2 |
| `{{previous_attempts}}` | `previousAttempts` | Step 1, textarea #3 |
| `{{current_tools}}` | `currentTools` | Step 1, textarea #4 (optional) |
| `{{solution_vision}}` | `solutionVision` | Step 2, textarea #1 |
| `{{user_story}}` | `userStory` | Step 2, textarea #2 |
| `{{success_indicators}}` | `successIndicators` | Step 2, textarea #3 |
| `{{stakeholders}}` | `stakeholders` | Step 3, textarea |
| `{{stakeholders_sponsor}}` | `stakeholdersSponsor` | Step 3, text input (optional) |
| `{{constraint_timeline}}` | `constraintTimeline` | Step 3, radio: `kurang-1-bulan` / `1-3-bulan` / `3-6-bulan` / `fleksibel` |
| `{{constraint_budget}}` | `constraintBudget` | Step 3, radio: `under-50jt` / `50-200jt` / `200-500jt` / `above-500jt` / `belum-ditentukan` |
| `{{constraint_systems}}` | `constraintSystems` | Step 3, textarea (optional) |
| `{{urgency_level}}` | `urgencyLevel` | Step 4, radio: `sangat-mendesak` / `penting-tidak-urgent` / `baru-eksplorasi` |
| `{{urgency_reason}}` | `urgencyReason` | Step 4, textarea |

**Expected API response** — must match the shape `renderResults()` already expects:
```json
{
  "preview": {
    "reframedProblem": "string",
    "completenessGaps": [
      { "gap": "string", "question": "string" }
    ],
    "maturity": {
      "level": "Incomplete | Developing | Ready",
      "reasoning": "string"
    },
    "tradeoffs": {
      "sendNow": ["string", "string", "string"],
      "addressFirst": ["string", "string", "string"]
    }
  }
}
```

Note: `renderResults()` reads from `response.preview.*` — keep this wrapper or update the render function accordingly.

**Loading / error handling:** already built — just make sure `generateBRD()` shows `#loading-state` while waiting and calls `renderResults()` on success, or shows `#error-state` on failure.

---

### 2. Store the assessment result in memory

After `renderResults()` runs, store the full response in `state.aiResponse` (already done in the current code). You'll need the preview data again when generating the full BRD PDF.

---

### 3. Replace `downloadPDF()` with `generatePDF()`

When the user submits name + email in the modal, instead of downloading `demo-brd.pdf`, call the backend again with the full brief + short assessment result to get the BRD JSON, then call `generatePDF()`.

**Step A — Call the backend** with `prompt-full-report.md`. Pass:
- All original form inputs (same as above)
- Short assessment preview from `state.aiResponse.preview`
- User identity from the modal (nama, email, perusahaan, role)

**Expected API response** — must match the shape `generatePDF()` already expects inside `state.aiResponse.fullBRD`:
```json
{
  "fullBRD": {
    "executiveSummary": "string",
    "problemStatement": "string",
    "businessObjectives": ["string"],
    "scope": {
      "inScope": ["string"],
      "outOfScope": ["string"]
    },
    "stakeholderMap": [
      { "role": "string", "involvement": "string", "impact": "string" }
    ],
    "functionalRequirements": [
      { "id": "string", "description": "string", "priority": "High | Medium | Low" }
    ],
    "constraints": ["string"],
    "successMetrics": [
      { "metric": "string", "target": "string", "measurement": "string" }
    ],
    "openItems": ["string"]
  }
}
```

Note: `generatePDF()` also reads `state.aiResponse.preview.maturity.level` for the PDF meta table — make sure this is still set from Step A.

**Step B — Call `generatePDF()`** — it's already wired to populate `#pdf-template` and trigger `html2pdf.js`. No changes needed to the function itself once the data is in `state.aiResponse`.

---

## API Config

Use the Claude API (Anthropic). Both prompts use:

| Setting | Short Assessment | Full BRD |
|---|---|---|
| Model | `claude-sonnet-4-6` | `claude-sonnet-4-6` |
| Max tokens | `1500` | `4000` |
| Temperature | `0.3` | `0.3` |

The API key must live server-side — never expose it in client-side JS.

---

## Recommended Architecture

```
Browser (index.html)
  │
  ├── Step 4 → Step 5: POST /api/brd-preview
  │     body: { ...state.formData }
  │     returns: { preview: { reframedProblem, completenessGaps, maturity, tradeoffs } }
  │     → renderResults(response)
  │
  └── Email submit: POST /api/brd-full
        body: { ...state.formData, preview: state.aiResponse.preview, nama, email, perusahaan, role }
        returns: { fullBRD: { executiveSummary, problemStatement, ... } }
        → state.aiResponse.fullBRD = response.fullBRD
        → generatePDF()

Backend (Node / Python / whatever)
  ├── /api/brd-preview  → calls Claude API with prompt-short-assessment
  │                        → returns preview JSON
  ├── /api/brd-full     → calls Claude API with prompt-full-report
  │                        → returns fullBRD JSON
  │                        → optionally sends PDF via email (Resend / Postmark)
  └── never exposes API key to the browser
```

---

## Files Reference

| File | Purpose |
|---|---|
| `index.html` | Frontend — full form, validation, Step 5 render, PDF template, email modal |
| `prompt-short-assessment.md` | System + user prompt for Step 5 preview (replaces `buildSampleResponse`) |
| `prompt-full-report.md` | System + user prompt for full BRD PDF (replaces `downloadPDF`) |
| `demo-brd.pdf` | Static placeholder — will be replaced by dynamic `generatePDF()` |
