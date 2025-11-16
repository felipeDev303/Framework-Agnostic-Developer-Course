# Creando una Interfaz de Usuario en la Web

## Lección 2.1: El DOM - La API que Todos los Frameworks Abstraen

### Entendiendo el Costo del DOM

El DOM (Document Object Model) es la representación en árbol de tu HTML que el navegador usa para renderizar la página. Manipular el DOM es costoso porque:

```javascript
// Anatomía de una actualización DOM
element.style.width = "200px";

// Lo que realmente sucede:
// 1. RECALCULATE STYLE
//    - El navegador recalcula todos los estilos afectados
//    - Cascada CSS debe re-evaluarse
//
// 2. LAYOUT (Reflow)
//    - Calcula posiciones y tamaños
//    - Puede afectar elementos padres/hijos/hermanos
//
// 3. PAINT
//    - Dibuja los píxeles en memoria
//    - Crea capas de renderizado
//
// 4. COMPOSITE
//    - Combina todas las capas
//    - Envía al GPU

// Ejemplo del problema de Layout Thrashing
const elements = document.querySelectorAll(".item");
elements.forEach((el) => {
  // MALO: Fuerza layout en cada iteración
  el.style.height = el.offsetHeight + 10 + "px";
  // offsetHeight fuerza un reflow para obtener el valor actual
});

// BUENO: Batch reads, then batch writes
const heights = Array.from(elements).map((el) => el.offsetHeight);
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + "px";
});
```

### Estrategias de Optimización del DOM

Los frameworks usan diferentes estrategias para minimizar las operaciones DOM:

```javascript
// 1. VIRTUAL DOM (React, Vue 2)
// Mantiene una representación en memoria y hace diff
const oldVDOM = {
  type: "div",
  props: { className: "container" },
  children: [
    { type: "span", props: {}, children: ["Hello"] },
    { type: "span", props: {}, children: ["World"] },
  ],
};

const newVDOM = {
  type: "div",
  props: { className: "container active" },
  children: [
    { type: "span", props: {}, children: ["Hello"] },
    { type: "span", props: {}, children: ["React"] },
  ],
};

// Algoritmo de diffing encuentra cambios mínimos
const patches = [
  { type: "UPDATE_PROPS", path: [], props: { className: "container active" } },
  { type: "UPDATE_TEXT", path: [1], text: "React" },
];

// 2. INCREMENTAL DOM (Angular Ivy)
// Actualiza el DOM directamente durante el renderizado
function renderComponent() {
  elementStart("div", 0, ["class", "container"]);
  text(1, "Hello");
  if (showWorld) {
    text(2, "World"); // Solo se crea/actualiza si necesario
  }
  elementEnd();
}

// 3. COMPILE-TIME OPTIMIZATION (Svelte)
// Genera código específico para cada actualización
function update(dirty) {
  if (dirty & 1) text0.textContent = state.name;
  if (dirty & 2) div0.className = state.active ? "active" : "";
}

// 4. FINE-GRAINED REACTIVITY (Solid, Vue 3)
// Solo actualiza exactamente lo que cambió
createEffect(() => {
  // Solo este nodo de texto se actualiza
  textNode.textContent = name();
});
```

---

## Lección 2.2: Templates vs JSX vs Funciones

### La Guerra de Sintaxis y Sus Implicaciones

