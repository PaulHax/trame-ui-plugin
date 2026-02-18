---
name: trame-ui
description: Trame UI development, Vuetify components, click handlers, refs, state management. Use for Trame widgets, vuetify3, trame-client JS context.
---

# Trame UI

## App Structure (Trame 3 / Vue 3)

```python
from trame.app import TrameApp
from trame.decorators import change
from trame.ui.vuetify3 import SinglePageLayout
from trame.widgets import vuetify3

class MyApp(TrameApp):
    def __init__(self, server=None, **kwargs):
        super().__init__(server, client_type="vue3", **kwargs)
        self.state.my_value = 10
        self._build_ui()

    def _build_ui(self):
        with SinglePageLayout(self.server, full_height=True) as self._ui:
            with self._ui.content:
                pass  # UI here
```

## @change Decorator

```python
@change("resolution")
def _on_resolution_change(self, resolution, **kwargs):
    self.cone_source.SetResolution(int(resolution))
    self.ctrl.view_update()

@change("contour_value", "opacity")  # Multiple variables
def _on_visual_change(self, contour_value, opacity, **kwargs): ...
```

## Controller Pattern

```python
view = vtk_widgets.VtkLocalView(self.render_window)
self.ctrl.view_update = view.update
self.ctrl.view_reset_camera = view.reset_camera
self.ctrl.on_server_ready.add(self.ctrl.view_update)

vuetify3.VBtn(icon="mdi-crop-free", click=self.ctrl.view_reset_camera)
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
vuetify3.VFileInput(v_model=("file_var", None), ref="fileInput", style="display: none;")
vuetify3.VBtn(click="trame.refs.fileInput.$el.querySelector('input').click()")
```

## State Binding

**Two-element tuple** - state with default:
```python
v_model=("search_query", "")  # Creates state.search_query
```

**Single-element tuple** - reactive expression (no state registration):
```python
model_value=("runs[id].value",)  # Note trailing comma
v_if=("items.length > 0",)
```

**Controlled binding** - intercept changes:
```python
vuetify3.VSelect(
    model_value=("runs[id].value",),
    update_modelValue=(self.ctrl.handler, r"[id, $event]"),
)
```

**flushState** for nested state sync:
```python
update_modelValue=f"values[idx] = $event; flushState('values');"
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
click="state_var = 1"                   # Inline JS
click="trigger('name', [args])"         # Named trigger
update_modelValue=(handler, "[$event]") # v-model event
```

**Register trigger**:
```python
@ctrl.trigger("exec_prog")
def exec_function(*args, **kwargs): ...
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

## raw_attrs

For Vue directives not wrapped by trame:

```python
html.Div(raw_attrs=["@click.stop", "@mousedown.stop"])
html.Div(raw_attrs=['v-click-outside="() => { expanded = null }"'])
```

**Slot destructuring** - quoting is critical:
```python
# CORRECT: double quotes around destructuring
with vuetify3.Template(
    raw_attrs=['v-slot:header.col="{ column, toggleSort }"'],
):
    html.Div(raw_attrs=["@click='toggleSort(column)'"])

# WRONG: missing quotes generates invalid template
raw_attrs=["v-slot:header.col={ column }"]  # Bad!
```

**Slot-scoped variables** use `raw_attrs`, not tuple syntax:
```python
vuetify3.VIcon(raw_attrs=["v-if='getSortIcon(column)'"])  # Correct
vuetify3.VIcon(v_if="getSortIcon(column)")  # Wrong - expects trame state
```

## Slots

```python
with vuetify3.VMenu():
    with vuetify3.Template(v_slot_activator="{ props }"):
        vuetify3.VBtn(v_bind="props")
    vuetify3.VList()  # Menu content
```

## __events

Expose non-default Vuetify events:
```python
vuetify3.VSlider(
    __events=["end"],
    end=(self.on_release, "[$event]"),
)
```

## Debug Template

```python
with vuetify3.Template(raw_attrs=['v-slot:item="{ item }"']) as t:
    html.Div("Content")
print("TEMPLATE:", t)
```

## Dynamic Bindings

```python
to=("`/bar/${item.id}`",)  # JS template literal in single-element tuple
```
