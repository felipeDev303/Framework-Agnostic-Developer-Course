# RPCs - Remote Procedure Calls

## Server Actions (React)

```javascript
// app/actions.ts
"use server";

export async function createPost(formData: FormData) {
  const title = formData.get("title");
  const content = formData.get("content");

  await db.post.create({
    data: { title, content },
  });

  revalidatePath("/posts");
}

// app/new-post/page.tsx
import { createPost } from "../actions";

export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit">Create</button>
    </form>
  );
}
```

## tRPC - Type-Safe APIs

```typescript
// server/router.ts
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.create();

export const appRouter = t.router({
  getUser: t.procedure
    .input(z.object({ id: z.string() }))
    .query(async ({ input }) => {
      return await db.user.findUnique({
        where: { id: input.id },
      });
    }),

  createPost: t.procedure
    .input(
      z.object({
        title: z.string(),
        content: z.string(),
      })
    )
    .mutation(async ({ input }) => {
      return await db.post.create({
        data: input,
      });
    }),
});

export type AppRouter = typeof appRouter;

// client/page.tsx
import { trpc } from "./trpc";

function UserProfile({ id }: { id: string }) {
  // Totalmente type-safe!
  const { data: user } = trpc.getUser.useQuery({ id });
  const createPost = trpc.createPost.useMutation();

  return (
    <div>
      <h1>{user?.name}</h1>
      <button
        onClick={() =>
          createPost.mutate({
            title: "New Post",
            content: "Content here",
          })
        }
      >
        Create Post
      </button>
    </div>
  );
}
```

## Qwik Server$

```typescript
import { component$, useSignal } from "@builder.io/qwik";
import { server$ } from "@builder.io/qwik-city";

// Función que se ejecuta SOLO en el servidor
const getUser = server$(async function (id: string) {
  // Acceso directo a DB, filesystem, etc.
  const user = await db.user.findUnique({ where: { id } });
  return user;
});

export default component$(() => {
  const userId = useSignal("123");
  const user = useSignal(null);

  return (
    <div>
      <button
        onClick$={async () => {
          // Llamada RPC al servidor
          user.value = await getUser(userId.value);
        }}
      >
        Load User
      </button>

      {user.value && <div>{user.value.name}</div>}
    </div>
  );
});
```

---

**Anterior**: [10 - Lazy Loading](./10-Lazy-Loading.md)  
**Siguiente**: [12 - Stack Dive](./12-Stack-Dive.md)
