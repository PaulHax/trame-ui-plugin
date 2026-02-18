# Trame UI Plugin

Claude Code plugin with best practices for [Trame](https://trame.readthedocs.io/) UI development with Vuetify 3 components.

Claude automatically uses this skill when working on Trame widgets, Vuetify components, click handlers, refs, and state management.

## Installation

```
/plugin install trame-ui@https://github.com/PaulHax/trame-ui-plugin
```

## What's covered

- App structure (Trame 3 / Vue 3)
- `@change` decorator
- Controller pattern
- Click handler sandboxed JS context (`trame.refs`, `utils.get`)
- State binding (tuples, `flushState`, `v_model` modifiers)
- Event handlers and triggers
- IIFE pitfalls in event handlers
- Client-side JavaScript (`client.Script`, `client.JSEval`)
- ES module imports via `client.Script(module=True)`
- `raw_attrs` for Vue directives, slot destructuring
- Slots, `__events`, dynamic bindings

## Authors

- Paul Elliott
