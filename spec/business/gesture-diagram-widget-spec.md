# Gesture-First Diagram Widget — Product Specification Draft (v0.4)

> **Purpose of this document.** Input for an AI-driven pipeline that will produce a detailed specification, requirements and backlog. It consolidates the original idea, adds missing requirements, proposes architectural defaults, and lists open questions.
> **Conventions.** Requirement IDs are `FR-*` (functional) and `NFR-*` (non-functional). Priority uses MoSCoW: **M**ust, **S**hould, **C**ould, **W**on't (now). Items marked **[ADDED]** were not in the original description. Items marked **[DECISION]** are proposed defaults that should be confirmed (see section 16).

---

## 0. Resolved Decisions Log (v0.4) — supersedes earlier proposals where they conflict

| # | Topic | Decision | Consequences for the spec |
|---|---|---|---|
| D1 | Audience | **Primary:** developers/architects documenting architecture. **Secondary:** whiteboard-style brainstorming. | Add a **Whiteboard profile** (sticky notes, simple nodes + connectors, free text, quick styling). First polish priority is architecture diagrams (clean alignment, containers, icons, auto-layout); the whiteboard mode favors speed, loose layout, and low ceremony. Both run on the same core. |
| D2 | License / model | **Open source, Apache-2.0** (recommended in v0.4, see D20 for rationale and the alternative), commercial use allowed. | Only permissive dependencies (MIT/BSD/Apache-2.0/ISC). Avoid GPL/AGPL; treat LGPL/EPL/MPL (e.g. the ELK layout engine is EPL-2.0) as *needs legal review*, with a permissive alternative (e.g. dagre-style layout) as default. Icon sets (AWS, Azure, GCP) are **trademark/terms-restricted** and must not be bundled by default; load them as optional, user-supplied or separately licensed library packs. Add DCO/CLA decision (D20), NOTICE file, SPDX headers, third-party license audit in CI. |
| D3 | Offline / local-first | **Plugin/adapter architecture.** A deployment picks its adapters: persistence (browser storage / file system / Google Drive (to be researched) / host system), collaboration, history, assets. A **local, in-browser "backend"** is part of the product. Larger systems (corporate KB portals) plug in a server adapter that owns the shared model and orchestrates users. | Removes the CRDT-vs-server dilemma as a *product-level* choice: the **op-based model + adapter interface** is the stable core; the sync strategy lives behind the Collaboration Adapter. Local adapter = single-user, no conflict resolution. Server adapter = server-sequenced ops (8.1). Offline queue becomes a feature of the sync adapter rather than a core requirement. A pure P2P/CRDT adapter remains possible later. Google Drive is a **candidate adapter to research** (OAuth, permissions, review requirements); not committed. |
| D4 | Backend scope | **v1: no backend.** A **placeholder/demo app** (single user) to run the widget and exercise draw/load/save. **Later:** a simple **reference backend** (server objects + endpoints) with its own plugin to demonstrate server-side integration and multi-user. | Add deliverable **DEMO-APP** to Phase 1 (see 15). Reference backend moves to Phase 3. The widget never requires a backend. |
| D5 | Tech stack | **TypeScript.** Focus on **large-screen devices**; must work in desktop browsers and, "if it comes cheap", mobile Chrome/Safari; **WebView support only if relatively easy.** | Mobile/phone-optimized UX is a **Could** (tablet/iPad with pen is the realistic touch target). Touch gizmos (FR-MAN-2) stay Must for large touch screens/tablets; phone layouts, small-screen palette variants are **Won't (now)**. No native shells in v1. Framework choice: see D18. |
| D6 | Rendering | **No special permissions/setup; fast for many objects; support viewport virtualization ("virtual window").** | **Canvas 2D** as the default renderer (no permissions, broad support, adequate for the target counts) with **viewport culling, spatial index (quadtree/R-tree), tiled/cached layers, level-of-detail**, and offscreen/worker rendering where available. WebGL is *not required* (and is optional later if benchmarks demand). DOM/SVG overlay only for gizmos and text editing. Rendering sits behind a **Renderer interface** so it can be swapped. |
| D7 | Text format | Custom native format, **YAML-based** (compact, readable, easy to edit), interoperable with popular formats through converter plugins. | See "Format clarification" and section 6.3. YAML replaces JSON as the *stored/authored* form; JSON may still be used internally as a wire/API encoding of the same model. |
| D8 | Hidden-layer connectors | **Accepted:** connector hides when either endpoint is hidden; a visible "hidden connections" indicator is shown. | Closes Q8; FR-CON-5 / FR-LAY-5 updated. |
| D9 | Groups across layers | **Confirmed:** a group lives in exactly one layer. | Closes Q9; FR-GRP-2 final. |
| D10 | First profiles | **Simple architecture diagrams first (AWS, other popular architecture types)**; no sample diagrams yet. | Profile order: Generic/Flowchart → **Architecture (AWS)** → Whiteboard → C4/UML. A **sample corpus** (draw.io AWS diagrams, Mermaid flowcharts, whiteboards) must be assembled as a task; until real samples exist, use public/permissively licensed examples and self-authored ones. |
| D11 | draw.io fidelity | **Start with basics**: common shapes, edges, text, styles, layers, groups. Import and export are **plugins**, configurable per embedding. Broader fidelity (stencils, pages, advanced styles) grows over time. | Import/export plugin interface with a declared capability/loss matrix. Baseline corpus is a task (no samples yet). Full-stencil and multi-page support moved to later phases. |
| D12 | Mermaid scope | Import: **flowchart + sequence first**. Export: **nice-to-have (Could)**, researched later. | Mermaid export removed from Phase 4 commitments; stays a research item. |
| D13 | LLM for guessing | **Context/guess engine is a plugin.** v1: **heuristics only, no LLM.** Later: optional LLM-aware plugin. | FR-CTX-5 becomes a future plugin; no external calls from core. Engine interface designed so an LLM plugin can be added without core changes. |
| D14 | Auth / permissions | **Out of scope.** The widget does not implement roles, per-layer permissions or anonymous-viewer rules; the host system handles access control. | Remove per-layer permissions and permission-filtered history. Keep: element **lock** (an editing aid, not security), read-only/viewer mode as a configuration flag, and identity (id, name, color) *passed in by the host* for attribution/presence later. |
| D15 | Scale | **One document = one diagram.** Size is adapted to the viewport on the client (virtualized rendering). Documents may carry optional **base/desired size properties** that the client may honor. No server scale targets now. | Drop concurrent-editor and document-count targets from the spec until the reference backend phase. Add document properties (see FR-DOC-1). |
| D16 | Compliance | **Out of scope** (WCAG/GDPR/SOC2/residency belong to the host system). | Keep only *technical* hygiene (input sanitization, no external calls by default). Keep basic keyboard/accessibility as a quality goal, not a compliance target. |
| D17 | Name / branding / theming | **Working name: `gesture-diagram`** (kept for now; npm scope `@gesture-diagram/*`). Theming via CSS custom properties; moderate depth. | Brand can change later; keep the name in one place (package metadata) to make renaming cheap. |
| D18 | Framework strategy | **Framework-agnostic headless TypeScript core + framework-based UI chrome + thin wrappers.** Not "pure TS everywhere" and not "a React app". No jQuery or other legacy libraries. | See section 12.1. Canvas hot path, model, ops, gestures and import/export carry **zero UI-framework dependency** (also runs in Node/headless). The surrounding UI (palette, panels, outline, dialogs) is built with a component framework and off-the-shelf headless primitives rather than hand-rolled DOM code. React hosts get a first-class `<GestureDiagram/>` wrapper; non-React hosts use the Web Component. |
| D19 | Delivery strategy | **Always-alive walking skeleton.** A runnable demo/container page exists from iteration 0 and is deployed on every merge; every iteration extends the same page and never breaks it. | See section 15.1: Definition of Done requires the demo to work, saved documents keep opening (schema migrations), and an automated smoke test passes in CI. |
| D20 | Project license | **Apache-2.0** recommended; MIT (or dual "MIT OR Apache-2.0") is the simplest alternative. Not legal advice. | See section 15.2 for the comparison. Decide DCO vs. CLA before the first external contribution. |

**Naming (D17).** Working-title suggestions, none checked for trademark/package-name conflicts: **Gestalt** (gesture + whole-shape perception; also a library-friendly name), **Scrawl** (quick, informal, whiteboard-friendly), **Sketchmark**, **Inkflow**, **Outline Pen**. Recommended for now: use the neutral codename **`gesture-diagram`** (already used for the `<gesture-diagram>` element) and pick a brand later; before choosing, check npm, GitHub, and trademark databases.

**Format clarification (D7).** Question 7 only asks *how a diagram is stored as a file*, not about the editor UX. The proposed answer, stated plainly:
- The **internal model** is the source of truth. Its canonical stored/authored form is **YAML** (decided: more compact and readable than JSON), with a compact one-element-per-line style for clean diffs in repositories/wikis and easy hand or LLM authoring. JSON remains only as an optional wire/API encoding of the identical model, so there is no second format to maintain by hand.
- Interoperability is done by **converters (plugins)** for draw.io, Mermaid, SVG etc.; the native format does not need to *be* Mermaid.
- **Defaults assumed unless you object:** hand-writing and LLM generation are *nice-to-have (Should)*, not a driver; LLMs can already produce Mermaid, so **Mermaid import** gives them a ready path into the tool. The text form is chosen for readable diffs first.

---

## 1. Vision and Positioning

A web-embeddable diagram editor where **drawing replaces tool selection**. The user sketches a rough gesture, the system recognizes the intent, places a clean shape, and offers context-aware choices immediately afterwards. Connecting, styling, grouping and layering follow the same principle: act first, refine second.

