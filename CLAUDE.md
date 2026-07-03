# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`reapy` is a pythonic Python wrapper around REAPER's ReaScript Python API. It lets users control REAPER from both inside (as a ReaScript) and outside (from a regular Python shell) using the same API.

## Commands

```bash
# Install in development mode
pip install -e .

# Check PEP style compliance
pycodestyle reapy/

# Build and publish docs (requires sphinx)
python update_docs.py
```

There is no test suite.

## Architecture

### Inside vs. outside REAPER

The central design concern is that the same code must work in two contexts:

- **Inside REAPER**: `reapy.is_inside_reaper()` returns `True`. API calls go directly to `reaper_python` (REAPER's built-in Python module). Functions execute synchronously with native performance.
- **Outside REAPER**: `reapy.is_inside_reaper()` returns `False`. API calls are serialized and sent over the network to a running REAPER instance. Throughput is limited to ~30–60 calls/second.

### Network transport (outside REAPER)

```
External Python → Client → WebInterface (port 2307) → reapy Server (port 2306) → REAPER
```

- `reapy/tools/network/web_interface.py` — discovers the reapy server port via REAPER's built-in HTTP interface
- `reapy/tools/network/client.py` — sends serialized function calls over TCP
- `reapy/tools/network/server.py` — runs inside REAPER in a `defer` loop, processes requests
- `reapy/tools/network/machines.py` — manages connections; `reapy.connect(host)` selects a target machine
- `reapy/reascripts/activate_reapy_server.py` — the ReaScript REAPER runs to start the server

### `@reapy.inside_reaper()` decorator / context manager (`reapy/tools/_inside_reaper.py`)

This is the key performance primitive. When wrapping a function or used as a context manager, it batches all reapy calls inside a single network round-trip (HOLD/RELEASE protocol). Properties on core objects use `DistProperty` (a subclass of `property`) to transparently dispatch to REAPER.

### `reapy.map()` (`reapy/core/map.py`)

For calling the same function many times (thousands+), `reapy.map()` sends all arguments in one network call and receives all results at once — much faster than a loop with `inside_reaper`.

### `reapy.reascript_api` (`reapy/reascript_api.py`)

- **Inside REAPER**: imports all `RPR_*` functions from `reaper_python`, strips the `RPR_` prefix, exposes them as `reapy.reascript_api.*`. Also imports SWS extension functions if available.
- **Outside REAPER**: dynamically generates proxy functions decorated with `@reapy.inside_reaper()` for each name in `__all__`.

### High-level API (`reapy/core/`)

Pythonic object wrappers over ReaScript primitives:

| Module | Classes |
|--------|---------|
| `core/project/` | `Project`, `Marker`, `Region`, `TimeSelection` |
| `core/track/` | `Track`, `Send`, `AutomationItem` |
| `core/item/` | `Item`, `Take`, `Source`, `MidiEvent` |
| `core/fx/` | `FX`, `FXParam` |
| `core/envelope.py` | `Envelope` |
| `core/audio_accessor.py` | `AudioAccessor` |
| `core/window/` | `Window`, `MIDIEditor`, `Tooltip` |
| `core/reaper/` | Module-level functions (`reaper.py`, `audio.py`, `defer.py`, `midi.py`, `ui.py`) |

All core objects inherit from `ReapyObject` (`core/reapy_object.py`), which provides `__repr__`, `__eq__`, and serialization via `_to_dict` (used by the network layer to reconstruct objects on the other side).

### Configuration (`reapy/config/`)

`reapy.configure_reaper()` edits REAPER's `.ini` files directly to:
1. Enable Python in ReaScript
2. Add a Web Interface on port 2307
3. Register `activate_reapy_server.py` in REAPER's Actions list

## Conventions

- **Style**: PEP 8 (check with `pycodestyle`)
- **Docstrings**: numpy docstring format
- **Type stubs**: Every `module.py` has a `module.pyi` sibling. `.pyi` files are not the top priority — contributors may leave them for maintainers to update.
- **Changelog**: All API changes (additions, removals, fixes) must be recorded in the `Unreleased` section of `CHANGELOG.md` following [keepachangelog.com](https://keepachangelog.com/en/1.0.0/) conventions.
