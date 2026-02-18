# Trame UI Plugin

Claude Code plugin with best practices for [Trame](https://trame.readthedocs.io/) UI development. Verified against trame source code and Sebastien Jourdain's latest patterns.

## Installation

```
/plugin install trame-ui@https://github.com/PaulHax/trame-ui-plugin
```

## Skills

### trame-ui (auto-invoked for all Trame work)

General Trame 3 / Vue 3 widget patterns:

- App structure with `TrameApp` base class
- `@change` decorator, controller pattern
- Click handler sandboxed JS context (`trame.refs`, `utils.get`)
- State binding (tuples, `flushState`, `v_model` modifiers)
- Event handlers, triggers, IIFE pitfalls
- Client-side JS injection (`client.Script`, `client.JSEval`)
- ES module imports via `client.Script(module=True)`
- `raw_attrs` for Vue directives, slot destructuring
- Slots, `__events`, dynamic bindings

### trame-vanilla-js (invoked on demand)

Setting up vanilla JavaScript in a Trame app with no build step:

- Trame module system (`serve`, `scripts`, `module_scripts`, `styles`)
- IIFE isolation pattern for private state
- `trame.utils` namespace for template-accessible JS
- Inline module dict for single-file JS
- External ESM from CDN via `module_scripts`
- `js_call` (Python → JS) and `trigger` (JS → Python) communication
- `utils.get()` for browser globals in sandboxed handlers
- `state.get()` and `state.watch()` from JS
- Serial script loading for dependency order
- VTK view lifecycle hooks (`on_ready`, `before_scene_loaded`, etc.)

### trame-vue-component (invoked on demand)

Building Vue.js components for Trame with Vite and npm:

- Directory structure (`vue-components/` → `module/serve/`)
- Vite UMD build config with `external: ["vue"]`
- Multi-entry builds with `ENTRY_NAME` env var
- `vue_use` registration for Vue plugins
- `trame.utils` namespace for non-plugin UMDs
- Python widget wrapper (`AbstractElement` subclass with `_attr_names`, `_event_names`)
- Module `setup()` with vue3 assertion

