---
name: trame-vue-component
description: Building Vue.js components for Trame with Vite and npm. UMD library build, trame module serving, component registration pattern.
---

# Trame Vue/JS Component (Vite Build)

For JS modules that need npm dependencies (vtk.js, maplibre-gl, etc.), use Vite to build a UMD bundle that trame serves.

## Directory Structure

```
my_app/
├── module/
│   ├── __init__.py                  # Trame module (lists built UMD files)
│   ├── serve/                       # Build output (UMD bundles land here)
│   │   ├── my_component.umd.js     # Built by Vite
│   │   └── my_component.css        # Built by Vite (if any)
│   └── vue-components/             # Source (npm project)
│       ├── package.json
│       ├── vite.config.js
│       └── src/
│           └── my_component.js     # Entry point
```

## package.json

```json
{
  "name": "my-trame-components",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "cross-env ENTRY_NAME=my_component vite build",
    "dev": "vite build --watch"
  },
  "dependencies": {
    "@kitware/vtk.js": "32.2.0"
  },
  "devDependencies": {
    "cross-env": "^7.0.3",
    "vite": "^5.0.0"
  }
}
```

For multiple entry points, chain builds:
```json
"build": "cross-env ENTRY_NAME=comp_a vite build && cross-env ENTRY_NAME=comp_b vite build"
```

## vite.config.js

```javascript
const entryName = process.env.ENTRY_NAME || "my_component";

export default {
  base: "./",
  build: {
    lib: {
      entry: `./src/${entryName}.js`,
      name: entryName,
      formats: ["umd"],
      fileName: (format) => `${entryName}.${format}.js`,
    },
    sourcemap: true,
    emptyOutDir: false,
    cssCodeSplit: false,
    rollupOptions: {
      external: [],
      output: {
        globals: {},
        assetFileNames: `${entryName}.[ext]`,
      },
    },
    outDir: "../serve",
    assetsDir: ".",
  },
};
```

Key settings:
- `formats: ["umd"]` — produces a self-contained bundle that works via `<script>` tag
- `outDir: "../serve"` — builds directly into the trame serve directory
- `emptyOutDir: false` — preserves other built files in serve/

## Entry Point Pattern

```javascript
import vtkTexture from "@kitware/vtk.js/Rendering/Core/Texture";

if (!window.trame) window.trame = {};
if (!window.trame.utils) window.trame.utils = {};

window.trame.utils.my_component = {
  initialized: false,

  actions: {
    init() {
      if (window.trame?.state) {
        this.setup();
      }
    },

    setup() {
      // Access VTK refs, set up scene objects, etc.
      const viewRef = window.trame?.refs?.["myView"];
      if (!viewRef) {
        setTimeout(() => this.setup(), 100);
        return;
      }
      window.trame.utils.my_component.initialized = true;
    },
  },
};

// Auto-initialize on load
window.trame.utils.my_component.actions.init();
```

Namespace under `window.trame.utils` so trame event handlers can access it:
```python
click="utils.my_component.actions.doSomething()"
```

## Trame Module Registration

`module/__init__.py`:
```python
from pathlib import Path

serve_path = str(Path(__file__).with_name("serve").resolve())
serve = {"__my_app": serve_path}
scripts = [
    "__my_app/my_component.umd.js",
]
styles = [
    "__my_app/my_component.css",
]

def setup(server, **kwargs):
    pass
```

## Build and Serve

```bash
cd module/vue-components && npm install && npm run build
```

Python side:
```python
from my_app import module as my_app_module
self.server.enable_module(my_app_module)
```
