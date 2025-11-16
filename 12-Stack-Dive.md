# Stack Dive - Proyectos Prácticos

## Proyecto 1: Mini Framework Reactivo

**Objetivo**: Construir un framework con signals desde cero

**Características**:

- Sistema de signals
- Computed values
- Effects con auto-tracking
- Batching de actualizaciones
- Renderizado eficiente del DOM

```javascript
// Starter code
class MiniFramework {
  constructor() {
    // Tu implementación aquí
  }

  signal(initialValue) {
    // Crear signal reactivo
  }

  computed(fn) {
    // Valor derivado con caché
  }

  effect(fn) {
    // Side effect con auto-tracking
  }

  render(component, container) {
    // Renderizar componente al DOM
  }
}
```

## Proyecto 2: Router con Code Splitting

**Objetivo**: Implementar un router con lazy loading

**Características**:

- Navegación declarativa
- Parámetros de ruta
- Lazy loading de componentes
- Prefetching inteligente
- Historial del navegador

```javascript
// Starter code
class Router {
  constructor(routes) {
    this.routes = routes;
    // Setup
  }

  navigate(path) {
    // Navegar a ruta
  }

  lazy(importer) {
    // Lazy load component
  }

  prefetch(path) {
    // Prefetch recursos
  }
}
```

## Proyecto 3: SSR + Hydration

**Objetivo**: Sistema de renderizado isomorfo

**Características**:

- Renderizado en servidor
- Hidratación en cliente
- Serialización de estado
- Progressive enhancement

```javascript
// Server
export function renderToString(component) {
  // Renderizar a HTML
}

export function serializeState(state) {
  // Serializar estado
}

// Client
export function hydrate(component, container) {
  // Hidratar HTML existente
}

export function deserializeState(serialized) {
  // Deserializar estado
}
```

## Proyecto 4: Virtual DOM Completo

**Objetivo**: Implementar VDOM con reconciliación

**Características**:

- Creación de VNodes
- Algoritmo de diffing
- Reconciliación con keys
- Batching de patches
- Event delegation

```javascript
// Starter code
function h(type, props, ...children) {
  // Crear VNode
}

function diff(oldVNode, newVNode) {
  // Calcular diferencias
}

function patch(parent, patches) {
  // Aplicar cambios al DOM
}
```

---

## Recursos de Aprendizaje

### Documentación Oficial

- [React](https://react.dev)
- [Vue](https://vuejs.org)
- [Solid](https://solidjs.com)
- [Svelte](https://svelte.dev)
- [Qwik](https://qwik.builder.io)

### Artículos Fundamentales

- Virtual DOM and Internals (React)
- Reactivity in Depth (Vue)
- Signals vs Observables
- Hydration vs Resumability

### Código Fuente

Estudia las implementaciones:

- `preact/preact` (Virtual DOM simple)
- `solidjs/solid` (Signals y compilación)
- `BuilderIO/qwik` (Resumability)

---

## Conclusión

**Principios Universales**:

1. Todo es un trade-off
2. La abstracción tiene costo
3. El rendimiento percibido importa
4. DX influye en la calidad
5. Los frameworks convergen

**El Futuro**:

- Compilación más agresiva
- Signals everywhere
- Server-first con gran UX
- AI-assisted development
- Web Components renaissance

> "No se trata de conocer todos los frameworks, sino de entender los principios que los gobiernan."

---

**Anterior**: [11 - RPCs](./11-RPCs.md)  
**Inicio**: [00 - Introducción](./00-Introduccion.md)
