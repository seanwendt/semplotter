# Semaglutide Plasma Concentration Plotter

A standalone, client-side pharmacokinetic (PK) modeling tool for visualizing estimated semaglutide plasma concentrations over time. Built as a single HTML app — no server, no build step, no dependencies to install.

> ⚠️ **For educational and informational use only.** This tool does not constitute medical advice. Individual pharmacokinetics vary substantially. Never adjust your dosing based on this model without consulting your prescribing physician.

## Background

Semaglutide (brand names Ozempic, Wegovy) is a GLP-1 receptor agonist administered as a once-weekly subcutaneous injection for diabetes and/or chronic weight management indications, depending on product and dose. Because of its long half-life (~1 week), drug accumulates over several weeks before reaching steady state — making it useful to visualize how plasma concentrations evolve across a titration schedule.

This tool models that behavior using a **one-compartment pharmacokinetic model with first-order absorption and elimination**, parameterized from FDA semaglutide clinical pharmacology review and labeling documents.

---

## Features

- **Injection schedule logging** — add multiple doses with individual dates and amounts
- **Per-injection body weight** — each dose records weight at time of injection; the PK model applies a simple weight-based exposure scalar per dose
- **Three chart views:**
  - `ng/mL` — plasma concentration in standard clinical units
  - `mg equiv.` — estimated total drug in the central compartment (C × Vd)
  - `Weight` — body weight over time, one point per injection
- **Configurable time range** — 30 days to 1 year
- **Steady state detection** — computed from consecutive pre-dose trough estimates (relative change < 5%), not a fixed heuristic
- **Key statistics** — current estimated concentration, Cmax, Cmin (trough), and steady state status, each with explanatory tooltips
- **Dose markers and TODAY line** overlaid on the PK chart when they fall inside the visible date range
- **Persistent storage** — auto-saves to `localStorage` on every change; status indicator shows whether browser storage is available
- **Shareable schedules** — generate a copyable URL that contains your full schedule; no backend required
- **Static hosting friendly** — works without a backend; CDN assets are used for charts, fonts, README rendering, and the donation button

---

## PK Model

Model inputs are sourced from FDA semaglutide review and labeling documents, with `ka` derived to place the model tmax around 48 hours within the reported 1–3 day range for subcutaneous dosing.

| Parameter | Value | Source |
|---|---|---|
| Elimination half-life (t½) | ~1 week (168 h) | FDA labeling |
| Absorption tmax range (SC) | 1–3 days | FDA labeling |
| Bioavailability (F) | ~89% | FDA clinical pharmacology review / labeling |
| Clearance (CL/F) | ~0.05 L/h | FDA labeling |
| Volume of distribution (Vd) | ~12.5 L | FDA labeling |
| Plasma protein binding | >99% (albumin) | FDA labeling / DrugBank |
| Steady state (once-weekly) | ~4–5 weeks | FDA labeling |
| Dose proportionality | Across evaluated injectable dose ranges | FDA clinical pharmacology review |

### Equations

Plasma concentration from a single dose is modeled using the **Bateman equation**:

```
C(t) = (F · D / Vd) · (ka / (ka − ke)) · (e^(−ke·t) − e^(−ka·t))
```

Where:
- `D` = dose in mg
- `ka` = absorption rate constant (0.0606 h⁻¹, derived to reproduce tmax ≈ 48 h within the reported 1–3 day range)
- `ke` = elimination rate constant = CL / Vd
- `t` = time since injection in hours

Concentrations from multiple doses are summed by **superposition**.

### Weight Scaling

Body weight affects semaglutide exposure; FDA labeling notes that exposure decreases as body weight increases. This tool uses a simple inverse body-weight scalar:

```
scalar = (weight_kg / 90) ^ (-1.0)
CL_eff = CL / scalar
```

Because each injection records its own weight, the model can show the effect of weight changes across a titration schedule. This is a simplified scalar, not the full FDA population PK model.

### Limitations

