# Streaming, Diferimiento y Suspense

## React Suspense

```javascript
import { Suspense } from "react";

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <ProfilePage />
    </Suspense>
  );
}

// Component que "suspende"
function ProfilePage() {
  // Esto lanza una promesa que React captura
  const user = use(fetchUser());

  return <div>{user.name}</div>;
}

// Hook use() de React 19
function use(promise) {
  if (promise.status === "fulfilled") {
    return promise.value;
  }

  if (promise.status === "rejected") {
    throw promise.reason;
  }

  // Suspender
  throw promise;
}
```

## Streaming SSR

```javascript
// Server
import { renderToReadableStream } from "react-dom/server";

async function handler(req, res) {
  const stream = await renderToReadableStream(<App />, {
    onError(error) {
      console.error(error);
    },
  });

  // Stream HTML al cliente
  res.setHeader("Content-Type", "text/html");
  stream.pipeTo(res);
}
```

## Diferimiento con defer

```javascript
// Remix - defer para datos no críticos
export async function loader() {
  return defer({
    // Datos críticos (await)
    post: await getPost(),

    // Datos no críticos (sin await)
    comments: getComments(),
    recommendations: getRecommendations(),
  });
}

function Post() {
  const { post, comments, recommendations } = useLoaderData();

  return (
    <div>
      <h1>{post.title}</h1>

      <Suspense fallback={<CommentsSkeleton />}>
        <Await resolve={comments}>
          {(comments) => <Comments data={comments} />}
        </Await>
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Await resolve={recommendations}>
          {(recs) => <Recommendations data={recs} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

---

**Anterior**: [08 - Enrutamiento](./08-Enrutamiento.md)  
**Siguiente**: [10 - Lazy Loading](./10-Lazy-Loading.md)
