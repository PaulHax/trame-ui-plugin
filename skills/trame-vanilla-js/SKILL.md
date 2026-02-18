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

The `serve` dict maps a URL prefix (`__my_feature`) to the local `serve/` directory. Scripts listed in `scripts` are auto-loaded by the client.

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

## IIFE Isolation Pattern

Wrap all JS in an IIFE to avoid polluting globals:

```javascript
(function () {
  // Private state — not visible outside
  let initialized = false;
  let myState = null;

  function helperFunction() { /* ... */ }

  // Expose public API via trame.refs
  window.trame = window.trame || {};
  window.trame.refs = window.trame.refs || {};
  window.trame.refs.myController = {
    doSomething(arg) { /* ... */ },
    reset() { /* ... */ },
  };

  // Expose lifecycle hooks as globals for VTK view callbacks
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

## Reading Trame State from JS

```javascript
const value = window.trame?.state?.get?.("my_state_var");
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
