# Pre-renderizado Parcial

## Conceptos de Pre-renderizado

El pre-renderizado genera HTML en tiempo de compilación o en el servidor, mejorando:

1. **SEO**: Contenido visible para crawlers
2. **Performance**: Primera carga más rápida
3. **UX**: Contenido inmediato sin spinners

---

## Estrategias de Pre-renderizado

### Static Site Generation (SSG)

```javascript
// Next.js - getStaticProps
export async function getStaticProps() {
  const posts = await fetchPosts();

  return {
    props: { posts },
    // Revalidar cada 60 segundos (ISR)
    revalidate: 60,
  };
}

function BlogPage({ posts }) {
  return (
    <div>
      {posts.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}

// Genera páginas dinámicas en build time
export async function getStaticPaths() {
  const posts = await fetchPosts();

  return {
    paths: posts.map((post) => ({
      params: { slug: post.slug },
    })),
    // fallback: false -> 404 si no existe
    // fallback: true -> genera bajo demanda
    // fallback: 'blocking' -> SSR para nuevas páginas
    fallback: "blocking",
  };
}
```

### Incremental Static Regeneration (ISR)

```javascript
// Combina SSG con actualizaciones automáticas
export async function getStaticProps() {
  const data = await fetchData();

  return {
    props: { data },
    revalidate: 10, // Revalidar cada 10 segundos
  };
}

// Flujo de ISR:
// 1. Primera request → Sirve HTML estático
// 2. Después de 10s → Background regeneration
// 3. Siguiente request → Sirve HTML actualizado
// 4. Repetir desde paso 2
```

### Partial Pre-rendering (PPR)

```javascript
// Next.js 14+ - PPR experimental
// Combina estático + dinámico en la misma página

import { Suspense } from "react";

// Layout es estático
export default function Page() {
  return (
    <div>
      {/* Estático - pre-renderizado */}
      <header>
        <nav>...</nav>
      </header>

      {/* Dinámico - streaming SSR */}
      <Suspense fallback={<Skeleton />}>
        <DynamicContent />
      </Suspense>

      {/* Estático otra vez */}
      <footer>...</footer>
    </div>
  );
}

// DynamicContent se renderiza en request-time
async function DynamicContent() {
  const user = await getCurrentUser();
  const notifications = await getNotifications(user.id);

  return (
    <div>
      <UserProfile user={user} />
      <Notifications items={notifications} />
    </div>
  );
}
```

---

**Anterior**: [06 - Signals](./06-Signals.md)  
**Siguiente**: [08 - Enrutamiento](./08-Enrutamiento.md)
