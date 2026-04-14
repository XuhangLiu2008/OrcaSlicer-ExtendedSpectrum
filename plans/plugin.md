# Controlled Variable Layer Height via Compiled `layer_height_profile`

## Summary
The workable solution in this repo is to build on `ModelObject::layer_height_profile`, not to invent a new slicing path and not to force this into `layer_config_ranges`.

Why this is the right path:
- `PrintObject::update_layer_height_profile()` already accepts a stored per-object profile and only falls back to `layer_config_ranges` when no valid profile exists.
- `PrintObject::slice()` already feeds that profile into `generate_object_layers()`, so the slicer backend already honors it.
- The profile is already persisted in `bbs_3mf` / `3mf` and already integrated with undo, background re-slicing, and the existing variable-layer-height editor.

Chosen product shape:
- Minimal UI first.
- Input is a user-defined layer table in the form `layer: layer_height`.
- Layer 1 is not editable by this feature; the table starts at layer 2 and reuses existing first-layer settings.

## Key Changes
- Add a small “Table…” entry point to the existing variable layer height UI in [src/slic3r/GUI/GLCanvas3D.cpp](/Users/thomasb/Library/Mobile Documents/com~apple~CloudDocs/Documents/develop/OrcaSlicer-ExtendedSpectrum/src/slic3r/GUI/GLCanvas3D.cpp), next to the current Adaptive / Smooth / Reset workflow.
- Open a minimal modal editor with multiline input like:
  ```text
  2: 0.16
  25: 0.12
  80: 0.20
  ```
- Define table semantics as change-points by layer index:
  - Each row means “starting at this layer, use this height”.
  - Omitted layers keep the last active height.
  - Layer numbering is 1-based in the UI, but row `1` is rejected in this first version because first-layer height remains controlled by existing print/object settings.
- Add backend helpers in [src/libslic3r/Slicing.cpp](/Users/thomasb/Library/Mobile Documents/com~apple~CloudDocs/Documents/develop/OrcaSlicer-ExtendedSpectrum/src/libslic3r/Slicing.cpp) and its header to:
  - Parse and validate the table.
  - Compile layer-index change-points into a valid stepped `layer_height_profile`.
  - Optionally reconstruct change-points from the current profile for dialog prefill by generating the current object layers and emitting rows when the effective layer height changes.
- On Apply:
  - Recompute `SlicingParameters` for the selected object.
  - Validate heights against current `min_layer_height` / `max_layer_height`.
  - Compile and store `model_object->layer_height_profile`.
  - Clear `model_object->layer_config_ranges` for that object so the object has one authoritative custom-layering source.
  - Take a snapshot, schedule background slicing, and refresh object-list variable-height state.
- Do not change the core slicing algorithm in [src/libslic3r/PrintObject.cpp](/Users/thomasb/Library/Mobile Documents/com~apple~CloudDocs/Documents/develop/OrcaSlicer-ExtendedSpectrum/src/libslic3r/PrintObject.cpp); reuse its existing profile validation and fallback behavior.

## Interfaces
- New UI input format:
  - One mapping per line: `layer_index: layer_height_mm`
  - Whitespace allowed around tokens.
  - Indices must be strictly increasing and unique.
- New backend helper surface:
  - A parser for `layer -> height` rows.
  - A compiler from layer-index change-points to `std::vector<coordf_t>` profile data.
  - A reverse helper for pre-populating the dialog from an existing profile.
- No new project file format is required; the compiled profile continues to use existing 3MF persistence.

## Test Plan
- Add unit tests near [tests/fff_print/test_printobject.cpp](/Users/thomasb/Library/Mobile Documents/com~apple~CloudDocs/Documents/develop/OrcaSlicer-ExtendedSpectrum/tests/fff_print/test_printobject.cpp) for:
  - A sparse change-point table producing the expected effective layer heights on a simple cube.
  - Repetition semantics for unspecified layers.
  - Rejection of layer `1` entries in this mode.
  - Rejection of non-monotonic indices, duplicates, malformed rows, and heights outside printer bounds.
  - Round-trip reconstruction from compiled profile back to table rows for a simple stepped case.
- Manual verification:
  - Apply a table to one FFF object, slice, and confirm preview layer progression changes as specified.
  - Save and reopen a `.3mf` and confirm the same profile remains active.
  - Confirm existing restrictions still behave as before for organic supports and multi-object prime-tower synchronization.

## Assumptions
- This feature is FFF-only.
- First-layer height remains authoritative in existing config and is out of scope for the table in v1.
- Controlled table mode and `layer_config_ranges` are mutually exclusive for a given object; applying a table clears ranges.
- Existing manual painting remains available because it uses the same `layer_height_profile`, but table application overwrites the current profile rather than merging with it.
- The topmost remainder layer continues to follow current slicer/profile behavior rather than introducing a second exact-fit algorithm in v1.