```javascript
// TEMPLATES (Vue, Angular, Svelte)
// Pros: Más cercano a HTML, análisis estático más fácil, optimizaciones en compile-time
// Contras: DSL nuevo que aprender, limitaciones de expresividad

// Vue Single File Component
<template>
  <div class="user-card" :class="{ active: isActive }">
    <img :src="user.avatar" :alt="user.name">
    <h2>{{ user.name }}</h2>
    <p v-if="user.bio">{{ user.bio }}</p>
    <button @click="toggleActive">
      {{ isActive ? 'Deactivate' : 'Activate' }}
    </button>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const props = defineProps(['user'])
const isActive = ref(false)

const toggleActive = () => {
  isActive.value = !isActive.value
}
</script>

// JSX (React, Solid, Preact)
// Pros: Es JavaScript, máxima flexibilidad, type-safe con TypeScript
// Contras: Mezcla lógica y presentación, más verboso

function UserCard({ user }) {
  const [isActive, setIsActive] = useState(false);

  return (
    <div className={`user-card ${isActive ? 'active' : ''}`}>
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      {user.bio && <p>{user.bio}</p>}
      <button onClick={() => setIsActive(!isActive)}>
        {isActive ? 'Deactivate' : 'Activate'}
      </button>
    </div>
  );
}

// TEMPLATE LITERALS (lit-html, µhtml)
// Pros: Estándares web, no requiere compilación
// Contras: Menos optimizaciones posibles, sintaxis puede ser verbosa

import { html, render } from 'lit-html';

const userCard = (user, isActive, toggleActive) => html`
  <div class="user-card ${isActive ? 'active' : ''}">
    <img src="${user.avatar}" alt="${user.name}">
    <h2>${user.name}</h2>
    ${user.bio ? html`<p>${user.bio}</p>` : ''}
    <button @click=${toggleActive}>
      ${isActive ? 'Deactivate' : 'Activate'}
    </button>
  </div>
`;

// HYPERSCRIPT (Mithril, HyperApp)
// Pros: No requiere transpilación, muy ligero
// Contras: Verboso, menos intuitivo

m('div.user-card', { class: isActive ? 'active' : '' }, [
  m('img', { src: user.avatar, alt: user.name }),
  m('h2', user.name),
  user.bio && m('p', user.bio),
  m('button', { onclick: toggleActive },
    isActive ? 'Deactivate' : 'Activate'
  )
])
```

### El Proceso de Compilación/Transformación

```javascript
// JSX se transforma en llamadas a funciones
// Entrada (JSX):
const element = (
  <div className="container">
    <h1>Hello, {name}!</h1>
    <Button onClick={handleClick}>Click me</Button>
  </div>
);

// Salida (JavaScript):
const element = React.createElement(
  'div',
  { className: 'container' },
  React.createElement('h1', null, 'Hello, ', name, '!'),
  React.createElement(Button, { onClick: handleClick }, 'Click me')
);

// Templates se compilan a funciones de renderizado
// Entrada (Vue template):
<div v-for="item in items" :key="item.id">
  {{ item.name }}
</div>

// Salida (función de renderizado):
function render() {
  return items.map(item =>
    h('div', { key: item.id }, item.name)
  )
}

// Svelte compila a DOM imperativo
// Entrada:
<button on:click={increment}>
  Count: {count}
</button>

// Salida:
function create_fragment(ctx) {
  let button;
  let t0;
  let t1;

  return {
    c() {
      button = element("button");
      t0 = text("Count: ");
      t1 = text(ctx[0]);
    },
    m(target, anchor) {
      insert(target, button, anchor);
      append(button, t0);
      append(button, t1);
      listen(button, "click", ctx[1]);
    },
    p(ctx, dirty) {
      if (dirty & 1) set_data(t1, ctx[0]);
    },
    d(detaching) {
      if (detaching) detach(button);
      unlisten(button, "click", ctx[1]);
    }
  };
}
```

---

## Lección 2.3: Reconciliación - El Corazón de la Reactividad

### Algoritmo de Diffing del Virtual DOM

