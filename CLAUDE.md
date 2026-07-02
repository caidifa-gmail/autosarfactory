# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

AutosarFactory is a Python library for reading, creating, and modifying AUTOSAR (schema r4.0) `.arxml` files. The top-level `autosarfactory/` package targets the latest AUTOSAR release (schema `AUTOSAR_00053` = R24-11); `autosar_releases/` holds frozen snapshots for older releases.

## Commands

```bash
# Install (dev) — pulls lxml, sv-ttk, mcp[cli]
python setup.py install

# Tests — run from repo root. pytest auto-discovers tests/.
pytest
pytest tests/test_autosarmodel.py::test_model_load   # single test

# Launch the Tk visualizer on a loaded model
#   autosarfactory.show_in_ui(root)  (see Examples/ for a runnable script)

# MCP server (separate project under mcp_server/, hatchling build)
pip install "mcp[cli]"
claude mcp add autosarfactory -e PYTHONPATH=./mcp_server -- python -m autosarfactory_mcp.server
# stdio/SSE/HTTP transport options — see mcp_server/README.md
```

Dependencies: the live code imports `lxml` (core) and `sv_ttk` (UI). Note `requirements.txt` is stale (`ttkthemes`) — `setup.py` is the source of truth (`lxml>=4.6.1`, `sv-ttk>=2.6.1`, `mcp[cli]`).

## Architecture

### `autosarfactory.py` is code-generated — do not hand-edit the bulk of it
[autosarfactory/autosarfactory.py](autosarfactory/autosarfactory.py) is ~440k lines: one Python class per AUTOSAR meta-class, generated from the AUTOSAR XSD (the generator is not in this repo). Treat the class bodies as output, not source. **Hand-written logic** lives in:
- The module-level functions and base classes at the top of `autosarfactory.py` (`AutosarNode` and the modeling API).
- [datatype_utils.py](autosarfactory/datatype_utils.py), [XmlElementDirtyTracker.py](autosarfactory/XmlElementDirtyTracker.py), [autosar_ui.py](autosarfactory/autosar_ui.py), [__init__.py](autosarfactory/__init__.py).

When fixing a bug, prefer a change in the hand-written base/runtime layer over editing a generated class.

### Module-level singleton API + global state
The library holds the entire loaded model in **module-global state** inside `autosarfactory.py`:
- `__autosarNode__` — the merged `AUTOSAR` root.
- `__pathsToNodeDict__` / `__autosarPathsToNodeDict__` — path → node lookup caches (powers `get_node`).
- `__refBaseToArPackageDict__` — `ReferenceBase` short-label → package.
- `XDT` — the single `XmlElementDirtyTracker` instance.

Public entry points operate on this state: `read(files)` (cumulative — multiple calls / multiple files **merge** into one in-memory tree), `new_file`, `get_node(path)`, `get_all_instances(node, clazz)`, `export_to_file`, `save`, `saveAs`, `get_root`, and `reinit()`.

**`read()` is stateful and cumulative**, and there is no per-session isolation. Call `autosarfactory.reinit()` to wipe all caches and the root before starting an independent operation — the test suite does this in every teardown. Failing to reinit causes cross-test leakage and stale path lookups.

### Modeling API conventions (generated, consistent across every class)
All schema classes derive from a small base hierarchy: `AutosarNode → ARObject → Referrable → Identifiable → CollectableElement / ARElement → … → ARPackage → AUTOSAR`. The generated methods follow strict naming patterns:
- `get_<attr>` / `set_<attr>` — attribute access. Multi-valued references also have `add_<ref>` / `remove_<ref>`.
- `new_<Element>(name=...)` — factory on the parent that creates and attaches a child (raises if a child of that name already exists, or if `name` is omitted for a `Referrable`).
- `get_children()`, `get_property_values()` — introspection.
- Enumerations: `<Name>Enum` (e.g. `IntervalTypeEnum`), members accessed as `autosarfactory.<Enum>.<MEMBER>`.

### lxml-backed nodes + schema ordering
Every model node wraps an lxml element (`self._node`). ARXML element order is dictated by the AUTOSAR schema sequence; insertions go through `_add_or_get_xml_node` / `_insert_element_after_given_tags` with `create_after_nodes` to stay schema-compliant. Don't append lxml children directly — they'll land in the wrong sequence and the file won't validate.

### Dirty tracking drives `save()` performance
`XmlElementDirtyTracker` (`XDT`) marks an element **and its ancestors** dirty on any mutation; `save()`/`saveAs()` only serialize dirty subtrees instead of rewriting the whole tree (the 0.6.2 perf optimization). This is subtle: 0.6.3 fixed referenced elements being dropped because dirty propagation missed the `referenced_by` back-edges. If you touch anything around `_save_*`, mutation, or reference resolution, verify that newly-created and newly-referenced elements still get written — `XDT.mark_dirty` must fire for both the element and anything that references it.

### Splittable / multi-file merge
`read()` deep-searches folders for `.arxml` and merges files by path. Splittable duplicates merge their children; a non-splitable `Identifiable` found in two files is logged as a warning and not duplicated. `ReferenceBase` entries are resolved up front and registered in `__refBaseToArPackageDict__`.

### Releases
`autosarfactory/` is always the latest release. `autosar_releases/autosarNNN/` (e.g. `autosar422` → `AUTOSAR_00046`, `autosar453` → `AUTOSAR_00053` = R24-11) are frozen snapshots — each mirrors the main package (`autosarfactory.py` + `datatype_utils.py` + `XmlElementDirtyTracker.py` + `__init__.py`). There is no runtime release switch; to model against an older schema, import that snapshot directly. `autosar_releases/` has no `__init__.py`.

### MCP server (separate concern)
[mcp_server/](mcp_server/) is its own hatchling package (`autosarfactory-mcp`) that exposes the autosarfactory **API reference** to AI agents. It does **not** introspect the live library — it serves pre-built JSON databases:
- `db/af_api_reference.json` — class/enum/method reference (the agent's source of truth).
- `db/ecuc_param_def.json` — ECUC/BSW module → container → parameter hierarchy (built by `paramDefDbBuilder/ecuc_module_def_to_json.py`).
- `ar_map/autosar_element_map.md` — which AUTOSAR elements belong to each modelling use case (edited as Markdown, no code change).
- `kb/*.pkl` — optional semantic-search knowledge base built from AUTOSAR spec PDFs via `kb_builder/build_knowledge_base.py` (needs the `kb` extra: `sentence-transformers`, `numpy`, `pdfplumber`).

Editing the MCP tools happens in [mcp_server/autosarfactory_mcp/server.py](mcp_server/autosarfactory_mcp/server.py); regenerating the reference DB happens in `kb_builder/`.

## Tests

[tests/test_autosarmodel.py](tests/test_autosarmodel.py) is the spec for the public API. Fixtures live in [tests/resources/](tests/resources/) (`components.arxml`, `datatypes.arxml`, `interfaces.arxml`, `diag_sw_mapping.arxml`, plus `split1/split2.arxml` and `invalid.arxml` for merge/merge and error cases). Every test ends by calling the module-level `teardown()` helper, which calls `autosarfactory.reinit()` — any new test must do the same to avoid polluting global state. A runnable end-to-end example is in [Examples/create_autosar_basic_communication.py](Examples/create_autosar_basic_communication.py).
