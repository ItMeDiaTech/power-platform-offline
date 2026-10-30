# power-platform-offline — Project Plan

**Goal.** An offline, desktop-first renderer for Power BI reports (PBIP / PBIR / TMDL, PBIX import) and Power Apps canvas apps (`*.pa.yaml`) whose visual output and interactive behavior are **indistinguishable from the Microsoft products**, plus a one-push deployment of the whole project to a Fabric workspace and a Dataverse environment.

**Method.** Identity is a measured property, not a design intent. Every phase below is driven by a *fidelity harness* that captures reference output from the real Microsoft products, renders the same inputs in our app, diffs the two, and scores the gap. The project iterates on the lowest score until the exit criteria hold. No phase is "done" by inspection.

**Language.** C# / .NET throughout. No Node, no TypeScript, no Python in the product or the harness.

---

## 1. Definition of "identical"

"Identical" is decomposed into five measurable layers. Each has a metric, a tolerance, and a reference source.

| Layer | What is compared | Metric | Target | Reference source |
|---|---|---|---|---|
| **L1 Layout** | Bounding box of every visual / control / page element | Max abs. pixel delta at 100% DPI | ≤ 1 px | DOM geometry from the Microsoft renderer (see §3) |
| **L2 Appearance** | Rasterized page / screen | Perceptual diff (SSIM) and per-region pixel mismatch | SSIM ≥ 0.995; mismatch ≤ 0.2 % of pixels outside anti-aliasing bands | Screenshots of the Microsoft renderer at fixed size and DPI |
| **L3 Data** | Every value displayed (cells, labels, axes, cards) and its formatted string | Exact match after format-string rendering | 100 % | DAX results from Power BI's own engine; Power Fx values from the Power Apps player |
| **L4 Behavior** | State after each scripted interaction (click, hover, select, type, navigate) | L1–L3 re-evaluated after each step of a scripted scenario | Same thresholds per step | Scripted sessions against the Microsoft product |
| **L5 Motion** | Transition and animation timing | Frame-sampled SSIM during transitions | Advisory only until L1–L4 hold | Screen recordings |

Anything the Microsoft product does not expose deterministically (anti-aliasing of text, sub-pixel rounding across GPUs) is masked, not chased. The mask list is a committed artifact and must shrink over time, never grow silently.

### Why Windows desktop first makes identity reachable

- Power BI Desktop and the Power BI service render reports in a Chromium-based web view with the Segoe UI font family. Our app hosts Blazor in **WebView2** (Chromium) on Windows and reads Segoe UI from the OS. Same layout engine, same text rasterizer: pixel identity is physically possible. A cross-platform build is a later concern and will carry the Selawik fallback.
- Power BI Desktop ships a local Analysis Services engine (`msmdsrv`). When Desktop is installed, our harness connects to it over XMLA on `localhost` to obtain **ground-truth DAX results** without any cloud call. This is the data oracle for L3.
- Power Apps has no offline player. Reference capture for canvas apps uses Microsoft's **Power Apps Test Engine** (Playwright, .NET) against the app published in the tenant. The capture step is online; the viewer is not.

---

## 2. Architecture (all .NET)

```
┌────────────────────────── Product ──────────────────────────┐
│ Ppo.Pbir        PBIR JSON → model (schemas vendored, pinned) │
│ Ppo.Tmdl        TOM TmdlSerializer wrapper → semantic model │
│ Ppo.Pbix        PBIX container, Xpress9/VertiPaq reader     │
│ Ppo.Data        DuckDB.NET engine, Parquet/CSV sample data  │
│ Ppo.Dax         DAX parser (ANTLR4 C#) → logical plan → SQL  │
│ Ppo.Query       Visual query planner (queryState + filters  │
│                 + slicers + bookmarks → SUMMARIZECOLUMNS)    │
│ Ppo.Rendering   Razor class library: page canvas, visuals,  │
│                 theme resolver, format-string engine        │
│ Ppo.Canvas      pa.yaml parser, control tree, formula graph  │
│ Ppo.PowerFxHost Microsoft.PowerFx.Interpreter in-process,   │
│                 canvas host functions, mock data sources    │
│ Ppo.Desktop     Photino.Blazor shell (WebView2 on Windows)  │
│ Ppo.Cli         ppo: validate, capture, diff, score, deploy │
├────────────────────────── Fidelity ─────────────────────────┤
│ Ppo.Fidelity.Capture  Drives Power BI Desktop (UI Automation│
│                       + XMLA/ADOMD) and Test Engine; stores  │
│                       goldens per corpus case               │
│ Ppo.Fidelity.Diff     Layout, image, data, behavior diffs   │
│ Ppo.Fidelity.Score    Scoreboard, thresholds, mask registry │
│ fidelity/corpus/      Cases: inputs + sample data + goldens │
└─────────────────────────────────────────────────────────────┘
```

