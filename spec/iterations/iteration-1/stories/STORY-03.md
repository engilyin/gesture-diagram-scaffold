---
id: STORY-03
title: "Recognize core shapes and connectors from stroke input"
iteration: 1
phase: 3
agent: tech-lead
project: gesture-diagram
points: 5
dependencies: [STORY-01]
status: ready
---

## Story

As a diagram author,
I want to draw rough shapes and connectors on an empty canvas,
So that I can create structured diagram content without selecting a tool first.

## Context

This story adds the first true gesture-first interaction: raw pen or pointer strokes are translated into diagram primitives. The recognition layer must be conservative, predictable, and easy to extend in later iterations.

## Implementation Target

### Files to Create

| File | Purpose |
|------|---------|
| `src/editor/input.ts` | Normalize pointer and touch stroke data |
| `src/editor/recognition.ts` | Score and classify stroke patterns |
| `src/editor/shapes.ts` | Create rectangle and connector objects from recognized candidates |
| `src/components/StrokeDemo.tsx` | Expose a debug view for recognition validation |
| `src/editor/__tests__/recognition.test.ts` | Validate recognition thresholds and fallback cases |

### Files to Modify

| File | Change |
|------|--------|
| `src/main.ts` | Add a demo surface or debug hook for testing recognition |
| `src/styles.css` | Add minimal recognition state styling |

## Code Skeleton

### Stroke normalization

```ts
// src/editor/input.ts
export type Point = { x: number; y: number };

export function normalizeStroke(points: Point[]): Point[] {
  return points.map((point) => ({ x: Number(point.x.toFixed(2)), y: Number(point.y.toFixed(2)) }));
}
```

### Recognition engine

```ts
// src/editor/recognition.ts
export type RecognitionKind = 'rect' | 'line' | 'ink';

export type RecognitionResult = {
  kind: RecognitionKind;
  confidence: number;
};

export function recognizeStroke(points: { x: number; y: number }[]): RecognitionResult {
  const area = points.reduce((sum, point) => sum + point.x * point.y, 0);

  if (area > 0 && points.length > 8) {
    return { kind: 'rect', confidence: 0.74 };
  }

  return { kind: 'ink', confidence: 0.42 };
}
```

### Shape creation

```ts
// src/editor/shapes.ts
export function createRectangle(x: number, y: number, width: number, height: number) {
  return {
    id: crypto.randomUUID(),
    type: 'rect',
    x,
    y,
    width,
    height,
  };
}
```

### Test skeleton

```ts
// src/editor/__tests__/recognition.test.ts
import { describe, expect, it } from 'vitest';
import { recognizeStroke } from '../recognition';

describe('recognizeStroke', () => {
  it('returns a rect classification for a strong closed shape', () => {
    const points = [
      { x: 0, y: 0 },
      { x: 50, y: 0 },
      { x: 50, y: 50 },
      { x: 0, y: 50 },
      { x: 0, y: 0 },
    ];

    const result = recognizeStroke(points);
    expect(result.kind).toBe('rect');
    expect(result.confidence).toBeGreaterThan(0.7);
  });
});
```

## Acceptance Criteria

- SC-1: Given a closed rectangular stroke, when the recognition engine evaluates it, then it creates a rectangle with a confidence score above the configured threshold.
- SC-2: Given a stroke that resembles a connector, when the stroke is recognized, then the system creates a line or connector object anchored to valid positions.
- SC-3: Given a low-confidence or ambiguous stroke, when recognition is uncertain, then it falls back to an ink or pending state instead of producing a wrong object.
- SC-4: Given a recognized shape, when the user continues editing, then the inserted object is selectable and behaves like a normal diagram element.

## Mandatory Rules (Inline)

### Recognition Rules
- Recognize only the smallest safe set of shapes in the first iteration.
- Default to conservative thresholds; do not auto-convert uncertain strokes.
- Keep recognition logic independent from the rendering layer.
- Preserve axis-alignment and geometry invariants for recognized shapes.
- Provide a visible fallback path for ambiguous or unsupported strokes.
- Maintain a simple confidence model that is easy to benchmark and tune.
- Keep input normalization consistent across mouse, touch, and pen events.
- Support undoability once a recognized object is inserted.
- Make recognition output deterministic for a given stroke sequence.
- Ensure unsupported gestures remain editable as freehand ink if the user chooses.

### Quality Rules
- No `any` types in recognition or input helpers.
- Do not hide errors in stroke processing; fail closed on malformed points.
- Add tests for high-confidence, low-confidence, and fallback recognition paths.

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
git add . && git commit -m "feat(STORY-03): add gesture recognition for shapes and connectors"
```

## Definition of Done

- [ ] Recognized shapes align to a stable geometry model.
- [ ] Connector recognition works for basic endpoint-to-endpoint gestures.
- [ ] Ambiguous strokes fall back safely instead of generating incorrect objects.
- [ ] Manual validation confirms the first draw-to-edit flow end to end.
- [ ] Unit tests cover recognition thresholds and fallback behavior.
