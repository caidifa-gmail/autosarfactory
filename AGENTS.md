# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## What this is

AutosarFactory is a Python library for reading, creating, and modifying AUTOSAR (schema r4.0) `.arxml` files. The `autosarfactory/` package targets the latest AUTOSAR release (schema `AUTOSAR_00053` = R24-11); `autosar_releases/` holds frozen snapshots for older releases. There is also a standalone MCP server under `mcp_server/` that exposes the API reference to AI agents.

## Build / Test / Lint

```bash
# Install (dev) — pulls lxml, sv-ttk, mcp[cli]
python setup.py install

# Run all tests (pytest auto-discovers tests/)
pytest

# Run a single test
pytest tests/test_autosarmodel.py::test_model_load

# Run the MCP server (separate hatchling package)
pip install "mcp[cli]"
PYTHONPATH=./mcp_server python -m autosarfactory_mcp.server

# Build the optional knowledge base for MCP semantic search
pip install sentence-transformers numpy pdfplumber
python mcp_server/kb_builder/build_knowledge_base.py --docs /path/to/specs/ --output autosar_kb.pkl
```

**Dependency note:** `requirements.txt` is stale (`ttkthemes`). The real dependencies are in `setup.py`: `lxml>=4.6.1`, `sv-ttk>=2.6.1`, `mcp[cli]`. Python >= 3.10 required. The MCP server has its own `pyproject.toml` with `mcp[cli]>=1.0` and optional `kb` extras.

## Architecture

### `autosarfactory.py` is code-generated — do NOT hand-edit the bulk of it

`autosarfactory/autosarfactory.py` is ~440k lines: one Python class per AUTOSAR meta-class, generated from the AUTOSAR XSD. The generator is not in this repo. **Treat the generated class bodies as output, not source.**

**Hand-written logic** lives in:
- The module-level functions and base classes at the top of `autosarfactory.py` (`AutosarNode`, `ARObject`, `Referrable`, `Identifiable`, `CollectableElement`, `ARElement`, plus the public API functions: `read`, `new_file`, `save`, `saveAs`, `get_node`, `get_all_instances`, `export_to_file`, `reinit`, `get_root`)
- `autosarfactory/datatype_utils.py` — int/float/bool/string value parsers
- `autosarfactory/XmlElementDirtyTracker.py` — dirty-tagging for incremental save
- `autosarfactory/autosar_ui.py` — Tk-based visualizer (sv-ttk themed)
- `autosarfactory/__init__.py` — re-exports `autosarfactory` module

**Bug-fix rule:** prefer a change in the hand-written base/runtime layer over editing a generated class.

### Class hierarchy (hand-written base classes)

```
AutosarNode          — root base: wraps lxml element, path, parent, file, referenced_by
  └─ ARObject        — adds checksum + timestamp attributes
       └─ Referrable — adds SHORT-NAME (+ fragments)
            └─ Identifiable  — adds category, UUID, admin-data, etc.
                 └─ CollectableElement / ARElement
                      └─ … (generated hierarchy) → ARPackage → AUTOSAR
```

### Module-level singleton API + global state

The library holds the entire loaded model in **module-global state** inside `autosarfactory/autosarfactory.py`:
- `__autosarNode__` — the merged `AUTOSAR` root
- `__pathsToNodeDict__` / `__autosarPathsToNodeDict__` — path → node lookup caches
- `__refBaseToArPackageDict__` — `ReferenceBase` short-label → package
- `XDT` — the single `XmlElementDirtyTracker` instance

All public functions (`read`, `get_node`, `save`, etc.) operate on this global state. **`read()` is cumulative** — multiple calls merge files into the same in-memory tree. There is no per-session isolation.

**Always call `autosarfactory.reinit()`** to wipe all caches and the root before starting an independent operation. Every test in the test suite does this in its teardown. Failing to reinit causes cross-test leakage and stale path lookups.

### lxml-backed nodes + schema ordering

Every model node wraps an lxml element (`self._node`). ARXML element order is dictated by the AUTOSAR schema sequence. Insertions use `_add_or_get_xml_node` / `_insert_element_after_given_tags` with `create_after_nodes`. **Do not** append lxml children directly — they will land in the wrong sequence and the file will not validate.

### Dirty tracking drives `save()` performance

`XmlElementDirtyTracker` (`XDT`) marks an element **and its ancestors** dirty on any mutation. `save()`/`saveAs()` only serialize dirty subtrees (the 0.6.2 optimization). When writing code that touches mutation or reference resolution, verify that:
- `XDT.mark_dirty` fires for both the mutated element and anything that references it
- The `referenced_by` back-edges are propagated (this was the 0.6.3 fix)

### Splittable / multi-file merge

`read()` deep-searches folders for `.arxml` and merges by path. Splittable duplicates merge their children; a non-splittable `Identifiable` found in two files logs a warning. `ReferenceBase` entries are resolved up front.

### Releases

`autosarfactory/` is always the latest release. `autosar_releases/autosarNNN/` are frozen snapshots — each mirrors the main package files. To model against an older schema, import that snapshot directly. `autosar_releases/` has no `__init__.py`.

### MCP server (separate concern)

