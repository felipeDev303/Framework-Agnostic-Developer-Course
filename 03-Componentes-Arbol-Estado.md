# Componentes, Árboles y Estado

## Lección 4.1: Componentes - La Unidad de Composición

### Evolución de los Componentes

```javascript
// 1. JQUERY PLUGINS (2000s)
// Encapsulación básica, sin estado real
$.fn.counter = function(options) {
  const settings = $.extend({
    initialValue: 0,
    step: 1
  }, options);

  return this.each(function() {
    let value = settings.initialValue;
    const $element = $(this);

    $element.html(`
      <span class="value">${value}</span>
      <button class="increment">+</button>
      <button class="decrement">-</button>
    `);

    $element.on('click', '.increment', () => {
      value += settings.step;
      $element.find('.value').text(value);
    });
  });
};

// 2. BACKBONE VIEWS (Early 2010s)
// Separación de concerns, pero verbose
const CounterView = Backbone.View.extend({
  template: _.template($('#counter-template').html()),

  events: {
    'click .increment': 'increment',
    'click .decrement': 'decrement'
  },

  initialize: function() {
    this.model.on('change', this.render, this);
  },

  render: function() {
    this.$el.html(this.template(this.model.toJSON()));
    return this;
  }
});

// 3. ANGULAR DIRECTIVES (2010-2016)
// Two-way data binding, pero complejo
angular.module('app').directive('counter', function() {
  return {
    restrict: 'E',
    scope: {
      value: '=',
      step: '@'
    },
    template: `
      <div>
        <span>{{value}}</span>
        <button ng-click="increment()">+</button>
      </div>
    `,
    controller: function($scope) {
      $scope.increment = function() {
        $scope.value += parseInt($scope.step);
      };
    }
  };
});

// 4. REACT CLASS COMPONENTS (2013-2018)
// Unidirectional data flow, lifecycle methods
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { value: 0 };
  }

  componentDidMount() {
    console.log('Mounted');
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.value !== this.state.value) {
      console.log('Value changed');
    }
  }

  render() {
    return (
      <div>
        <span>{this.state.value}</span>
        <button onClick={() => this.setState({
          value: this.state.value + 1
        })}>+</button>
      </div>
    );
  }
}

// 5. FUNCTION COMPONENTS + HOOKS (2018+)
// Simplicidad y composabilidad
function Counter({ initialValue = 0, step = 1 }) {
  const [value, setValue] = useState(initialValue);

  useEffect(() => {
    console.log('Value changed:', value);
  }, [value]);

  return (
    <div>
      <span>{value}</span>
      <button onClick={() => setValue(v => v + step)}>+</button>
    </div>
  );
}

// 6. COMPILED COMPONENTS (Svelte, Solid)
// Desaparece la abstracción en runtime
// Svelte
<script>
  export let step = 1;
  let value = 0;

  $: console.log('Value changed:', value);
</script>

<div>
  <span>{value}</span>
  <button on:click={() => value += step}>+</button>
</div>

// Se compila a:
function create_fragment(ctx) {
  let span, t, button;

  return {
    c() {
      span = element("span");
      t = text(ctx[0]);
      button = element("button");
      button.textContent = "+";
    },
    m(target, anchor) {
      insert(target, span, anchor);
      append(span, t);
      insert(target, button, anchor);
      listen(button, "click", ctx[1]);
    },
    p(ctx, dirty) {
      if (dirty & 1) set_data(t, ctx[0]);
    }
  };
}
```

### Anatomía de un Componente Moderno

```javascript
// COMPONENTE COMPLETO MODERNO
// Muestra todas las características de un componente

// TypeScript para type safety
interface TodoItemProps {
  id: string;
  text: string;
  completed: boolean;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

// Componente con todas las features modernas
const TodoItem: React.FC<TodoItemProps> = memo(
  ({ id, text, completed, onToggle, onDelete }) => {
    // Estado local
    const [isEditing, setIsEditing] = useState(false);
    const [editText, setEditText] = useState(text);

    // Refs para acceso DOM
    const inputRef = useRef < HTMLInputElement > null;

    // Contexto global
    const { theme, user } = useContext(AppContext);

    // Estado derivado
    const isOwner = useMemo(
      () => user?.id === todo.userId,
      [user, todo.userId]
    );

    // Side effects
    useEffect(() => {
      if (isEditing && inputRef.current) {
        inputRef.current.focus();
        inputRef.current.select();
      }
    }, [isEditing]);

    // Custom hooks
    const { mutate: updateTodo } = useUpdateTodo();

    // Event handlers
    const handleSave = useCallback(() => {
      if (editText.trim() !== text) {
        updateTodo({ id, text: editText });
      }
      setIsEditing(false);
    }, [editText, text, id, updateTodo]);

    const handleKeyDown = useCallback(
      (e: KeyboardEvent) => {
        if (e.key === "Enter") handleSave();
        if (e.key === "Escape") {
          setEditText(text);
          setIsEditing(false);
        }
      },
      [handleSave, text]
    );

    // Render
    return (
      <div
        className={cn(
          "todo-item",
          completed && "completed",
          theme === "dark" && "dark-theme"
        )}
      >
        <input
          type="checkbox"
          checked={completed}
          onChange={() => onToggle(id)}
          aria-label="Toggle completion"
        />

        {isEditing ? (
          <input
            ref={inputRef}
            value={editText}
            onChange={(e) => setEditText(e.target.value)}
            onBlur={handleSave}
            onKeyDown={handleKeyDown}
            className="edit-input"
          />
        ) : (
          <span
            onDoubleClick={() => isOwner && setIsEditing(true)}
            className="todo-text"
          >
            {text}
          </span>
        )}

        {isOwner && (
          <button onClick={() => onDelete(id)} aria-label="Delete todo">
            ×
          </button>
        )}
      </div>
    );
  }
);

TodoItem.displayName = "TodoItem";

export default TodoItem;
```

