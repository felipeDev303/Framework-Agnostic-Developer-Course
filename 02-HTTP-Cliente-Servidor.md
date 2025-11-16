# HTTP y el Límite Servidor/Cliente

## Lección 3.1: El Modelo Mental Cliente-Servidor

### Los Paradigmas de Renderizado

```javascript
// 1. CLIENT-SIDE RENDERING (CSR)
// Todo el renderizado ocurre en el navegador
// index.html
<!DOCTYPE html>
<html>
<body>
  <div id="root"></div>
  <script src="bundle.js"></script> <!-- ~500KB -->
</body>
</html>

// bundle.js
fetch('/api/data')
  .then(res => res.json())
  .then(data => {
    ReactDOM.render(<App data={data} />, document.getElementById('root'));
  });

// Timeline:
// 1. Download HTML (pequeño)
// 2. Download JS (grande)
// 3. Parse & Execute JS
// 4. Fetch Data
// 5. Render
// Total: 2-5 segundos hasta contenido interactivo

// 2. SERVER-SIDE RENDERING (SSR)
// HTML se genera en el servidor
app.get('/', async (req, res) => {
  const data = await fetchData();
  const html = ReactDOMServer.renderToString(<App data={data} />);

  res.send(`
    <!DOCTYPE html>
    <html>
    <body>
      <div id="root">${html}</div>
      <script>
        window.__INITIAL_DATA__ = ${JSON.stringify(data)};
      </script>
      <script src="bundle.js"></script>
    </body>
    </html>
  `);
});

// Cliente hidrata el HTML existente
ReactDOM.hydrate(
  <App data={window.__INITIAL_DATA__} />,
  document.getElementById('root')
);

// Timeline:
// 1. Server fetch data + render HTML
// 2. Download HTML (con contenido)
// 3. Contenido visible inmediatamente
// 4. Download JS
// 5. Hydrate (hacer interactivo)
// Total: 0.5-1 segundo hasta contenido, 2-3 hasta interactivo

// 3. STATIC SITE GENERATION (SSG)
// HTML se genera en build time
// build.js
const pages = await getAllBlogPosts();
pages.forEach(async (page) => {
  const html = await renderPage(page);
  writeFileSync(`dist/${page.slug}.html`, html);
});

// Servir archivos estáticos (CDN)
// Timeline:
// 1. Download HTML (desde CDN, muy rápido)
// 2. Contenido visible inmediatamente
// 3. Download JS (si necesario)
// 4. Enhance (progressive enhancement)
// Total: <0.5 segundos hasta contenido

// 4. INCREMENTAL STATIC REGENERATION (ISR)
// Combina SSG con actualizaciones bajo demanda
async function getStaticProps() {
  const data = await fetchData();

  return {
    props: { data },
    revalidate: 60 // Revalidar cada 60 segundos
  };
}

// 5. EDGE RENDERING
// SSR en edge locations (cerca del usuario)
export default {
  async fetch(request, env) {
    const data = await env.KV.get('data');
    const html = renderApp(data);

    return new Response(html, {
      headers: {
        'content-type': 'text/html',
        'cache-control': 'max-age=60'
      }
    });
  }
};
```

### Comparación de Estrategias

| Estrategia | TTFB | FCP | TTI | SEO | Costo Servidor | Complejidad |
| ---------- | ---- | --- | --- | --- | -------------- | ----------- |
| CSR        | ✓✓✓  | ✗   | ✗   | ✗   | ✓✓✓            | ✓           |
| SSR        | ✓    | ✓✓  | ✓   | ✓✓✓ | ✗              | ✓✓          |
| SSG        | ✓✓✓  | ✓✓✓ | ✓✓  | ✓✓✓ | ✓✓✓            | ✓           |
| ISR        | ✓✓✓  | ✓✓✓ | ✓✓  | ✓✓✓ | ✓✓             | ✓✓          |
| Edge       | ✓✓   | ✓✓  | ✓✓  | ✓✓✓ | ✓✓             | ✓✓✓         |

---

## Lección 3.2: Hidratación vs Resumability

### El Problema de la Hidratación

