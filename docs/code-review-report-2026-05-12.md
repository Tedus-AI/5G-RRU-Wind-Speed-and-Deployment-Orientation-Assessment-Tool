# Full Code Review Report

Project: **5G RRU Wind Speed and Deployment Orientation Assessment Tool**  
Review date: **2026-05-12**  
Reviewer: Codex (GPT-5.3-Codex)

---

## 1) Scope & Method

This is a full repository review focused on:

- Security and trust boundaries (DOM injection / third-party API use)
- Correctness and methodology consistency
- Performance and reliability
- Maintainability and code organization
- UX and accessibility risks

Reviewed files:

- `index.html`
- `index.legacy.html`
- `README.md`
- `docs/methodology.md`

---

## 2) Executive Summary

The project is product-complete and unusually well documented for a browser-only tool.  
However, there are several **high-priority engineering risks** to address:

1. **Potential DOM XSS vectors** caused by unescaped template interpolation into `innerHTML`.
2. **No timeout/retry/backoff policy** for external APIs (Nominatim / Open-Meteo / Overpass).
3. **Single-file architecture** in `index.html` creates high maintenance and regression risk.
4. **Methodology wording mismatch risk** between docs and UI narrative (hourly vs daytime filter wording).

---

## 3) Findings by Severity

## High Severity

### H-01: Unsafe `innerHTML` interpolation with externally sourced data

**Why it matters**  
Values from geocoding/API/localStorage flows are later inserted into HTML templates without escaping. Any malicious content that reaches localStorage or an API field can become executable DOM content.

**Evidence**

- Search history rendering uses `innerHTML` with dynamic `label`/`short` strings.  
  `row.innerHTML = history.map(... title="${label}">${short}</button>...`.
- Multi-site table renders `s.name`, `s.prevailing` through string templates and then assigns `wrap.innerHTML = html`.

**References**

- `index.html` search history rendering: lines 3703–3719.  
- `index.html` multi-site rendering: lines 3425–3440.

**Recommendation**

- Prefer DOM APIs (`createElement`, `textContent`, `setAttribute`) for user/API data.
- If templating is unavoidable, add strict `escapeHtml()` and apply to every interpolated token.
- Treat `localStorage` payload as untrusted input.

---

### H-02: External API calls have no timeout and no retry strategy

**Why it matters**  
Network instability, 429/5xx responses, and transient outages are expected for free public APIs. Without timeout/retry/backoff, user workflows fail unpredictably.

**Evidence**

- Direct `fetch` calls to:
  - Nominatim geocoding
  - Open-Meteo geocoding
  - Open-Meteo archive climate data
  - Overpass building density API
- Calls are wrapped in broad `try/catch`, but do not include `AbortController` timeout, retry budget, or status-based handling.

**References**

- `index.html` Nominatim fetch: lines 1913–1918.
- `index.html` Open-Meteo geocoding fetch: lines 1971–1976.
- `index.html` Open-Meteo archive fetch: lines 2018–2026.
- `index.html` Overpass fetch: lines 2057–2063.

**Recommendation**

- Build a shared `fetchWithPolicy()` helper:
  - timeout (8–12s)
  - retry for `429`, `503`, select network errors (max 2 retries)
  - exponential backoff + jitter
- Centralize API error-to-UI mapping for actionable user messages.

---

## Medium Severity

### M-01: Single large page file increases change risk

**Why it matters**  
`index.html` mixes style tokens, layout, localization text, geocoding logic, climate data I/O, analytics, charting, and report generation. This raises coupling and makes future bugfixes expensive.

**Evidence**

- Core app logic, API clients, renderers, state and controls all exist in one document.

**References**

- `index.html` includes all major concerns in one file (representative logic spans lines 1890–3850).

**Recommendation**

- Keep no-build workflow, but modularize with native ES modules:
  - `js/state.js`
  - `js/api.js`
  - `js/calc.js`
  - `js/render.js`
  - `js/events.js`
- Split CSS (`base.css`, `components.css`) while preserving static hosting simplicity.

---

### M-02: Methodology wording consistency risk (hourly vs daytime wording)

