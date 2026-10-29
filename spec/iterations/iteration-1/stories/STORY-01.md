---
id: STORY-01
title: "Render an editable canvas with pan, zoom, and selection"
iteration: 1
phase: 2
agent: react-frontend-developer
project: gesture-diagram
points: 5
dependencies: []
status: ready
---

## Story

As a diagram author,
I want to pan, zoom, and select elements on a large interactive canvas,
So that I can work comfortably with diagrams larger than the viewport.

## Context

This story focuses on the core editing surface. It establishes the viewport, pointer handling, and selection model needed before gesture recognition or persistence is introduced.

## Implementation Target

### Files to Create

| File | Purpose |
|------|---------|
| `src/editor/viewport.ts` | Manage view transforms, pan offsets, and zoom bounds |
| `src/editor/selection.ts` | Hold selection state and minimal hit-testing helpers |
| `src/editor/canvas.ts` | Wire pointer events to view and selection state |
| `src/components/EditorSurface.tsx` | Render the browser canvas and host the editor interactions |
| `src/components/__tests__/EditorSurface.test.tsx` | Verify pan/zoom and selection logic with render tests |

### Files to Modify

| File | Change |
|------|--------|
| `src/main.ts` | Mount the editor surface into the demo app |
| `src/styles.css` | Add canvas styling and viewport affordances |

## Code Skeleton

### Viewport model

```ts
// src/editor/viewport.ts
export type ViewState = {
  x: number;
  y: number;
  scale: number;
};

export function clampScale(scale: number): number {
  return Math.min(Math.max(scale, 0.2), 2.5);
}

export function createViewState(): ViewState {
  return { x: 0, y: 0, scale: 1 };
}
```

### Selection model

```ts
// src/editor/selection.ts
export type Selection = {
  ids: string[];
};

export function makeSelection(ids: string[] = []): Selection {
  return { ids };
}
```

### Canvas demo component

```tsx
// src/components/EditorSurface.tsx
import { useMemo, useState } from 'react';
import { createViewState, clampScale } from '../editor/viewport';
import { makeSelection } from '../editor/selection';

export function EditorSurface() {
  const [view, setView] = useState(createViewState());
  const [selection, setSelection] = useState(makeSelection());

  const summary = useMemo(
    () => ({ scale: view.scale, selected: selection.ids.length }),
    [selection.ids.length, view.scale],
  );

  return (
    <div className="editor-surface" aria-label="Diagram canvas">
      <div aria-live="polite">Zoom: {summary.scale.toFixed(2)} | Selected: {summary.selected}</div>
    </div>
  );
}
```

## Acceptance Criteria

- SC-1: Given a loaded diagram, when the user uses wheel or touch gestures, then the viewport pans and zooms smoothly within safe bounds.
- SC-2: Given multiple objects are rendered, when the user clicks or drags to select them, then only the intended objects remain selected.
- SC-3: Given an active selection, when the user performs a transform action, then only selected elements are affected.
- SC-4: Given large diagrams, when the user scrolls or pans, then the canvas remains responsive and does not flicker or block input.

## Mandatory Rules (Inline)

### Component Rules
- Keep the canvas interaction model purely client-side.
- Use pointer events as the primary source of input.
- Avoid blocking the main thread during basic pan/zoom operations.
- Ensure selection remains deterministic and keyboard accessible.
- Respect viewport bounds to keep the canvas stable.
- Preserve a single source of truth for view transforms.
- Keep the selection state decoupled from rendering logic.

### UI Quality Rules
- Support touch and mouse input without relying on browser-specific frameworks.
- Use accessible labels and non-visual affordances where possible.
- Keep the editor usable on large screens and tablets.
- No business logic in component render functions; derive state in hooks or utility modules.

## Validation Commands

```bash
npx prettier --write src/
npx eslint src/
npx tsc --noEmit
npm run build
npx vitest run
```

## Commit Command

```bash
git add . && git commit -m "feat(STORY-01): add canvas viewport and selection model"
```

## Definition of Done

- [ ] Canvas pan and zoom behavior works without jitter or visual corruption.
- [ ] Selection state updates correctly for single and multi-select behaviors.
- [ ] The editor surface remains responsive for typical large-diagram use.
- [ ] Tests cover viewport bounds and selection logic.
- [ ] The browser demo accepts direct user interaction.
