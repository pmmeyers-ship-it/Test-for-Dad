# OSV-BLA — Open Source Business Legitimacy Assessment Workbench v1.0
## Documentation: User Guide · Connector Reference · Testing Checklist · Limitations

**Designation:** OPEN SOURCE ASSESSMENT
**Deliverable:** `business_legitimacy_assessment_tool.html` — a single self-contained HTML file. No installation, no CDNs, no remote fonts, no telemetry, no analytics, no cookies. Implements Option A (browser-only with manual fallback) and Option C (local public-dataset import). Option B (approved proxy) is described at the end as an optional enhancement only.

---

## 1. Quick start

1. Copy the single HTML file into the secure environment (USB, file share, or paste into a new file).
2. Open it in any modern browser (Chrome, Edge, Firefox). It works from `file://` or any internal web server. Nothing is downloaded at load time and zero network requests occur until you press a Run button.
3. Work the tabs in numbered order: **Intake → Source Checks → Manual Evidence → Identity Consistency → Red Flags → Source Log → Scoring → Network Log → Report.** This matches the recommended analyst workflow built into the Methodology tab.
4. The assessment gauge at the top updates live: overall score, verdict band, and six sub-scores (evidence completeness, source quality, consistency, red-flag severity, automation coverage, manual dependency).

### Controls in the header
- **External calls enabled** — master switch. When off, every connector records a SKIPPED entry in the Network Log and nothing leaves the machine.
- **Local autosave** — optional. Saves the case to the browser's local storage on this machine only. Off by default. "Clear all locally stored data" on the Network Log tab wipes it.

### API keys
Only one connector pair uses a key (System for Award Management, SAM.gov). The key is entered at runtime on the Source Checks tab, lives only in that field, and is never written to disk by the tool. Without a key, both SAM connectors fall back to manual links.

### Case management
- **Save case (JSON)** / **Import case (JSON)** — full round trip of intake, evidence, flags, logs, settings, and connector run history. A malformed file produces a clear error and changes nothing.
- **Export summary (.txt)**, **Export source log (.csv)**, **Export full report (.html)**, **Print / save as PDF** (print stylesheet included).
- **Compare tab** — load a second saved case for a read-only side-by-side of scores, flags, and sub-scores.
- **Re-run (preserve prior results)** — connector run history is kept with timestamps; each connector card shows its history.

---

## 2. Connector reference

Every connector states, on its card, exactly what it sends and where, before you run it. Every attempt — success, failure, timeout, rate-limit, or skip — is recorded in the Network Activity Log. In the source code, every external call site is marked with the comment `EXTERNAL CALL`; all fetches pass through one gateway function (`netFetch`) that enforces the toggle, a 12-second timeout, `credentials: omit`, and `no-referrer`.

| # | Connector | Mode | Category | Endpoint / method | Data sent | Reliability class |
|---|-----------|------|----------|-------------------|-----------|-------------------|
| 1 | SAM.gov Entity Registration | API key | Registration / procurement | `api.sam.gov/entity-information/v3/entities` | Legal name + your key | Official |
| 2 | SAM.gov Exclusions | API key | Sanctions / debarment | `api.sam.gov/entity-information/v4/exclusions` | Legal name + your key | Official |
| 3 | USAspending.gov Federal Awards | Automatic (no key) | Procurement | `api.usaspending.gov/api/v2/search/spending_by_award/` (POST) | Recipient name only | Official |
| 4 | Domain Registration (RDAP) | Automatic | Website / domain | `rdap.org/domain/{domain}` → authoritative registry | Domain name only | Non-official (registry-operated) |
| 5 | Website Reachability | Automatic (limited) | Website | `HEAD` on the business's own site, `no-cors` | The request itself | Direct observation |
| 6 | State Business Registry | Manual | Registration | Official state search link from a built-in 17-state directory (plus NASS index) | Nothing | Official |
| 7 | Licensing Authority Check | Manual | Licensing | Analyst-identified issuing authority | Nothing | Official |
| 8 | OFAC SDN List | Local file import | Sanctions | None — you download `sdn.csv` from Treasury separately; matching runs locally | Nothing | Official |
| 9 | Public Dataset Import | Local file import | Any | None — parses CSV/JSON you downloaded from official sources | Nothing | Official |

Connector behaviors worth knowing:

- **Failure handling.** CORS blocks, timeouts, HTTP errors, and rate limits (429) never break the run; the connector records a FAILED source-log entry and exposes its manual fallback link. A failed check scores as "unknown," never as negative evidence.
- **Automatic red flags.** Connectors raise AUTO flags only for: SAM exclusion match (critical), OFAC SDN name match (critical, with an explicit over-matching caveat), domain not found in registry data (high), domain under one year old (medium), and domain age far younger than claimed years in business (medium). AUTO flags are de-duplicated and deletable.
- **RDAP matching note.** The bootstrap service at rdap.org redirects to the authoritative registry's own RDAP server — the same data that backs WHOIS, without scraping.
- **OFAC matching note.** The importer does substring name matching, which deliberately over-matches; the tool says so on every hit and requires identity confirmation before the flag is relied on.

