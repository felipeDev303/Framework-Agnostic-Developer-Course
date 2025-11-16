# Enrutamiento

## Client-Side Routing

### React Router v6

```javascript
import { BrowserRouter, Routes, Route, Link } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/users/:id" element={<UserProfile />} />
      </Routes>
    </BrowserRouter>
  );
}

// Acceder a parámetros
function UserProfile() {
  const { id } = useParams();
  const navigate = useNavigate();
  const location = useLocation();

  return <div>User {id}</div>;
}
```

## Server-Side Routing

### Next.js App Router

```javascript
// app/page.tsx - Ruta: /
export default function HomePage() {
  return <h1>Home</h1>;
}

// app/about/page.tsx - Ruta: /about
export default function AboutPage() {
  return <h1>About</h1>;
}

// app/blog/[slug]/page.tsx - Ruta: /blog/:slug
export default function BlogPost({ params }) {
  return <h1>Post: {params.slug}</h1>;
}
```

## File-Based Routing

Los frameworks modernos usan convenciones de archivos:

- Next.js: `pages/` o `app/` directory
- SvelteKit: `src/routes/`
- Remix: `app/routes/`
- Nuxt: `pages/`

---

**Anterior**: [07 - Pre-Render Parcial](./07-PreRender-Parcial.md)  
**Siguiente**: [09 - Streaming y Suspense](./09-Streaming-Suspense.md)
