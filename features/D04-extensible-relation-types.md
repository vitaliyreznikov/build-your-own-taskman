---
id: D04
epic: D — Task relations
title: Extensible relation types
size: S
requires: [D01]
novel: false
---

## What
The set of relation types is open-ended. New types are added through a
`RelationType` definition and a `KNOWN_TYPES` list; a relation whose type isn't
recognized is simply ignored rather than breaking the parse. So `blocks` and
`spawned` ship as known types, and more can be added without a schema migration.

## Why it exists
The relationship vocabulary will grow. Keeping types data-driven — and treating
unknown types as inert rather than fatal — means `relations.md` stays
forward-compatible: an older app can read a file that uses a newer type without
crashing.

## Acceptance criteria (EARS)
- The system shall define recognized relation types via `RelationType` /
  `KNOWN_TYPES`, and adding a new type shall not require changing the file format.
- When a relation row uses a known type, the system shall handle it fully
  (display, inverse view, editing).
- When a relation row uses an unknown type, the system shall ignore that row
  rather than failing to parse `relations.md`.

## Build notes
- `KNOWN_TYPES` is the single place to register a type; the `From | Type | To`
  shape (D01) does not change when the vocabulary does.
- "Ignore unknown" is deliberate forward-compatibility, mirroring the model's
  general tolerance of external edits — a newer type written by another tool must
  not corrupt the reader.
