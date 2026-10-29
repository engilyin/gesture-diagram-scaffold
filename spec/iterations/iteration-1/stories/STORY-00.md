---
id: STORY-00
title: "Phase 0: project foundation"
iteration: 1
phase: 1
agent: tech-lead
project: gesture-diagram
points: 3
dependencies: []
status: ready
---

## Story

As a product team,
I want a clean browser-first project foundation,
So that the editor can be built iteratively without coupling to a backend or host runtime.

## Context

This story establishes the baseline workspace for the gesture-first diagram editor. The project intentionally prioritizes a local-first client experience with adapter-ready interfaces for future host embedding and persistence.

## Implementation Target

### Files to Create

| File | Purpose |
|------|---------|
| `package.json` | Define scripts, dependencies, and build/test entrypoints |
| `tsconfig.json` | Configure TypeScript for a strict browser-first codebase |
| `src/main.ts` | Launch the demo UI and initialize the editor shell |
| `src/editor/index.ts` | Export the document model and editor primitives |
| `src/styles.css` | Provide the base canvas and UI styling |
| `src/editor/__tests__/index.test.ts` | Verify empty-document initialization and default configuration |

### Files to Modify

| File | Change |
|------|--------|
| None | New project scaffold only |

## Code Skeleton

### Project bootstrap

```ts
// src/editor/index.ts
export type DiagramDocument = {
  version: string;
  elements: DiagramElement[];
};

export type DiagramElement = {
  id: string;
  type: 'rect' | 'line' | 'text';
  x: number;
  y: number;
  width: number;
  height: number;
};

export function createDocument(): DiagramDocument {
  return {
    version: '0.1.0',
    elements: [],
  };
}
```

```ts
// src/main.ts
import './styles.css';
import { createDocument } from './editor';

const doc = createDocument();
console.log('Diagram document initialized', doc);
```

```css
/* src/styles.css */
:root {
  color-scheme: light;
  font-family: Inter, system-ui, sans-serif;
}

body {
  margin: 0;
  background: #f4f7fb;
}
```

### Test skeleton

```ts
// src/editor/__tests__/index.test.ts
import { describe, expect, it } from 'vitest';
import { createDocument } from '../index';

describe('createDocument', () => {
  it('creates an empty versioned document', () => {
    const doc = createDocument();
    expect(doc.version).toBe('0.1.0');
    expect(doc.elements).toEqual([]);
  });
});
```

## Acceptance Criteria

- SC-1: Given a clean workspace, when the project is bootstrapped, then the browser app starts without a backend dependency.
- SC-2: Given a default document model, when the editor initializes, then it exposes a valid empty diagram structure.
- SC-3: Given the project layout, when a developer opens the workspace, then the system exposes clear boundaries between editor core, browser UI, and future adapter layers.
- SC-4: Given the initial test coverage, when the project is validated, then the empty-document behavior passes with no regressions.

## Mandatory Rules (Inline)

### Layer Rules
- Keep the foundation browser-first and backend-free for this iteration.
- Use TypeScript with `strict: true` and no `any` types in new code.
- Keep model/state logic separate from rendering concerns.

### Quality Rules
- Keep the document model versioned from the first commit.
- Prefer small, testable modules over framework-specific abstractions.
- Preserve a clear adapter boundary for future persistence and host embedding.
- Use DOM or canvas APIs without requiring network calls or server-side services.
- Keep the demo app minimal, reproducible, and safe to run locally.
- Ensure the project build runs without hidden environment variables.
- Add minimal tests for initialization and empty-document creation.
- No commented-out code, no dead files, and no placeholder logic that leaks into production code.

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
git add . && git commit -m "feat(STORY-00): initialize editor foundation"
```

## Definition of Done

- [ ] Workspace config and scripts are in place.
- [ ] Empty diagram model is initialized and versioned.
- [ ] Browser demo boots in a local environment.
- [ ] Lint, tests, and build all pass.
- [ ] Architecture boundaries for canvas, model, and adapter layers are documented.
