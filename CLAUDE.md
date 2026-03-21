# CLAUDE.md — MCI DCE Repository Guide

## Project Overview

**MCI DCE** is a web-based research survey tool for the [Male Contraceptive Initiative](https://malecontraceptive.org/). It uses a **Discrete Choice Experiment (DCE)** methodology to capture participant preferences across male contraceptive pipeline products and generate a personalized product recommendation.

The entire application is a **single self-contained HTML file** (`mci-dce.html`) with no build step, no external dependencies, and no backend server. It runs directly in any modern browser.

---

## Repository Structure

```
mci-dce/
├── mci-dce.html   # Entire application (HTML + CSS + JavaScript, ~1 400 lines)
└── CLAUDE.md      # This file
```

There are no subdirectories, no package manager manifests, no build configs, and no test files.

---

## Technology Stack

| Layer | Choice |
|---|---|
| Language | HTML5, CSS3, JavaScript (ES6+) |
| Framework | Vanilla JS (zero external libraries) |
| Styling | Inline `<style>` block with CSS custom properties |
| State | Plain global object (`S`) |
| Persistence | `localStorage` (key: `mci_dce`) + Google Apps Script (POST) |
| Remote data | Google Sheets via a deployed Apps Script web app |

---

## Running the Application

Open `mci-dce.html` directly in a modern browser — no server or build required:

```bash
open mci-dce.html          # macOS
xdg-open mci-dce.html      # Linux
start mci-dce.html         # Windows
```

Or serve it over HTTP if CORS is needed for the Apps Script endpoint:

```bash
python3 -m http.server 8080
# then visit http://localhost:8080/mci-dce.html
```

---

## Application Architecture

### Screen Flow

```
Welcome → DCE (5 rounds) → Transition → Builder → Summary
```

All screens are rendered in the same DOM; `showScreen(id)` hides/shows the relevant `.screen` div.

### Global State Object (`S`)

```js
S = {
  sessionId,    // UUID generated at load
  startTime,    // ISO timestamp
  pairs,        // Randomised DCE product pairs (5 rounds)
  dceRounds,    // Array of {chosen, rejected} per round
  idealProfile, // { efficacy, administration, onset, fertility,
                //   hormonal, provider, sexDrive, testes,
                //   ejaculation, energyMood }
}
```

### Key Functions

| Function | Purpose |
|---|---|
| `showScreen(id)` | Navigate between screens |
| `startDCE()` | Generate randomised product pairs |
| `renderRound()` | Display a DCE comparison card |
| `recordChoice(chosen, rejected)` | Persist a single DCE response |
| `initBuilder()` | Build the drag-and-drop attribute UI |
| `setIdeal(attrId, val)` | Record an attribute preference |
| `scoreProduct(product)` | Score a product against `S.idealProfile` |
| `getTopMatches()` | Return top-3 scored products |
| `buildSummary()` | Render the results/summary screen |
| `sendData()` | POST session data to Apps Script endpoint |

### Configuration Constants

Defined near the top of the `<script>` block:

```js
const CONFIG = {
  ENDPOINT_URL: 'https://script.google.com/macros/s/.../exec',
  DCE_ROUNDS: 5,
};
```

Change `ENDPOINT_URL` when redeploying the Apps Script, and `DCE_ROUNDS` to alter experiment length.

### Product Definitions (`PRODUCTS` array)

Each product has:

```js
{
  id,           // kebab-case string
  name,         // Short display name
  realName,     // Full clinical/brand name
  developer,    // Organisation name
  stage,        // Development stage string
  icon,         // Emoji icon
  tagline,      // One-line description
  attrs,        // { efficacy, administration, onset, fertility,
                //   hormonal, provider, sexDrive, testes,
                //   ejaculation, energyMood }
  showInResults // false = DCE pair-generation only, hidden from recommendations
}
```

Currently **10 products** are defined; 7 have `showInResults: true`.

### Attribute System (`ATTR_META`)

Eight core attributes, each with ordered levels:

1. **efficacy** — 86–90 % / 95 % / 99 % / 99.9 %
2. **administration** — on-demand pill / daily pill / implant / injection / topical gel
3. **onset** — same day / within a week / within a month / several months
4. **fertility** — immediate / weeks / months / years
5. **hormonal** — hormonal / non-hormonal / no preference
6. **provider** — fully self-administered / provider for placement / provider for placement & removal
7. **sexDrive, testes, ejaculation, energyMood** — individual side-effect sliders

### Scoring Logic

`scoreProduct(product)` compares each attribute in `S.idealProfile` against `SCORING_MAP[attrId][idealVal]` to get a compatibility set, then checks if the product's value is in that set. Maximum score = 8.

---

## Data Collection

### Google Apps Script Setup

The setup instructions (with sample Apps Script code) are embedded as an HTML comment in `mci-dce.html` at lines 8–60. Key points:

1. Create a Google Sheet.
2. Paste the provided Apps Script.
3. Deploy as a web app (access: **Anyone**).
4. Copy the deployment URL into `CONFIG.ENDPOINT_URL`.

### Submitted Payload

```json
{
  "sessionId": "<uuid>",
  "startTime": "<ISO 8601>",
  "completedAt": "<ISO 8601>",
  "userAgent": "<browser UA>",
  "dceRounds": [{ "chosen": "<id>", "rejected": "<id>" }, ...],
  "idealProfile": { "efficacy": "...", ... }
}
```

Data is also saved to `localStorage` as a fallback.

---

## Code Conventions

### Naming

| Category | Convention | Example |
|---|---|---|
| Constants | UPPER_CASE | `CONFIG`, `PRODUCTS`, `ATTR_META` |
| Functions / variables | camelCase | `showScreen`, `sessionId` |
| Product IDs | kebab-case | `'hormonal-implant'` |
| Attribute keys | camelCase | `sexDrive`, `returnToFertility` |
| DOM IDs | `screen-{name}`, `card-{side}` | `screen-dce`, `card-left` |
| CSS classes | BEM-like | `match-card`, `builder-wrap` |

### Formatting

- **Indentation:** 2 spaces (JS and HTML)
- **Strings:** Single quotes in JS
- **Section separators:** `// ══════════════════` visual headers in JS

### CSS

- All styles are in a single `<style>` block in `<head>`.
- Design tokens live at the top as CSS custom properties (`--color-*`, `--space-*`).
- MCI brand colours: primary `#0077B6`, accent `#00B4D8`, dark `#03045E`.
- Layout uses flexbox and CSS Grid; the design is mobile-responsive.

---

## Editing Guidelines for AI Assistants

1. **Do not introduce a build step or external dependencies.** The application must remain a single deployable HTML file.
2. **Preserve the global `S` state object.** All screens and functions read from it; do not refactor it away without updating every reference.
3. **When adding a new product**, add it to `PRODUCTS` with all required keys and set `showInResults` appropriately. Also verify `SCORING_MAP` covers its attribute values.
4. **When adding a new attribute**, update `ATTR_META`, every product's `attrs` object, `SCORING_MAP`, `initBuilder()`, and the summary rendering in `buildSummary()`.
5. **Test manually in a browser** after every change — there is no automated test suite.
6. **Keep inline documentation.** The setup instructions in the HTML comment block (lines 8–60) are the primary onboarding docs; keep them accurate.
7. **Config changes** (new endpoint, different round count) live only in `CONFIG` — do not hard-code values elsewhere.

---

## Git Workflow

- Default development branch: `master`
- Feature/agent branches follow the pattern: `claude/<description>-<id>`
- Commit messages are imperative and descriptive (see existing log for style).
- There is no CI/CD pipeline; deployment is manual (copy the HTML file to a static host or share directly).

---

## No Test Suite

There are no automated tests. Manual testing checklist:

- [ ] Welcome screen renders and "Start" button advances to DCE
- [ ] DCE shows 5 rounds; progress bar increments correctly
- [ ] Both card selections work; skipping via the skip button works
- [ ] Builder drag-and-drop assigns chips to slots; progress bar fills
- [ ] Summary shows top-3 product matches with correct percentages
- [ ] Data payload is POSTed to the Apps Script endpoint (check Network tab)
- [ ] Data falls back to `localStorage` if endpoint is unreachable
