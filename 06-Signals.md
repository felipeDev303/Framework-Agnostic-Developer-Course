# Renderizado, Reactividad y Signals

## Signals: El Futuro de la Reactividad

Los Signals representan una nueva primitiva de reactividad que está siendo adoptada por múltiples frameworks modernos (Solid, Preact, Vue 3, Angular 16+, Qwik).

**Ventajas sobre Virtual DOM:**

1. **Fine-grained reactivity**: Solo actualiza lo que cambió
2. **Sin reconciliación**: No necesita diffing del árbol completo
3. **Menor overhead**: No mantiene copias del árbol en memoria
4. **Mejor rendimiento**: Actualizaciones O(1) en lugar de O(n)

---

## Implementación de Signals

### Signal Básico

```javascript
// Sistema de signals desde cero
let currentObserver = null;
const observerStack = [];

class Signal {
  constructor(value) {
    this._value = value;
    this._observers = new Set();
  }

  get value() {
    // Auto-tracking: registrar observador actual
    if (currentObserver) {
      this._observers.add(currentObserver);
      currentObserver.dependencies.add(this);
    }
    return this._value;
  }

  set value(newValue) {
    if (this._value === newValue) return;

    this._value = newValue;

    // Notificar a todos los observadores
    this._notify();
  }

  _notify() {
    // Batch notifications
    batch(() => {
      this._observers.forEach((observer) => {
        observer.execute();
      });
    });
  }

  peek() {
    // Leer sin tracking
    return this._value;
  }
}

// Factory function
function signal(initialValue) {
  return new Signal(initialValue);
}

// Ejemplo de uso
const count = signal(0);
const double = signal(0);

effect(() => {
  // Auto-tracking: esta función se re-ejecuta cuando count cambia
  console.log("Count:", count.value);
  double.value = count.value * 2;
});

count.value = 5; // Log: "Count: 5", double ahora es 10
count.value = 10; // Log: "Count: 10", double ahora es 20
```

### Computed Values

```javascript
class Computed {
  constructor(computation) {
    this._computation = computation;
    this._value = undefined;
    this._observers = new Set();
    this._dependencies = new Set();
    this._dirty = true;

    // Es tanto un observer (de sus dependencias)
    // como un observable (para sus consumidores)
    this.execute = () => {
      this._dirty = true;
      this._notify();
    };
    this.dependencies = this._dependencies;
  }

  get value() {
    if (this._dirty) {
      this._update();
    }

    // Registrarse como dependencia
    if (currentObserver) {
      this._observers.add(currentObserver);
      currentObserver.dependencies.add(this);
    }

    return this._value;
  }

  _update() {
    // Limpiar dependencias anteriores
    this._dependencies.forEach((dep) => {
      dep._observers.delete(this);
    });
    this._dependencies.clear();

    // Ejecutar con tracking
    const prevObserver = currentObserver;
    currentObserver = this;

    try {
      this._value = this._computation();
    } finally {
      currentObserver = prevObserver;
    }

    this._dirty = false;
  }

  _notify() {
    this._observers.forEach((observer) => {
      observer.execute();
    });
  }
}

function computed(computation) {
  return new Computed(computation);
}

// Ejemplo de uso
const firstName = signal("John");
const lastName = signal("Doe");

const fullName = computed(() => {
  console.log("Computing full name...");
  return `${firstName.value} ${lastName.value}`;
});

console.log(fullName.value); // "Computing full name..." → "John Doe"
console.log(fullName.value); // No recomputa, usa caché → "John Doe"

firstName.value = "Jane";
console.log(fullName.value); // "Computing full name..." → "Jane Doe"
```

### Effects

```javascript
class Effect {
  constructor(fn, options = {}) {
    this._fn = fn;
    this._cleanup = null;
    this.dependencies = new Set();
    this._scheduler = options.scheduler;

    // Ejecutar efecto inmediatamente
    this.execute();
  }

  execute() {
    // Cleanup anterior
    if (this._cleanup) {
      this._cleanup();
      this._cleanup = null;
    }

    // Limpiar dependencias anteriores
    this.dependencies.forEach((dep) => {
      dep._observers.delete(this);
    });
    this.dependencies.clear();

    // Ejecutar con tracking
    const prevObserver = currentObserver;
    currentObserver = this;

    try {
      const result = this._fn();
      // El effect puede retornar una función cleanup
      if (typeof result === "function") {
        this._cleanup = result;
      }
    } finally {
      currentObserver = prevObserver;
    }
  }

  dispose() {
    if (this._cleanup) {
      this._cleanup();
    }

    // Desuscribirse de todas las dependencias
    this.dependencies.forEach((dep) => {
      dep._observers.delete(this);
    });
    this.dependencies.clear();
  }
}

function effect(fn, options) {
  return new Effect(fn, options);
}

// Ejemplo con cleanup
const timer = signal(0);

const dispose = effect(() => {
  console.log("Timer:", timer.value);

  const interval = setInterval(() => {
    timer.value++;
  }, 1000);

  // Cleanup function
  return () => {
    clearInterval(interval);
    console.log("Cleaned up");
  };
});

// Más tarde...
dispose.dispose(); // "Cleaned up"
```

