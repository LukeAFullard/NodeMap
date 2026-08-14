# Graph Builder App — Architecture & Plan (v2)

A frictionless, client-side web application for dynamic system mapping and static sports tactics boards.

---

## 1. Concept Overview

A browser-based tool for building connected node-link graphs with zero backend, zero installs, and zero sign-up. Two distinct modes, driven by the same underlying engine but presented as two thin, purpose-built front ends:

- **Systems Mode (Policy & Comms):** Force-directed graphs for visualizing dependencies, environmental reporting ecosystems, and stakeholder networks.
- **Tactics Mode (Sports):** Fixed-position boards over a pitch/court background, for mapping formations, movement, and set-pieces across Soccer, Cricket, Rugby, and Hockey.

**Design note carried over from review:** these two audiences (policy analysts vs. coaches) don't share a mental model, so the plan below treats them as **one shared engine, two separate landing experiences** rather than a single UI trying to serve both. This keeps the "click a link, zero friction" promise intact for each audience.

---

## 2. Architecture & Tech Stack

### Core Technologies
- **Rendering engine:** `vis-network` for Systems Mode (mature physics, drag/drop, minimal setup).
  - **Open question to resolve in Phase 0:** whether `vis-network`'s edge model can cleanly support Tactics Mode's annotation needs (see §4). If not, Tactics Mode may need a lighter custom canvas/SVG layer instead of forcing it through a graph library built for node-link diagrams. Prototype this before committing — it's the single biggest architectural risk in the plan.
- **Styling:** Tailwind CSS for rapid, consistent UI across both modes.
- **State management:** In-memory app state + `localStorage` autosave, with manual JSON export/import as the "real" persistence and sharing mechanism.
- **History:** A simple undo/redo stack (command pattern) sitting above the DataSets — see §5, Step 6.
- **Mobile strategy:** Responsive layout with explicit interaction modes (see §4). Data structures are kept UI-agnostic so a future React Native/Flutter shell could reuse them if needed.

### Pros & Cons

**Pros**
- Zero friction: no installs, accounts, or servers.
- Free hosting via GitHub Pages, consistent with TimeTag/BunkerPDF.
- Privacy by default: all editing happens client-side, which matters for local-government or internal-club data.
- Shareable output: a single JSON file is the whole "save game."

**Cons**
- Performance ceiling in the thousands-of-nodes range with physics enabled.
- Persistence is manual — no auto-sync across devices without the user handling files themselves.
- No collaboration/multi-user editing without adding a backend (out of scope for v1).

---

## 3. Use Cases

### Use Case A — Environmental Reporting & Local Government Policy
- **Setup:** Physics on. Nodes color-coded by category (e.g., blue = hydrological data, green = catchment zones, red = compliance metrics).
- **Value:** Turns static reports into an explorable ecosystem — click "Nitrate Limits" and see every connected monitoring station, agricultural policy, and stakeholder in one view.
- **Data sensitivity:** Likely to include internal or pre-release government data. Client-side-only architecture is a genuine selling point here, not just a technical convenience — worth stating explicitly in the app's own copy.

### Use Case B — Sports Tactics (Soccer, Cricket, Rugby, Hockey)
- **Setup:** Physics off. Background image of the relevant field/pitch/court. Nodes represent players; annotations represent movement.
- **Value:**
  - Soccer/Rugby/Hockey: show how a defensive line shifts during a passage of play.
  - Cricket: map fielding placements (slips, gully, cover) against a specific batter.
  - Sharing: export a lightweight JSON file to a team chat before match day.
