---
iteration: 1
title: "Gesture-first authoring foundation"
goal: "By the end of Iteration 1, we will have a browser-first diagram editor foundation that allows authors to create and manipulate shapes on a canvas with gesture recognition and a stable local-first workflow."
start_date: 2026-10-29
end_date: 2026-11-05
velocity_target: 12
mode: dark_factory
status: planned
---

## Iteration Goal

By the end of Iteration 1, we will have a working browser-first diagram editor foundation that enables a user to open a blank canvas, pan and zoom, draw basic shapes, and see those gestures recognized into editable diagram objects without requiring a backend.

## Selected Stories

| Story | Title | Agent | Points | Phase | Dependencies |
|-------|-------|-------|--------|-------|-------------|
| STORY-00 | Phase 0: project foundation | tech-lead | 3 | 1 | none |
| STORY-01 | Render an editable canvas with pan, zoom, and selection | react-frontend-developer | 5 | 2 | none |
| STORY-03 | Recognize core shapes and connectors from stroke input | tech-lead | 5 | 3 | STORY-01 |

**Total:** 3 stories, 13 story points

## Execution Phases

### Phase 1 — Foundation
- STORY-00: Phase 0 project foundation
  - Establish the browser-first project layout, TypeScript configuration, and initial editor shell.
  - Define the document model and adapter boundaries for future storage and host integration.

### Phase 2 — Core Editing Surface
- STORY-01: Render an editable canvas with pan, zoom, and selection
  - Implement the viewport and selection model needed for editing.
  - Validate smooth, reliable interactions before introducing recognition logic.

### Phase 3 — Gesture Recognition
- STORY-03: Recognize core shapes and connectors from stroke input
  - Add narrow gesture recognition for rectangles and connectors.
  - Confirm the first end-to-end loop: draw → recognize → edit.

## Dark Factory Configuration

```yaml
dark_factory:
  enabled: true
  mode: layered
  max_parallel_agents: 2
  quality_gates:
    unit_tests: required
    linter: required
    build: required
    security_scan: required
    accessibility_check: required
  auto_merge_on_pass: false
  time_box_hours: 40
```

## Success Criteria

The iteration is successful when:
- [ ] The browser-first project skeleton compiles and runs locally.
- [ ] The user can pan, zoom, and select diagram objects on an editable canvas.
- [ ] A rough stroke can be recognized as a basic shape or connector.
- [ ] The workflow is documented enough for the next iteration to extend the editor.
- [ ] All quality gates pass before the story is marked done.