### Batching de Actualizaciones

```javascript
let batchDepth = 0;
let batchedEffects = new Set();

function batch(fn) {
  batchDepth++;

  try {
    fn();
  } finally {
    batchDepth--;

    if (batchDepth === 0) {
      // Flush all batched effects
      const effects = Array.from(batchedEffects);
      batchedEffects.clear();

      effects.forEach((effect) => effect.execute());
    }
  }
}

// Modificar Signal para usar batching
Signal.prototype._notify = function () {
  if (batchDepth > 0) {
    // Batching activo, acumular effects
    this._observers.forEach((observer) => {
      batchedEffects.add(observer);
    });
  } else {
    // Ejecutar inmediatamente
    this._observers.forEach((observer) => {
      observer.execute();
    });
  }
};

// Uso
const a = signal(1);
const b = signal(2);
const c = computed(() => a.value + b.value);

effect(() => {
  console.log("Sum:", c.value);
});

// Sin batching: 3 logs
// a.value = 10; // Log: "Sum: 12"
// b.value = 20; // Log: "Sum: 30"

// Con batching: 1 log
batch(() => {
  a.value = 10;
  b.value = 20;
}); // Log: "Sum: 30" (solo una vez)
```

---

## Signals en Diferentes Frameworks

### Solid.js

```javascript
import { createSignal, createEffect, createMemo } from "solid-js";

function Counter() {
  const [count, setCount] = createSignal(0);
  const [multiplier, setMultiplier] = createSignal(2);

  // Computed value
  const result = createMemo(() => {
    console.log("Computing...");
    return count() * multiplier();
  });

  // Effect
  createEffect(() => {
    console.log("Result changed:", result());
  });

  return (
    <div>
      <button onClick={() => setCount(count() + 1)}>Count: {count()}</button>
      <button onClick={() => setMultiplier(multiplier() + 1)}>
        Multiplier: {multiplier()}
      </button>
      <p>Result: {result()}</p>
    </div>
  );
}

// Características de Solid:
// - No re-renderiza componentes, solo actualiza DOM
// - JSX se compila a inserciones de DOM directas
// - Signals son la única fuente de reactividad
```

### Preact Signals

```javascript
import { signal, computed, effect, batch } from "@preact/signals";

// Signals globales (fuera de componentes)
const count = signal(0);
const doubled = computed(() => count.value * 2);

effect(() => {
  console.log("Count:", count.value);
});

function Counter() {
  // También signals locales
  const localCount = signal(0);

  return (
    <div>
      {/* Auto-subscription en JSX */}
      <p>Global: {count}</p>
      <p>Doubled: {doubled}</p>
      <p>Local: {localCount}</p>

      <button
        onClick={() => {
          batch(() => {
            count.value++;
            localCount.value++;
          });
        }}
      >
        Increment Both
      </button>
    </div>
  );
}

// Características de Preact Signals:
// - Integración automática con Preact
// - Puede usar signals fuera de componentes
// - No necesita hooks, solo signals
```

### Vue 3 Composition API

```javascript
import { ref, reactive, computed, watchEffect } from "vue";

export default {
  setup() {
    // ref() es como signal()
    const count = ref(0);
    const multiplier = ref(2);

    // reactive() para objetos
    const state = reactive({
      user: {
        name: "John",
        age: 30,
      },
    });

    // computed() con caché automático
    const result = computed(() => {
      console.log("Computing...");
      return count.value * multiplier.value;
    });

    // watchEffect() es como effect()
    watchEffect(() => {
      console.log("Result:", result.value);
    });

    // watch() para observar valores específicos
    watch([count, multiplier], ([newCount, newMult], [oldCount, oldMult]) => {
      console.log("Count changed from", oldCount, "to", newCount);
    });

    return {
      count,
      multiplier,
      result,
      increment: () => count.value++,
    };
  },
};

// Características de Vue 3:
// - ref() para valores primitivos
// - reactive() para objetos (usa Proxy)
// - Integrado con template compiler
// - Tracking granular de dependencias
```

### Angular Signals (v16+)

