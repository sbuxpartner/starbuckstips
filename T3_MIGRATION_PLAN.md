# T3 Stack Migration Plan

## Overview

Converting this Express + Vite + React application to a T3 Stack application (Next.js, tRPC, Prisma, Tailwind CSS).

## Current Stack

- **Frontend**: React 18 + Vite + Wouter (routing)
- **Backend**: Express + Node.js
- **Database**: PostgreSQL (via Neon/Supabase) with Drizzle ORM
- **API**: REST endpoints
- **OCR**: Azure Computer Vision + Tesseract.js
- **UI**: Radix UI + Tailwind CSS + shadcn/ui

## Target T3 Stack

- **Framework**: Next.js 14 (App Router)
- **API**: tRPC v11
- **Database**: PostgreSQL with Prisma ORM
- **Auth**: NextAuth.js (optional, for future)
- **UI**: Tailwind CSS + Radix UI (keep existing)
- **Type Safety**: End-to-end TypeScript

## Migration Strategy

### Phase 1: Project Setup
1. Create new Next.js 14 app with T3 structure
2. Copy existing UI components
3. Configure Prisma to connect to existing Supabase database
4. Set up tRPC with Next.js App Router

### Phase 2: Database Migration
1. Generate Prisma schema from existing Drizzle schema
2. Introspect existing database
3. Create Prisma client configuration
4. Test database connectivity

### Phase 3: API Migration
1. Convert Express routes to tRPC procedures
2. Migrate OCR endpoint → tRPC mutation
3. Migrate distribution endpoints → tRPC router
4. Migrate partner endpoints → tRPC router
5. Set up file upload handling in Next.js

### Phase 4: Frontend Migration
1. Convert pages from Wouter to Next.js App Router
2. Update components to use tRPC hooks
3. Replace React Query usage with tRPC's built-in client
4. Update state management (TipContext → tRPC)

### Phase 5: Services Migration
1. Move Azure OCR service to server-side only
2. Move Tesseract service to server-side only
3. Move table parser to server-side only
4. Update all utility functions

### Phase 6: Testing & Deployment
1. Test all functionality
2. Update environment variables
3. Build and verify production build
4. Deploy to Vercel (recommended for T3)

## Key Changes

### Routing
```typescript
// OLD: Wouter
<Route path="/" component={Home} />
<Route path="/partners" component={Partners} />

// NEW: Next.js App Router
app/page.tsx              // Home page
app/partners/page.tsx     // Partners page
```

### API Calls
```typescript
// OLD: REST with fetch
const response = await fetch('/api/ocr', {
  method: 'POST',
  body: formData
});

// NEW: tRPC
const result = await trpc.ocr.analyzeImage.mutate({
  imageBuffer: buffer
});
```

### Database
```typescript
// OLD: Drizzle
const partners = await db.select().from(partners);

// NEW: Prisma
const partners = await prisma.partner.findMany();
```

## File Structure

```
t3-app/
├── prisma/
│   └── schema.prisma
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx              # Home
│   │   └── partners/
│   │       └── page.tsx          # Partners
│   ├── components/
│   │   └── ui/                   # shadcn/ui components (copy)
│   ├── server/
│   │   ├── api/
│   │   │   ├── root.ts           # Root tRPC router
│   │   │   ├── routers/
│   │   │   │   ├── ocr.ts
│   │   │   │   ├── distributions.ts
│   │   │   │   └── partners.ts
│   │   │   └── trpc.ts           # tRPC setup
│   │   ├── db.ts                 # Prisma client
│   │   └── services/
│   │       ├── azure-ocr.ts
│   │       ├── tesseract.ts
│   │       └── table-parser.ts
│   ├── lib/
│   │   └── utils.ts
│   └── trpc/
│       ├── client.tsx
│       └── server.ts
├── .env
└── package.json
```

## Dependencies to Add

```json
{
  "dependencies": {
    "next": "^14.2.0",
    "@trpc/server": "^11.0.0",
    "@trpc/client": "^11.0.0",
    "@trpc/react-query": "^11.0.0",
    "@trpc/next": "^11.0.0",
    "@prisma/client": "^5.0.0",
    "superjson": "^2.2.0"
  },
  "devDependencies": {
    "prisma": "^5.0.0"
  }
}
```

## Dependencies to Remove

- express
- express-session
- wouter
- drizzle-orm
- drizzle-kit
- vite
- @vitejs/plugin-react
- multer
- passport (not used yet)

## Benefits of T3 Stack

1. **Type Safety**: End-to-end type safety from database to UI
2. **Better DX**: No need to manually write API types
3. **Performance**: Next.js SSR and App Router optimizations
4. **Deployment**: Easy deployment to Vercel
5. **Simplified State**: tRPC provides built-in caching and state management
6. **File Uploads**: Built-in Next.js API routes for file handling
7. **Auto Reload**: Better HMR and development experience

## Environment Variables

```env
# Database
DATABASE_URL="postgresql://..."

# Azure Computer Vision
AZURE_CV_KEY="..."
AZURE_CV_ENDPOINT="..."

# OCR Engine
OCR_ENGINE="auto"

# Next.js
NEXTAUTH_SECRET="..." # for future auth
NEXTAUTH_URL="http://localhost:3000"
```

## Migration Steps (Detailed)

### Step 1: Initialize T3 App
```bash
npx create-t3-app@latest tipjar-t3 \
  --tailwind \
  --trpc \
  --prisma \
  --noAuth
```

### Step 2: Set Up Prisma Schema
```prisma
// prisma/schema.prisma
model Partner {
  id    Int    @id @default(autoincrement())
  name  String
}

model Distribution {
  id           Int      @id @default(autoincrement())
  date         DateTime @default(now())
  totalAmount  Float
  totalHours   Float
  hourlyRate   Float
  partnerData  Json
}
```

### Step 3: Create tRPC Routers
```typescript
// src/server/api/routers/ocr.ts
export const ocrRouter = createTRPCRouter({
  analyzeImage: publicProcedure
    .input(z.object({ imageBase64: z.string() }))
    .mutation(async ({ input }) => {
      // OCR logic here
    })
});
```

### Step 4: Migrate Components
```typescript
// src/app/page.tsx
import { api } from "~/trpc/server";

export default async function Home() {
  const partners = await api.partners.list();
  return <HomePage partners={partners} />;
}
```

## Testing Checklist

- [ ] OCR image upload and processing
- [ ] Partner management (CRUD)
- [ ] Distribution calculations
- [ ] Distribution history
- [ ] Bill breakdown calculations
- [ ] Azure Computer Vision integration
- [ ] Tesseract fallback
- [ ] Database persistence
- [ ] Environment variables
- [ ] Production build

## Deployment

Recommended: **Vercel** (built by Next.js creators)

```bash
# Deploy to Vercel
vercel deploy

# Or connect GitHub repo for auto-deployments
```

## Timeline Estimate

- Phase 1: 2 hours (Project setup)
- Phase 2: 1 hour (Database migration)
- Phase 3: 3 hours (API migration)
- Phase 4: 3 hours (Frontend migration)
- Phase 5: 2 hours (Services migration)
- Phase 6: 2 hours (Testing & deployment)

**Total: ~13 hours**

## Questions to Consider

1. Do you want to migrate everything or keep existing app alongside?
2. Do you want authentication (NextAuth.js)?
3. Deploy to Vercel or keep current hosting?
4. Keep Supabase database or migrate to Vercel Postgres?

## Next Steps

1. Review this plan
2. Approve migration approach
3. Start with Phase 1: Create T3 app structure
4. Migrate incrementally with testing at each phase
