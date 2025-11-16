# Transpiladores, Compiladores y Empaquetado

## Conceptos Fundamentales

Los frameworks modernos dependen de herramientas de compilación para:

1. **Transformar sintaxis moderna** (JSX, TypeScript, etc.) a JavaScript compatible
2. **Optimizar el código** mediante análisis estático
3. **Empaquetar módulos** en bundles eficientes
4. **Code splitting** para carga bajo demanda

---

## El Pipeline de Compilación

```javascript
// 1. CÓDIGO FUENTE
// Component.tsx
import { useState } from "react";

export function Counter({ initial = 0 }: { initial?: number }) {
  const [count, setCount] = useState(initial);

  return (
    <div className="counter">
      <button onClick={() => setCount(count - 1)}>-</button>
      <span>{count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// 2. TRANSPILACIÓN (Babel / SWC)
// TypeScript → JavaScript
// JSX → createElement calls
import { useState } from "react";
import { jsx as _jsx } from "react/jsx-runtime";

export function Counter({ initial = 0 }) {
  const [count, setCount] = useState(initial);

  return _jsx("div", {
    className: "counter",
    children: [
      _jsx("button", {
        onClick: () => setCount(count - 1),
        children: "-",
      }),
      _jsx("span", { children: count }),
      _jsx("button", {
        onClick: () => setCount(count + 1),
        children: "+",
      }),
    ],
  });
}

// 3. BUNDLING (Webpack / Rollup / Vite)
// Combina módulos, tree-shaking, minificación
(function () {
  var e = require("react");
  function t(t) {
    var n = t.initial,
      r = void 0 === n ? 0 : n,
      o = e.useState(r),
      u = o[0],
      c = o[1];
    return e.createElement(
      "div",
      { className: "counter" },
      e.createElement(
        "button",
        {
          onClick: function () {
            return c(u - 1);
          },
        },
        "-"
      ),
      e.createElement("span", null, u),
      e.createElement(
        "button",
        {
          onClick: function () {
            return c(u + 1);
          },
        },
        "+"
      )
    );
  }
  exports.Counter = t;
})();

// 4. OPTIMIZACIÓN
// Code splitting, lazy loading, prefetch hints
import(/* webpackChunkName: "counter" */ "./Counter").then((module) => {
  const Counter = module.Counter;
  // render Counter
});
```

---

## Herramientas de Compilación

### Babel - El Transpilador Universal

```javascript
// .babelrc
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.25%, not dead",
      "useBuiltIns": "usage",
      "corejs": 3
    }],
    "@babel/preset-react",
    "@babel/preset-typescript"
  ],
  "plugins": [
    "@babel/plugin-proposal-class-properties",
    ["@babel/plugin-transform-runtime", {
      "regenerator": true
    }]
  ]
}

// Plugin personalizado de Babel
module.exports = function ({ types: t }) {
  return {
    visitor: {
      // Transformar console.log a desarrollo solamente
      CallExpression(path) {
        if (
          path.node.callee.object?.name === 'console' &&
          path.node.callee.property?.name === 'log' &&
          process.env.NODE_ENV === 'production'
        ) {
          path.remove();
        }
      }
    }
  };
};
```

### SWC - Compilador en Rust (Ultra Rápido)

```javascript
// .swcrc
{
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "tsx": true,
      "decorators": true
    },
    "transform": {
      "react": {
        "runtime": "automatic",
        "development": false,
        "refresh": true
      }
    },
    "target": "es2020"
  },
  "module": {
    "type": "es6"
  },
  "minify": true
}

// Benchmark: SWC vs Babel
// Babel:  ~2000ms para 1000 archivos
// SWC:    ~100ms para 1000 archivos (20x más rápido)
```

### Vite - Bundler Moderno

```javascript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],

  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ["react", "react-dom"],
          ui: ["@mui/material"],
        },
      },
    },
  },

  optimizeDeps: {
    include: ["react", "react-dom"],
  },

  server: {
    port: 3000,
    hmr: {
      overlay: true,
    },
  },
});
```

---

## Optimizaciones del Compilador

### Tree Shaking - Eliminar Código No Usado

```javascript
// utils.js - Exporta muchas funciones
export function add(a, b) {
  return a + b;
}
export function subtract(a, b) {
  return a - b;
}
export function multiply(a, b) {
  return a * b;
}
export function divide(a, b) {
  return a / b;
}
export function power(a, b) {
  return Math.pow(a, b);
}

// app.js - Solo usa una función
import { add } from "./utils";
console.log(add(2, 3));

// Bundle final (con tree-shaking):
// Solo incluye la función add
function add(a, b) {
  return a + b;
}
console.log(add(2, 3));

// Sin tree-shaking, todas las funciones se incluirían
```