**Why it matters**  
This tool’s credibility depends on consistent method disclosure. In one location, prevailing wind is described with hot-season hourly samples; another text mentions daytime window emphasis.

**Evidence**

- Methodology says hot-season hourly sample set for wind-frequency distribution.
- UI methodology paragraph emphasizes daytime (10:00–18:00) hot-season wind usage.

**References**

- `docs/methodology.md`: lines 76–80.
- `index.html`: line 1575.

**Recommendation**

- Make the implementation rule explicit as a named config constant and sync documentation wording.
- Add a “calculation assumptions” export block in report output for traceability.

---

### M-03: Legacy file duplication (`index.legacy.html`) may drift

**Why it matters**  
The project has two full app files with largely mirrored logic. Feature/security fixes can diverge if not updated in both.

**Evidence**

- `index.legacy.html` contains parallel geocoding/fetch/history/rendering logic.

**References**

- `index.legacy.html` geocoding/fetch blocks: lines 1214–1360.
- `index.legacy.html` history rendering: lines 2904–2931.

**Recommendation**

- Share core logic modules and keep legacy as UI shell only, or formally deprecate one entrypoint.

---

## Low Severity

### L-01: Font requests are heavier than necessary

**Why it matters**  
Two Google Fonts families are loaded with overlapping scripts and multiple weights, increasing first render cost.

**References**

- `index.html`: lines 10–14.

**Recommendation**

- Trim to exact families/weights in use.
- Consider self-hosting critical fonts if performance becomes a target metric.

---

### L-02: Cache key precision may cause coarse reuse between nearby points

**Why it matters**  
Climate cache key rounds lat/lon to 2 decimals (~1.1 km scale at equator). Nearby distinct rooftop/microclimate selections may share cached data.

**Evidence**

- `climateCacheKey` uses `lat.toFixed(2)` and `lon.toFixed(2)`.

**References**

- `index.html`: lines 1992–1995.

**Recommendation**

- Increase precision (e.g., 3 decimals) or include user-selected resolution metadata.

---

## 4) Positive Notes

1. Methodology disclosure and limitations are clearly documented, which is strong engineering practice.
2. README is bilingual and deployment-friendly for browser-only users.
3. Climate caching strategy exists and has expiration semantics.

**References**

- `README.md`: lines 25–40, 46–60, 77–89.
- `docs/methodology.md`: lines 181–188.
- `index.html`: lines 1997–2004.

---

## 5) Recommended Remediation Plan

### Phase 1 (Immediate, 1–2 days)

- Replace unsafe `innerHTML` interpolation for search history and multi-site table paths.
- Introduce `escapeHtml()` for any residual HTML-template use.
- Add timeout + retry policy helper for all external fetches.

### Phase 2 (Near-term, 3–5 days)

- Extract core runtime into ES modules while keeping static/no-build deployment.
- Add minimal smoke tests (manual checklist + scripted lint/static checks).

### Phase 3 (Follow-up)

- Align methodology wording and UI/report assumption blocks.
- Decide legacy strategy (`index.legacy.html` as maintained target vs deprecated fallback).

---

## 6) Suggested Tracking Table

| ID | Severity | Topic | Owner | Target Date | Status |
|---|---|---|---|---|---|
| H-01 | High | DOM injection hardening | Frontend | +2 days | Open |
| H-02 | High | API fetch resilience | Frontend | +2 days | Open |
| M-01 | Medium | Modularization | Frontend | +1 sprint | Open |
| M-02 | Medium | Method wording sync | PM+Eng | +1 sprint | Open |
| M-03 | Medium | Legacy entrypoint strategy | Tech Lead | +1 sprint | Open |
| L-01 | Low | Font optimization | Frontend | Backlog | Open |
| L-02 | Low | Cache key precision | Frontend | Backlog | Open |

---

## 7) Final Recommendation

Proceed with this tool for internal decision support, but do **not** treat current build as security-hardened production UI until H-01 and H-02 are addressed. After those fixes, prioritize modularization to preserve delivery speed and reduce regression risk.