Adding a connector: each is a self-contained object in the `CONNECTORS` array with `id, name, mode, category, cls, describe(), manualLink(), run()` (and `fileHandler()` for import mode). Copy an existing one and route any network call through `netFetch` so logging, the toggle, and the timeout apply automatically.

---

## 3. Scoring reference

`Score = Σ (category weight × capped category credit) − Σ red-flag penalties`, clamped to 0–100.

- Per-item credit = strength (strong 1.0 / moderate 0.6 / weak 0.3) × source class (official 1.0 / non-official 0.7 / analyst observation 0.6 / business self-claim 0.2). Weakening items subtract. Category credit caps at 1.0.
- Default weights (sum 100): registration 20, licensing 15, physical presence 15, website/domain 10, contactability 10, reputation 10, procurement 10, identity consistency 10. The sanctions category carries no points — sanctions findings act through critical penalties, and a "no match" result counts toward completeness only.
- Default penalties: low −2, medium −5, high −10, critical −20. Weights and penalties are editable on the Scoring tab; the report flags any deviation from defaults and prints the recorded justification (or its absence).
- Bands: 85–100 high confidence the business is real; 65–84 moderate confidence; 40–64 low confidence / more evidence required; 0–39 high concern / potentially suspicious.
- The identity-consistency category is driven by the consistency matrix: credit = consistent fields ÷ fields reviewed. Marking a field inconsistent also raises a medium AUTO flag.

---

## 4. Testing checklist (executed before delivery)

Automated test harness (jsdom, stubbed network): **22 of 22 checks passed.**

| Requirement | Result |
|---|---|
| Application loads without installation; selects and 9 connector cards populate | PASS |
| Zero network calls at page load | PASS (verified by instrumented fetch) |
| External calls are visible and documented (per-card description + Network Log + `EXTERNAL CALL` code comments) | PASS |
| External-call toggle blocks all fetches and logs SKIPPED entries | PASS |
| Failed source calls do not break the app; manual fallback offered | PASS (failure path exercised) |
| CORS-blocked sources fall back to manual entry | PASS (catch path records failure + manual link) |
| Score recalculates when data changes (evidence add, flag add, weight change) | PASS |
| Red-flag penalty arithmetic correct (observed 18 → 8 after high flag) | PASS |
| JSON export/import round trip preserves the case | PASS |
| Bad JSON import produces a clear error and leaves state untouched | PASS |
| CSV export with proper escaping of quotes, commas, newlines | PASS |
| Report generation includes executive summary, scoring appendix, failed-checks section, OPEN SOURCE ASSESSMENT designation | PASS |
| Print-to-PDF supported (print stylesheet isolates the report) | PASS (stylesheet present; visual print check is environment-dependent) |
| Reset clears evidence, flags, logs | PASS |
| No hidden telemetry; every observed request went to a documented source | PASS |

Manual checks recommended on first use in your environment: open from `file://`, run RDAP against a known domain, confirm the Network Log entry, print the report, and confirm the SAM connectors against your own key (live CORS behavior of api.sam.gov can vary; the manual fallback covers a block).

---

## 5. Limitations

1. **Confidence, not proof.** The output is an evidence-based confidence assessment. A high score lowers risk; it cannot eliminate it, because sophisticated fraud invests in exactly the signals measured here.
2. **CORS constrains browser automation.** Most state registries and many federal pages block direct browser requests by design. Automation coverage will rarely approach 100%; the manual-dependency sub-score discloses this on every report.
3. **Name matching is fuzzy.** Exact-name queries miss variants; substring matching (OFAC importer) over-matches. Every list hit requires human identity confirmation, and the tool says so at the point of the hit.
4. **Open sources lag reality.** Dissolved entities can show active for months; new licenses can take weeks to appear.
5. **Jurisdiction coverage is uneven.** The built-in registry link directory covers 17 U.S. states plus a national index; other jurisdictions route through manual entry.
6. **Single-source claims are unconfirmed.** Consistent with standing tradecraft, treat any conclusion resting on one source as unconfirmed; the consistency matrix exists to force corroboration.
7. **The tool respects access controls.** It will not bypass CAPTCHAs, logins, paywalls, robots restrictions, or rate limits; a blocked source is a manual task, not a target.
8. **Browser storage is local and unencrypted.** Autosave and any pasted key live in the browser profile on that machine. Use "Clear all locally stored data" when finished on shared workstations.

---

## 6. Option B (optional enhancement, not included by default)

If the secure environment approves a proxy, a minimal design that preserves the tool's guarantees: a small local HTTP service that (a) forwards requests only to a hard-coded allowlist of public official endpoints, (b) appends nothing to the request, (c) logs every request to an append-only file, (d) stores no response data, and (e) strips credentials. The application would need only one change: pointing connector base URLs at the proxy. Do not deploy a general-purpose CORS proxy; allowlisting is the control that keeps the network story auditable.