- Population-average model only — individual variability is not represented
- Single-compartment approximation — semaglutide PK is more precisely described by a two-compartment model, but the one-compartment fit is adequate for visualization purposes given the available parameters
- Injection site effects (abdomen vs. thigh vs. upper arm) are minor and not separately modeled per FDA findings
- Subcutaneous absorption is modeled as first-order; actual SC absorption can be more complex

---

## Usage

The tool is hosted and ready to use — no installation required. Simply open it in any modern browser.

If you prefer to self-host or run it locally:

1. Clone or download the repository
2. Deploy the `web/` directory to any static host (Netlify, GitHub Pages, etc.)
3. Or open `web/index.html` directly from your local filesystem. The core app works as a local file; the README/LICENSE popup depends on the browser allowing local `fetch()` access to adjacent files.

Once open:

1. Enter a date, dose amount, and your body weight at the time of injection
2. Click **+ Add Injection**
3. Repeat for each injection in your schedule
4. Use the **ng/mL / mg equiv. / Weight** tabs to switch chart views
5. Use **Share** to copy a URL that can restore the same schedule on another browser or machine

### Data Persistence

| Browser | localStorage | Notes |
|---|---|---|
| Chrome | ✅ Supported | Recommended |
| Firefox | ✅ Supported | Works normally |
| Edge | ✅ Supported | Works normally |
| Safari | ✅ Supported | Works normally when hosted |
| Private/Incognito | ⚠️ Limited | Storage may be temporary or unavailable; use Share before closing |

> **Note:** If running from a local `file://` path rather than a hosted URL, Safari may restrict localStorage. Use the **Share** button to preserve your schedule in that case.

The storage status note in the Injection Schedule panel shows whether auto-save is active. If it shows an amber warning, use **Share** before closing.

### Share URL Format

Share URLs store a compact, encoded copy of the same schedule data directly in the `share` query parameter. The decoded payload uses a versioned format:

```json
{
  "v": 1,
  "d": [
    ["2026-04-13T12:00:00.000Z", 0.25, 89.8]
  ]
}
```

Each dose row is `[dateISO, amount, weightKg]`. The compact format keeps URLs shorter while preserving backward compatibility for older shared links.

---

## Dependencies

No package installation is required. Runtime assets are loaded from CDNs or adjacent static files.

| Library | Version | Purpose |
|---|---|---|
| [Chart.js](https://www.chartjs.org/) | 4.4.1 | Chart rendering |
| [marked.js](https://marked.js.org/) | 9.1.6 | README Markdown rendering |
| [Google Fonts](https://fonts.google.com/) | — | DM Sans, DM Mono typefaces |
| [PayPal Donate SDK](https://www.paypal.com/) | — | Donation button rendering |

The app can be opened without a backend, but it is not fully self-contained offline: charts, fonts, Markdown rendering, and the PayPal button rely on CDN or remote assets unless they are already cached by the browser.

## Tests

The single HTML file includes a small built-in unit test runner for parsing, serialization, example schedule generation, weight scaling, PK contribution math, concentration summation, and steady-state detection.

Run it from the browser console:

```js
runUnitTests()
```

Or open:

```text
index.html?test=1
```

## Repository Structure

```
_semplotter/
├── web/             ← deployed static web app
│   ├── index.html
│   ├── README.md
│   ├── LICENSE
│   └── assets/
└── netlify.toml
```

---

## References

- FDA NDA 209637 Clinical Pharmacology Review — Semaglutide (Ozempic), Novo Nordisk, 2017. Available at: https://www.accessdata.fda.gov/drugsatfda_docs/nda/2017/209637Orig1s000ClinPharmR.pdf
- DrugBank: Semaglutide (DB13928) — https://go.drugbank.com/drugs/DB13928

---

## License

[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

You are free to share and adapt this work for non-commercial purposes, provided you give appropriate credit and distribute any derivatives under the same license. Commercial use is prohibited. See `LICENSE` for full terms.

---

## Disclaimer

This project is not affiliated with, endorsed by, or sponsored by Novo Nordisk or any regulatory authority. It is an independent educational tool. The pharmacokinetic model is a simplification intended for informational visualization only and should not be used for clinical decision-making.
