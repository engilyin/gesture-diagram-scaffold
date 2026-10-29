---
spec_id: SPEC-011
title: "Gesture Diagram API Contracts"
version: "1.0.0"
status: draft
type: technical
created: 2026-10-29
updated: 2026-10-29
authors:
  - Solution Architect Agent
  - Tech Lead Agent
related_specs:
  - SPEC-005
  - SPEC-006
---

## Overview

This document defines the public programming interface for the diagram editor and its host integrations. Because the first release is intentionally backend-optional, the primary API is the host integration contract for the widget rather than a remote REST service. Any future server-backed collaboration will implement the same interface through a backend adapter.

## Host Integration API

### `loadDocument(payload: DocumentPayload): void`
- **Purpose:** Load a diagram document into the editor.
- **Request schema:**
  ```ts
  type DocumentPayload = {
    id?: string;
    format?: 'yaml' | 'json';
    content: string;
    profile?: string;
    title?: string;
  };
  ```
- **Response:** No direct response; emits `document:loaded` or `error` events.
- **Validation:** Content must parse successfully as YAML or JSON matching the document schema.
- **Errors:** `INVALID_DOCUMENT`, `UNSUPPORTED_FORMAT`, `SCHEMA_VERSION_MISMATCH`

### `saveDocument(): string`
- **Purpose:** Return the current document as the native YAML format.
- **Response schema:**
  ```ts
  type SaveResult = string;
  ```
- **Errors:** `NO_DOCUMENT`, `SERIALIZATION_ERROR`

### `exportDocument(format: 'svg' | 'png' | 'yaml' | 'json'): Blob | string`
- **Purpose:** Export the current diagram using a supported format.
- **Request schema:**
  ```ts
  type ExportFormat = 'svg' | 'png' | 'yaml' | 'json';
  ```
- **Response:** Returns a binary blob for bitmap formats or string content for text formats.
- **Validation:** Format must be enabled in the configured plugin registry.
- **Errors:** `UNSUPPORTED_FORMAT`, `EXPORT_FAILED`

### `setProfile(name: string): void`
- **Purpose:** Switch the active profile for diagram semantics and palette behavior.
- **Request schema:**
  ```ts
  type ProfileName = string;
  ```
- **Response:** Emits `profile:changed` event.
- **Errors:** `UNKNOWN_PROFILE`

### `setTheme(theme: 'light' | 'dark' | string): void`
- **Purpose:** Update the current visual theme.
- **Request schema:**
  ```ts
  type ThemeName = 'light' | 'dark' | string;
  ```
- **Errors:** `UNKNOWN_THEME`

### `setReadOnly(readOnly: boolean): void`
- **Purpose:** Toggle edit mode for the host application.
- **Request schema:**
  ```ts
  type ReadOnlyState = boolean;
  ```
- **Response:** Emits `mode:changed` event.

## Event Model

### `document:loaded`
- **Direction:** from editor to host
- **Payload:**
  ```ts
  { id?: string; title?: string; profile?: string; }
  ```

### `document:changed`
- **Direction:** from editor to host
- **Payload:**
  ```ts
  { id?: string; revision?: number; timestamp: string; }
  ```

### `selection:changed`
- **Direction:** from editor to host
- **Payload:**
  ```ts
  { selectedIds: string[]; }
  ```

### `error`
- **Direction:** from editor to host
- **Payload:**
  ```ts
  { code: string; message: string; }
  ```

### `profile:changed`
- **Direction:** from editor to host
- **Payload:**
  ```ts
  { profile: string; }
  ```

## Adapter Contracts

### Persistence Adapter

```ts
interface PersistenceAdapter {
  init?(config?: Record<string, unknown>): Promise<void>;
  load(id?: string): Promise<string | null>;
  save(id: string, content: string): Promise<void>;
  list?(): Promise<Array<{ id: string; title?: string }>>;
  dispose?(): Promise<void>;
}
```

### Collaboration Adapter

```ts
interface CollaborationAdapter {
  connect?(url: string, identity: { id: string; name?: string; color?: string }): Promise<void>;
  sendOp(op: unknown): void;
  subscribe(listener: (op: unknown) => void): () => void;
  disconnect?(): Promise<void>;
}
```

### Importer / Exporter Plugin

```ts
interface DiagramPlugin {
  id: string;
  type: 'import' | 'export';
  format: string;
  capabilities: string[];
  load?(input: unknown): Promise<unknown>;
  save?(document: unknown): Promise<unknown>;
}
```

## Example Document Payload

```yaml
diagram: 1
profile: flowchart
meta:
  title: Signup flow
  size: [1200, 800]
layers:
  - id: main
    nodes:
      - id: start
        kind: terminal
        text: Start
        at: [40, 40]
        size: [120, 48]
    edges:
      - id: e1
        from: start
        to: next
        routing: orthogonal
```

## Error Contract

| Code | Meaning |
|------|---------|
| INVALID_DOCUMENT | Failed to parse or validate the supplied document |
| UNSUPPORTED_FORMAT | The format is not configured or available |
| SCHEMA_VERSION_MISMATCH | The document uses an unsupported schema version |
| UNKNOWN_PROFILE | The requested profile is not registered |
| EXPORT_FAILED | Export plugin failed to generate the requested artifact |
| NO_DOCUMENT | No current document is available to save or export |

## Notes
- This API is intentionally host-centric, not server-centric.
- A future backend backend can be adapted by implementing the same persistence and collaboration contracts.
- All public APIs are versioned using semver semantics and should be considered stable for the library’s supported release line.
