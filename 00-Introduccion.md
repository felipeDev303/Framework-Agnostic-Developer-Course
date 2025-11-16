# Introducción

> **Objetivo del Curso**: No aprender a usar frameworks específicos, sino entender **cómo** y **por qué** funcionan, preparándote para cualquier framework presente o futuro.

---

## Lección 1.1: La Evolución del Desarrollo Web y Por Qué Existen los Frameworks

### El Problema Fundamental

Antes de entender los frameworks, necesitamos entender el problema que resuelven. El desarrollo web comenzó con manipulación directa del DOM:

```javascript
// Era pre-framework (jQuery)
$(document).ready(function () {
  let todos = [];

  $("#add-todo").click(function () {
    const text = $("#todo-input").val();
    todos.push({ id: Date.now(), text, completed: false });
    renderTodos();
  });

  function renderTodos() {
    $("#todo-list").empty();
    todos.forEach((todo) => {
      const li = $("<li>")
        .text(todo.text)
        .addClass(todo.completed ? "completed" : "")
        .click(() => {
          todo.completed = !todo.completed;
          renderTodos(); // Re-renderizar todo
        });
      $("#todo-list").append(li);
    });
  }
});
```

**Problemas con este enfoque:**

1. **Estado desincronizado**: El estado (datos) y la vista (DOM) se manejan separadamente
2. **Renderizado ineficiente**: Re-renderizamos todo cuando algo cambia
3. **Difícil de mantener**: La lógica de UI está mezclada con manipulación DOM
4. **No composable**: Difícil reutilizar componentes

### Los Tres Paradigmas de Solución

Los frameworks modernos abordan estos problemas con tres enfoques principales:

```javascript
// 1. TEMPLATE-BASED (Vue, Angular)
// Extiende HTML con directivas y bindings
<template>
  <div v-for="todo in todos" :key="todo.id">
    <span :class="{ completed: todo.completed }">{{ todo.text }}</span>
    <button @click="todo.completed = !todo.completed">Toggle</button>
  </div>
</template>

// 2. COMPONENT-BASED (React)
// JavaScript que genera HTML
function TodoList({ todos }) {
  return todos.map(todo => (
    <div key={todo.id}>
      <span className={todo.completed ? 'completed' : ''}>
        {todo.text}
      </span>
      <button onClick={() => toggleTodo(todo.id)}>Toggle</button>
    </div>
  ));
}

// 3. COMPILER-BASED (Svelte, Solid)
// Transforma el código en tiempo de compilación
// Svelte
{#each todos as todo (todo.id)}
  <div>
    <span class:completed={todo.completed}>{todo.text}</span>
    <button on:click={() => todo.completed = !todo.completed}>
      Toggle
    </button>
  </div>
{/each}
// Se compila a JavaScript vanilla optimizado
```

### La Convergencia de Ideas

Interesantemente, todos los frameworks están convergiendo hacia ideas similares:

| Concepto          | React  | Vue             | Angular   | Svelte    | Solid     | Qwik    |
| ----------------- | ------ | --------------- | --------- | --------- | --------- | ------- |
| Componentes       | ✓      | ✓               | ✓         | ✓         | ✓         | ✓       |
| Reactividad       | Hooks  | Composition API | RxJS      | Compilada | Signals   | Signals |
| Templates         | JSX    | Templates/JSX   | Templates | Templates | JSX       | JSX     |
| Compilación       | Mínima | Parcial         | AOT       | Total     | Parcial   | Total   |
| Server Components | ✓      | En desarrollo   | -         | ✓ (Kit)   | ✓ (Start) | Nativo  |

---

## Lección 1.2: Stack Dive - Tu Primera Exploración

### Desafío: Construye Tu Propio Sistema de Reactividad

Vamos a implementar un sistema de reactividad básico para entender cómo funcionan:

```javascript
// mini-framework.js
class MiniFramework {
  constructor() {
    this.state = {};
    this.effects = {};
    this.currentEffect = null;
  }

  // Crear estado reactivo
  createState(initialValue) {
    const key = Symbol("state");
    let value = initialValue;
    const subscribers = new Set();

    this.state[key] = {
      get: () => {
        // Auto-tracking: si hay un efecto activo, suscribirlo
        if (this.currentEffect) {
          subscribers.add(this.currentEffect);
        }
        return value;
      },
      set: (newValue) => {
        if (value !== newValue) {
          value = newValue;
          // Notificar a todos los suscriptores
          subscribers.forEach((effect) => effect());
        }
      },
    };

    return [
      () => this.state[key].get(), // getter
      (v) => this.state[key].set(v), // setter
    ];
  }

  // Crear efecto reactivo
  createEffect(fn) {
    const effect = () => {
      this.currentEffect = effect;
      fn();
      this.currentEffect = null;
    };
    effect(); // Ejecutar inmediatamente
    return effect;
  }

  // Crear computed value
  createComputed(fn) {
    let cache;
    let dirty = true;

    const computed = this.createEffect(() => {
      dirty = true;
    });

    return () => {
      if (dirty) {
        this.currentEffect = computed;
        cache = fn();
        this.currentEffect = null;
        dirty = false;
      }
      return cache;
    };
  }
}

// Uso del mini-framework
const framework = new MiniFramework();

// Estado reactivo
const [count, setCount] = framework.createState(0);
const [multiplier, setMultiplier] = framework.createState(2);

// Computed value
const doubled = framework.createComputed(() => count() * multiplier());

// Efecto que actualiza el DOM
framework.createEffect(() => {
  document.getElementById(
    "output"
  ).textContent = `Count: ${count()}, Doubled: ${doubled()}`;
});

// Actualizar estado dispara efectos automáticamente
setCount(5); // DOM se actualiza
setMultiplier(3); // DOM se actualiza
```

**Ejercicio**: Extiende este framework para soportar:

1. Arrays reactivos
2. Objetos anidados
3. Batching de actualizaciones
4. Cleanup de efectos

---

## Los Principios Universales

1. **Todo es un trade-off**: No hay soluciones perfectas, solo apropiadas para cada caso
2. **La abstracción tiene costo**: Cada capa añade complejidad y overhead
3. **El rendimiento percibido importa más que el absoluto**
4. **La developer experience influye en la calidad del producto**
5. **Los frameworks convergen hacia las mejores ideas**

---

**Siguiente**: [01 - Creando una Interfaz de Usuario en la Web](./01-UI-Web.md)
