# Renderizado, Reactividad y Virtual DOM

## Conceptos Fundamentales

El Virtual DOM es una representación en JavaScript del DOM real que permite:

1. **Reconciliación eficiente**: Calcular cambios mínimos necesarios
2. **Batching de actualizaciones**: Agrupar cambios múltiples
3. **Abstracción multiplataforma**: React Native, React Three Fiber, etc.

---

## El Problema que Resuelve el VDOM

```javascript
// MANIPULACIÓN DIRECTA DEL DOM (lenta)
function updateUI(items) {
  const list = document.getElementById("list");
  list.innerHTML = ""; // Fuerza reflow completo

  items.forEach((item) => {
    const li = document.createElement("li");
    li.textContent = item.text;
    li.className = item.completed ? "completed" : "";
    list.appendChild(li); // Reflow por cada item
  });
}

// VIRTUAL DOM (optimizado)
function render(items) {
  return items.map((item) =>
    h(
      "li",
      {
        class: item.completed ? "completed" : "",
      },
      item.text
    )
  );
}

// El framework calcula el diff y aplica cambios mínimos
// Solo actualiza los nodos que realmente cambiaron
```

---

## Implementación del Virtual DOM

### Estructura de un VNode

```javascript
// Representación de Virtual DOM Node
interface VNode {
  type: string | Function;
  props: Record<string, any>;
  children: Array<VNode | string>;
  key?: string | number;

  // Metadata interna
  _dom?: HTMLElement;
  _parent?: VNode;
  _depth?: number;
}

// Ejemplo de VNode
const vnode = {
  type: "div",
  props: {
    className: "container",
    id: "main",
    onClick: handleClick,
  },
  children: [
    {
      type: "h1",
      props: {},
      children: ["Hello World"],
    },
    {
      type: "button",
      props: { className: "btn" },
      children: ["Click me"],
    },
  ],
};

// Función para crear VNodes
function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: children.flat(),
    key: props?.key,
  };
}

// JSX se compila a llamadas h()
// <div className="container">
//   <h1>Hello</h1>
// </div>
//
// Becomes:
// h('div', { className: 'container' },
//   h('h1', {}, 'Hello')
// )
```

### Algoritmo de Diffing

```javascript
function diff(oldVNode, newVNode, parent) {
  // Caso 1: Nodo nuevo
  if (!oldVNode) {
    return mount(newVNode, parent);
  }

  // Caso 2: Nodo eliminado
  if (!newVNode) {
    return unmount(oldVNode, parent);
  }

  // Caso 3: Tipo diferente - reemplazo completo
  if (oldVNode.type !== newVNode.type) {
    unmount(oldVNode, parent);
    return mount(newVNode, parent);
  }

  // Caso 4: Mismo tipo - actualizar
  const dom = (newVNode._dom = oldVNode._dom);

  // Actualizar props
  updateProps(dom, oldVNode.props, newVNode.props);

  // Reconciliar hijos
  reconcileChildren(oldVNode, newVNode);

  return dom;
}

function updateProps(dom, oldProps, newProps) {
  // Remover props viejos
  Object.keys(oldProps).forEach((name) => {
    if (name !== "children" && !(name in newProps)) {
      if (name.startsWith("on")) {
        const eventType = name.substring(2).toLowerCase();
        dom.removeEventListener(eventType, oldProps[name]);
      } else if (name === "className") {
        dom.className = "";
      } else if (name === "style") {
        dom.style.cssText = "";
      } else {
        dom.removeAttribute(name);
      }
    }
  });

  // Establecer nuevos props
  Object.keys(newProps).forEach((name) => {
    if (name !== "children" && oldProps[name] !== newProps[name]) {
      if (name.startsWith("on")) {
        const eventType = name.substring(2).toLowerCase();
        if (oldProps[name]) {
          dom.removeEventListener(eventType, oldProps[name]);
        }
        dom.addEventListener(eventType, newProps[name]);
      } else if (name === "className") {
        dom.className = newProps[name] || "";
      } else if (name === "style") {
        if (typeof newProps[name] === "string") {
          dom.style.cssText = newProps[name];
        } else {
          Object.assign(dom.style, newProps[name]);
        }
      } else if (name === "value" || name === "checked") {
        dom[name] = newProps[name];
      } else {
        dom.setAttribute(name, newProps[name]);
      }
    }
  });
}

function reconcileChildren(oldVNode, newVNode) {
  const oldChildren = oldVNode.children;
  const newChildren = newVNode.children;
  const dom = newVNode._dom;

  // Reconciliación con keys
  if (newChildren.some((child) => child.key != null)) {
    reconcileChildrenWithKeys(oldChildren, newChildren, dom);
  } else {
    reconcileChildrenByIndex(oldChildren, newChildren, dom);
  }
}

function reconcileChildrenWithKeys(oldChildren, newChildren, parent) {
  // Crear mapas de children por key
  const oldKeyedChildren = new Map();
  const newKeyedChildren = new Map();

  oldChildren.forEach((child, i) => {
    if (child.key) {
      oldKeyedChildren.set(child.key, { child, index: i });
    }
  });

  newChildren.forEach((child, i) => {
    if (child.key) {
      newKeyedChildren.set(child.key, { child, index: i });
    }
  });

  // Procesar cada child nuevo
  newChildren.forEach((newChild, newIndex) => {
    if (!newChild.key) return;

    const oldEntry = oldKeyedChildren.get(newChild.key);

    if (oldEntry) {
      // Existe, hacer diff
      const oldChild = oldEntry.child;
      diff(oldChild, newChild, parent);

      // Mover si es necesario
      if (oldEntry.index !== newIndex) {
        const currentNode = newChild._dom;
        const referenceNode = parent.childNodes[newIndex];
        if (currentNode !== referenceNode) {
          parent.insertBefore(currentNode, referenceNode);
        }
      }
    } else {
      // Nuevo child
      const newNode = mount(newChild, parent);
      const referenceNode = parent.childNodes[newIndex];
      parent.insertBefore(newNode, referenceNode);
    }
  });

  // Eliminar children que ya no existen
  oldKeyedChildren.forEach((entry, key) => {
    if (!newKeyedChildren.has(key)) {
      unmount(entry.child, parent);
    }
  });
}
```

