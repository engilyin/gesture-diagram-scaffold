---
spec_id: SPEC-012
title: "Gesture Diagram Data Model"
version: "1.0.0"
status: draft
type: technical
created: 2026-10-29
updated: 2026-10-29
authors:
  - Tech Lead Agent
  - Solution Architect Agent
related_specs:
  - SPEC-005
  - SPEC-006
---

## Overview

The data model describes the canonical document state used by the editor. It is designed to be stable, serializable, and extensible while remaining easy to reason about in text and code.

## Core Entities

### Diagram
- **Purpose:** Top-level document container.
- **Fields:**
  - `id: string` — stable document ID
  - `schemaVersion: number` — version of the model
  - `profile: string` — active diagram profile
  - `layers: Layer[]` — ordered collection of layers
  - `assets: Asset[]` — attached media and libraries
  - `meta: DiagramMeta` — title, size hints, background, theme
- **Relationships:** Owns `layers`, `assets`, `edges`, and `groups` indirectly through the model graph.

### Layer
- **Purpose:** An ordered, toggleable container for top-level items.
- **Fields:**
  - `id: string`
  - `name: string`
  - `visible: boolean`
  - `locked: boolean`
  - `order: string` — fractional-index ordering
  - `opacity: number`
- **Relationships:** Contains `nodes` and `edges` by reference or nested lookup.

### Node
- **Purpose:** A visual semantic object with geometry and styling.
- **Fields:**
  - `id: string`
  - `kind: string`
  - `layerId: string`
  - `parentId?: string`
  - `geometry: Geometry`
  - `style: Style`
  - `text?: string`
  - `ports?: Port[]`
  - `data?: Record<string, unknown>`
- **Relationships:** Can belong to a layer or be nested in a container/parent node.

### Edge
- **Purpose:** A connector or relationship between two endpoints.
- **Fields:**
  - `id: string`
  - `source: Endpoint`
  - `target: Endpoint`
  - `routing?: 'orthogonal' | 'straight' | 'curve'`
  - `waypoints?: Point[]`
  - `style: Style`
  - `label?: string`
- **Relationships:** References node IDs via source and target endpoints.

### Group
- **Purpose:** A logical bundle of nodes or edges that move together.
- **Fields:**
  - `id: string`
  - `memberIds: string[]`
  - `label?: string`
- **Relationships:** Not a layout container; purely organizational.

### Asset
- **Purpose:** External media or embedded file references.
- **Fields:**
  - `id: string`
  - `mimeType: string`
  - `source: 'embedded' | 'url'`
  - `url?: string`
  - `hash?: string`
  - `license?: string`

### Style
- **Purpose:** Encapsulates visual appearance.
- **Fields:**
  - `fill?: string`
  - `stroke?: string`
  - `strokeWidth?: number`
  - `fontFamily?: string`
  - `fontSize?: number`
  - `fontWeight?: number`
  - `opacity?: number`
  - `themeToken?: string`

## Core Geometry and Metadata

### Geometry
```ts
type Geometry = {
  x: number;
  y: number;
  width: number;
  height: number;
  rotation?: number;
};
```

### Endpoint
```ts
type Endpoint = {
  nodeId: string;
  portId?: string;
};
```

### DiagramMeta
```ts
type DiagramMeta = {
  title?: string;
  background?: string;
  baseSize?: { width: number; height: number };
  theme?: string;
  activeProfile?: string;
};
```

## Constraints and Invariants

- Each entity must have a stable ID.
- No containment cycles are permitted between parent and child nodes.
- A group must remain a logical grouping and must not impose layout semantics by itself.
- Connectors must reference valid source and target endpoints.
- Ordering is represented with fractional indexes rather than integer indices where concurrent reorder is expected.
- The document must support versioned schema migrations.

## Indexes and Storage

- **Storage:** YAML document on disk or in browser storage; optional JSON transport representation.
- **Recommended indexes:**
  - by `id`
  - by `layerId`
  - by `parentId`
  - by `source.nodeId` and `target.nodeId` for connector lookup

## Lifecycle

- The model is created from a native YAML document or a parsed importer result.
- The renderer reads the model and updates visual output.
- User interactions produce operations that mutate the model.
- Save/export serializes the model in canonical order.

## Summary

The data model is intentionally structured for adaptability and future collaboration. It keeps the runtime model clean, preserves semantic relationships, and allows the document to move between local storage, import/export plugins, and future server-backed adapters without major rewrites.
