# Carga y Prefetch Perezosa

## Lazy Loading de Componentes

```javascript
import { lazy, Suspense } from "react";

// Lazy load component
const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}

// Con named exports
const Chart = lazy(() =>
  import("./Charts").then((module) => ({
    default: module.LineChart,
  }))
);
```

## Route-Based Code Splitting

```javascript
import { lazy } from "react";
import { Routes, Route } from "react-router-dom";

const Home = lazy(() => import("./pages/Home"));
const About = lazy(() => import("./pages/About"));
const Dashboard = lazy(() => import("./pages/Dashboard"));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

## Prefetching

```javascript
// Manual prefetch
function ProductCard({ product }) {
  const prefetchDetails = () => {
    // Prefetch el chunk antes de navegar
    import("./ProductDetails");
  };

  return (
    <Link to={`/products/${product.id}`} onMouseEnter={prefetchDetails}>
      {product.name}
    </Link>
  );
}

// Next.js Link con prefetch automático
import Link from "next/link";

<Link href="/dashboard" prefetch>
  Dashboard
</Link>;

// Vite - Dynamic import con prefetch hint
const module = import(
  /* webpackPrefetch: true */
  "./OptionalFeature"
);
```

## Intersection Observer para Lazy Loading

```javascript
function LazyImage({ src, alt }) {
  const [isVisible, setIsVisible] = useState(false);
  const imgRef = useRef();

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <img
      ref={imgRef}
      src={isVisible ? src : placeholder}
      alt={alt}
      loading="lazy"
    />
  );
}
```

---

**Anterior**: [09 - Streaming y Suspense](./09-Streaming-Suspense.md)  
**Siguiente**: [11 - RPCs](./11-RPCs.md)