```javascript
// HIDRATACIÓN TRADICIONAL
// Problema: Duplicación de trabajo entre servidor y cliente

// 1. Servidor renderiza componente
function ServerComponent({ data }) {
  const [count, setCount] = useState(0); // Estado inicial
  const expensive = useMemo(() => compute(data), [data]); // Se calcula en servidor

  return (
    <div>
      <h1>{expensive}</h1>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
    </div>
  );
}

// 2. Servidor envía HTML + Estado
<div>
  <h1>Computed Value</h1>
  <button>Count: 0</button>
</div>
<script>
  window.__STATE__ = { count: 0, data: {...} };
</script>

// 3. Cliente debe recrear TODO
// - Volver a ejecutar el componente
// - Recalcular expensive (desperdicio!)
// - Crear event listeners
// - Verificar que el HTML coincide

// PROBLEMA: El cliente hace trabajo innecesario
// - Re-ejecuta toda la lógica del componente
// - Descarga todo el JavaScript de la app
// - No puede interactuar hasta que TODO esté listo
```

### Qwik: Resumability en Acción

```javascript
// RESUMABILITY (Qwik)
// El HTML incluye toda la información necesaria para continuar

// Servidor genera HTML con estado serializado inline
<button
  on:click="./chunk-abc123.js#handleClick"
  q:id="1"
  q:obj="0">
  Count: <span q:obj="1" q:key="count">0</span>
</button>
<script type="qwik/json">
  {
    "obj": {
      "0": { "count": 0 },
      "1": { "ref": "0", "prop": "count" }
    },
    "subs": [["1", "0", "count"]]
  }
</script>

// Cuando el usuario hace click:
// 1. Se carga SOLO chunk-abc123.js (pequeño)
// 2. Se ejecuta handleClick
// 3. Se actualiza el estado
// 4. Se re-renderiza SOLO lo necesario

// Implementación del sistema de Qwik
class QwikLoader {
  constructor() {
    this.chunks = new Map();
    this.state = new Map();
    this.subscriptions = new Map();
  }

  async handleEvent(element, eventType) {
    const handler = element.getAttribute(`on:${eventType}`);
    if (!handler) return;

    const [chunkPath, functionName] = handler.split('#');

    // Cargar chunk solo cuando se necesita
    if (!this.chunks.has(chunkPath)) {
      const module = await import(chunkPath);
      this.chunks.set(chunkPath, module);
    }

    const chunk = this.chunks.get(chunkPath);
    const fn = chunk[functionName];

    // Obtener estado asociado
    const stateId = element.getAttribute('q:obj');
    const state = this.state.get(stateId);

    // Ejecutar handler con estado
    fn(state, element);
  }

  serialize(component) {
    // Serializar estado y suscripciones
    return {
      obj: Object.fromEntries(this.state),
      subs: Array.from(this.subscriptions)
    };
  }

  resume(serialized) {
    // Restaurar estado desde HTML
    Object.entries(serialized.obj).forEach(([id, obj]) => {
      this.state.set(id, obj);
    });

    // Restaurar suscripciones
    serialized.subs.forEach(([elementId, stateId, prop]) => {
      this.subscribe(elementId, stateId, prop);
    });
  }
}
```

### Partial Hydration y Progressive Enhancement

```javascript
// ASTRO - Islands Architecture
// Solo hidrata componentes interactivos

// Layout.astro - Nunca se hidrata
---
const navigation = await getNavigation();
---
<html>
  <body>
    <nav>
      {navigation.map(item => (
        <a href={item.url}>{item.label}</a>
      ))}
    </nav>

    <!-- Contenido estático -->
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>

    <!-- Solo este componente se hidrata -->
    <CommentSection client:visible />

    <!-- Se hidrata cuando el usuario interactúa -->
    <ShareButtons client:idle />

    <!-- Se hidrata solo en móvil -->
    <MobileMenu client:media="(max-width: 768px)" />
  </body>
</html>

// Estrategias de hidratación
const hydrationStrategies = {
  // Hidrata inmediatamente al cargar
  load: () => import('./Component.js'),

  // Hidrata cuando el navegador está idle
  idle: () => {
    if ('requestIdleCallback' in window) {
      requestIdleCallback(() => import('./Component.js'));
    } else {
      setTimeout(() => import('./Component.js'), 200);
    }
  },

  // Hidrata cuando es visible
  visible: (element) => {
    const observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting) {
        import('./Component.js').then(({ default: Component }) => {
          hydrate(Component, element);
        });
        observer.disconnect();
      }
    });
    observer.observe(element);
  },

  // Hidrata cuando el usuario muestra intención
  hover: (element) => {
    element.addEventListener('mouseenter', () => {
      import('./Component.js');
    }, { once: true });
  }
};
```