```javascript
// Implementación simplificada del algoritmo de diffing
class VirtualDOM {
  diff(oldVNode, newVNode) {
    // Caso 1: Nodo nuevo
    if (!oldVNode) {
      return { type: "CREATE", node: newVNode };
    }

    // Caso 2: Nodo eliminado
    if (!newVNode) {
      return { type: "REMOVE" };
    }

    // Caso 3: Nodo reemplazado (tipo diferente)
    if (oldVNode.type !== newVNode.type) {
      return { type: "REPLACE", node: newVNode };
    }

    // Caso 4: Nodo actualizado (mismo tipo)
    const propsDiff = this.diffProps(oldVNode.props, newVNode.props);
    const childrenDiff = this.diffChildren(
      oldVNode.children,
      newVNode.children
    );

    if (propsDiff.length > 0 || childrenDiff.length > 0) {
      return {
        type: "UPDATE",
        props: propsDiff,
        children: childrenDiff,
      };
    }

    return null; // Sin cambios
  }

  diffProps(oldProps = {}, newProps = {}) {
    const patches = [];

    // Props agregados o modificados
    for (const [key, value] of Object.entries(newProps)) {
      if (oldProps[key] !== value) {
        patches.push({ type: "SET_PROP", key, value });
      }
    }

    // Props eliminados
    for (const key in oldProps) {
      if (!(key in newProps)) {
        patches.push({ type: "REMOVE_PROP", key });
      }
    }

    return patches;
  }

  diffChildren(oldChildren = [], newChildren = []) {
    const patches = [];
    const additionalPatches = [];

    // Usando keys para optimizar
    const oldKeyed = {};
    const newKeyed = {};

    // Crear mapas con keys
    oldChildren.forEach((child, i) => {
      if (child.key) oldKeyed[child.key] = { node: child, index: i };
    });

    newChildren.forEach((child, i) => {
      if (child.key) newKeyed[child.key] = { node: child, index: i };
    });

    // Diff con keys
    for (const key in newKeyed) {
      if (key in oldKeyed) {
        // Mismo key, verificar si se movió
        const oldIndex = oldKeyed[key].index;
        const newIndex = newKeyed[key].index;

        if (oldIndex !== newIndex) {
          patches.push({
            type: "MOVE",
            from: oldIndex,
            to: newIndex,
          });
        }

        // Diff recursivo del nodo
        const nodePatch = this.diff(oldKeyed[key].node, newKeyed[key].node);

        if (nodePatch) {
          patches.push({ ...nodePatch, index: newIndex });
        }
      } else {
        // Nuevo nodo con key
        patches.push({
          type: "INSERT",
          node: newKeyed[key].node,
          index: newKeyed[key].index,
        });
      }
    }

    // Eliminar nodos viejos que no están en los nuevos
    for (const key in oldKeyed) {
      if (!(key in newKeyed)) {
        patches.push({
          type: "REMOVE",
          index: oldKeyed[key].index,
        });
      }
    }

    return patches;
  }
}

// Aplicar patches al DOM real
function applyPatch(parent, patch, index = 0) {
  switch (patch.type) {
    case "CREATE":
      parent.appendChild(createElement(patch.node));
      break;

    case "REMOVE":
      parent.removeChild(parent.childNodes[index]);
      break;

    case "REPLACE":
      parent.replaceChild(createElement(patch.node), parent.childNodes[index]);
      break;

    case "UPDATE":
      const node = parent.childNodes[index];

      // Actualizar props
      patch.props?.forEach((propPatch) => {
        if (propPatch.type === "SET_PROP") {
          setProp(node, propPatch.key, propPatch.value);
        } else if (propPatch.type === "REMOVE_PROP") {
          removeProp(node, propPatch.key);
        }
      });

      // Actualizar hijos recursivamente
      patch.children?.forEach((childPatch) => {
        applyPatch(node, childPatch, childPatch.index);
      });
      break;
  }
}
```

### Optimizaciones de Reconciliación

