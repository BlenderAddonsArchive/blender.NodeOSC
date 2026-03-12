# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NodeOSC is a Blender addon (v2.4.1) enabling real-time OSC (Open Sound Control) communication between Blender and external applications via UDP. It supports both receiving (mapping OSC messages to Blender properties) and sending (broadcasting Blender property values as OSC messages).

No build system, package manager, or test runner — this is a traditional Blender addon. Development cycle is: edit files → reload addon in Blender (Edit > Preferences > Addons, or use a script reloader).

All OSC library dependencies are **vendored** inside `server/oscpy/` and `server/pythonosc/`. No `pip install` required.

## Installation / Development Setup

1. Symlink or copy this folder into Blender's addons directory: `<blender>/scripts/addons/NodeOSC/`
2. Enable the addon in Blender: Edit > Preferences > Add-ons > search "NodeOSC"
3. Optional: Install [Animation Nodes](https://github.com/JacquesLucke/animation_nodes) or [Sorcar](https://github.com/aachman98/Sorcar) for node integration

To reload after code changes without restarting Blender, use the [Blender Development VSCode extension](https://marketplace.visualstudio.com/items?itemName=JacquesLucke.blender-development) or run in Blender's Python console:
```python
import importlib, NodeOSC
importlib.reload(NodeOSC)
```

## Architecture

### Layered Structure

```
__init__.py          → Addon entry point, class registration orchestration
preferences.py       → NodeOSCPreferences (addon-level) + NodeOSCEnvVarSettings (scene-level)
server/
  _base.py           → OSC_OT_OSCServer: base modal operator, core message routing logic
  server.py          → OSC_OT_PythonOSCServer + OSC_OT_OSCPyServer (two library backends)
  callbacks.py       → Thread-safe LIFO queue + callback execution system
  operators.py       → UI operators (create/delete/move handlers, import/export JSON)
  oscpy/             → Vendored oscpy library
  pythonosc/         → Vendored python-osc library
ui/panels.py         → Three View3D sidebar panels: Settings, Operations (custom handlers), Nodes
nodes/
  nodes.py           → Scans all node trees, builds NodeOSC_nodes / NodeOSC_outputs collections
  AN/                → Animation Nodes integration (auto-loaded if AN installed)
  sorcar/            → Sorcar integration (platform-aware imports)
utils/
  keys.py            → NodeOSCMsgValues PropertyGroup — the data model for each handler
  utils.py           → Shared enums (direction, node type, etc.)
```

### Data Flow

**Input (OSC → Blender):**
OSC packet received on UDP → `OSC_callback_pythonosc()` [network thread] → `OSC_callback_queue.put()` [LIFO queue] → Blender timer fires every `input_rate` ms → `execute_queued_OSC_callbacks()` [main thread] → `eval()`/`exec()` modifies `bpy.data` properties

**Output (Blender → OSC):**
Blender modal timer fires every `output_rate` ms → `make_osc_messages()` reads `bpy.data` datapaths → optional `dp_format` evaluation → repeat filter check → UDP send

### Handler Type System

Handlers are categorized by integer type codes (1–11) in `_base.py`:
- **1**: Custom properties with `getValue()`
- **2**: Single scalar property (float/int/string)
- **3**: Dict-style custom properties `obj['key']`
- **4**: Array/vector properties
- **5–6**: Node-based handlers (single/list)
- **7**: Function calls
- **10**: Format-string enabled handlers with loop support
- **11**: Script function handlers (`script(textblock).function`)

### Key Data Model (`utils/keys.py`)

`NodeOSCMsgValues` PropertyGroup fields:
- `osc_address`: OSC address pattern (e.g., `/instrument/1/level`)
- `osc_direction`: INPUT / OUTPUT / BOTH
- `data_path`: Full Blender datapath (e.g., `bpy.data.objects['Cube'].location[0]`)
- `dp_format_enable` + `dp_format`: Python expression for value transformation
- `filter_enable` + `filter_eval`: Python expression to accept/reject messages
- `loop_enable` + `loop_range`: Process multiple arguments iteratively

Collections stored on the scene:
- `bpy.context.scene.NodeOSC_keys` — user-defined handlers
- `bpy.context.scene.NodeOSC_nodes` — auto-generated from node trees
- `bpy.context.scene.NodeOSC_outputs` — merged set of all output handlers
- `bpy.context.scene.nodeosc_envars` — network config and runtime state

### Threading Model

The OSC server runs in a background thread. **Never modify `bpy.data` from the OSC callback thread.** All callbacks push to `OSC_callback_queue` (a `queue.LifoQueue`) and are processed in the main thread by the modal operator's timer handler. The LIFO ordering ensures the latest message wins when messages arrive faster than Blender processes them.

### Node Integration Pattern

Both AN and Sorcar nodes implement:
- `setValue(value)` — called by the OSC callback to update node data
- `getValue()` — called by the output system to read node data

Node trees are scanned at server start and after changes via `nodes_createCollections()` in `nodes/nodes.py`.

## OSC Library Selection

The addon supports two vendored OSC libraries switchable in addon preferences (`NodeOSCPreferences.usePyLiblo`):
- **python-osc** (default): More mature, production-ready
- **oscpy**: Simpler, supports wildcard address matching with `*`

## Format Strings and Filters

In `dp_format` and `filter_eval` expressions, these variables are available:
- `args` — list of all OSC arguments
- `addr` — OSC address split by `/` (list of segments)
- `address` — full OSC address string
- `length` — number of arguments
- `index` — current loop index (when looping)