### Montaje y Desmontaje

```javascript
function mount(vnode, parent) {
  // Nodos de texto
  if (typeof vnode === "string" || typeof vnode === "number") {
    const textNode = document.createTextNode(vnode);
    parent.appendChild(textNode);
    return textNode;
  }

  // Componentes
  if (typeof vnode.type === "function") {
    return mountComponent(vnode, parent);
  }

  // Elementos HTML
  const dom = document.createElement(vnode.type);
  vnode._dom = dom;

  // Establecer props
  if (vnode.props) {
    updateProps(dom, {}, vnode.props);
  }

  // Montar children
  if (vnode.children) {
    vnode.children.forEach((child) => mount(child, dom));
  }

  parent.appendChild(dom);
  return dom;
}

function unmount(vnode, parent) {
  // Componentes: ejecutar cleanup
  if (typeof vnode.type === "function") {
    unmountComponent(vnode);
  }

  // Remover del DOM
  if (vnode._dom) {
    parent.removeChild(vnode._dom);
  }

  // Cleanup de children
  if (vnode.children) {
    vnode.children.forEach((child) => {
      if (typeof child === "object") {
        unmount(child, vnode._dom);
      }
    });
  }
}
```

---

## React Fiber - Reconciliación Incremental

```javascript
// Fiber Node Structure
class FiberNode {
  constructor(tag, pendingProps, key) {
    this.tag = tag; // Tipo de nodo
    this.key = key; // Key para reconciliación
    this.type = null; // Tipo del componente
    this.stateNode = null; // Referencia al DOM node

    // Relaciones en el árbol
    this.return = null; // Padre
    this.child = null; // Primer hijo
    this.sibling = null; // Hermano siguiente

    // Props y estado
    this.pendingProps = pendingProps;
    this.memoizedProps = null;
    this.memoizedState = null;

    // Effects
    this.flags = NoFlags; // Side effects
    this.subtreeFlags = NoFlags;

    // Scheduling
    this.lanes = NoLanes;

    // Alternate para double buffering
    this.alternate = null;
  }
}

// Work Loop - Corazón de Fiber
function workLoopConcurrent() {
  // Mientras haya trabajo y tiempo disponible
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork) {
  const current = unitOfWork.alternate;

  // Fase BEGIN: Procesar este fiber
  let next = beginWork(current, unitOfWork, renderLanes);

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // No hay hijo, completar este fiber
    completeUnitOfWork(unitOfWork);
  } else {
    // Continuar con el hijo
    workInProgress = next;
  }
}

function beginWork(current, workInProgress, renderLanes) {
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(
        current,
        workInProgress,
        workInProgress.type,
        workInProgress.pendingProps,
        renderLanes
      );
    case ClassComponent:
      return updateClassComponent(
        current,
        workInProgress,
        workInProgress.type,
        workInProgress.pendingProps,
        renderLanes
      );
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    // ... otros tipos
  }
}

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;

  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // Fase COMPLETE: Construir DOM, recolectar effects
    completeWork(current, completedWork, renderLanes);

    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // Continuar con hermano
      workInProgress = siblingFiber;
      return;
    }

    // Subir al padre
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);

  // Hemos alcanzado la raíz
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

### Prioridades y Scheduling

```javascript
// Lanes - Sistema de prioridades en React 18+
const SyncLane = 0b0000000000000000000000000000001;
const InputContinuousLane = 0b0000000000000000000000000000100;
const DefaultLane = 0b0000000000000000000000000010000;
const IdleLane = 0b0100000000000000000000000000000;