---

## Lección 3.3: Server Components - El Nuevo Paradigma

### React Server Components Deep Dive

```javascript
// SERVER COMPONENTS vs CLIENT COMPONENTS

// user-profile.server.jsx - Server Component
// Se ejecuta SOLO en el servidor, nunca se envía al cliente
async function UserProfile({ userId }) {
  // Puede hacer queries a DB directamente
  const user = await db.query(`SELECT * FROM users WHERE id = ?`, [userId]);
  const posts = await db.query(`SELECT * FROM posts WHERE userId = ?`, [
    userId,
  ]);

  // Puede usar Node.js APIs
  const fs = require("fs");
  const config = JSON.parse(fs.readFileSync("./config.json"));

  // El componente en sí nunca llega al cliente
  return (
    <div>
      <UserHeader user={user} />
      <PostList posts={posts} />
      {/* Client Component dentro de Server Component */}
      <LikeButton postId={posts[0].id} />
    </div>
  );
}

// like-button.client.jsx - Client Component
("use client"); // Directiva que marca componente cliente

function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);

  // Puede usar hooks y browser APIs
  useEffect(() => {
    const stored = localStorage.getItem(`liked-${postId}`);
    setLiked(stored === "true");
  }, [postId]);

  return (
    <button onClick={() => setLiked(!liked)}>{liked ? "❤️" : "🤍"}</button>
  );
}

// EL PROTOCOLO RSC
// Los Server Components no envían HTML, envían un formato especial

// RSC Wire Format (simplificado)
const rscPayload = {
  // Definición de componentes y sus props
  1: {
    type: "div",
    props: {
      children: ["2", "3", "4"],
    },
  },
  2: {
    type: "UserHeader",
    props: {
      user: { id: 1, name: "John" },
    },
  },
  3: {
    type: "PostList",
    props: {
      posts: [
        /* ... */
      ],
    },
  },
  4: {
    type: "module-reference",
    name: "LikeButton",
    chunks: ["client-chunk-123.js"],
  },
};

// Cliente procesa el payload
function processRSCPayload(payload, container) {
  function createElement(descriptor) {
    if (descriptor.type === "module-reference") {
      // Es un Client Component, cargar y renderizar
      return loadClientComponent(descriptor);
    }

    // Es elemento HTML o Server Component ya procesado
    const element = document.createElement(descriptor.type);

    if (descriptor.props.children) {
      descriptor.props.children.forEach((childId) => {
        const child = createElement(payload[childId]);
        element.appendChild(child);
      });
    }

    return element;
  }

  const root = createElement(payload["1"]);
  container.appendChild(root);
}
```

### Server Components vs SSR

```javascript
// DIFERENCIAS CLAVE

// SSR Tradicional
// 1. Renderiza TODO en el servidor
// 2. Envía HTML
// 3. Envía TODO el JavaScript
// 4. Hidrata TODO

function TraditionalSSR() {
  return (
    <div>
      <ExpensiveComponent /> {/* Se renderiza en servidor Y se hidrata */}
      <InteractiveWidget /> {/* Se renderiza en servidor Y se hidrata */}
    </div>
  );
}

// Server Components
// 1. Server Components se ejecutan en servidor
// 2. Client Components se envían como referencias
// 3. Solo el JS necesario llega al cliente
// 4. Solo Client Components se hidratan

function ServerComponent() {
  return (
    <div>
      <ExpensiveComponent /> {/* SOLO servidor, 0 JS al cliente */}
      <InteractiveWidget />  {/* Client Component, sí envía JS */}
    </div>
  );
}

// VENTAJAS DE SERVER COMPONENTS

// 1. Zero Bundle Size para lógica del servidor
// Antes: Librería de markdown (100KB) llega al cliente
import MarkdownIt from 'markdown-it'; // 100KB
function Post({ content }) {
  const md = new MarkdownIt();
  return <div dangerouslySetInnerHTML={{ __html: md.render(content) }} />;
}

// Después: La librería solo existe en el servidor
// post.server.jsx
import MarkdownIt from 'markdown-it'; // 0KB para el cliente
function Post({ content }) {
  const md = new MarkdownIt();
  const html = md.render(content); // Se ejecuta en servidor
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// 2. Acceso directo a recursos del servidor
async function ProductList() {
  // Sin APIs, directo a la DB
  const products = await prisma.product.findMany();

  // Sin waterfall de requests
  const [reviews, inventory, pricing] = await Promise.all([
    getReviews(products),
    getInventory(products),
    getPricing(products)
  ]);

  return products.map(product => (
    <ProductCard
      product={product}
      reviews={reviews[product.id]}
      inventory={inventory[product.id]}
      price={pricing[product.id]}
    />
  ));
}

// 3. Composición automática
function Layout({ children }) {
  // Layout es Server Component
  const user = await getUser(); // Server data

  return (
    <div>
      <nav>...</nav>
      {children} {/* Puede contener Client Components */}
      <Footer />
    </div>
  );
}
```