### Code Splitting - Dividir el Bundle

```javascript
// Sin code splitting
import HeavyComponent from "./HeavyComponent";
import AnotherHeavy from "./AnotherHeavy";

function App() {
  return (
    <>
      <HeavyComponent />
      <AnotherHeavy />
    </>
  );
}
// Bundle: 500KB (todo se carga al inicio)

// Con code splitting dinámico
const HeavyComponent = lazy(() => import("./HeavyComponent"));
const AnotherHeavy = lazy(() => import("./AnotherHeavy"));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <HeavyComponent />
      <AnotherHeavy />
    </Suspense>
  );
}
// Bundle inicial: 50KB
// Chunks separados: heavy-1.js (250KB), heavy-2.js (200KB)
// Se cargan bajo demanda
```

### Minificación y Compresión

```javascript
// Código original
function calculateTotalPrice(items, taxRate, discount) {
  const subtotal = items.reduce((sum, item) => {
    return sum + item.price * item.quantity;
  }, 0);

  const discountAmount = subtotal * (discount / 100);
  const afterDiscount = subtotal - discountAmount;
  const tax = afterDiscount * (taxRate / 100);

  return afterDiscount + tax;
}

// Después de minificación (Terser)
function calculateTotalPrice(e, t, a) {
  const c = e.reduce((e, t) => e + t.price * t.quantity, 0),
    n = c * (a / 100),
    r = c - n;
  return r + r * (t / 100);
}

// Después de compresión gzip
// Tamaño: 500 bytes → 150 bytes (70% reducción)
```

---

## Compiladores de Framework

### Svelte Compiler

```javascript
// Input: Svelte component
<script>
  let count = 0;

  function increment() {
    count += 1;
  }
</script>

<button on:click={increment}>
  Clicks: {count}
</button>

// Output: JavaScript optimizado
function create_fragment(ctx) {
  let button;
  let t0;
  let t1;
  let mounted;
  let dispose;

  return {
    c() {
      button = element("button");
      t0 = text("Clicks: ");
      t1 = text(ctx[0]);
    },
    m(target, anchor) {
      insert(target, button, anchor);
      append(button, t0);
      append(button, t1);

      if (!mounted) {
        dispose = listen(button, "click", ctx[1]);
        mounted = true;
      }
    },
    p(ctx, [dirty]) {
      if (dirty & 1) set_data(t1, ctx[0]);
    },
    d(detaching) {
      if (detaching) detach(button);
      mounted = false;
      dispose();
    }
  };
}

function instance($$self, $$props, $$invalidate) {
  let count = 0;

  function increment() {
    $$invalidate(0, count += 1);
  }

  return [count, increment];
}
```

### Solid Compiler

```javascript
// Input: Solid component
function Counter() {
  const [count, setCount] = createSignal(0);

  return (
    <button onClick={() => setCount(count() + 1)}>Count: {count()}</button>
  );
}

// Output: Optimizado con fine-grained reactivity
function Counter() {
  const [count, setCount] = createSignal(0);

  return (() => {
    const _el$ = _tmpl$();
    _el$.$$click = () => setCount(count() + 1);
    insert(
      _el$,
      createComponent(Show, {
        get children() {
          return ["Count: ", count()];
        },
      }),
      null
    );
    return _el$;
  })();
}
```

---

## Module Federation - Compartir Código Entre Apps

```javascript
// webpack.config.js - App Host
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "host",
      remotes: {
        app1: "app1@http://localhost:3001/remoteEntry.js",
        app2: "app2@http://localhost:3002/remoteEntry.js",
      },
      shared: {
        react: { singleton: true },
        "react-dom": { singleton: true },
      },
    }),
  ],
};

// webpack.config.js - App Remota
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: "app1",
      filename: "remoteEntry.js",
      exposes: {
        "./Counter": "./src/Counter",
        "./Button": "./src/Button",
      },
      shared: {
        react: { singleton: true },
        "react-dom": { singleton: true },
      },
    }),
  ],
};

// Uso en Host
const RemoteCounter = lazy(() => import("app1/Counter"));

function App() {
  return (
    <Suspense fallback="Loading...">
      <RemoteCounter />
    </Suspense>
  );
}
```

---

**Anterior**: [03 - Componentes, Árboles y Estado](./03-Componentes-Arbol-Estado.md)  
**Siguiente**: [05 - Virtual DOM](./05-VDOM.md)