function scheduleSyncCallback(callback) {
  // Prioridad más alta - renderizado inmediato
  if (syncQueue === null) {
    syncQueue = [callback];
    scheduleCallback(ImmediatePriority, flushSyncQueue);
  } else {
    syncQueue.push(callback);
  }
}

function scheduleCallback(priorityLevel, callback) {
  const currentTime = getCurrentTime();

  const timeout = timeoutForPriorityLevel(priorityLevel);
  const expirationTime = currentTime + timeout;

  const newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime: currentTime,
    expirationTime,
    sortIndex: -1,
  };

  // Insertar en la cola de prioridad
  push(taskQueue, newTask);

  // Programar flush
  requestHostCallback(flushWork);

  return newTask;
}

// Interrumpir trabajo si hay algo más urgente
function shouldYield() {
  const currentTime = getCurrentTime();

  if (currentTime >= deadline) {
    // Frame deadline exceeded
    return true;
  }

  if (
    scheduledHostCallback !== null &&
    currentTask !== null &&
    currentTask.priorityLevel < SchedulerPriority
  ) {
    // Hay trabajo más urgente
    return true;
  }

  return false;
}
```

---

## Optimizaciones Avanzadas

### Memoización y Bailout

```javascript
// React.memo - Memoizar componentes
const MemoizedComponent = React.memo(
  function Component(props) {
    return <div>{props.value}</div>;
  },
  (prevProps, nextProps) => {
    // Retornar true si props son iguales (skip re-render)
    return prevProps.value === nextProps.value;
  }
);

// useMemo - Memoizar valores computados
function ExpensiveComponent({ data }) {
  const processed = useMemo(() => {
    console.log("Processing...");
    return data.map((item) => expensiveOperation(item));
  }, [data]); // Solo recomputar si data cambia

  return <div>{processed}</div>;
}

// useCallback - Memoizar funciones
function Parent() {
  const [count, setCount] = useState(0);

  // Sin useCallback, nueva función en cada render
  const increment = useCallback(() => {
    setCount((c) => c + 1);
  }, []); // Función estable

  return <Child onIncrement={increment} />;
}

const Child = React.memo(({ onIncrement }) => {
  console.log("Child render");
  return <button onClick={onIncrement}>+</button>;
});
```

### Batching de Actualizaciones

```javascript
// React 17 y anterior: Solo batch en event handlers
function handleClick() {
  setCount((c) => c + 1); // \
  setFlag((f) => !f); //  } Batch - un solo re-render
  setData([]); // /
}

setTimeout(() => {
  setCount((c) => c + 1); // Re-render
  setFlag((f) => !f); // Re-render
  setData([]); // Re-render
}, 1000);

// React 18+: Automatic batching everywhere
import { flushSync } from "react-dom";

setTimeout(() => {
  setCount((c) => c + 1); // \
  setFlag((f) => !f); //  } Batch - un solo re-render
  setData([]); // /
}, 1000);

// Opt-out del batching
flushSync(() => {
  setCount((c) => c + 1); // Re-render inmediato
});
setFlag((f) => !f); // Re-render inmediato
```

---

**Anterior**: [04 - Compiladores](./04-Compiladores.md)  
**Siguiente**: [06 - Signals](./06-Signals.md)
