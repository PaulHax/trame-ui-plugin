---
name: trame-vanilla-js
description: Setting up a vanilla JavaScript module in a Trame app with no build step. IIFE isolation, trame module serving, js_call/trigger communication, lifecycle hooks.
---

# Trame Vanilla JS Module (No Build Step)

For adding isolated JavaScript to a Trame app without npm/Vite.

## Directory Structure

```
my_feature/
├── my_feature.py              # Python class
└── module/
    ├── __init__.py            # Trame module registration
    └── serve/
        └── my_feature.js      # Vanilla JavaScript
```

## Module Registration

`module/__init__.py`:
```python
from pathlib import Path

serve_path = str(Path(__file__).with_name("serve").resolve())
serve = {"__my_feature": serve_path}
scripts = ["__my_feature/my_feature.js"]
styles = []

def setup(server, **kwargs):
    pass
```

The `serve` dict maps a URL prefix to the local `serve/` directory. The double-underscore prefix (`__my_feature`) is a convention for namespacing, not required. Scripts listed in `scripts` are loaded as `<script>` tags. Scripts load in parallel by default — if load order matters, use serial groups:
```python
scripts = [
    ["__my_feature/dependency.js", {"serial": "my_group"}],
    ["__my_feature/my_feature.js", {"serial": "my_group"}],
]
```

There is also `module_scripts` for loading as `<script type="module">`, but `scripts` with IIFE is the standard Kitware pattern.

## Enabling the Module in Python

```python
from my_feature import module as my_feature_module

def _inject_scripts(self):
    self.server.enable_module(my_feature_module)
    # Inline CSS if needed:
    self.server.enable_module(
        {"styles": ["data:text/css,.my-class { overflow: hidden; }"]}
    )
```

## Inline Module Dict (Simple Alternative)

For a single JS file, skip the `module/` directory and use an inline dict:

```python
from pathlib import Path

def _inject_scripts(self):
    js_file = Path(__file__).with_name("my_utils.js").resolve()
    self.server.enable_module(
        dict(
            serve={"my_code": str(js_file.parent)},
            scripts=[f"my_code/{js_file.name}"],
        )
    )
```

## External ESM from CDN

Load external ES modules without any local files:

```python
self.server.enable_module({"module_scripts": ["https://esm.sh/canvas-confetti@1.9.3"]})
```

## trame.utils Namespace Pattern

Extend `window.trame.utils` so trame event handlers can access your code directly:

```javascript
window.trame.utils.my_code = {
  actions: {
    hello(name) { alert("Hello " + name); },
    compute(x, y) { return x + y; },
  },
  rules: {
    number(v) { return !isNaN(parseFloat(v)) || "Must be a number"; },
  },
};
```

Access in trame templates:
```python
click="utils.my_code.actions.hello('world')"
rules=("utils.my_code.rules.number",)
```

## IIFE Isolation Pattern

For larger JS with private state, wrap in an IIFE:

```javascript
(function () {
  let initialized = false;
  let myState = null;

  function helperFunction() { /* ... */ }

  // Expose public API via trame.refs (for js_call from Python)
  window.trame = window.trame || {};
  window.trame.refs = window.trame.refs || {};
  window.trame.refs.myController = {
    doSomething(arg) { /* ... */ },
    reset() { /* ... */ },
  };

  // Also expose via trame.utils (for event handler access)
  window.trame.utils = window.trame.utils || {};
  window.trame.utils.my_feature = { actions: { /* ... */ } };

  // Lifecycle hooks as globals for VTK view callbacks
  window.initMyFeature = async function () {
    const vtkViewRef = window.trame?.refs?.["myVtkView"];
    if (!vtkViewRef) {
      setTimeout(window.initMyFeature, 100);
      return;
    }
    initialized = true;
  };

  window.onMyFeatureStateChange = function (state) { /* ... */ };
})();
```

## JS → Python Communication (trigger)

```javascript
if (window.trame?.trigger) {
  window.trame.trigger("my_event", [], { key: "value" });
}
```

Python handler:
```python
@self.ctrl.trigger("my_event")
def _on_my_event(key=None, **kwargs):
    print(f"Got: {key}")
```

## Python → JS Communication (js_call)

Call methods on objects exposed via `window.trame.refs`:

```python
self.server.js_call("myController", "doSomething", arg1, arg2)
```

This calls `window.trame.refs.myController.doSomething(arg1, arg2)`.

## Accessing Browser Globals from Event Handlers

Event handlers run in a sandbox where `window`, `document`, `fetch` are undefined. Use `utils.get()`:

```python
click="utils.get('fetch')(url).then(v => v.json()).then(v => (response = v))"
click="utils.get('document').getElementById('myId').click()"
click="utils.get('window').myGlobal()"
```

## Reading Trame State from JS

```javascript
const value = window.trame?.state?.get?.("my_state_var");

// Watch for state changes:
window.trame.state.watch(["my_var"], (my_var) => { /* react */ });
```

## VTK View Lifecycle Hooks

Connect JS functions to VTK view events via inline expressions:

```python
vtk_widgets.VtkSharedSyncView(
    self.render_window,
    ref="myVtkView",
    on_ready="window.initMyFeature && window.initMyFeature()",
    before_scene_loaded="window.onBeforeScene && window.onBeforeScene()",
    after_scene_loaded="window.onAfterScene && window.onAfterScene()",
    view_state_change="window.onMyFeatureStateChange && window.onMyFeatureStateChange($event)",
)
```

The `&&` guard prevents errors if the JS hasn't loaded yet.