```typescript
import { Component, signal, computed, effect } from "@angular/core";

@Component({
  selector: "app-counter",
  template: `
    <div>
      <button (click)="increment()">Count: {{ count() }}</button>
      <p>Doubled: {{ doubled() }}</p>
    </div>
  `,
})
export class CounterComponent {
  // Signal primitivo
  count = signal(0);

  // Computed signal
  doubled = computed(() => this.count() * 2);

  constructor() {
    // Effect
    effect(() => {
      console.log("Count changed:", this.count());
    });
  }

  increment() {
    // Actualizar signal
    this.count.update((c) => c + 1);

    // O set directo
    // this.count.set(10);
  }
}

// Características de Angular Signals:
// - Integración con Zone.js opcional
// - API similar a Solid
// - Detección de cambios más eficiente
// - Interoperabilidad con RxJS
```

---

## Signals vs Virtual DOM: Comparación

```javascript
// VIRTUAL DOM (React)
function TodoList({ items }) {
  const [filter, setFilter] = useState("all");

  // TODO el componente se re-ejecuta cuando filter cambia
  const filtered = items.filter((item) => {
    if (filter === "active") return !item.completed;
    if (filter === "completed") return item.completed;
    return true;
  });

  return (
    <div>
      {/* TODOS los items se reconcilian, incluso los que no cambiaron */}
      {filtered.map((item) => (
        <TodoItem key={item.id} item={item} />
      ))}
    </div>
  );
}

// SIGNALS (Solid)
function TodoList(props) {
  const [filter, setFilter] = createSignal("all");

  // Computed se actualiza SOLO cuando filter o props.items cambian
  const filtered = createMemo(() => {
    return props.items.filter((item) => {
      if (filter() === "active") return !item.completed;
      if (filter() === "completed") return item.completed;
      return true;
    });
  });

  return (
    <div>
      {/* For re-renderiza SOLO los items que realmente cambiaron */}
      <For each={filtered()}>{(item) => <TodoItem item={item} />}</For>
    </div>
  );
}

// Benchmark (1000 items, cambio en filter):
// React VDOM:    ~15ms (reconcilia 1000 vnodes)
// Solid Signals: ~2ms  (solo actualiza el DOM afectado)
```

### Cuándo Usar Cada Uno

**Virtual DOM es mejor cuando:**

- Aplicaciones con muchas actualizaciones simultáneas
- Necesitas Time Travel debugging
- Quieres React DevTools
- Tu equipo ya conoce React

**Signals son mejores cuando:**

- Necesitas máximo rendimiento
- Actualizaciones frecuentes y granulares
- Aplicaciones de tiempo real
- Menor bundle size

---

## Signals Avanzados

### Untracked Reads

```javascript
// Leer sin crear dependencia
function Logger() {
  const count = signal(0);

  effect(() => {
    // Esto se ejecuta cuando count cambia
    console.log("Count changed to:", count.value);

    // Pero esto NO crea dependencia
    const untracked = untrack(() => count.value);
    console.log("Untracked read:", untracked);
  });

  return () => count.value++;
}

function untrack(fn) {
  const prev = currentObserver;
  currentObserver = null;

  try {
    return fn();
  } finally {
    currentObserver = prev;
  }
}
```

### Signal Arrays y Objetos

```javascript
// Array reactivo
function createArraySignal(initial) {
  const signal = createSignal(initial);
  const [get, set] = signal;

  return [
    get,
    {
      push: (item) => set([...get(), item]),
      pop: () => {
        const arr = get();
        set(arr.slice(0, -1));
        return arr[arr.length - 1];
      },
      map: (fn) => get().map(fn),
      filter: (fn) => get().filter(fn),
    },
  ];
}

// Store reactivo profundo
function createStore(initial) {
  const signals = new Map();

  function get(path) {
    const key = path.join(".");
    if (!signals.has(key)) {
      const value = path.reduce((obj, prop) => obj[prop], initial);
      signals.set(key, signal(value));
    }
    return signals.get(key)[0]();
  }

  function set(path, value) {
    const key = path.join(".");
    if (!signals.has(key)) {
      signals.set(key, signal(value));
    }
    signals.get(key)[1](value);
  }

  return { get, set };
}

// Uso
const store = createStore({
  user: {
    name: "John",
    age: 30,
    settings: {
      theme: "dark",
    },
  },
});

effect(() => {
  console.log("Theme:", store.get(["user", "settings", "theme"]));
});

store.set(["user", "settings", "theme"], "light"); // Trigger effect
```

---

**Anterior**: [05 - Virtual DOM](./05-VDOM.md)  
**Siguiente**: [07 - Pre-Render Parcial](./07-PreRender-Parcial.md)