Key decisions:

- **Two DAX execution paths, one offline.** `Ppo.Dax` compiles a growing DAX subset to DuckDB SQL. When Power BI Desktop is present, an optional *oracle mode* also runs the same query against Desktop's local engine. In development the oracle verifies; in the product it can serve as a local fallback for DAX we do not yet compile. Both are offline. Nothing from Desktop is redistributed.
- **Microsoft's own query text is the spec.** Power BI Desktop's Performance Analyzer exports the exact DAX each visual issues. The harness stores those queries per visual; `Ppo.Query` must produce a semantically equivalent query, verified by running both against the oracle and comparing result sets.
- **Power Fx is Microsoft's code.** Formula semantics come from `Microsoft.PowerFx.Interpreter`; we build the canvas host (control symbols, `ThisItem`/`Parent`/`Self`, `Navigate`, `Set`, `UpdateContext`, `Collect`, `Patch`, `Notify`, `Reset`, `SubmitForm`, delegation emulation). Behavior identity is validated against the real player via Test Engine.
- **Rendering is SVG/HTML authored in Razor**, matching the technology the Microsoft renderer uses, rather than a third-party charting library that would impose its own geometry.

---

## 3. The fidelity harness (built first)

### 3.1 Corpus

`fidelity/corpus/<case>/` contains:

- `project/` — the PBIP (Report + SemanticModel) and/or solution folder with the canvas app source.
- `data/` — Parquet per table, exported once from Power BI Desktop (or extracted via `Ppo.Pbix`).
- `scenario.json` — the interaction script: viewport size, DPI, page/screen order, and steps (`click visual X datapoint Y`, `select slicer value`, `hover`, `type into control`, `navigate`).
- `golden/` — captured references: `layout.json` (element boxes), `*.png` (screenshots per step), `queries/*.dax` (Performance Analyzer export), `results/*.json` (oracle result sets), `values/*.json` (Power Fx property values per step).
- `masks.json` — regions excluded from pixel comparison, each with a reason.

The corpus starts small and deliberately: one report per visual type, one app per control type, then composite real-world cases. Every bug fix adds or extends a case.

### 3.2 Capture

- **Power BI:** `ppo capture pbi <case>` opens the PBIP in Power BI Desktop, sets the window to the scenario's viewport, walks the scenario with UI Automation, screenshots each step, exports Performance Analyzer, and queries the local engine via ADOMD for every stored query. Element geometry is read from Desktop's embedded web view accessibility tree.
- **Power Apps:** `ppo capture pa <case>` runs a Test Engine plan derived from `scenario.json` against the published app, recording screenshots and control property values per step.

### 3.3 Diff and score

`ppo score <case>` renders the case in our engine headlessly (WebView2 off-screen), replays the scenario, and produces per-layer scores against the goldens. `ppo score --all` writes `fidelity/SCOREBOARD.md` and fails CI when any case regresses below its recorded best. The scoreboard is the project's single progress indicator.

---

## 4. Phases and exit criteria

Each phase lists what is built and the measured condition that closes it. Phases overlap in practice; exit criteria do not.

### Phase 0 — Harness and corpus seed
Build `Ppo.Fidelity.*`, the `ppo` CLI, and ten seed cases (five reports, five apps). Capture goldens.
**Exit:** `ppo score --all` runs end to end on CI (Windows runner) and prints a scoreboard, even if every score is zero.

### Phase 1 — Report structure and static appearance
PBIR and TMDL parsing with schema validation; page canvas at true size; theme resolution (Fluent 2, Classic 2026, Classic 2018, custom themes, `visualStyles`); text classes; visual containers with titles, backgrounds, borders, shadows; shapes, text boxes, images, buttons; table and matrix with placeholder data.
**Exit:** L1 ≤ 1 px and L2 ≥ 0.99 on every seed report with data visuals masked.