**Problem.** Tools such as draw.io and Visio require *select tool → place → configure*. Connecting shapes and restyling are slow, and touch support is usually an afterthought.

**Differentiators**
1. Gesture recognition and snapping into clean shapes.
2. Context-aware prediction (next shape, style, connector, library).
3. Direct-manipulation gizmos instead of toolbars and property panels (mouse, touch and pen are all first-class).
4. A collaboration-ready model with zero perceived latency and a real change history.
5. Open, text-based format with draw.io and Mermaid interoperability.
6. Pluggable **diagram profiles** (UML, C4, AWS, flowchart, ...) that make context guessing domain-aware.

**Target users (confirmed, D1).** Primary: developers and architects documenting systems. Secondary: whiteboard-style brainstorming (sticky notes, simple nodes and connectors). Embedding hosts: portals, wikis, knowledge bases. Business-user flowcharting is a side benefit, not a driver.

**Non-goals (initial).** Full vector illustration (Illustrator/Inkscape scope), pixel editing, desktop-publishing precision, video/audio conferencing.

---

## 2. Glossary

| Term | Meaning |
|---|---|
| **Diagram** | A document: layers, nodes, edges, metadata. |
| **Node / Shape** | A visual element with geometry, style, text, and optional children. |
| **Edge / Connector** | A link between two nodes (or ports/points) with routing, style, labels. |
| **Port / Anchor** | A named connection point on a node. |
| **Layer** | An ordered, toggleable, lockable container of top-level elements. |
| **Group** | A purely organizational set of elements, moved/selected together. No layout semantics. |
| **Container (parent shape)** | A shape that owns children and may auto-size and auto-layout around them. |
| **Gizmo** | A contextual handle/overlay for manipulating a shape (resize, rotate, connect, ...). |
| **Palette (contextual)** | The pop-up of ranked suggestions shown after an action. |
| **Library** | A named collection of shape definitions, icons and stencils. |
| **Diagram profile** | A pluggable package defining a diagram type: vocabulary, rules, libraries, prediction hints, layout, import/export mappers. |
| **Op / Change** | An atomic model mutation, the unit of sync. |
| **Commit** | A user-facing, named group of ops with authorship (see section 9). |
| **Ink** | Raw stroke data captured before recognition. |

---

## 3. Scope Overview (Epics)

| # | Epic | Priority |
|---|---|---|
| E1 | Canvas, rendering, viewport | M |
| E2 | Gesture recognition and shape placement | M |
| E3 | Contextual palette and context engine | M |
| E4 | Shape manipulation (gizmos), text editing | M |
| E5 | Connectors | M |
| E6 | Layers, groups, containers, hierarchies | M |
| E7 | Libraries, icons, images | S |
| E8 | Document model and text format | M |
| E9 | Import / export (draw.io, Mermaid, SVG, PNG, PDF) | M/S |
| E10 | Real-time collaboration | S (M if SaaS target) |
| E11 | Versioning and history | S |
| E12 | Diagram profiles (plugin system) | S |
| E13 | Embedding and host integration API | M |
| E14 | Accessibility, i18n, input devices | M |
| E15 | Input security and sanitization (permissions/compliance out of scope: D14, D16) | M |
| E16 | Quality: performance, testing, telemetry | M |

---

## 4. Functional Requirements

### 4.1 Canvas and Viewport (E1)

- **FR-CAN-1 (M)** Infinite (or very large) canvas with pan and zoom (wheel/pinch/trackpad/keyboard). Zoom range at least 5%–800%.
- **FR-CAN-2 (M)** Rendering must stay interactive (≥ 60 fps while dragging) with at least 5,000 elements. **[DECIDED, D6]** Canvas 2D main surface with viewport culling, spatial index, cached tiles and level-of-detail ("virtual window"), plus an SVG/DOM overlay for gizmos and text editing; WebGL only if benchmarks demand. Renderer is behind an interface.
- **FR-CAN-3 (S) [ADDED]** Grid, snapping to grid, snapping to other shapes' edges/centers, smart guides with distance hints.
- **FR-CAN-4 (S) [ADDED]** Minimap and "fit to content / fit selection".
- **FR-CAN-5 (M) [ADDED]** Undo/redo (see 4.14 for multi-user semantics).
- **FR-CAN-6 (S) [ADDED]** Rulers, optional page/frame boundaries for printing and export.
- **FR-CAN-7 (M) [ADDED]** Selection model: click, shift-click, marquee, lasso, select-by-type/style/layer, select all in group.
- **FR-DOC-1 (S) [ADDED, D15]** Document properties: title, optional **base/desired size** and background, default style/theme, active profile. The canvas is unbounded and adapts to the viewport; the size hint is advisory (initial view, export framing) and a host or client may ignore it.
- **FR-CAN-8 (M) [ADDED]** Clipboard: copy/cut/paste/duplicate within and across diagrams; paste from draw.io clipboard payloads, plain text (creates a labeled shape), images.

### 4.2 Gesture Recognition and Shape Placement (E2)

- **FR-GES-1 (M)** Drawing a stroke on empty canvas is interpreted, not stored as ink. Initial recognized set: rectangle, rounded rectangle, ellipse/circle, diamond, triangle, line/arrow (→ connector or free line), cylinder, cloud, actor (stick figure), text (scribble or tap).
- **FR-GES-2 (M)** On recognition, a clean shape is placed with size and position fitted to the stroke's bounding geometry (including rotation where applicable, e.g. rotated rectangle or diamond).
- **FR-GES-3 (M)** Recognition happens on **stroke end** with a short timeout for multi-stroke shapes (e.g. a cross-hatched box or two-stroke shapes). Show a transient preview (ghost) of the interpretation while drawing. **[DECISION]** 150–300 ms end-of-stroke debounce, configurable.
- **FR-GES-4 (M)** Ambiguity handling: if confidence is below a threshold or top-2 candidates are close, place the best guess and show the alternatives in the palette. The user can always switch shape type afterwards.
- **FR-GES-5 (M) [ADDED]** **Escape hatch for freehand.** Users must be able to keep a raw stroke (e.g. hold modifier, a "freehand" toggle, or an immediate "keep as ink" option in the palette). Ink is stored as a stroke element on an ink-capable layer.
- **FR-GES-6 (M) [ADDED]** **Gesture disambiguation by context.** A stroke may mean: create a shape, move or lasso an existing shape, draw a connector, scribble-to-delete, write text, or ink. Rules:
  - Starting on a shape's connection gizmo → connector.
  - Starting inside a shape with the select/move affordance → move (not draw).
  - Starting on empty canvas → create.
  - Starting inside a container/frame on empty interior area → create as child (see FR-HIE-3).
- **FR-GES-7 (S) [ADDED]** Command gestures: scribble-over-shape to delete, circle-around-multiple to select (lasso), two-finger tap for undo, three-finger tap for redo (touch), drag-a-line-through-a-connector to cut/disconnect. Each must be undoable and discoverable (see FR-ONB-1).
- **FR-GES-8 (S) [ADDED]** (large-screen touch/tablet; phones out of scope per D5) Pen vs finger vs mouse: when a stylus is detected, finger = pan/zoom, pen = draw (palm rejection). Without a stylus: touch single-finger on empty = draw, two fingers = pan/zoom (configurable).
- **FR-GES-9 (S)** Recognizer is pluggable (interface: `strokes[] + context → ranked candidates with confidence`). Profiles can register extra gestures (e.g. UML "lifeline", C4 "person").
- **FR-GES-10 (C) [ADDED]** User-trainable or user-defined gestures (map custom stroke → shape from a library).
- **FR-GES-11 (M) [ADDED]** Recognition accuracy target: ≥ 95% top-1 on the supported set on a project-owned test corpus (mouse, touch, pen); latency < 50 ms from stroke end to ghost/placement on mid-range hardware.

> **Implementation hints for the downstream spec.** Candidate approaches: geometric heuristics (stroke simplification such as Ramer–Douglas–Peucker, corner detection, ellipse/polygon fitting, closedness test), template matchers (`$1`/`$P`/`$N` family), a small on-device ML classifier, or a hybrid (heuristics first, ML for ties). All must run client-side without network round trips. A labeled gesture corpus is a project deliverable.

### 4.3 Contextual Palette and Context Engine (E3)

