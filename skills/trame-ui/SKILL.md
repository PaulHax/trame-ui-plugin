---
name: trame-ui
description: Trame Python-to-web UI patterns. Read BEFORE writing any trame widget code, inline JS event handlers, Vuetify components, state binding, or client-side JavaScript.
---

# Trame UI

## Related Skills

- When setting up a **vanilla JS module with no build step** (IIFE, trame module serving, js_call/trigger), invoke the **trame-vanilla-js** skill.
- When building a **Vue.js/JS component with Vite/npm** (UMD bundle, npm dependencies), invoke the **trame-vue-component** skill.

## App Structure (Trame 3 / Vue 3)

```python
from trame.app import TrameApp
from trame.decorators import change
from trame.ui.vuetify3 import SinglePageLayout
from trame.widgets import vuetify3 as v3

class MyApp(TrameApp):
    def __init__(self, server=None, **kwargs):
        super().__init__(server, client_type="vue3", **kwargs)
        self.state.my_value = 10
        self._build_ui()

    def _build_ui(self):
        with SinglePageLayout(self.server, full_height=True) as self.ui:
            with self.ui.content:
                pass  # UI here
```

## @change Decorator

```python
@change("resolution")
def _on_resolution_change(self, resolution, **_):
    self.cone_source.SetResolution(int(resolution))
    self.ctrl.view_update()

@change("contour_value", "opacity")  # Multiple variables
def _on_visual_change(self, contour_value, opacity, **_): ...
```

## Controller Pattern

```python
view = vtk_widgets.VtkLocalView(self.render_window)
self.ctrl.view_update = view.update
self.ctrl.view_reset_camera = view.reset_camera

v3.VBtn(icon="mdi-crop-free", click=self.ctrl.view_reset_camera)
```

## Context Pattern

```python
vtk_widgets.VtkLocalView(self.render_window, ctx_name="view")

v3.VBtn(icon="mdi-crop-free", click=self.ctx.view.reset_camera)
```

## Click Handler Context

Click handlers run in sandboxed JS. `$refs`, `document`, `window` are undefined.

```python
click="trame.refs.myRef.$el.querySelector('input').click()"  # Access refs
click="utils.get('document').getElementById('myId').click()"  # Access DOM
click="utils.download('data.zip', trigger('get_data'), 'application/zip')"
data=utils.safe(obj)  # Strip non-serializable properties
```

## File Input with Custom Button

```python
v3.VFileInput(v_model=("file_var", None), ref="fileInput", style="display: none;")
v3.VBtn(click="trame.refs.fileInput.$el.querySelector('input').click()")
```

## State Binding

**Two-element tuple** - state with default:

```python
v_model=("search_query", "")  # Creates state.search_query
```

**Single-element tuple** - reactive expression (no state registration):

```python
model_value=("runs[id].value",)  # Note trailing comma
v_if=("items.length > 0",) # Optional since directive (v_*) are always expressions
```

**Controlled binding** - intercept changes:
```python
v3.VSelect(
    model_value=("runs[id].value",),
    update_modelValue=(self.ctrl.handler, "[id, $event]"),
)
```

**flushState** for nested state sync:

```python
update_modelValue="values[idx] = $event; flushState('values');"
```

**v_model modifiers**:

```python
v_model_number=("maxValue", 10)  # v-model.number
v_model_lazy=("text", "")        # v-model.lazy
v_model_trim=("input", "")       # v-model.trim
v_model_title=("pageTitle",)     # v-model:title (named, Vue 3)
```

## Event Handlers

```python
click=handler                           # No args
click=(handler, "[a, b]")               # With JS args
click=(handler, "[a, b]", "{c, d}")     # Call handler with args and kwargs
click="state_var = 1"                   # Inline JS
click="trigger('name', [args])"         # Named trigger
update_modelValue=(handler, "[$event]") # v-model event
```

**Register trigger**:

```python
from trame.app import TrameApp
from trame.decorators import trigger

class App(TrameApp):
    @trigger("exec_prog")
    def exec_function(self, *args, **kwargs): ...
```

**IIFE patterns don't work** - Immediately Invoked Function Expressions fail silently in event handlers:
```python
# WRONG - IIFE doesn't work, handler won't fire
click="((f) => { trigger('foo', [], {val: f}); })($event)"

# CORRECT - use inline statements instead
click="if (condition) { trigger('foo', [], {val: $event}); }"
```

**Throttle example** using `utils.get('window')` for state:
```python
update_model_value="if (!utils.get('window')._lastCall || Date.now() - utils.get('window')._lastCall > 66) { utils.get('window')._lastCall = Date.now(); trigger('handler', [], {value: $event}); }"
```

## Client-Side JavaScript

For any non-trivial JS, use a **trame module** to serve `.js` files directly instead of inline strings. See the **trame-vanilla-js** skill for the full pattern (IIFE isolation, `trame.utils` namespace, `js_call`/`trigger` communication).

Quick inline module loading for external ESM from CDN:
```python
self.server.enable_module({"module_scripts": ["https://esm.sh/canvas-confetti@1.9.3"]})
```

**`client.JSEval`** - does NOT auto-execute. Only runs when `.exec()` is called from the server:
```python
client.JSEval(ctx_name="js_eval", exec="window.doStuff($event)")
self.ctx.js_eval.exec("hello")  # Trigger execution
```

## raw_attrs

For Vue directives not wrapped by trame:

```python
html.Div(raw_attrs=["@click.stop", "@mousedown.stop"])
html.Div(raw_attrs=['v-click-outside="() => { expanded = null }"'])
```

**Slot destructuring** - quoting is critical:
```python
# CORRECT: double quotes around the right side of the expression
with v3.Template(
    raw_attrs=['v-slot:header.col="{ column, toggleSort }"'],
):
    html.Div(raw_attrs=['@click="toggleSort(column)"'])

# WRONG: missing quotes generates invalid template
raw_attrs=["v-slot:header.col={ column }"]  # Bad!
```

## Slots

```python
with v3.VMenu():
    with v3.Template(v_slot_activator="{ props }"):
        v3.VBtn(v_bind="props")
    v3.VList()  # Menu content
```

## v_on

Expose any event handling

```python
v3.VSlider(
    v_on_end=..., # @end=
    v_on_end_stop=..., # @end.stop=
)
```


## __events

Expose non-default Vuetify events:
```python
v3.VSlider(
    __events=["end"],
    end=(self.on_release, "[$event]"),
)
```

## Debug Template

```python
with v3.Template(raw_attrs=['v-slot:item="{ item }"']) as t:
    html.Div("Content")
print("TEMPLATE:", t)
```

## Dynamic Bindings

```python
to=("`/bar/${item.id}`",)  # JS template literal in single-element tuple
```