- **Key gap from v1 plan:** node-link *edges* alone don't cover what tactics boards need. Coaches think in terms of:
  - **Movement arrows** (solid = pass/ball movement, dashed = off-ball run, curved = specific paths)
  - **Player roles/numbers**, not just labels
  - **Zones** (freehand or shaped regions, e.g., marking a pressing trap or fielding arc)
  - **Locked vs. movable elements** (pitch markings and fixed reference points shouldn't drag with the player being moved)

---

## 4. UI/UX

### Desktop Layout
- **Floating toolbar** (top-left): Add Node, Add Edge/Arrow, Add Zone, Mode Switch, Undo/Redo, Import/Export.
- **Contextual side panel:** click a node/edge to slide out a right-hand editor for labels, color, shape, role, arrow style.
- **Keyboard shortcuts:** `Del`/`Backspace` to remove selection, `Ctrl+Z`/`Ctrl+Shift+Z` for undo/redo, `Ctrl+S` to trigger export.

### Mobile Experience (the "swipe problem")
Panning a canvas and dragging a node use the same gesture, so they conflict constantly on touch. Mitigations:
- **Explicit modes via FAB:** hard toggle between "Pan/Zoom" and "Edit/Drag" — no gesture-guessing.
- **Bottom sheets** instead of side panels for node/edge editing on small screens.
- **Inflated hitboxes** around thin elements (edges, arrows) so taps don't need pixel precision.
- **Long-press to enter edit mode on a single node** as an alternative to global mode-switching, worth A/B testing against the FAB approach once there's a working prototype.

### Locking & Grouping (new)
- Individual nodes/edges can be marked **locked** (excluded from drag and from physics recalculation).
- Useful for: fixed pitch markers in Tactics Mode, or an "anchor" policy node in Systems Mode that shouldn't drift.
- **Grid snapping** (optional toggle) for precise tactical placement — off by default in Systems Mode, on by default in Tactics Mode.

---

## 5. Step-by-Step Implementation Blueprint

### [ ] Step 0: De-risk the Annotation Model (new — do this first)
Before writing any app scaffolding, spike a throwaway prototype that answers: *can `vis-network` edges be styled/curved/dashed enough to serve as tactical movement arrows, and can zones be layered on top via an SVG/canvas overlay?*
- If yes → proceed with `vis-network` as the single engine for both modes.
- If no → consider `Cytoscape.js` (more actively maintained, more flexible edge styling, no built-in physics toggle as clean as `vis-network`'s) or a custom lightweight SVG canvas for Tactics Mode specifically, sharing only the data model and export format with Systems Mode.

This step exists because retrofitting annotation support after Phase 2 is much more expensive than deciding it up front.

### [ ] Step 1: Initialize the Canvas & Data Sets

```javascript
// Core state
const nodes = new vis.DataSet([]);
const edges = new vis.DataSet([]);
let network = null;

function initNetwork(containerId) {
    const container = document.getElementById(containerId);
    const data = { nodes, edges };

    const options = {
        interaction: { dragNodes: true, hover: true },
        physics: {
            enabled: true,
            barnesHut: { gravitationalConstant: -2000 }
        }
    };

    network = new vis.Network(container, data, options);
}
```

### [ ] Step 2: Mode Switching (Systems vs. Tactics)

```javascript
function setMode(mode, backgroundUrl = null) {
    const container = document.getElementById('network-canvas');

    if (mode === 'tactics') {
        network.setOptions({ physics: { enabled: false } });

        if (backgroundUrl) {
            container.style.backgroundImage = `url('${backgroundUrl}')`;
            container.style.backgroundSize = 'contain';
            container.style.backgroundRepeat = 'no-repeat';
            container.style.backgroundPosition = 'center';
        }
    } else {
        network.setOptions({ physics: { enabled: true } });
        container.style.backgroundImage = 'none';
    }
}
```

### [ ] Step 3: Locking Individual Elements (new)

```javascript
function setNodeLocked(nodeId, locked) {
    nodes.update({ id: nodeId, fixed: { x: locked, y: locked } });
}
```
`vis-network` supports per-node `fixed` properties independent of the global physics toggle — use this for pinning pitch markers or anchor nodes without disabling physics for everything else.

### [ ] Step 4: JSON Export & Import

```javascript
function exportGraph() {
    const payload = {
        version: 1,
        mode: currentMode, // 'systems' | 'tactics'
        nodes: nodes.get(),
        edges: edges.get(),
        zones: zones.get(),        // see Step 5
        isPhysicsEnabled: network.physics.options.enabled,
        backgroundUrl: currentBackgroundUrl || null
    };

    const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(payload));
    const a = document.createElement('a');
    a.setAttribute("href", dataStr);
    a.setAttribute("download", "graph_workspace.json");
    document.body.appendChild(a);
    a.click();
    a.remove();
}

function importGraph(jsonString) {
    const payload = JSON.parse(jsonString);

    nodes.clear();
    edges.clear();
    zones.clear();

    nodes.add(payload.nodes);
    edges.add(payload.edges);
    zones.add(payload.zones || []);

    setMode(payload.mode, payload.backgroundUrl);
}
```
Including a `version` field from day one avoids painful migration logic later if the schema changes.

**Also add a PNG/image export, not just JSON.** JSON export is great for re-editing but useless for sharing with someone who doesn't have the app open — and "send the coach a picture of the formation" or "paste this into a council report" are two of the most common real-world sharing actions for this tool. `vis-network` exposes the underlying `<canvas>` element, so a static snapshot is nearly free:

```javascript
function exportPng() {
    const canvas = document.querySelector('#network-canvas canvas');
    const link = document.createElement('a');
    link.download = 'graph_snapshot.png';
    link.href = canvas.toDataURL('image/png');
    link.click();
}
```
For Tactics Mode this needs to composite the SVG zone/arrow overlay (Step 5) on top of the canvas before export — draw both to an offscreen `<canvas>` and export that instead of the raw network canvas alone.

### [ ] Step 5: Zones & Annotation Layer (new)

For Tactics Mode's freehand regions (pressing traps, fielding arcs) and directional arrows that aren't simple graph edges, layer a lightweight SVG overlay on top of the `vis-network` canvas rather than forcing every annotation into the node/edge model:

```javascript
const zones = new vis.DataSet([]); // { id, points: [[x,y], ...], color, label }

function renderZoneOverlay(svgElement) {
    svgElement.innerHTML = '';
    zones.get().forEach(zone => {
        const poly = document.createElementNS('http://www.w3.org/2000/svg', 'polygon');
        poly.setAttribute('points', zone.points.map(p => p.join(',')).join(' '));
        poly.setAttribute('fill', zone.color || 'rgba(255,0,0,0.15)');
        poly.setAttribute('stroke', zone.color || 'red');
        svgElement.appendChild(poly);
    });
}
```
This overlay is synced to the network's viewport transform on every `zoom`/`dragging` event so zones stay pinned to their canvas coordinates.

### [ ] Step 4b: Bulk Import from CSV / XLSX / JSON (new)

Manual node-by-node creation doesn't scale for the Systems Mode audience — a policy analyst or ops team is more likely to already have their data in a spreadsheet than to want to click-build a 50-node graph by hand. This matters less for Tactics Mode (a handful of players) and more for Systems Mode (potentially hundreds of stakeholders/policies), so prioritize it alongside Phase 2, not as a later nice-to-have.

**Supported formats:**
- **JSON** — accept the app's own `{nodes, edges}` schema, and ideally also the common `node-link` JSON shape other tools export, so users aren't locked in.
- **CSV, edge-list style** — columns like `from, to, label`; nodes are auto-created the first time an ID appears. Lowest-friction option for non-technical users.
- **CSV, two-file style** — separate `nodes.csv` (id, label, color, group) and `edges.csv` (from, to, label) for richer node attributes.
- **XLSX** — same shape as two-file CSV, but as sheets within one workbook ("Nodes" + "Edges" tabs), parsed client-side via SheetJS. This is the most natural fit for how people actually keep this data in Excel/Google Sheets.

**Critical UX step — column mapping.** Real-world spreadsheets won't match your exact column names, so raw "upload and go" will fail constantly. After upload, show the detected columns and let the user map them to `id` / `label` / `from` / `to` / `color` / `group` before committing:

```javascript
// CSV parsing (PapaParse)
import Papa from 'papaparse';

function parseCsvForMapping(file, onParsed) {
    Papa.parse(file, {
        header: true,
        skipEmptyLines: true,
        complete: (results) => onParsed(results.meta.fields, results.data)
        // onParsed gives you the column headers + raw rows,
        // which drives the mapping UI before any nodes/edges are created
    });
}

// XLSX parsing (SheetJS)
import * as XLSX from 'xlsx';

function parseXlsxForMapping(file, onParsed) {
    const reader = new FileReader();
    reader.onload = (e) => {
        const workbook = XLSX.read(e.target.result, { type: 'binary' });
        const sheets = {};
        workbook.SheetNames.forEach(name => {
            sheets[name] = XLSX.utils.sheet_to_json(workbook.Sheets[name]);
        });
        onParsed(sheets); // e.g. { Nodes: [...], Edges: [...] }
    };
    reader.readAsBinaryString(file);
}

// Commit step, after the user has confirmed the column mapping
function commitImportedData(mappedNodes, mappedEdges) {
    const seenIds = new Set();
    const errors = [];

    mappedNodes.forEach(n => {
        if (seenIds.has(n.id)) errors.push(`Duplicate node id: ${n.id}`);
        seenIds.add(n.id);
    });
    mappedEdges.forEach(e => {
        if (!seenIds.has(e.from) || !seenIds.has(e.to)) {
            errors.push(`Edge references missing node: ${e.from} → ${e.to}`);
        }
    });

    if (errors.length) return { success: false, errors };

    pushHistory(); // so a bad import is one undo away
    nodes.add(mappedNodes);
    edges.add(mappedEdges);
    return { success: true };
}
```

**Validation before commit, not after:** catch duplicate IDs, edges referencing nodes that don't exist, and empty required fields, and surface them in the mapping UI rather than silently dropping bad rows or creating a broken graph.

**Ties into the existing performance ceiling (§2):** bulk import is the most likely path to hit the "thousands of nodes" physics limit, since it's a single action rather than the gradual buildup of manual editing. Worth adding a soft warning ("this will import 2,400 nodes — physics may be slow, continue?") rather than a hard cap, so large-but-valid imports (like a genuinely big stakeholder network) aren't blocked outright.

### [ ] Step 5b: Generate Pitch Backgrounds Programmatically (new — recommended over sourced images)

Rather than sourcing background images for the four sports (licensing overhead, inconsistent style, fixed aspect ratios), draw each pitch as parametric SVG using official regulation dimensions:

| Sport | Standard dimensions | Key markings |
|---|---|---|
| Soccer | 105m × 68m (FIFA) | Center circle, penalty boxes, goal areas |
| Rugby (Union) | 100m × 70m in-goal +22m each end | Try lines, 22m/10m/halfway lines |
| Hockey | 91.4m × 55m | 23m lines, shooting circles |
| Cricket | Oval, ~137–150m diameter (varies by ground) | Pitch strip, crease markings, boundary |

Benefits over sourced images:
- **Zero licensing risk** — you own every pixel.
- **Infinitely scalable** — no blurring at any zoom level, unlike raster images.
- **Themeable** — swap grass green for a dark-mode or print-friendly palette with a CSS variable, rather than needing a second image asset per theme.
- **Consistent line-weight/coordinate system** across all four sports, which also makes node-snapping-to-markings (e.g., "snap defender to the 22m line") straightforward since the markings are just SVG coordinates you already know.

If a hand-illustrated look is preferred over clean regulation-line diagrams, freesvg.org hosts CC0 (public domain) pitch diagrams for soccer that can be used as a starting reference without licensing concerns — but a generated parametric version remains the more maintainable long-term choice since it covers all four sports from one code path and never needs re-sourcing.

### [ ] Step 6: Undo/Redo Stack (new)

```javascript
const history = { past: [], future: [] };

function snapshot() {
    return JSON.stringify({ nodes: nodes.get(), edges: edges.get(), zones: zones.get() });
}

function pushHistory() {
    history.past.push(snapshot());
    history.future = [];
    if (history.past.length > 50) history.past.shift(); // cap memory use
}

function undo() {
    if (!history.past.length) return;
    history.future.push(snapshot());
    restoreSnapshot(history.past.pop());
}

function redo() {
    if (!history.future.length) return;
    history.past.push(snapshot());
    restoreSnapshot(history.future.pop());
}

function restoreSnapshot(snap) {
    const state = JSON.parse(snap);
    nodes.clear(); edges.clear(); zones.clear();
    nodes.add(state.nodes); edges.add(state.edges); zones.add(state.zones);
}
```
Call `pushHistory()` before any mutating action (add/remove/move-commit, not on every drag frame — snapshot on drag-end, not on every mousemove).

---

## 6. Execution Phases

- [x] 1. **Phase 0: De-risk.** Spike the annotation model (Step 0). Decide `vis-network`-only vs. hybrid SVG overlay before writing app scaffolding.
- [x] 2. **Phase 1: MVP.** `index.html` + CDN setup. Canvas rendering, Add Node/Edge, physics toggle, basic mode switch.
- [x] 3. **Phase 2: Persistence.** Export/import JSON (Step 4), PNG snapshot export, versioned schema, verify exact coordinate restoration with physics disabled.
- [x] 4. **Phase 2b: Bulk import.** CSV/XLSX/JSON upload with column-mapping UI and validation (Step 4b) — prioritize alongside Phase 2 since it's high-value for Systems Mode specifically.
- [x] 5. **Phase 3: Tactics-specific features.** Zone overlay (Step 5), locking (Step 3), background images for the four sports, arrow styling for movement.
- [x] 6. **Phase 4: Editing quality-of-life.** Undo/redo (Step 6), keyboard shortcuts, contextual side panel / bottom sheet editor.
- [x] 7. **Phase 5: Mobile optimization.** FAB-based Pan/Edit toggle, bottom sheets, inflated hitboxes, grid snapping.
- [ ] 8. **Phase 6 (stretch): Two landing pages.** Split Systems and Tactics into separate entry points/branding sharing the same engine and JSON schema, so each audience gets a purpose-fit first impression.

---

## 7. High-Value Use Case Extensions

The two original use cases (policy/systems mapping, sports tactics) sit at opposite ends of a spectrum: one is dynamic/exploratory, one is static/precise. The engine built for those two extremes happens to cover a wide middle ground cheaply. Worth considering, roughly in order of how well they fit the *existing* feature set with little extra work:

- [ ] 1. **Org charts & stakeholder mapping (consulting, HR, NGOs).** Almost identical to the policy use-case UI (physics-off or hierarchical layout, categorized node colors) but a much bigger addressable market — every consultant, HR team, and NGO program manager has drawn a stakeholder map in PowerPoint at some point. `vis-network` has a built-in hierarchical layout mode, so this may need close to zero new engineering, just a different landing page and example templates (Phase 6 territory).
- [ ] 2. **Mind mapping / brainstorming.** Physics-on, free-form, no background — essentially Systems Mode with a lighter visual style. High-volume, low-stakes use case that's good for organic discovery/virality (people screenshot and share mind maps constantly), and the PNG export feature (Step 4) is what makes this shareable to people who'll never open the app.
- [ ] 3. **Incident postmortems / root-cause / dependency mapping (software & ops teams).** Same shape as the policy use case — "click the failed service, see what it's connected to." Technical audience, likely to want the JSON export/import to integrate with their own tooling later, and tolerant of a slightly rougher UI than a public-facing tool would need.
- [ ] 4. **Curriculum & prerequisite mapping (education).** Physics-off, hierarchical/layered layout (topic A must be learned before topic B). Same core engine as org charts. Smaller market but a genuinely underserved one — most "prerequisite map" tools are static images someone made once in a diagramming tool and never updates.
- [ ] 5. **Family trees / genealogy.** Worth naming as a *contrast* case: family trees have strict layout conventions (generations as rows) that fight against both physics-on and free-drag layouts. Only pursue this if there's specific demand — it would need its own layout algorithm, not just a mode toggle.

**Recommendation:** don't build for all five. Ship Systems + Tactics first (as planned), watch which one gets organic traction or requests, and treat #1 (org charts) as the natural next add — it's the closest to free, reusing the Systems Mode UI almost as-is.

---

## 8. Resolved & Remaining Unknowns

Research findings for the open questions from the previous version of this plan:

### Resolved
- **Can `vis-network` combine dashed lines *and* arrowheads on the same edge?** Yes. This was an open GitHub issue in the library's early years, but current `vis-network` supports `dashes: true` and `arrows: { to: { enabled: true } }` as independent edge options, so a dashed arrow (useful for "off-ball run" in Tactics Mode) is a standard combination, not a workaround. Edges also support a `smooth.type` of `curvedCW`/`curvedCCW` for curved paths. This substantially de-risks Phase 0 — the annotation model may not need a separate SVG overlay for arrows specifically, only for filled zones/regions, which vis-network has no native concept of.
- **Is `vis-network` the right engine given the editing-heavy, drag-and-drop use case?** Current comparisons (mid-2026) consistently position `vis-network` as the better fit for *interactive, editable* diagrams with physics and drag/drop, while `Cytoscape.js` is stronger for graph *analysis* (algorithms, large-scale layout, bioinformatics-style workloads) and `Sigma.js` for large-scale WebGL rendering (100k+ nodes). Since this app is about people building and editing graphs by hand rather than analyzing pre-existing large datasets, `vis-network` remains the right choice — this closes out the Phase 0 "should we switch engines" question in favor of the original plan.
- **Where do sports pitch backgrounds come from without licensing risk?** CC0 (public domain) SVG pitch diagrams exist (e.g., freesvg.org) if a sourced-image approach is preferred, but the stronger recommendation is to generate pitches parametrically from regulation dimensions (see §5, Step 5b) — this sidesteps licensing entirely and covers all four sports from one code path.

### Still open — needs a hands-on spike, not just research
- [ ] **Zone/region drawing UX.** No amount of library research substitutes for actually prototyping freehand or click-to-place polygon zones on top of the vis-network canvas and seeing if the coordinate-syncing (Step 5) feels smooth during pan/zoom. This is the one piece of Phase 0 that still needs real prototyping, not just documentation review.
- [ ] **Mobile performance with the SVG overlay.** Compositing a live-syncing SVG layer over a canvas-rendered network, on a mid-range phone, during physics simulation — no published benchmark answers this for your specific case. Test on real devices in Phase 0/1, not just desktop Chrome.
- [ ] **Exact regulation dimensions per sport variant.** Cricket grounds vary in size (no fixed boundary distance), and rugby/hockey have minor variations by governing body (e.g., World Rugby vs. school/club-level pitches). Pick one authoritative source per sport (e.g., World Rugby's official law book, ICC ground regulations) before hardcoding dimensions, rather than approximating from general web images.

---

## 9. Open Risks & Decisions to Revisit

| Risk | Why it matters | Resolve by |
|---|---|---|
| [ ] Zone/region drawing UX on top of vis-network is unproven | Could still force a hybrid rendering approach even though the arrow/dash question is resolved | Phase 0 hands-on spike |
| [ ] Two very different audiences (policy vs. sports) under one brand | Diluted positioning, confusing landing page | Phase 6 decision |
| [ ] No collaboration/multi-user support | Fine for v1, but a likely feature request from teams | Explicitly out of scope — state this in the README |
| [ ] SVG overlay + canvas compositing performance on mobile | Could make Tactics Mode feel sluggish on phones specifically | Real-device testing in Phase 0/1 |
| [ ] localStorage autosave only, no cross-device sync | Users must remember to export JSON | Consider a "you have unsaved changes" banner reminding users to export |
| [ ] Scope creep across 5 possible use cases (§7) | Spreads effort thin before any one use case gets traction | Ship Systems + Tactics only for v1; treat org charts as the sole "next" candidate |
| [ ] Bulk import (CSV/XLSX) is the most likely path to hit the physics performance ceiling | A single large import can degrade the experience immediately, unlike gradual manual editing | Soft warning at import time above a node-count threshold, rather than a hard cap (§5, Step 4b) |
| [ ] Column-mapping UX for imports is unproven | Real-world spreadsheets vary enough that a naive "auto-detect columns" approach may frustrate users | Prototype with a few real sample spreadsheets (not synthetic test data) during Phase 2b |