---

## Lección 4.2: Props vs Estado vs Contexto

### El Flujo de Datos

```javascript
// PROPS - Datos que fluyen de padre a hijo
// Inmutables desde la perspectiva del hijo
function Parent() {
  const [parentState, setParentState] = useState("data");

  return (
    <Child
      data={parentState}
      onUpdate={setParentState}
      config={{ readOnly: true }}
    />
  );
}

function Child({ data, onUpdate, config }) {
  // ❌ No puedes hacer esto:
  // data = 'new value';

  // ✅ Pero puedes notificar al padre:
  return <button onClick={() => onUpdate("new value")}>Update Parent</button>;
}

// ESTADO - Datos locales del componente
// Mutable a través de setState
function StatefulComponent() {
  // Estado primitivo
  const [count, setCount] = useState(0);

  // Estado complejo
  const [user, setUser] = useState({
    name: "John",
    age: 30,
    preferences: {
      theme: "dark",
    },
  });

  // ❌ Mutación directa (no funcionará)
  const wrongUpdate = () => {
    user.age = 31;
    setUser(user); // React no detecta el cambio
  };

  // ✅ Crear nuevo objeto
  const correctUpdate = () => {
    setUser({
      ...user,
      age: 31,
    });
  };

  // ✅ Para objetos anidados
  const updateTheme = (theme) => {
    setUser({
      ...user,
      preferences: {
        ...user.preferences,
        theme,
      },
    });
  };

  // ✅ O usar Immer para sintaxis más limpia
  const updateWithImmer = () => {
    setUser(
      produce((draft) => {
        draft.age = 31;
        draft.preferences.theme = "light";
      })
    );
  };
}

// CONTEXTO - Datos que atraviesan el árbol
// Evita prop drilling
const ThemeContext = createContext();
const UserContext = createContext();
const ConfigContext = createContext();

// Provider Pattern
function App() {
  const [theme, setTheme] = useState("light");
  const [user, setUser] = useState(null);

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <UserContext.Provider value={user}>
        <ConfigContext.Provider value={config}>
          <DeepChildComponent />
        </ConfigContext.Provider>
      </UserContext.Provider>
    </ThemeContext.Provider>
  );
}

// Consumir múltiples contextos
function DeepChildComponent() {
  const { theme, setTheme } = useContext(ThemeContext);
  const user = useContext(UserContext);
  const config = useContext(ConfigContext);

  // Ahora tiene acceso a todo sin prop drilling
  return <div className={theme}>Welcome, {user?.name}!</div>;
}
```

### Patrones Avanzados de Estado

```javascript
// REDUCER PATTERN - Estado complejo con lógica
function todoReducer(state, action) {
  switch (action.type) {
    case 'ADD':
      return [...state, {
        id: Date.now(),
        text: action.payload,
        completed: false
      }];

    case 'TOGGLE':
      return state.map(todo =>
        todo.id === action.payload
          ? { ...todo, completed: !todo.completed }
          : todo
      );

    case 'DELETE':
      return state.filter(todo => todo.id !== action.payload);

    case 'EDIT':
      return state.map(todo =>
        todo.id === action.payload.id
          ? { ...todo, text: action.payload.text }
          : todo
      );

    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function TodoList() {
  const [todos, dispatch] = useReducer(todoReducer, []);

  // Acciones más semánticas
  const addTodo = (text) => dispatch({ type: 'ADD', payload: text });
  const toggleTodo = (id) => dispatch({ type: 'TOGGLE', payload: id });
  const deleteTodo = (id) => dispatch({ type: 'DELETE', payload: id });

  return (/*...*/);
}

// STATE MACHINES - Estado con transiciones explícitas
const authMachine = {
  initial: 'idle',
  states: {
    idle: {
      on: {
        LOGIN: 'loading'
      }
    },
    loading: {
      on: {
        SUCCESS: 'authenticated',
        ERROR: 'error'
      }
    },
    authenticated: {
      on: {
        LOGOUT: 'idle'
      }
    },
    error: {
      on: {
        RETRY: 'loading',
        CANCEL: 'idle'
      }
    }
  }
};

function useStateMachine(machine) {
  const [state, setState] = useState(machine.initial);

  const send = (event) => {
    const currentStateConfig = machine.states[state];
    const nextState = currentStateConfig.on[event];

    if (nextState) {
      setState(nextState);
    }
  };

  return [state, send];
}

// URL STATE - Estado sincronizado con URL
function useUrlState(key, defaultValue) {
  const [searchParams, setSearchParams] = useSearchParams();

  const value = searchParams.get(key) || defaultValue;

  const setValue = useCallback((newValue) => {
    setSearchParams(prev => {
      if (newValue === defaultValue) {
        prev.delete(key);
      } else {
        prev.set(key, newValue);
      }
      return prev;
    });
  }, [key, defaultValue]);

  return [value, setValue];
}

// Uso
function SearchPage() {
  const [query, setQuery] = useUrlState('q', '');
  const [filter, setFilter] = useUrlState('filter', 'all');
  const [page, setPage] = useUrlState('page', '1');

  // Estado persistido en URL:
  // /search?q=react&filter=recent&page=2
}
```

---

**Anterior**: [02 - HTTP y el Límite Servidor/Cliente](./02-HTTP-Cliente-Servidor.md)  
**Siguiente**: [04 - Compiladores](./04-Compiladores.md)