### Phase 2 — Data identity
`Ppo.Data` loads corpus data into DuckDB; `Ppo.Query` builds per-visual queries; `Ppo.Dax` compiles the first subset (aggregations, CALCULATE with column filters, ALL/ALLEXCEPT/REMOVEFILTERS, FILTER, relationships, RELATED, SELECTEDVALUE, VALUES, SUMX/AVERAGEX, DIVIDE, IF/SWITCH, VAR, basic time intelligence); format strings and display units; oracle mode.
**Exit:** L3 = 100 % against oracle result sets for every seed report; every Performance Analyzer query reproduced with an identical result set.

### Phase 3 — Visual identity
Per visual type, in this order: card, multi-row card, table, matrix, slicer, column/bar families, line/area, combo, pie/donut, scatter, waterfall, funnel, gauge, KPI, treemap, map (local tiles). Axis layout, label density and collision rules, legends, data labels, conditional formatting, tooltips.
**Exit:** L2 ≥ 0.995 unmasked on each visual's dedicated case; mask registry contains only anti-aliasing bands.

### Phase 4 — Interaction identity
Cross-filter and cross-highlight (including `drillFilterOtherVisuals` and visual interaction edits), slicer sync, sort, drill up/down/expand, drillthrough, bookmarks (data, display, current page), page navigation, selection state, keyboard focus and tab order, mobile layout.
**Exit:** L4 holds across every scenario step in the corpus.

### Phase 5 — Canvas app identity
pa.yaml v3 parsing; classic and modern controls as Razor components with Microsoft's default property values; responsive containers; Power Fx data properties with reactive recalculation; behavior formulas; galleries, forms and data cards; components and component libraries; mock Dataverse/SharePoint/Excel sources seeded from `data/`; delegation warnings and the data-row limit.
**Exit:** L1–L4 against Test Engine goldens for every seed app.

### Phase 6 — One-push deployment and round trip
`ppo deploy <stage>` validates (schemas, Fab Inspector rules, Tabular Editor BPA, `pac canvas validate`, solution checker), publishes Fabric items via the Fabric REST API, imports the Power Platform solution via `pac` with a deployment-settings file, binds and refreshes. GitHub Actions workflow with dev/test/prod environments.
**Exit:** deploy a corpus case, capture it from the service and the player, and score it against our local render with the same thresholds. The round trip is the final identity test.

### Phase 7 — Breadth
Custom visuals (`.pbiviz` host), PCF controls, model-driven form preview, cross-platform shell, motion (L5).

---

## 5. Prerequisites on the development machine

- Windows 11, .NET SDK (latest LTS), WebView2 runtime.
- Power BI Desktop (current monthly release) — required for the data oracle and Power BI capture.
- Access to a Power Platform environment and Fabric/PPU workspace — required only for Power Apps capture and for Phase 6.
- Power Platform CLI (`pac`), Tabular Editor 2, Fab Inspector.

---

## 6. Working agreement

- The scoreboard is the backlog. Work on the lowest-scoring case or layer unless a prerequisite blocks it.
- A change that lowers any score fails CI. A mask may only be added with a reason and an issue link.
- Every fix lands with a corpus case that would have caught it.
- Schema and control versions are pinned and vendored; unknown versions render with a warning, never a crash.
- Nothing from Power BI Desktop, the Power Query engine, Segoe fonts or AppSource visuals is redistributed. The product reads what is on the user's machine.
- Conventional commits (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`), one logical change per commit.

---

## 7. Risks and how the plan contains them

| Risk | Containment |
|---|---|
| DAX subset never reaches the long tail | Oracle fallback keeps the product correct while coverage grows; coverage is a scoreboard line item |
| Microsoft changes rendering between monthly releases | Goldens are versioned with the Desktop build that produced them; re-capture is a scripted command |
| Power Fx host functions diverge from the player | Test Engine goldens cover each host function; the interpreter itself is Microsoft's |
| Anti-aliasing and GPU differences poison pixel scores | Fixed DPI, software rendering in capture and score, masks limited to anti-aliasing bands |
| Schema/version churn in PBIR and pa.yaml | Pinned, vendored schemas; warn-and-render on unknown versions |
| Power Apps capture needs the tenant | Only the capture needs it; goldens are committed, so scoring stays offline |