---

## Lección 3.4: Islands Architecture

### Implementación de Islands

```javascript
// CONCEPTO DE ISLANDS
// HTML estático con islas de interactividad

// island-framework.js
class IslandFramework {
  constructor() {
    this.islands = new Map();
  }

  // Registrar un componente como island
  registerIsland(name, component, hydrationStrategy = "load") {
    this.islands.set(name, {
      component,
      hydrationStrategy,
      instances: [],
    });
  }

  // Descubrir islands en el DOM
  discoverIslands() {
    document.querySelectorAll("[data-island]").forEach((element) => {
      const name = element.dataset.island;
      const props = JSON.parse(element.dataset.props || "{}");
      const strategy = element.dataset.hydrate || "load";

      const island = this.islands.get(name);
      if (!island) return;

      island.instances.push({ element, props });

      // Aplicar estrategia de hidratación
      this.applyHydrationStrategy(island, element, props, strategy);
    });
  }

  applyHydrationStrategy(island, element, props, strategy) {
    const hydrate = () => {
      import(island.component).then((module) => {
        const Component = module.default;

        // React
        ReactDOM.hydrate(<Component {...props} />, element);

        // O Vue
        // createApp(Component, props).mount(element);

        // O Vanilla
        // new Component(element, props);
      });
    };

    switch (strategy) {
      case "load":
        hydrate();
        break;

      case "idle":
        if ("requestIdleCallback" in window) {
          requestIdleCallback(hydrate);
        } else {
          setTimeout(hydrate, 1);
        }
        break;

      case "visible":
        const observer = new IntersectionObserver((entries) => {
          if (entries[0].isIntersecting) {
            hydrate();
            observer.disconnect();
          }
        });
        observer.observe(element);
        break;

      case "media":
        const mediaQuery = element.dataset.media;
        const mq = window.matchMedia(mediaQuery);

        if (mq.matches) {
          hydrate();
        } else {
          mq.addEventListener("change", (e) => {
            if (e.matches) hydrate();
          });
        }
        break;
    }
  }
}

// Uso en HTML generado
/*
<div class="page">
  <!-- Contenido estático -->
  <header>
    <h1>Mi Blog</h1>
    <nav>...</nav>
  </header>
  
  <!-- Island: Búsqueda interactiva -->
  <div data-island="SearchBar" 
       data-hydrate="idle"
       data-props='{"placeholder":"Buscar..."}'>
    <!-- HTML pre-renderizado -->
    <input type="search" placeholder="Buscar..." disabled>
  </div>
  
  <!-- Más contenido estático -->
  <article>...</article>
  
  <!-- Island: Comentarios (solo cuando visible) -->
  <div data-island="Comments" 
       data-hydrate="visible"
       data-props='{"postId":123}'>
    <!-- Placeholder mientras carga -->
    <div class="comments-skeleton">Cargando comentarios...</div>
  </div>
</div>
*/
```

---

**Anterior**: [01 - UI Web](./01-UI-Web.md)  
**Siguiente**: [03 - Componentes, Árboles y Estado](./03-Componentes-Arbol-Estado.md)