```javascript
// React Fiber - Reconciliación Incremental
// Divide el trabajo en unidades pequeñas (fibers)
class Fiber {
  constructor(element, parent) {
    this.type = element.type;
    this.props = element.props;
    this.parent = parent;
    this.child = null;
    this.sibling = null;
    this.effectTag = null; // 'PLACEMENT', 'UPDATE', 'DELETION'
    this.alternate = null; // Referencia al fiber anterior
  }
}

function workLoop(deadline) {
  let shouldYield = false;

  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot(); // Aplicar todos los cambios al DOM
  }

  requestIdleCallback(workLoop);
}

// Vue 3 - Optimización con Flags
// Marca qué tipo de actualizaciones son posibles
const PatchFlags = {
  TEXT: 1, // Texto dinámico
  CLASS: 2, // Clase dinámica
  STYLE: 4, // Estilo dinámico
  PROPS: 8, // Props dinámicos (no clase/estilo)
  FULL_PROPS: 16, // Props con keys dinámicos
  HYDRATE_EVENTS: 32,
  STABLE_FRAGMENT: 64,
  KEYED_FRAGMENT: 128,
  UNKEYED_FRAGMENT: 256,
  NEED_PATCH: 512,
  DYNAMIC_SLOTS: 1024,
  HOISTED: -1, // Estático, no necesita diff
  BAIL: -2, // Siempre hacer full diff
};

// El compilador añade flags
function render() {
  return createVNode(
    "div",
    { class: dynamicClass }, // Flag: CLASS
    [
      createVNode("span", null, staticText, PatchFlags.HOISTED),
      createVNode("span", null, dynamicText, PatchFlags.TEXT),
    ]
  );
}
```

---

## Lección 2.4: Event Handling y Delegation

### Sistema de Eventos Sintéticos

```javascript
// React - Sistema de Eventos Sintéticos
class SyntheticEventSystem {
  constructor() {
    this.eventHandlers = new Map();
    this.rootElement = null;
  }

  initialize(root) {
    this.rootElement = root;

    // Registrar un solo listener por tipo de evento en el root
    ["click", "change", "input", "focus", "blur"].forEach((eventType) => {
      root.addEventListener(eventType, (e) => this.handleEvent(e), true);
    });
  }

  handleEvent(nativeEvent) {
    // Crear evento sintético
    const syntheticEvent = this.createSyntheticEvent(nativeEvent);

    // Encontrar el path desde el target hasta el root
    const path = this.getEventPath(nativeEvent.target);

    // Fase de captura
    for (let i = path.length - 1; i >= 0; i--) {
      const handlers = this.eventHandlers.get(path[i]);
      if (handlers?.capture?.[nativeEvent.type]) {
        handlers.capture[nativeEvent.type](syntheticEvent);
        if (syntheticEvent.propagationStopped) return;
      }
    }

    // Fase de burbujeo
    for (let i = 0; i < path.length; i++) {
      const handlers = this.eventHandlers.get(path[i]);
      if (handlers?.bubble?.[nativeEvent.type]) {
        handlers.bubble[nativeEvent.type](syntheticEvent);
        if (syntheticEvent.propagationStopped) return;
      }
    }
  }

  createSyntheticEvent(nativeEvent) {
    return {
      type: nativeEvent.type,
      target: nativeEvent.target,
      currentTarget: null,
      preventDefault: () => nativeEvent.preventDefault(),
      stopPropagation: () => (this.propagationStopped = true),
      nativeEvent,
      // Propiedades específicas del evento
      ...this.extractEventProperties(nativeEvent),
    };
  }

  registerHandler(element, eventType, handler, capture = false) {
    if (!this.eventHandlers.has(element)) {
      this.eventHandlers.set(element, { capture: {}, bubble: {} });
    }

    const handlers = this.eventHandlers.get(element);
    const phase = capture ? "capture" : "bubble";
    handlers[phase][eventType] = handler;
  }
}

// Svelte - Event handling directo con modificadores
// Compilado a:
button.addEventListener("click", handleClick, { once: true });

// Vue - Sistema de eventos con modificadores
// <button @click.stop.prevent="handleClick">
// Se traduce a:
element.addEventListener("click", (e) => {
  e.stopPropagation();
  e.preventDefault();
  handleClick(e);
});
```

---

**Anterior**: [00 - Introducción](./00-Introduccion.md)  
**Siguiente**: [02 - HTTP y el Límite Servidor/Cliente](./02-HTTP-Cliente-Servidor.md)