- **FR-PAL-1 (M)** After a shape is placed (or selected), a **contextual palette** opens near the shape with ranked quick actions: change shape type, fill/stroke color, font, icon/library items, text alignment, connector styles. Dismisses on click-away or Escape; never blocks drawing.
- **FR-PAL-2 (M)** Palette content and order are driven by the **context engine** (FR-CTX-*). First items must be the most likely next choices.
- **FR-PAL-3 (M)** Any shape can be converted into any other shape type while keeping text, style where applicable, connections, children and position/size (best-effort mapping of ports).
- **FR-PAL-4 (S)** Palette is searchable (type-to-filter) and keyboard navigable.
- **FR-PAL-5 (S)** Palette adapts to device: radial/compact on touch, anchored popover on desktop.
- **FR-CTX-1 (M)** The engine keeps a **context state**: last created shape, its type/icon/library/style/font/size, current layer, selection, current profile, recent connector style, and recent palette choices.
- **FR-CTX-2 (M)** **Style inheritance for new shapes.** New shapes inherit from the last created shape of the same *kind* by default (the user's example), with a visible, easily togglable behavior ("sticky style" on/off).
- **FR-CTX-3 (S) [ADDED]** **Next-shape prediction** (ranking): combine (a) recency/frequency in this diagram, (b) sequence patterns (e.g. after "Decision" often comes "Process"), (c) spatial/relational context (shape near a container, at the end of a flow, under a lifeline), (d) the active profile's grammar/rules, (e) user history across diagrams (opt-in).
- **FR-CTX-4 (S) [ADDED]** Context engine API is pluggable: `rank(candidates, contextState) → scored list`, with explainability data (why was this suggested) for debugging and tuning.
- **FR-CTX-5 (C) [ADDED, D13]** The guess engine is a **plugin** (`rank(candidates, context) → scored list`). The default plugin is **heuristic-only (no LLM, no network)**. An LLM-aware plugin may be added later; it must be opt-in and have the heuristic fallback.
- **FR-CTX-6 (S) [ADDED]** **Label-driven guessing**: typing a label can re-rank or auto-suggest an icon/shape (typing "PostgreSQL" suggests a cylinder + DB icon, "User" suggests an actor).

### 4.4 Shape Manipulation and Gizmos (E4)

- **FR-MAN-1 (M)** Unlocked shapes are interactive. Hover (mouse) shows context-specific gizmos depending on **hover zone**: corners/edges → resize, outside corner → rotate, center → move, edge midpoints → connect, label zone → move text, a dedicated handle → shape-specific parameters (corner radius, arrow tip, etc.).
- **FR-MAN-2 (M)** **Touch alternative.** Tap selects and reveals the gizmo set sized for touch (≥ 44 px targets); tap on a specific region (corner, edge, center) or a long-press enables the relevant mode; double-tap edits text. Gizmos must never depend on hover.
- **FR-MAN-3 (M)** Resize (free, proportional with modifier/two-finger), rotate (with snap angles), scale, flip, move with nudge by arrow keys.
- **FR-MAN-4 (M)** Lock/unlock per element, group, and layer. Locked elements are not hit-testable for manipulation (but may remain connectable if configured).
- **FR-MAN-5 (M)** **Inline text editing**: click (or tap) inside the shape and type immediately; no separate text tool. Horizontal and vertical text alignment decided by *where the user clicks inside the shape* (e.g. click near the top-left → top-left aligned) and adjustable afterwards via palette/gizmo.
- **FR-MAN-6 (S) [ADDED]** Text capabilities: rich text (bold/italic/underline/links, lists), auto-fit (shrink text vs grow shape), wrapping, overflow modes, multi-line, text on connectors, text in any script (RTL, CJK, emoji), font fallback and web-font loading.
- **FR-MAN-7 (S) [ADDED]** Multi-selection manipulation: align, distribute, equalize size, match style (format painter), arrange z-order, flip, rotate as a group.
- **FR-MAN-8 (M) [ADDED]** Keyboard operability for every action (see 4.13).
- **FR-MAN-9 (S) [ADDED]** Custom shape parameters (e.g. rounded corners, cylinder depth) exposed as gizmo handles defined in the shape definition.

### 4.5 Connectors (E5)

- **FR-CON-1 (M)** Hovering/selecting a shape shows **connection gizmos** at ports. Dragging from a gizmo starts a connector; **all valid targets are highlighted** during drag.
- **FR-CON-2 (M)** Dropping on a target shape connects to the nearest/best port (or "floating" to the nearest edge point, configurable). Dropping on empty canvas creates a dangling connector and offers the palette to create a new target shape immediately ("drag out to create the next node" — high-value flow).
- **FR-CON-3 (M)** After connection, the palette opens to configure the connector: line style (straight, orthogonal, curved), arrowheads, dash, color, label.
- **FR-CON-4 (M)** Connector routing: straight, orthogonal (with obstacle avoidance), curved, with manual waypoints and auto re-routing when connected shapes move.
- **FR-CON-5 (M)** Connectors can attach to shapes, ports, other connectors (midpoint), and to elements in **other layers**. **Decided (D8):** a connector is hidden when either endpoint is hidden, and a visible "hidden connections" indicator is shown.
- **FR-CON-6 (S) [ADDED]** Connectors attach to containers and to children inside collapsed containers (re-routed to the container border).
- **FR-CON-7 (S) [ADDED]** Semantic edge types and constraints from profiles (e.g. UML association vs. inheritance, sequence-diagram message vs. return). Profile can reject or warn on invalid connections.
- **FR-CON-8 (S) [ADDED]** Edge labels, multiple labels (start/middle/end), cardinalities.
- **FR-CON-9 (C) [ADDED]** Parallel-edge bundling, jump-over/line-hop at crossings, edge bundling for dense graphs.

### 4.6 Layers (E6)

- **FR-LAY-1 (M)** Layers like Photoshop/GIMP: create, rename, reorder, delete, merge, duplicate; toggle **visibility**, **lock**, and **opacity**; assign color tag.
- **FR-LAY-2 (M)** Move shapes between layers by drag in layer panel, "send to layer" command, or palette action.
- **FR-LAY-3 (M)** Connections across layers remain valid; moving shapes between layers must never break connectors.
- **FR-LAY-4 (S) [ADDED]** Layer behaviors: "dim others" (focus), "solo" a layer, per-layer snapping, per-layer export inclusion, per-layer default style/profile (e.g. an "AWS" layer and a "notes" layer).
- **FR-LAY-5 (S) [ADDED]** Hiding a layer: define selection/hit-testing behavior (hidden = not selectable), connector rendering to hidden endpoints (see Q8), and what exports include.
- **FR-LAY-6 (W) [ADDED, D14]** ~~Layer-level permissions~~ — out of scope; access control belongs to the host. Layer **lock** remains an editing aid only.

### 4.7 Groups, Containers and Hierarchies (E6)

- **FR-GRP-1 (M)** Group/ungroup; groups can be nested; moving/resizing/rotating a group affects all members; enter-group editing (double-click) for editing members.
- **FR-GRP-2 (M)** Groups can span or live inside a single layer. **[DECIDED, D9]** A group belongs to exactly one layer; members inherit it. Cross-layer grouping is **not** supported, but cross-layer *relations* are via connectors/links.
- **FR-HIE-1 (M)** **Container shapes** own child shapes (a parent-child tree separate from group membership). Moving the parent moves children; deleting the parent asks what to do with children (delete/release).
- **FR-HIE-2 (M)** Containers support **auto-size** around children (padding, min size) and optional **auto-layout** (row/column/grid/flow/tree, alignment, spacing, wrap), similar to flexbox/auto-layout in design tools.
- **FR-HIE-3 (M)** Dragging a shape into a container (or drawing inside one) re-parents it; dragging out releases it. Drop target is visibly highlighted.
- **FR-HIE-4 (S)** Collapse/expand containers; collapsed state keeps connections routed to the container.
- **FR-HIE-5 (S) [ADDED]** **Semantic hierarchy relations** distinct from visual containment (e.g. org-chart "reports to", C4 "container within system", tree/mind-map). A hierarchy relation may be rendered either as nesting, or as edges, or both. **[DECISION]** Model relations as typed edges; containment as `parentId`.
- **FR-TOL-1 (M)** **Organization tools:** outline/tree view of layers → groups → shapes, drag-drop to reparent/regroup/move between layers, multi-select in outline, rename, search, filter by type/style, lock/hide from outline.
- **FR-TOL-2 (M)** Align/distribute/space, "tidy up", "distribute along path", "match size".
- **FR-TOL-3 (S) [ADDED]** Auto-layout algorithms on selection or whole diagram (layered/DAG, tree, force, radial, grid, orthogonal) — non-destructive "apply layout" with undo; **layout locking** for hand-placed elements.

### 4.8 Libraries, Icons and Images (E7)

- **FR-LIB-1 (S)** Libraries of shapes/stencils/icons; the active libraries are **context-dependent**; new diagrams start with a minimal default set.
- **FR-LIB-2 (S)** Load libraries after diagram creation, from the host, URL, or file. Library definition format must be documented and versioned (shape geometry, ports, text area, parameters, default style, tags/keywords for search and context ranking).
- **FR-LIB-3 (S)** Support user-crafted **custom libraries** (authoring: draw a shape → "save to library", import SVG as shape with defined ports and text area).
- **FR-LIB-4 (M) [ADDED]** Import compatibility goal: draw.io stencil/libraries (`mxlibrary`) and common icon sets (e.g. SVG icon packs; AWS/Azure/GCP/C4/UML sets subject to licensing).
- **FR-IMG-1 (S)** Load images/icons from **external sources**: file upload, drag & drop, paste, URL, host-provided asset provider. Reuse via "my assets"/recent.
- **FR-IMG-2 (M) [ADDED]** Store assets by content hash (dedupe), keep them out of the document body (referenced by ID/URL), support embed vs link modes and sanitization of SVG (see 8).
- **FR-IMG-3 (S) [ADDED]** Image as shape fill, as a standalone node, or as a icon inside a node; crop/mask, tinting for monochrome SVG icons.
- **FR-LIB-5 (C) [ADDED]** Library marketplace/registry interface for hosts; licensing metadata per library.

### 4.9 Document Model and Text Format (E8)

See section 6.

### 4.10 Import and Export (E9)

See section 7.

### 4.11 Collaboration (E10) and Versioning (E11)

See sections 8 and 9.

### 4.12 Diagram Profiles / Diagram Types (E12)

See section 10.

### 4.13 Embedding and Integration (E13)

- **FR-EMB-1 (M)** Distributed as (a) a headless TypeScript core package, (b) a **framework-agnostic Web Component** `<gesture-diagram>` (bundles its own UI chrome, shadow-DOM isolated, works in any page), and (c) a first-class **React wrapper** (`<GestureDiagram/>` with props, ref/imperative handle and hooks) that is the *primary integration path* for React portals; Vue/Angular wrappers are Could. **[DECIDED, D18]**
- **FR-EMB-2 (M)** Embeddable modes: full editor, compact editor, read-only viewer, headless (render/convert without UI).
- **FR-EMB-3 (M)** Host API: load/save document (YAML text format; JSON as wire encoding), events (change, selection, commit, error), commands (zoom, select, export), configuration (enabled tools, libraries, profile, theme, locale, read-only flag), and **extension points** (asset provider, library provider, persistence adapter, collaboration adapter, auth/identity provider, context-engine hook).
- **FR-EMB-4 (S)** Optional sandboxed **iframe** embedding with a `postMessage` protocol for hosts that need isolation (CSP, third-party content).
- **FR-EMB-5 (M)** Theming via CSS custom properties; dark mode; shadow-DOM style isolation; host design-token mapping.
- **FR-EMB-6 (S) [ADDED]** Server-side rendering/export service (headless) for thumbnails, PDF generation, CI doc builds. **[DECISION]** Same core library runs in Node.
- **FR-EMB-7 (M) [ADDED]** Persistence is **adapter-based** (D3/D4): the widget never requires a backend. Built-in adapters: in-memory, **browser storage (IndexedDB/localStorage)**, **file open/save** (File System Access API where available, download/upload fallback). Candidate adapter to research: Google Drive. Host systems supply their own adapter.
- **FR-EMB-8 (M) [ADDED, D3]** **Plugin/adapter registry and configuration:** a deployment declares which adapters/plugins are active (persistence, collaboration, history, assets, libraries, profiles, importers/exporters, context engine), with a typed, versioned interface for each and a documented lifecycle (init, capabilities, dispose).
- **FR-DEMO-1 (M) [ADDED, D4]** **Demo/placeholder app** (single user): a static web page that hosts the widget and demonstrates draw, save, load, import/export, adapter switching (memory / browser storage / file), profile selection, and theme. **[DECISION]** Plain web app first (also deployable as static site / GitHub Pages); an Electron/Tauri wrapper is optional and *not* needed for v1 because the File System Access API plus download/upload covers file use.

### 4.14 Input, Accessibility, i18n, Onboarding (E14)

- **FR-A11Y-1 (S) [ADDED, D16]** Keyboard operation (create via palette/search, move/resize with arrows, connect via keyboard, navigate shapes in logical order), screen-reader semantics (diagram structure announced, shape labels, relations), high-contrast and reduced-motion modes. Target WCAG 2.2 AA for UI chrome; document the limits for the canvas.
- **FR-A11Y-2 (S) [ADDED]** Text alternative / structured outline export of diagram for assistive tech.
- **FR-I18N-1 (M) [ADDED]** Localizable UI strings, RTL layout, locale-aware number/date, IME composition support in text editing.
- **FR-ONB-1 (S) [ADDED]** Discoverability of gestures: interactive tutorial, gesture cheat-sheet overlay, subtle hints, undo always available after a recognition surprise.
- **FR-UND-1 (M) [ADDED]** **Undo/redo semantics:** local undo stack per user that only undoes *that user's* ops (inverse ops applied on top of the shared state); conflicts degrade gracefully (e.g. element deleted by someone else → undo skipped with notice).

---

## 5. Non-Functional Requirements

| ID | Requirement | Target (to validate) |
|---|---|---|
| NFR-PERF-1 | Input-to-visual latency for drawing/dragging | < 16 ms frame budget, no network dependency |
| NFR-PERF-2 | Diagram size | 5k elements smooth, 20k usable (with culling/LOD) |
| NFR-PERF-3 | Gesture recognition latency | < 50 ms p95 |
| NFR-PERF-4 | Load time | < 1 s to interactive for typical diagram; core bundle budget (e.g. ≤ 250 KB gzip, libs/profiles lazy-loaded) |
| NFR-COLLAB-1 | **Local-first responsiveness**: every local edit is applied optimistically and rendered immediately; remote latency is never in the interaction loop | Always |
| NFR-COLLAB-2 | Remote change visible to others | p95 < 150 ms same region |
| NFR-COLLAB-3 | Concurrent editors per diagram | *Deferred to the collaboration phase (D15)* |
| NFR-REL-1 | No data loss: local persistence + resend on reconnect; autosave | Always |
| NFR-COMPAT-1 | Browsers | Latest 2 versions of Chrome, Edge, Firefox, Safari on **desktop and tablets (large screens)**; phone browsers and WebView are best-effort/Could (D5) |
| NFR-SEC-1 | XSS-safe: sanitize SVG, HTML in labels, links; strict CSP compatibility | Always |
| NFR-SEC-2 | External resource loading controlled by host allow-list; CORS handled | Always |
| NFR-PRIV-1 | Core and default plugins make **no external network calls**; any plugin that does (e.g. future LLM plugin) is opt-in by the host | Always |
| NFR-A11Y-1 | Reasonable accessibility for non-canvas UI (keyboard, focus, labels). **No formal WCAG compliance target** (D16) | Best effort |
| NFR-EXT-1 | Public API stable under semver; plugin API versioned | Always |
| NFR-TEST-1 | Round-trip fixtures for all formats; gesture corpus; visual regression; collaboration convergence tests (fuzz/property-based) | Always |
| NFR-OBS-1 | Opt-in telemetry for gesture mis-recognitions (anonymized stroke + correction), perf metrics | Opt-in |
| NFR-LIC-1 | **Apache-2.0** project; only permissive dependencies (MIT/BSD/Apache/ISC); automated license audit in CI; weak-copyleft (LGPL/EPL/MPL) only after legal review; no bundled trademark-restricted icon sets (D2) | Always |

---

## 6. Document Model and Native Text Format

### 6.1 Design principles
1. **Stable IDs** for every entity (layers, nodes, edges, groups, assets). Required for sync, semantic diffs, and imports.
2. **Tree-plus-graph:** a *containment tree* (layer → group/container → node) plus a *graph* (edges referencing node/port IDs).
3. **Operation-friendly:** every user action maps to small, composable, serializable ops (see 8.2).
4. **Fine-grained properties**, so concurrent edits of different properties of one shape merge cleanly (position vs text vs color).
5. **Ordering via fractional indexes** (not integer indices) for z-order and sibling order, so concurrent reorders and inserts do not conflict.
6. **Geometry and semantics separated:** geometry (x, y, w, h, rotation) lives apart from semantic data (type, label, profile-specific attributes), so layout can be recomputed and semantic exports (Mermaid) can ignore geometry.
7. **Deterministic canonical serialization** (stable key order, normalized numbers) so text diffs and content hashes are stable.
8. **Extensible:** namespaced metadata (`x-vendor.*`) preserved on round trip.
9. **Versioned schema** with migrations.

### 6.2 Conceptual entities
`Diagram { id, schemaVersion, profile, layers[], assets[], meta }`
`Layer { id, name, visible, locked, opacity, order, children[] }`
`Node { id, parent, layer, type (shape ref: library/shape), geometry, style, text, ports?, params, data{}, children[] }`
`Edge { id, source{node,port?}, target{node,port?}, routing, waypoints[], style, labels[], semanticType }`
`Group { id, members[] }`
`Style { fill, stroke, strokeWidth, font, ... }` with **named style tokens/themes** referenced by nodes, so a restyle of a theme changes all shapes.
`Asset { id, hash, mime, source (embedded|url), license? }`

### 6.3 Text format (decided direction: YAML)
Goals: human-readable, compact, line-diff-friendly, easy to hand-edit and to generate programmatically. The text form is a **serialization of the same model** (YAML, canonical key order), not a separate language.

Illustrative example (not final schema):

```yaml
diagram: 1                 # schema version
profile: flowchart
meta: { title: Signup flow, size: [1200, 800] }   # optional base size hint
layers:
  - id: main
    nodes:
      - { id: start, kind: terminal, text: Start,        at: [40, 40],   size: [120, 48], style: { fill: "#d1fae5" } }
      - { id: check, kind: decision, text: Valid input?,  at: [40, 140],  size: [140, 80] }
      - { id: save,  kind: process,  text: Save record,   at: [260, 150], size: [140, 60] }
      - { id: err,   kind: process,  text: Show error,    at: [40, 280],  size: [140, 60], style: { fill: "#fee2e2" } }
    edges:
      - { from: start, to: check }
      - { from: check, to: save, text: "yes", routing: orthogonal }
      - { from: check, to: err,  text: "no" }
    groups:
      - { id: g1, members: [save, err] }
```

Rules to specify:
- Geometry (`at`, `size`) is **optional**; nodes without it are auto-laid out (supports hand-written files and Mermaid imports).
- **Use a safe YAML subset** (YAML 1.2 core schema; **no custom tags, no anchors/aliases, no merge keys, no code execution**; limits on depth, size and alias expansion). Quote ambiguous scalars (colors, `yes/no`, version-like strings) to avoid YAML typing pitfalls; the serializer always quotes them.
- Deterministic canonical output (stable key and element order, normalized numbers, one element per line in flow style) so diffs and content hashes are stable; comments and unknown namespaced keys (`x-*`) are preserved on round trip where the YAML library allows.
- Large assets are never inlined by default; they are referenced by hash/URL.
- A formal **JSON Schema** describes the structure (also used to validate YAML and to guide LLM generation).
- Parser/serializer library choice is a spike (must be permissive-licensed and support comment preservation if feasible).

### 6.4 Persistence building blocks
- Canonical YAML snapshot (JSON wire encoding optional) + op log (see 9).
- Binary/compact encoding for transport (optional) — distinct from the human-readable storage format.

---

## 7. Import / Export / Interoperability

### 7.1 draw.io (diagrams.net) — **Must (basic fidelity first, D11)**
**Initial scope:** common shapes, edges with waypoints, text/labels, basic styles, layers, groups/containers, images. **Later:** wider stencil coverage, multiple pages, advanced styles. Import and export are **plugins** selected when the widget is embedded.

- **Import:** `.drawio` / `.xml` (mxfile → diagram → mxGraphModel → mxCell). Handle both uncompressed XML and the compressed variant (base64 + raw deflate + URL-encoded). Map: vertices → nodes, edges → connectors, `parent` → containment, **layers** (top-level cells under root) → layers, groups, styles (`style="rounded=1;fillColor=..."` strings → style model with unmapped keys preserved under `x-drawio.*`), waypoints and edge routing (`edgeStyle`, `orthogonalLoop`), labels (HTML labels sanitized), embedded images, multiple pages (→ separate diagrams or "pages"; **[DECISION]** treat pages as separate diagrams in a "bundle").
- **Export:** to mxfile XML (uncompressed by default, compressed optional). Acceptable loss: features draw.io lacks (e.g. contextual-palette state, some layer semantics — layers map to draw.io layers where possible, so loss is smaller than expected; gesture ink may be rasterized or converted to a path shape).
- **Fidelity goals:** define a measurable **round-trip fidelity suite**: draw.io → ours → draw.io produces a semantically equal graph (same nodes, edges, text, geometry within tolerance) on a corpus of sample diagrams, and a rendered comparison threshold.
- **Risk:** draw.io's shape vocabulary (`shape=mxgraph.aws4...`, stencil XML) is huge. Plan: map the most common ~100 shapes natively, fall back to rendering unknown stencils as generic shapes using the embedded stencil geometry, or as an SVG image.
- **Legal check:** review license/terms for draw.io libraries and trademarks (AWS/Azure icons).

### 7.2 Mermaid — import (**Should**: flowchart + sequence first, D12) and export (**Could**, research)
**Fundamental difference.** Mermaid is a *semantic, layout-free* language: no coordinates, no per-element styling in most diagram types. Our model is geometric and richer. Hence:

- **Import (feasible, S):** Parse Mermaid (use the official parser to get the structured model rather than writing our own grammar), then create nodes/edges and run **auto-layout** (layered/ELK-style) to assign geometry. Mark the result with the originating profile (flowchart, sequence, class, state, ER, ...). Start with: flowchart/graph, sequence, class, state, ER; later: gantt, mindmap, C4, architecture, others. Styling via `classDef`/`style`/`linkStyle` mapped to the style model where possible.
- **Export (partially feasible, research task):** Possible only for diagrams that conform to a **Mermaid-expressible profile**:
  1. The diagram is tagged with a profile that has a Mermaid mapper (e.g. flowchart, sequence, class, state, ER).
  2. Elements use shapes/edges that exist in that Mermaid diagram type (e.g. flowchart: rectangle, rounded, stadium, diamond, circle, cylinder, subroutine, hexagon, etc.; edge types: `-->`, `---`, `-.->`, `==>`, labels; subgraphs ← containers).
  3. Geometry, free-form styling, layers, images, ink are **dropped** (or partially mapped to `classDef`).
  4. Provide a **lossiness report** to the user ("12 shapes mapped, 3 shapes approximated, layers ignored, geometry ignored") and a **"Mermaid-compatible" lint mode** in the editor that warns during editing.
  5. Sequence diagrams may be derived from horizontal/vertical positions only if the profile enforces the structure (lifelines, ordered messages).
- **Research tasks (R-MERM):** (a) enumerate Mermaid diagram types/versions and syntax stability; (b) feasibility matrix: shape/edge/feature → Mermaid support; (c) prototype flowchart + sequence export; (d) decide on a "semantic core" shared by profile ↔ Mermaid mapping; (e) Mermaid is also evolving (new diagram types like architecture, block, kanban, etc.) so mappers must be versioned plugins.
- **Strategic option [ADDED]:** make our **native text format a superset-compatible dialect** of Mermaid for the semantic part (so a valid Mermaid flowchart is valid input), with geometry/style as an extension block that Mermaid renderers ignore (e.g. in comments or a trailing directive). Evaluate feasibility and whether it creates confusion or maintenance cost.

### 7.3 Other formats
- **SVG export (M)** — clean, with optional embedded model metadata (round-trippable "editable SVG").
- **PNG export (M)** — scale factor, transparent background, selection/layer-based.
- **PDF export (S)** — via headless render or SVG→PDF; page sizing.
- **YAML (M)** — native format (D7); **JSON (S)** — same model as wire/API encoding.
- **Import: SVG as shapes/library (S)**, Visio `.vsdx` (C), Excalidraw/tldraw JSON (C), PlantUML text (C, via profile), Lucidchart (W), CSV-to-diagram for org charts/graphs (C).
- **Interop principle:** every importer/exporter is a **plugin** with a declared **capability/loss matrix** and automated round-trip tests.

---

## 8. Real-Time Collaboration (E10)

References: <https://engilyin.github.io/kb/SystemDesign/real-time-collaborative-editor>.

### 8.1 Recommendation for pattern selection **[DECISION]**
The KB lists LWW, centralized OT, CRDTs, differential sync, event sourcing, and client-side prediction + reconciliation, and recommends choosing the *simplest conflict strategy that fits the domain*. It also describes the drawing-tool scenario explicitly (list of shapes with UUIDs; per-object LWW; delete beats move; append-only strokes; optimistic rendering; snapshots; viewport-based partitioning) and cites Zoom Whiteboard's latency issue as the anti-pattern (waiting for server round trips before rendering).

Diagrams are **coarse-grained structured objects**, not linear text. So a full text-OT or sequence-CRDT is mostly unnecessary. Proposed hybrid:

| Concern | Strategy |
|---|---|
| Shapes/edges/layers (structured data) | **Per-property LWW registers on stable object IDs** (Figma-style): each property of each object is independently last-writer-wins, using a hybrid logical clock (HLC) + client ID as tie-breaker. Concurrent edits of *different* properties of a shape both survive. |
| Create / delete | Creates never conflict (unique IDs). **Delete wins over move/edit** (tombstone); undelete = new op with higher clock. |
| Ordering (z-order, sibling order) | **Fractional indexing** (or equivalent sequence CRDT) so concurrent reorders converge. |
| Containment/reparenting | Parent pointer as an LWW property with **cycle prevention** (server/validator rejects or repairs cycles deterministically). |
| Rich text *inside* a shape | Text CRDT or OT for the label field only (e.g. Yjs `Y.Text` or equivalent), since concurrent typing in one label needs character-level merge. |
| Freehand ink | Append-only point logs per stroke; only finalized strokes persisted. |
| Presence (cursors, selection, "who is dragging what") | Ephemeral channel, LWW with TTL, throttled (~50 ms), separate from document ops. |
| Server role | **Server-authoritative ordering + relay** (single total order per document, via a document "room" owner/sequencer), **clients apply ops optimistically** (client-side prediction) and reconcile against server order. |
| Offline | Local-first op queue persisted in IndexedDB; on reconnect send last-known server version and ops, receive missed ops; deterministic merge. |
| Transport | **WebSocket** default; optional WebRTC data channels (P2P/SFU) for ultra-low-latency presence/drag streaming; HTTP fallback. |

**Alternative to evaluate in a spike:** adopt an existing CRDT library (e.g. Yjs / Automerge style maps + arrays) as the engine and layer our schema on top. Trade-off: less custom code and strong offline/P2P story, vs. tombstone growth, less control over server authority, and semantic validation (cycles, constraints) being harder. The decision matrix should consider: offline requirement, central server availability, op-log/commit design (section 9), bundle size, and team expertise. **Choose before finalizing the model, since ops shape the whole system.**

### 8.2 Requirements
- **FR-COL-1 (M)** The model exposes mutations only as **serializable ops** (set-property, create, delete, reparent, reorder, text-edit, asset-ref). No hidden mutation paths.
- **FR-COL-2 (M)** **Optimistic local apply**; remote ops are rebased/merged without blocking local interaction. UI never waits for server acknowledgment.
- **FR-COL-3 (M)** **Interaction streaming vs. committing:** during drag/resize/rotate, stream *transient* previews (coalesced, throttled, unreliable-OK) to peers; commit **one** op at gesture end (avoids op-log bloat and keeps undo/commit granularity sensible).
- **FR-COL-4 (M)** Conflict rules documented and property-tested (convergence under random interleavings, delay, duplication, reordering of messages).
- **FR-COL-5 (S)** Soft **selection/edit indicators**: show who selected/is dragging what; optional soft locks (advisory, auto-expire) to reduce surprises; optional hard locks via the lock feature.
- **FR-COL-6 (S)** Remote-change UX: subtle animation of remote moves, per-user color, avatar/cursor labels, "followed user" viewport mode.
- **FR-COL-7 (M)** Ops are validated (schema, invariants such as no containment cycles) before sequencing. **Authorization is the host's responsibility** (D14); the adapter receives the user identity and may reject ops, but the widget defines no roles or permission model.
- **FR-COL-8 (S)** Comments and annotations (anchored to elements; threaded; resolved state) — modeled as separate, lower-priority entities so they do not bloat the diagram document.
- **FR-COL-9 (S)** Scalability: sticky routing per document or shared pub/sub between nodes; snapshot/compaction worker; spatial/viewport-aware streaming for very large diagrams (optional).
- **FR-COL-10 (M)** The widget talks to a **Collaboration Adapter interface** (connect, send op, receive ops, presence), so hosts can supply their own backend; ship a **reference server** implementation.
- **FR-COL-11 (M) [ADDED]** **Multi-user undo** — see FR-UND-1; undo = inverse ops by the same author as new ops.
- **FR-COL-12 (S) [ADDED]** Handling of **semantic conflicts** (two users reparent into each other; connector endpoint deleted concurrently → dangling connector policy; layer deleted while others edit it → move children to a recovery layer).
- **FR-COL-13 (S) [ADDED]** Single-user mode must work with **zero** collaboration infrastructure and the same model.

---

## 9. Versioning and History (E11)

References: <https://engilyin.github.io/kb/SystemDesign/content-history-and-versioning>.

### 9.1 Recommendation **[DECISION]**
The KB recommends: snapshots for small content (<~1 MB); **hybrid snapshot + delta** at scale; content-addressable storage (hash) for dedup; **separate the internal operation log from user-facing versions**; **restore creates a new version**; **semantic diffs for structured data (node/edge level via stable IDs)**; tiered retention; Git-style DAG only when branching/merging is needed; and, for diagrams specifically, store a full state snapshot plus a precomputed node-level change record for fast display, with explicit branch-merge conflict rules.

Apply it as:

1. **Operation log** (fine-grained, internal, ordered per document): the same ops used by collaboration; each op carries `author`, `clientId`, `timestamp (HLC)`, `serverSeq`.
2. **Commits** (user-facing history): a commit is a **named, authored grouping of one or more ops** — "granular commits which can hold a series of changes of different people", as in your description. A commit has: `id`, `parents[]`, `message/auto-summary`, `authors[]` (contributors with per-author op counts), `time range`, `opRange`, `snapshotRef?`, and `tags/labels`.
   - **Auto-commit** policy: close a commit after N seconds idle, on explicit save/"checkpoint", on session end, or when authorship composition changes by policy (configurable). Auto-generated summaries ("Alice added 3 shapes, Bob restyled 2").
   - **Manual named versions** ("v1.0 approved"), protected from pruning.
3. **Snapshots:** periodic materialized snapshots (every N ops / time / size), stored content-addressed; reconstruct any version = nearest snapshot + replay of ops.
4. **Semantic diff:** match elements by ID → added/removed/changed (field-level)/moved/reparented/reordered; visual diff overlay in the editor (highlight added/changed/deleted, with author attribution). Move detection is cheap with stable IDs (position change = property change).
5. **Blame / attribution:** per-element last-modified-by and creator (derived from the op log), "who did this?" on hover/inspector, activity feed.
6. **Restore** = new commit that applies the inverse/target state (history never rewritten).
7. **Branching and merging (C, later):** commit DAG with parents; three-way semantic merge using the rules above (one-sided additions kept; same-property divergent changes → conflict UI; delete-vs-modify → conflict). Keep the model DAG-ready (commits have `parents[]`) even if the first release is linear.
8. **Retention:** tiered — keep all named versions; compact auto-commits older than X to daily/weekly checkpoints; configurable per host; GDPR-style author data handling.
9. **Storage backend** is behind a **History Adapter** (host-supplied or reference implementation); format-agnostic (YAML/JSON snapshots).

### 9.2 Requirements
- **FR-VER-1 (S)** Commit list with author(s), time, summary; open any version read-only; visual compare of any two versions.
- **FR-VER-2 (S)** Named versions/labels, restore as new commit.
- **FR-VER-3 (S)** Per-element history ("show history of this shape").
- **FR-VER-4 (S)** Multi-author commits (series of changes by different people) retain per-op authorship.
- **FR-VER-5 (C)** Branch/merge, with review ("suggest changes" mode where edits are proposals accepted later).
- **FR-VER-6 (S)** Export history summary (audit log) for compliance.
- **FR-VER-7 (W) [D14]** ~~Permission-filtered history~~ — out of scope; host adapters decide who can read history.

---

## 10. Diagram Types — Generalized "Profile" Approach (E12)

### 10.1 Idea
Rather than hard-coding UML, C4, AWS and so on, define a **Diagram Profile**: a declarative (plus optional code) plugin that tells the core *what vocabulary and rules apply*, and **drives the context engine**. The core stays generic: nodes, edges, containers, layers, ops.

### 10.2 What a profile defines
| Facet | Content |
|---|---|
| **Vocabulary** | Node kinds (e.g. UML Class, Actor, Lifeline; C4 Person/System/Container/Component; AWS VPC/Subnet/EC2), edge kinds, container kinds, port definitions, default styles, icons. |
| **Grammar / constraints** | Which edge kinds may connect which node kinds, cardinalities, allowed containment (e.g. EC2 ∈ Subnet ∈ VPC), required properties, validation rules (errors/warnings/info). |
| **Context hints** | Prior probabilities and transitions for the next shape/edge given neighborhood (e.g. after dragging out from "Decision" suggest "Yes/No" labeled edges; inside a VPC suggest Subnet), keyword → shape mapping, spatial heuristics (sequence diagrams: lifelines are vertical, aligned in a row; messages are ordered top→bottom). |
| **Gesture extensions** | Extra gestures or gesture→shape mapping (e.g. vertical dashed line = lifeline). |
| **Layout** | Preferred layout algorithm and options (layered for flowchart, tree for org chart, sequence layout with timeline ordering, swimlanes). |
| **Interaction behaviors** | Special behaviors: sequence diagrams reorder messages by vertical drag; "activation bars"; swimlane membership by position; auto-numbering. |
| **I/O mappers** | draw.io shape/style mappings, Mermaid mapper (if any), PlantUML/other text mappers, and semantic export (e.g. to JSON/IaC graph). |
| **Libraries** | Associated shape/icon libraries (lazy loaded). |
| **Metadata/schema** | Typed `data` attributes per node/edge kind (e.g. class members, C4 technology/description). |

### 10.3 Core ↔ profile contract
- **Semantic type layer:** every node/edge has a `kind` (profile-defined) in addition to its visual shape — separation of **what it means** from **how it looks**. This enables: restyle without changing meaning, convert between visual notations, Mermaid/PlantUML export, validation and smarter prediction.
- **Mixed-profile diagrams:** allow layers or containers to carry a different profile (e.g. an AWS layer inside a C4 diagram); define precedence rules.
- **Profile detection:** auto-suggest a profile from first shapes/labels/imports ("looks like a sequence diagram"); user can always override; "generic" is the default profile.
- **Profile packaging/versioning:** declarative manifest (JSON/YAML) + optional sandboxed code module; semver; registry; host can allow-list profiles.

### 10.4 More advanced context handling (your question)
Layered scoring for "which shape does the user mean here?":
1. **Gesture evidence** (stroke geometry) → candidate primitives.
2. **Spatial context** (inside which container/lane, near which shapes, alignment to existing rows/columns, direction of flow).
3. **Profile grammar** (what is legal/likely here).
4. **Sequence/history** (what was drawn recently; n-gram/Markov over kinds).
5. **Textual context** (label typed so far, neighbors' labels).
6. **User/org preference** (frequently used shapes; opt-in cross-diagram learning).
7. *(Optional)* **LLM re-ranker** on structured summaries of the local neighborhood, opt-in, never blocking, with deterministic fallback.
Output: ranked candidates with confidence and "reason" metadata; top candidate auto-applied only if confidence ≥ threshold, else shown for one-tap choice. Provide an **offline evaluation harness** (recorded sessions → measure top-1/top-3 hit rates) so tuning is data-driven.

### 10.5 Initial profile set (proposal)
Generic/Flowchart (MVP), then: UML (class, sequence, activity/state), C4, ERD, BPMN-lite/swimlane, org chart/mind map, cloud infrastructure (AWS first), network topology. Each profile is a mini-project with its own acceptance criteria.

---

## 11. UX Principles (guidance for design specs)

1. **Act first, configure second:** no mandatory tool selection; the default action on canvas is "draw/recognize".
2. **One gesture, one meaning:** disambiguation rules must be deterministic and visualized (cursor changes, ghost preview).
3. **Everything is undoable**, especially recognition results.
4. **Palettes assist, never block.** Ignoring the palette must be zero-cost.
5. **Touch is first-class:** gizmo sizing, no hover-only features, long-press and tap-zone alternatives, pen support.
6. **Progressive disclosure:** beginner mode hides advanced layers/hierarchy tools; power users get keyboard shortcuts and outline views.
7. **Predictability over cleverness:** style-inheritance and prediction must be explainable and switchable.
8. **Consistent visual language for collaboration** (per-user colors, presence, remote change animations).
9. **Performance is a feature:** no jank; heavy operations (layout, import) in workers with progress and cancel.

---

## 12. Architecture Outline (non-binding)

```
┌───────────────────────── Host page / portal ────────────────────────┐
│  <gesture-diagram> Web Component  (Public API, events, adapters)    │
├──────────────────────────────────────────────────────────────────────┤
│ UI layer: canvas renderer · gizmos · palette · outline · panels      │
│ Interaction engine: input normalization (mouse/touch/pen) · gesture  │
│   recognizer · hit-testing · selection · snapping · connector tool   │
│ Context engine: state · ranking · profile hints · (optional LLM)     │
│ Document core (headless, UI-free, runs in Node too):                 │
│   model · ops · validation · layout · routing · styles · profiles    │
│ Sync layer: op queue · optimistic apply · HLC · presence · adapters  │
│ History layer: commits · snapshots · semantic diff · adapters        │
│ I/O plugins: drawio · mermaid · svg · png · pdf · json · text DSL    │
└──────────────────────────────────────────────────────────────────────┘
      ▲ Persistence/Collab/History/Asset/Auth adapters (host-provided)
      ▼
 Reference backend: WebSocket sequencer · snapshot worker · storage
```

Key architectural rules: **headless core**; all mutation through ops; rendering is a pure function of the model + view state; web workers for recognition, layout, import/export; adapters for every external dependency.

### 12.1 Technology Stack and Framework Strategy (D18)

**The question:** pure TypeScript, a React component, or legacy-style libraries (e.g. jQuery)?

**Recommendation: three layers, one framework-free core.**

| Layer | Package (proposed) | Technology | Why |
|---|---|---|---|
| **Core (headless)** | `@gesture-diagram/core` | Plain TypeScript, no DOM-framework dependency | Model, ops, YAML (de)serialization, validation, layout, routing, profiles, context/guess engine, history, import/export. Must run in Node (server-side export, CI, tests) and be shareable by any UI. |
| **Engine (browser)** | `@gesture-diagram/engine` | TypeScript + Canvas 2D + Pointer Events | Input normalization, gesture recognizer, hit-testing, spatial index, viewport virtualization, rendering, gizmos drawing. **Per-frame work must not go through a UI framework's reconciler.** |
| **UI chrome** | `@gesture-diagram/ui` | Component framework (see below) + headless UI primitives | Contextual palette, library panel, layers/outline panel, property popovers, dialogs, menus, toasts. This is ordinary application UI where reinventing the wheel has no payoff. |
| **Wrappers** | `@gesture-diagram/react`, `<gesture-diagram>` element | Thin | React component and Web Component over the same engine+ui. |
| **Adapters/plugins** | `@gesture-diagram/adapter-*`, `plugin-*` | TypeScript | Persistence, collaboration, history, import/export, profiles, guess engines. |

**Why not "everything in React":**
- Canvas rendering of thousands of objects and 60 fps drag/gesture loops should not depend on React state updates; the usual pattern in successful editors is *canvas/engine imperative, chrome declarative*.
- A React-only core would lock out non-React hosts and the headless/Node use cases (export service, CI).
- Collaboration needs deterministic, UI-independent ops; keeping the model outside React makes that natural.

**Why not "pure TypeScript with hand-built DOM UI":**
- Palettes, panels, outline trees, menus and dialogs are exactly where a component framework and ready-made accessible primitives save the most time. Hand-writing focus management, popover positioning, keyboard navigation and tree views is wasted effort and is the usual source of accessibility bugs.

**Why not jQuery-style libraries:** obsolete for this problem (imperative DOM manipulation, no component model, poor fit for reactive state). Not recommended.

**UI chrome framework choice (spike, then ADR):**
- **Option A (recommended default): React** for `@gesture-diagram/ui`, since most target hosts are React portals and it has the largest ecosystem of accessible headless primitives. The Web Component build bundles it (or a lighter drop-in like **Preact** through a compatibility alias) so non-React hosts need no React of their own. Measure bundle size in the spike; the React wrapper should treat React as a **peer dependency** so React hosts do not ship two copies.
- **Option B:** a small, dependency-light framework (e.g. Preact, Lit, or Solid) for the chrome with a thin React wrapper; smaller bundle, slightly smaller ecosystem.
- Whichever is chosen, the UI talks to the engine only through a **small public store/command API** (subscribe + dispatch), so the chrome can be replaced or re-skinned by a host.

**State management without reinventing it:**
- The **document model is the store** (ops in, snapshots out); the UI subscribes through a tiny external-store interface (compatible with React's `useSyncExternalStore`). No Redux-style duplicate of document state in the UI.
- Ephemeral UI state (open panels, hover, palette position) lives in a small UI store (e.g. a minimal library such as Zustand, or component state). This is deliberately separate from the document and from collaboration.

**Build vs. reuse (candidate list; every item needs a license + bundle-size check, versions not verified here):**
| Need | Reuse (permissive-license candidates) | Build ourselves |
|---|---|---|
| Popover/menu/dialog/tooltip primitives, focus management | Headless primitives libraries (e.g. Radix-style, Ariakit-style) and a positioning library (e.g. Floating UI) | Styling/theming |
| Spatial index | An R-tree/quadtree library (e.g. rbush) | Viewport/LOD logic |
| Rich text editing inside shapes | An editor framework (e.g. ProseMirror/Tiptap/Lexical family) as overlay — **spike**, versus a minimal custom contenteditable for plain text first | Text-layout metrics for canvas rendering |
| YAML | A YAML 1.2 library supporting comment preservation | Canonical serializer rules |
| Mermaid parsing | The official Mermaid parser (import) | Mapping to our model |
| Auto-layout | A permissive layered/tree layout library (e.g. dagre-style); ELK only after license review | Constraints/profile integration |
| Collaboration engine (later) | Optionally a CRDT library behind the adapter | Op schema, adapters |
| Gesture recognition | Evaluate template recognizers and small ML models | Disambiguation rules, context integration |
| Icons / UI | An open icon set with permissive license | Diagram-specific gizmos |
| Canvas drawing | Evaluate a 2D canvas library vs. direct Canvas2D | Virtualized scene, hit-testing (likely custom for performance control) |
| Tooling | pnpm workspaces monorepo, Vite, Vitest, Playwright, Storybook-style component workshop | — |

**Rule:** reuse for commodity infrastructure; build only the differentiators (gesture engine, context engine, profile system, op model, interop).

---

## 13. Input Security and Hygiene (E15) **[ADDED]** — permissions and compliance are out of scope (D14, D16)

- Sanitize imported SVG/HTML labels; block scripts, `foreignObject` abuse, external references; image proxying/allow-lists; CSP-compatible (no `eval`).
- Untrusted import files (draw.io XML, Mermaid text): defend against XML entity expansion, decompression bombs, huge payloads, cycle/loop structures.
- Profiles/plugins: sandboxed execution (iframe/worker) or declarative-only for untrusted sources; signed packages for marketplaces.
- AuthN/AuthZ is entirely the host's concern (D14); the widget may receive an identity (id, display name, color) for attribution and a read-only flag.
- Privacy: no external calls from the core or default plugins; telemetry opt-in. Data retention and regulatory compliance (GDPR, SOC2, residency) belong to the host system (D16).
- Licensing: icon sets (AWS/Azure/GCP trademarks and usage terms), draw.io stencils, Mermaid and any CRDT/OSS dependency licenses; the project license is Apache-2.0 (D2); see NFR-LIC-1.

---

## 14. Gaps Identified in the Original Description (what was missed or needs clarification)

**Product/UX**
1. **Gesture conflicts and escape hatch** — drawing vs. moving vs. lasso vs. freehand ink vs. pan on touch. Not covered; now FR-GES-5/6/8.
2. **Discoverability and onboarding** — gesture-driven UIs are invisible; need cheat-sheet/hints (FR-ONB-1).
3. **Mis-recognition recovery** — one-tap alternatives and undo (FR-GES-4).
4. **Multi-stroke gestures and timing**, and **stylus/palm rejection**.
5. **Keyboard-only and screen-reader use** — absent; accessibility is also frequently a procurement requirement for portals.
6. **Text** — rich text, RTL/IME, overflow, text on connectors, label-first workflow.
7. **Snapping, guides, alignment** — essential for "clean" diagrams.
8. **Copy/paste/duplicate, templates, themes/style tokens, format painter.**
9. **Auto-layout and connector routing** — core to diagram quality, especially after Mermaid import and for containers/hierarchies.
10. **Comments, mentions, sharing, presentation/read-only mode.**
11. **Print/page/frames** concept, if exports to PDF are needed.

**Model/Technical**
12. **Undo/redo in multi-user mode** — the hardest "invisible" requirement; must be designed with the op model.
13. **Dangling and cross-layer connector semantics** (hidden/deleted/locked layers); **group vs container vs hierarchy** distinctions — made explicit in this draft; please confirm.
14. **Semantic layer** (node `kind`) distinct from visual shape — needed for profiles, Mermaid export, and better guessing.
15. **Fractional ordering, cycle prevention in hierarchies, tombstones/GC** — collaboration details.
16. **Transient interaction streaming vs. committed ops** — necessary to meet the "no Zoom-whiteboard-lag" goal while keeping history clean.
17. **Offline/local-first** — decide whether required (affects CRDT vs. server-authoritative choice).
18. **Performance model** for large diagrams (culling, LOD, workers, rendering technology).
19. **Plugin/extension security**, API stability, versioning/migrations of the document schema.
20. **Testing strategy** — gesture corpus, convergence fuzzing, round-trip suites, visual regression. Not optional for this scope.
21. **Draw.io fidelity** — define "what is lost" up front; draw.io's styles/stencils are extensive; plan a loss matrix.
22. **Mermaid export feasibility** — see 7.2; realistic only as profile-constrained, lossy export.
23. **Legal/licensing** of third-party shape libraries and icon packs; product licensing model.
24. **Business/ops:** hosting model for the reference server, scale targets, SLAs, analytics.

**Terminology ambiguities in the original text** (please confirm my interpretation)
- "Scale" vs. "resize" gizmos — treated here as: resize changes geometry; *scale* scales content including text/stroke proportionally.
- "Change the shape" gizmo = convert shape type (FR-PAL-3).
- "Groups" vs "children shapes" vs "hierarchies" — treated as three different concepts (section 4.7).
- "Handle connections between layers" — treated as connectors may cross layers; hiding semantics TBD.
- "Palette" = both *shape library browser* and *contextual quick action pop-up*; separated here into **contextual palette** (quick) and **library panel** (browse).

---

## 15. Phased Roadmap (proposal)

| Phase | Goal | Contents |
|---|---|---|
| **0 – Walking skeleton + spikes (2–4 wks)** | De-risk | **Iteration 0 per section 15.1 (demo app live from the start)**;  Gesture recognition accuracy spike; **Canvas 2D virtualization benchmark** with 5k–20k nodes (D6); collaboration spike (per-property LWW vs. CRDT lib) behind the adapter interface with latency simulation; draw.io import of a first sample corpus (AWS-focused); Mermaid parse + auto-layout PoC; text-format syntax comparison; **license audit of candidate dependencies (D2)**; Google Drive adapter feasibility note. |
| **1 – MVP single-user** | Prove the interaction | Canvas, gesture→shapes (basic set), contextual palette with style inheritance, gizmos (mouse + large-screen touch), inline text, connectors, undo/redo, SVG/PNG/JSON export, text format v1, embed API with adapter/plugin registry, **browser-storage + file persistence adapters, and the DEMO-APP (FR-DEMO-1)**, keyboard basics, sticky-note/whiteboard basics. |
| **2 – Structure & interop** | Real diagrams | Layers, groups, containers + auto-size/layout, outline view, libraries and icons/images, draw.io import/export, Mermaid import, snapping/guides, auto-layout, accessibility pass. |
| **3 – Collaboration & history** | Teamwork | Op model hardened, collaboration adapter + sync engine, presence, **simple reference backend (server objects + endpoints + its plugin) and a multi-user demo (D4)**, commits/history/semantic diff, multi-user undo, comments. |
| **4 – Intelligence & profiles** | Differentiate | Profile SDK, **Architecture/AWS and Whiteboard profiles first (D10)**, then Flowchart/UML/C4, context engine v2 (grammar + sequence), Mermaid export (profile-constrained), library authoring, optional LLM assist. |
| **5 – Scale & ecosystem** | Grow | Branch/merge, marketplace, perf at 20k elements, advanced layouts, more importers (PlantUML/Visio/Excalidraw), analytics. |

> Note: ops/model design (Phase 0–1) must anticipate Phase 3, so the collaboration spike is in Phase 0 even though the feature ships later.

### 15.1 Always-Alive Delivery Strategy (D19)

**Principle:** there is always a runnable, deployed container page (the **demo app**, FR-DEMO-1) that shows the current state of the widget. It evolves in place each iteration and never sits broken.

**Iteration 0 — walking skeleton (first deliverable, intentionally thin but end-to-end):**
1. Monorepo (pnpm workspaces) with packages: `core`, `engine`, `ui`, `react`, `demo`.
2. Demo page with: infinite canvas (pan/zoom), draw a stroke → recognized as **rectangle or ellipse** (crude heuristics are fine) → clean shape placed, move/resize via basic gizmo, text typed inside, connector drag between two shapes.
3. **Live YAML pane** beside the canvas showing the document (edit YAML → canvas updates, and vice versa, via the real model and ops).
4. Save/Load through the **browser-storage adapter** and **file open/save**; undo/redo.
5. Dev panel: op log, recognition candidates with confidence, FPS/object count (doubles as the benchmark harness).
6. CI: build, unit tests, a Playwright smoke test that draws a shape, saves, reloads and checks the result; **auto-deploy to a public preview URL** (e.g. static hosting) on every merge to main, plus per-PR previews.

**Rules that keep it alive every iteration (Definition of Done additions):**
- The demo app builds and its smoke test passes before merge; new features appear in the demo, behind a **feature flag** if unfinished (demo has a flags panel).
- **Saved documents keep loading:** every schema change ships with a migration and a fixture file from the previous version (the fixture folder grows into the regression corpus).
- Plugins (adapters, importers, profiles) are registered in the demo through the same plugin registry hosts will use, so the demo is also the reference integration.
- A second tiny host page ("embed example") mounts the **React wrapper** and one mounts the **Web Component** in plain HTML, so integration regressions surface early.
- Each iteration is a **vertical slice** (gesture → model → render → persist → test), never a horizontal layer built in isolation.

**Suggested iteration order after 0** (each ends with a working demo): 1 gesture set + palette with style inheritance → 2 text editing + gizmos incl. touch → 3 connectors with routing → 4 layers/groups/containers + outline → 5 YAML format hardening + draw.io import/export plugin (basic) → 6 libraries/icons/images + Architecture(AWS) profile → 7 Mermaid import (flowchart, sequence) + auto-layout → 8 Whiteboard profile (stickers) → 9 history/commits → 10 collaboration adapter + reference backend.

### 15.2 License Choice (D20) — Apache-2.0 vs. MIT/BSD (not legal advice)

| | **MIT / BSD-2** | **BSD-3** | **Apache-2.0** |
|---|---|---|---|
| Commercial use, closed-source embedding | Yes | Yes | Yes |
| Length/simplicity | Shortest | Short | Longer |
| **Explicit patent grant** (and termination if a user sues over patents) | No | No | **Yes** |
| Trademark/name use | Silent | No endorsement clause | **Explicit: no trademark rights granted** |
| Change notices / NOTICE file | No | No | Yes (modest obligations) |
| Compatible with GPL-3 projects | Yes | Yes | Yes (one-way; **not** GPL-2-only) |
| Typical corporate-legal comfort | High | High | **Very high** (often preferred) |

**Recommendation: Apache-2.0.**
- A gesture-driven editor sits in an area with many software patents; the **explicit patent grant** is valuable for contributors and for corporate adopters embedding it in portals, which is your target.
- It protects the **name** (`gesture-diagram`) from implied trademark use.
- Large corporate users are generally at ease with it (draw.io is Apache-2.0 as well; many widely used editor libraries are MIT, so both are common in this space — verify before relying on any specific project's license).

**When MIT would be better:** if you want the shortest license and maximum compatibility with every other license, including GPL-2-only code. A common compromise used in some ecosystems is **dual licensing "MIT OR Apache-2.0"** (users choose), giving the maximum flexibility at the cost of two license files.

**Practical notes:**
- Because the project is *permissively* licensed, nothing prevents you from later offering a **hosted/paid service** or proprietary add-ons built on top. If you might want to **relicense future versions**, collect contributions under a **CLA** (a DCO sign-off alone does not grant relicensing rights); otherwise a lightweight DCO is friendlier to contributors. Decide before accepting outside contributions.
- Add `LICENSE`, `NOTICE`, SPDX headers, and a dependency license check in CI (NFR-LIC-1). Do **not** copy code from other projects without checking license compatibility (e.g. draw.io code is Apache-2.0, but icon sets and shape libraries carry their own terms).
- Have a lawyer review before the first public release if the project matters commercially.

---

## 16. Open Questions

**All questions Q1–Q17 are now answered or explicitly descoped; D18–D20 added in v0.4** (see section 0, D1–D17). Remaining items are *spec-phase details*, not blockers:
1. Build the **sample corpus** (10–20 diagrams: AWS architecture, flowcharts, sequence, whiteboard) — no real samples exist yet; use public or self-authored ones.
2. Choose the **working name** (D17) before the first public repository/package is created.
3. Define the **YAML schema** concretely (JSON Schema) and the safe-subset rules (6.3).
4. Spike-level technology choices: **UI chrome framework (React vs. lighter alternative, D18)**, gesture recognizer, YAML library, layout library (license-checked), Canvas spatial index, rich-text overlay approach.
5b. Decide **DCO vs. CLA** and **Apache-2.0 vs. dual MIT/Apache** (D20).
5. Detailed **whiteboard/sticky-note** behavior (colors, auto-size, quick-create gesture, clustering) to be specified in the Whiteboard profile.

## 17. Suggested Initial Backlog Skeleton (for the downstream AI to expand)

Each item below should become epics → stories with acceptance criteria (Given/When/Then), dependencies, estimates, and test notes.

- **E0 Spikes:** S0.1 Gesture recognizer accuracy; S0.2 Rendering benchmark; S0.3 Collab engine comparison; S0.4 draw.io import corpus; S0.5 Mermaid import/export feasibility (R-MERM); S0.6 Text format syntax; S0.7 Touch gizmo UX prototype.
- **E1 Core model:** schema, IDs, ops, validation, canonical serialization, fractional indexing, style tokens, migrations.
- **E2 Rendering and viewport:** canvas, culling, hit-testing, zoom/pan, selection, snapping.
- **E3 Gesture engine:** input normalization, stroke capture, recognizer interface, primitive shapes, ghost preview, ambiguity UI, ink fallback, command gestures, test corpus and harness.
- **E4 Contextual palette and context engine:** state tracking, style inheritance, ranking API, palette UI (desktop/touch), shape conversion.
- **E5 Gizmos and text editing:** hover zones, touch activation, resize/rotate/scale/move-text, inline text, text alignment by click-position, rich text.
- **E6 Connectors:** ports, drag-to-connect with target highlight, drag-to-create-target, routing, waypoints, labels, cross-layer behavior.
- **E7 Layers, groups, containers:** layer panel and ops, groups, containers with auto-size/auto-layout, outline tool, align/distribute, reparenting UX.
- **E8 Libraries/assets:** library format, loader, search, authoring, asset store with hashing and sanitization, icons/images.
- **E9 Interop:** draw.io import/export, Mermaid import, Mermaid export (R), SVG/PNG/PDF, loss matrix + round-trip tests.
- **E10 Embedding API:** Web Component, adapters, events, theming, iframe protocol, viewer/headless modes.
- **E11 Collaboration:** sync engine, presence, transient streaming, server sequencer, offline queue, convergence test harness, reference server.
- **E12 History:** commit model, snapshots, semantic diff UI, blame, restore, retention, adapters.
- **E13 Profiles:** SDK, manifest schema, validation engine, flowchart profile, UML profile(s), C4, AWS, profile detection, mixed-profile rules, evaluation harness.
- **E14 Accessibility/i18n/onboarding.**
- **E15 Security and privacy hardening.**
- **E16 Quality, telemetry, docs, samples, developer portal.**

---

## 18. Candidate Architecture Decision Records (ADRs) to Produce

1. Rendering technology.
2. Collaboration conflict strategy and library vs. custom.
3. Text format (YAML schema, canonical serialization, Mermaid relationship).
4. Containment/grouping/relations model.
5. Layer semantics (connectors, groups).
6. Gesture recognizer approach (heuristic vs. ML vs. hybrid).
7. Context engine and optional LLM use.
8. Profile plugin model and sandboxing.
9. History: commit granularity and retention.
10. Undo model under collaboration.
11. Packaging (Web Component, iframe, SSR/headless).
12. Interop fidelity policy and loss reporting.

---

## 19. Acceptance of this Draft

This document is a *draft for AI-assisted expansion*. Before generating the full specification and backlog, the downstream process should: (1) resolve or explicitly assume defaults for the open questions in section 16 (the proposed `[DECISION]` defaults may be used otherwise), (2) treat sections 4–10 as the requirement source of truth, (3) keep requirement IDs stable, and (4) mark any new requirement it invents as `[ADDED-AI]` for human review.