`mcp_server/` is its own hatchling package (`autosarfactory-mcp`) that exposes the autosarfactory **API reference** to AI agents via pre-built JSON databases:
- `db/af_api_reference.json` — class/enum/method reference
- `db/ecuc_param_def.json` — ECUC/BSW module hierarchy (built by `paramDefDbBuilder/ecuc_module_def_to_json.py`)
- `ar_map/autosar_element_map.md` — modelling use-case element maps
- `kb/*.pkl` — optional semantic-search knowledge base from AUTOSAR spec PDFs

The MCP server does **not** introspect the live library. Tools are defined in `mcp_server/autosarfactory_mcp/server.py`.

## Key Files & Directories

| Path | Purpose |
|------|---------|
| `autosarfactory/autosarfactory.py` | ~440k lines. Generated classes + hand-written base/runtime at the top |
| `autosarfactory/XmlElementDirtyTracker.py` | Dirty-tagging engine for incremental save |
| `autosarfactory/datatype_utils.py` | Value parsers (hex, binary, octal, float, bool) |
| `autosarfactory/autosar_ui.py` | Tk-based model visualizer (sv-ttk themed) |
| `autosarfactory/__init__.py` | Re-exports the `autosarfactory` module |
| `autosar_releases/autosarNNN/` | Frozen snapshots for older AUTOSAR schema releases |
| `tests/test_autosarmodel.py` | Public API spec; ~934 lines of pytest tests |
| `tests/resources/` | Test fixtures: `.arxml` files |
| `Examples/create_autosar_basic_communication.py` | Runnable end-to-end example |
| `mcp_server/` | Standalone MCP server (separate hatchling package) |
| `setup.py` | Package build; **real dependency source** (ignore `requirements.txt`) |
| `.github/workflows/autosarfactory_testing.yml` | CI: install deps + `pytest` on push/PR to `main` |

## Coding Conventions

### Generated method naming (consistent across all classes)
- `get_<attr>` / `set_<attr>` — scalar attribute access; multi-valued refs also have `add_<ref>` / `remove_<ref>`
- `new_<Element>(name=...)` — factory that creates and attaches a child (raises if duplicate name, or if `name` omitted for `Referrable`)
- `get_children()` / `get_property_values()` — introspection
- Enumerations: `<Name>Enum` accessed as `autosarfactory.<Enum>.<MEMBER>`

### Test patterns
- Every test imports `from autosarfactory import autosarfactory`
- Every test ends with `teardown()` calling `autosarfactory.reinit()`, asserting root is None, and cleaning up temp dirs
- Test fixtures in `tests/resources/` are copied to temp dirs for save-tests, then restored
- Use `filecmp.cmp` to verify save diffs
- Tests are the public API spec — they cover load, access, modify, create, save, saveAs, path resolution, references, all instances, schema ordering, split-file merge, invalid input, reference-base, and export

### Error handling
- `InvalidInputException`, `FileExistsError`, `InvalidRefOrChildNodeException` for API errors
- `logging.info`/`logging.warning`/`logging.error` for operational messages
- `lxml` parsing errors propagate as `etree.XMLSyntaxError`

## Git Workflow

- Branch: `main`
- Remote: `https://github.com/caidifa-gmail/autosarfactory.git` (original: `girishchandranc/autosarfactory`)
- Commit style: concise imperative descriptions (e.g. "Fixed issues with not saving attributes...", "Added Autosarfactory mcp server")
- Version tags follow `v0.7.0` style (see `setup.py` version: `0.7.0`)

## CI/CD

GitHub Actions (`autosarfactory_testing.yml`): triggers on push/PR to `main`, sets up Python 3.10 on Ubuntu, installs pytest + requirements.txt, runs `pytest`.

## Tips for AI Agents

- **Global state trap:** `autosarfactory.read()` is stateful and cumulative. Always call `autosarfactory.reinit()` before independent operations. The test suite does this in every teardown — if your test doesn't, it will leak state into the next test.
- **Don't edit generated code:** `autosarfactory.py` classes are generated. Fix bugs in the hand-written base classes or helper modules instead.
- **Dirty tracking is subtle:** If you add a new mutation path, ensure `XDT.mark_dirty` fires and propagates to ancestors AND `referenced_by` back-edges. The 0.6.3 fix is the canonical example of what breaks when this is missed.
- **Schema ordering:** Never append lxml children directly. Use `_add_or_get_xml_node` or `_insert_element_after_given_tags`. Incorrect order = invalid ARXML.
- **Dependency source of truth:** `requirements.txt` is stale. Use `setup.py` for actual dependencies.
- **MCP server is read-only:** It serves pre-built JSON databases — it never imports or calls the autosarfactory library at runtime. To update the API reference the agent sees, regenerate `af_api_reference.json`.
- **Enum access pattern:** `autosarfactory.IntervalTypeEnum.VALUE_CLOSED` (module-level, not class-level).
- **Path-based access:** `autosarfactory.get_node('/Swcs/asw1/outPort')` relies on the path cache built during `read()`/`new_file()`. Paths follow the AR-PACKAGE → element → child hierarchy.

## Model State Flow

```
read(files) ──► __autosarNode__ (merged tree)
                    │
new_file(path) ──► merges into __autosarNode__
                    │
API calls ──► modify lxml nodes + XDT.mark_dirty()
                    │
save([files]) ──► only dirty subtrees serialized
saveAs(file)  ──► full merged tree to single file
                    │
reinit() ──► wipes everything
```
