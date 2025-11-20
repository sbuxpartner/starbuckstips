# T3 Stack Quick Reference

## What You Need to Know

### 1. tRPC Replaces REST API

**Before (REST with Express)**:
```typescript
// server/routes.ts
app.post("/api/partners", async (req, res) => {
  const partner = await storage.createPartner({ name: req.body.name });
  res.json(partner);
});

// client
const response = await fetch("/api/partners", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "John" })
});
const partner = await response.json();
```

**After (tRPC)**:
```typescript
// src/server/api/routers/partners.ts
export const partnersRouter = createTRPCRouter({
  create: publicProcedure
    .input(z.object({ name: z.string() }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.partner.create({ data: { name: input.name } });
    }),
});

// Client component
"use client";
export function CreatePartner() {
  const createPartner = api.partners.create.useMutation();

  const handleSubmit = async (name: string) => {
    await createPartner.mutateAsync({ name });
    // ✅ Fully type-safe! No manual types needed!
  };
}
```

### 2. Prisma Replaces Drizzle

**Before (Drizzle)**:
```typescript
import { db } from "./db";
import { partners } from "./schema";

const allPartners = await db.select().from(partners);
const newPartner = await db.insert(partners).values({ name: "John" });
```

**After (Prisma)**:
```typescript
import { db } from "~/server/db";

const allPartners = await db.partner.findMany();
const newPartner = await db.partner.create({ data: { name: "John" } });
```

### 3. Next.js App Router Replaces Wouter

**Before (Wouter)**:
```typescript
// client/src/App.tsx
import { Route } from "wouter";

function App() {
  return (
    <>
      <Route path="/" component={Home} />
      <Route path="/partners" component={Partners} />
    </>
  );
}
```

**After (Next.js App Router)**:
```
src/app/
├── layout.tsx          # Root layout
├── page.tsx            # Home page (/)
└── partners/
    └── page.tsx        # Partners page (/partners)
```

```typescript
// src/app/page.tsx
export default function HomePage() {
  return <Home />;
}

// src/app/partners/page.tsx
export default function PartnersPage() {
  return <Partners />;
}
```

### 4. tRPC Client Replaces React Query

**Before (React Query + fetch)**:
```typescript
const { data, isLoading } = useQuery({
  queryKey: ["partners"],
  queryFn: async () => {
    const res = await fetch("/api/partners");
    return res.json();
  }
});
```

**After (tRPC - React Query built-in)**:
```typescript
const { data, isLoading } = api.partners.list.useQuery();
// That's it! Fully typed, cached, and optimized
```

### 5. Server Components vs Client Components

**Server Component** (can fetch data directly):
```typescript
// src/app/page.tsx (no "use client")
import { api } from "~/trpc/server";

export default async function HomePage() {
  const partners = await api.partners.list();
  return <div>{partners.map(p => p.name)}</div>;
}
```

**Client Component** (for interactivity):
```typescript
// src/components/create-partner.tsx
"use client";

import { api } from "~/trpc/client";

export function CreatePartner() {
  const createPartner = api.partners.create.useMutation();

  return (
    <button onClick={() => createPartner.mutate({ name: "John" })}>
      Create Partner
    </button>
  );
}
```

## Key Commands

```bash
# Development
npm run dev              # Start Next.js dev server (port 3000)

# Database
npx prisma generate      # Generate Prisma client
npx prisma db push       # Push schema to database
npx prisma studio        # Open database admin UI

# Build & Deploy
npm run build            # Build for production
npm run start            # Start production server
npx vercel deploy        # Deploy to Vercel
```

## Environment Variables

```env
# .env
DATABASE_URL="postgresql://..."           # Required
AZURE_CV_KEY="..."                        # Required
AZURE_CV_ENDPOINT="..."                   # Required
OCR_ENGINE="auto"                         # Optional
NODE_ENV="development"                    # Auto-set by Next.js
```

## File Upload with tRPC

```typescript
// Client
const file = e.target.files[0];
const reader = new FileReader();
reader.onload = async (e) => {
  const base64 = e.target?.result?.toString().split(',')[1];
  await api.ocr.analyzeImage.mutate({ imageBase64: base64 });
};
reader.readAsDataURL(file);

// Server (tRPC procedure)
export const ocrRouter = createTRPCRouter({
  analyzeImage: publicProcedure
    .input(z.object({ imageBase64: z.string() }))
    .mutation(async ({ input }) => {
      const buffer = Buffer.from(input.imageBase64, "base64");
      // Process buffer
    }),
});
```

## Common Patterns

### Query (GET data)
```typescript
// Server
export const partnersRouter = createTRPCRouter({
  list: publicProcedure.query(async ({ ctx }) => {
    return ctx.db.partner.findMany();
  }),
});

// Client
const { data } = api.partners.list.useQuery();
```

### Mutation (POST/PUT/DELETE)
```typescript
// Server
export const partnersRouter = createTRPCRouter({
  create: publicProcedure
    .input(z.object({ name: z.string() }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.partner.create({ data: input });
    }),
});

// Client
const createPartner = api.partners.create.useMutation();
await createPartner.mutateAsync({ name: "John" });
```

### With Optimistic Updates
```typescript
const utils = api.useUtils();
const createPartner = api.partners.create.useMutation({
  onSuccess: () => {
    utils.partners.list.invalidate(); // Refresh list
  },
});
```

## Folder Structure at a Glance

```
t3-app/
├── prisma/
│   └── schema.prisma           # Database schema
├── src/
│   ├── app/                    # Next.js pages (App Router)
│   │   ├── api/trpc/[trpc]/route.ts  # tRPC endpoint
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── server/
│   │   ├── api/
│   │   │   ├── root.ts         # Main router
│   │   │   ├── trpc.ts         # tRPC config
│   │   │   └── routers/        # Feature routers
│   │   ├── db.ts               # Prisma client
│   │   └── services/           # Business logic
│   ├── trpc/
│   │   ├── client.tsx          # Client-side tRPC
│   │   └── server.ts           # Server-side tRPC
│   ├── components/             # React components
│   ├── lib/                    # Utilities
│   └── styles/                 # CSS
├── .env                        # Environment variables
└── package.json
```

## Migration Steps (TL;DR)

1. **Set up**: `cd t3-app && npm install`
2. **Database**: Add `DATABASE_URL` to `.env` and run `npx prisma db push`
3. **Copy services**: Move Azure OCR, Tesseract, etc. to `src/server/services/`
4. **Copy components**: Move UI components to `src/components/`
5. **Update imports**: Change to use tRPC hooks instead of fetch
6. **Test**: `npm run dev` and verify all features work
7. **Deploy**: `npx vercel deploy`

## Benefits Recap

✅ **Type Safety**: Database → API → UI (zero manual types)
✅ **Simpler**: One codebase, one server, one build
✅ **Faster**: Server Components, built-in caching, edge deployment
✅ **Better DX**: Auto-complete everywhere, instant errors
✅ **Easy Deploy**: One command to Vercel

## Resources

- Guide: `/T3_CONVERSION_COMPLETE_GUIDE.md` (detailed walkthrough)
- T3 Docs: https://create.t3.gg/
- tRPC Docs: https://trpc.io/
- Next.js: https://nextjs.org/docs
- Prisma: https://www.prisma.io/docs

---

**Ready?** Start with the complete guide and migrate step by step!
