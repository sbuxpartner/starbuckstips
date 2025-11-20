# Complete T3 Stack Conversion Guide

## Executive Summary

This guide shows you exactly how to convert your current Express + Vite + React application to a modern T3 Stack (Next.js + tRPC + Prisma + Tailwind) application.

I've created a starter T3 structure in the `/t3-app` directory with:
- ✅ package.json with all T3 dependencies
- ✅ Prisma schema matching your current database
- ✅ Directory structure for Next.js App Router
- ✅ Environment variable template

## What is T3 Stack?

The T3 Stack is a modern web development stack focused on **simplicity**, **modularity**, and **full-stack type safety**. Created by Theo Browne (t3.gg), it includes:

- **Next.js 15**: React framework with App Router (SSR, RSC, API routes)
- **tRPC 11**: End-to-end type-safe APIs (no code generation needed)
- **Prisma**: Type-safe database ORM
- **Tailwind CSS**: Utility-first CSS framework
- **TypeScript**: Type safety across the entire stack

## Why Convert to T3?

### Current Stack Pain Points
1. **No type safety** between frontend and backend
2. **Manual API type definitions** (duplication)
3. **Separate server** (Express) and **client** (Vite) setups
4. **Complex state management** (Context + React Query)
5. **Deployment complexity** (need to run both Express and serve static files)

### T3 Stack Benefits
1. **End-to-end type safety**: Database → API → UI (zero manual types)
2. **Single codebase**: Next.js handles both frontend and API
3. **Better DX**: Auto-complete everywhere, instant error feedback
4. **Simpler deployment**: One command to Vercel/Railway/Fly.io
5. **Built-in caching**: tRPC + React Query integration
6. **Server Components**: Faster page loads with RSC

## Architecture Comparison

### Current Architecture
```
┌─────────────────┐     HTTP/REST     ┌─────────────────┐
│                 │ ←────────────────→ │                 │
│  React (Vite)   │                    │  Express Server │
│  Port 5173      │                    │  Port 3000      │
│                 │                    │                 │
└─────────────────┘                    └─────────────────┘
         ↓                                       ↓
    React Query                              Drizzle
         ↓                                       ↓
    Manual Types                           PostgreSQL
```

### T3 Architecture
```
┌──────────────────────────────────────────────────────┐
│               Next.js (Port 3000)                    │
├──────────────────────────────────────────────────────┤
│  React Server Components  │  API (tRPC)              │
│  Client Components        │  Procedures              │
│  Pages (App Router)       │  Routers                 │
└───────────────────────────┴──────────────────────────┘
         ↓                             ↓
    tRPC Client                   Prisma ORM
         └─────────────┬─────────────┘
                       ↓
                  PostgreSQL
```

##Directory Structure Comparison

### Current Structure
```
project/
├── client/src/
│   ├── components/
│   ├── pages/
│   ├── lib/
│   └── context/
├── server/
│   ├── api/
│   ├── lib/
│   └── routes.ts
├── shared/
│   └── schema.ts
└── package.json
```

### T3 Structure
```
t3-app/
├── src/
│   ├── app/                    # Next.js App Router pages
│   │   ├── layout.tsx
│   │   ├── page.tsx            # Home page
│   │   └── partners/
│   │       └── page.tsx        # Partners page
│   ├── server/
│   │   ├── api/
│   │   │   ├── root.ts         # Root tRPC router
│   │   │   ├── trpc.ts         # tRPC setup
│   │   │   └── routers/
│   │   │       ├── ocr.ts
│   │   │       ├── distributions.ts
│   │   │       └── partners.ts
│   │   ├── db.ts               # Prisma client
│   │   └── services/           # Azure OCR, Tesseract, etc.
│   ├── trpc/
│   │   ├── client.tsx          # tRPC React client
│   │   └── server.ts           # tRPC server-side caller
│   ├── components/
│   │   └── ui/                 # shadcn/ui components (copy existing)
│   └── lib/
│       └── utils.ts            # Utility functions
├── prisma/
│   └── schema.prisma           # Database schema
├── public/
└── package.json
```

## Step-by-Step Conversion

### Phase 1: Set Up T3 Base (30 minutes)

#### 1. Install Dependencies
```bash
cd t3-app
npm install
```

#### 2. Set Up Environment Variables
```bash
# Copy from existing .env and add DATABASE_URL
cp ../.env .env

# You need to get DATABASE_URL from your hosting provider
# For Supabase, go to: Project Settings → Database → Connection String
# Add this to .env:
DATABASE_URL="postgresql://postgres:[YOUR-PASSWORD]@[HOST]:5432/postgres"
```

#### 3. Set Up Prisma
```bash
npx prisma generate
npx prisma db push
```

#### 4. Verify Database Connection
```bash
npx prisma studio
# This should open a database admin UI at localhost:5555
```

### Phase 2: Create tRPC Setup (1 hour)

The tRPC setup requires several interconnected files. Here's the complete implementation:

#### File 1: `src/server/db.ts`
```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;
```

#### File 2: `src/server/api/trpc.ts`
```typescript
import { initTRPC } from "@trpc/server";
import superjson from "superjson";
import { ZodError } from "zod";
import { db } from "~/server/db";

export const createTRPCContext = async (opts: { headers: Headers }) => {
  return {
    db,
    ...opts,
  };
};

const t = initTRPC.context<typeof createTRPCContext>().create({
  transformer: superjson,
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    };
  },
});

export const createCallerFactory = t.createCallerFactory;
export const createTRPCRouter = t.router;
export const publicProcedure = t.procedure;
```

#### File 3: `src/server/api/routers/partners.ts`
```typescript
import { z } from "zod";
import { createTRPCRouter, publicProcedure } from "~/server/api/trpc";

export const partnersRouter = createTRPCRouter({
  list: publicProcedure.query(async ({ ctx }) => {
    return ctx.db.partner.findMany({
      orderBy: { id: "asc" },
    });
  }),

  create: publicProcedure
    .input(
      z.object({
        name: z.string().min(1, "Partner name is required"),
      })
    )
    .mutation(async ({ ctx, input }) => {
      return ctx.db.partner.create({
        data: {
          name: input.name.trim(),
        },
      });
    }),
});
```

#### File 4: `src/server/api/routers/distributions.ts`
```typescript
import { z } from "zod";
import { createTRPCRouter, publicProcedure } from "~/server/api/trpc";

const partnerPayoutSchema = z.object({
  name: z.string(),
  hours: z.number(),
  payout: z.number(),
  rounded: z.number(),
  billBreakdown: z.array(
    z.object({
      quantity: z.number(),
      denomination: z.number(),
    })
  ),
});

export const distributionsRouter = createTRPCRouter({
  list: publicProcedure.query(async ({ ctx }) => {
    return ctx.db.distribution.findMany({
      orderBy: { date: "desc" },
    });
  }),

  calculate: publicProcedure
    .input(
      z.object({
        partnerHours: z.array(
          z.object({
            name: z.string().min(1),
            hours: z.number().positive(),
          })
        ),
        totalAmount: z.number().positive(),
      })
    )
    .mutation(async ({ input }) => {
      const { partnerHours, totalAmount } = input;

      // Calculate total hours
      const totalHours = partnerHours.reduce((sum, p) => sum + p.hours, 0);
      const hourlyRate = totalAmount / totalHours;

      // Calculate payouts (copy logic from client/src/lib/utils.ts)
      const partnerPayouts = partnerHours.map((partner) => {
        const payout = partner.hours * hourlyRate;
        const rounded = Math.round(payout);

        // Bill breakdown logic (from client/src/lib/billCalc.ts)
        const billBreakdown = calculateBillBreakdown(rounded);

        return {
          name: partner.name,
          hours: partner.hours,
          payout,
          rounded,
          billBreakdown,
        };
      });

      return {
        totalAmount,
        totalHours,
        hourlyRate,
        partnerPayouts,
      };
    }),

  save: publicProcedure
    .input(
      z.object({
        totalAmount: z.number(),
        totalHours: z.number(),
        hourlyRate: z.number(),
        partnerPayouts: z.array(partnerPayoutSchema),
      })
    )
    .mutation(async ({ ctx, input }) => {
      return ctx.db.distribution.create({
        data: {
          totalAmount: input.totalAmount,
          totalHours: input.totalHours,
          hourlyRate: input.hourlyRate,
          partnerData: input.partnerPayouts,
        },
      });
    }),
});

// Helper function - copy from client/src/lib/billCalc.ts
function calculateBillBreakdown(amount: number) {
  const denominations = [100, 50, 20, 10, 5, 1];
  const breakdown: Array<{ quantity: number; denomination: number }> = [];
  let remaining = amount;

  for (const denom of denominations) {
    if (remaining >= denom) {
      const quantity = Math.floor(remaining / denom);
      breakdown.push({ quantity, denomination: denom });
      remaining -= quantity * denom;
    }
  }

  return breakdown;
}
```

#### File 5: `src/server/api/routers/ocr.ts`
```typescript
import { z } from "zod";
import { createTRPCRouter, publicProcedure } from "~/server/api/trpc";
import { analyzeImageWithService } from "~/server/services/ocr-service";

export const ocrRouter = createTRPCRouter({
  analyzeImage: publicProcedure
    .input(
      z.object({
        imageBase64: z.string(),
      })
    )
    .mutation(async ({ input }) => {
      // Convert base64 to buffer
      const buffer = Buffer.from(input.imageBase64, "base64");

      // Use OCR service (copy from server/lib/ocrService.ts)
      const result = await analyzeImageWithService(buffer);

      if (!result.text || !result.partnerData || result.partnerData.length === 0) {
        throw new Error(result.error || "Failed to extract partner data from image");
      }

      return {
        extractedText: result.text,
        partnerHours: result.partnerData,
        confidence: result.confidence,
        engine: result.engine,
      };
    }),
});
```

#### File 6: `src/server/api/root.ts`
```typescript
import { createCallerFactory, createTRPCRouter } from "~/server/api/trpc";
import { ocrRouter } from "./routers/ocr";
import { distributionsRouter } from "./routers/distributions";
import { partnersRouter } from "./routers/partners";

export const appRouter = createTRPCRouter({
  ocr: ocrRouter,
  distributions: distributionsRouter,
  partners: partnersRouter,
});

export type AppRouter = typeof appRouter;

export const createCaller = createCallerFactory(appRouter);
```

### Phase 3: Set Up tRPC Client (30 minutes)

#### File 7: `src/trpc/server.ts`
```typescript
import "server-only";
import { headers } from "next/headers";
import { cache } from "react";
import { createCaller } from "~/server/api/root";
import { createTRPCContext } from "~/server/api/trpc";

const createContext = cache(async () => {
  const heads = new Headers(await headers());
  heads.set("x-trpc-source", "rsc");

  return createTRPCContext({
    headers: heads,
  });
});

export const api = createCaller(createContext);
```

#### File 8: `src/trpc/client.tsx`
```typescript
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { httpBatchLink, loggerLink } from "@trpc/client";
import { createTRPCReact } from "@trpc/react-query";
import { type inferRouterInputs, type inferRouterOutputs } from "@trpc/server";
import { useState } from "react";
import superjson from "superjson";
import { type AppRouter } from "~/server/api/root";

const createQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
      },
    },
  });

let clientQueryClientSingleton: QueryClient | undefined = undefined;
const getQueryClient = () => {
  if (typeof window === "undefined") {
    return createQueryClient();
  }
  return (clientQueryClientSingleton ??= createQueryClient());
};

export const api = createTRPCReact<AppRouter>();

export function TRPCReactProvider(props: { children: React.ReactNode }) {
  const queryClient = getQueryClient();

  const [trpcClient] = useState(() =>
    api.createClient({
      links: [
        loggerLink({
          enabled: (op) =>
            process.env.NODE_ENV === "development" ||
            (op.direction === "down" && op.result instanceof Error),
        }),
        httpBatchLink({
          transformer: superjson,
          url: getBaseUrl() + "/api/trpc",
          headers: () => {
            const headers = new Headers();
            headers.set("x-trpc-source", "nextjs-react");
            return headers;
          },
        }),
      ],
    })
  );

  return (
    <QueryClientProvider client={queryClient}>
      <api.Provider client={trpcClient} queryClient={queryClient}>
        {props.children}
      </api.Provider>
    </QueryClientProvider>
  );
}

function getBaseUrl() {
  if (typeof window !== "undefined") return window.location.origin;
  if (process.env.VERCEL_URL) return `https://${process.env.VERCEL_URL}`;
  return `http://localhost:${process.env.PORT ?? 3000}`;
}

export type RouterInputs = inferRouterInputs<AppRouter>;
export type RouterOutputs = inferRouterOutputs<AppRouter>;
```

### Phase 4: Create Next.js Pages (1 hour)

#### File 9: `src/app/layout.tsx`
```typescript
import "~/styles/globals.css";
import { GeistSans } from "geist/font/sans";
import { type Metadata } from "next";
import { TRPCReactProvider } from "~/trpc/client";

export const metadata: Metadata = {
  title: "TipJar - Starbucks Tip Distribution",
  description: "Calculate and distribute tips fairly among partners",
  icons: [{ rel: "icon", url: "/favicon.ico" }],
};

export default function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="en" className={`${GeistSans.variable}`}>
      <body>
        <TRPCReactProvider>{children}</TRPCReactProvider>
      </body>
    </html>
  );
}
```

#### File 10: `src/app/page.tsx`
```typescript
import { HydrateClient } from "~/trpc/server";
import { HomePage } from "~/components/home-page";

export default async function Home() {
  return (
    <HydrateClient>
      <HomePage />
    </HydrateClient>
  );
}
```

#### File 11: `src/app/api/trpc/[trpc]/route.ts`
```typescript
import { fetchRequestHandler } from "@trpc/server/adapters/fetch";
import { type NextRequest } from "next/server";
import { appRouter } from "~/server/api/root";
import { createTRPCContext } from "~/server/api/trpc";

const handler = (req: NextRequest) =>
  fetchRequestHandler({
    endpoint: "/api/trpc",
    req,
    router: appRouter,
    createContext: () => createTRPCContext({ headers: req.headers }),
    onError:
      process.env.NODE_ENV === "development"
        ? ({ path, error }) => {
            console.error(
              `❌ tRPC failed on ${path ?? "<no-path>"}: ${error.message}`
            );
          }
        : undefined,
  });

export { handler as GET, handler as POST };
```

### Phase 5: Copy Services (30 minutes)

You need to copy these files from the current `server/` directory to `t3-app/src/server/services/`:

1. **Azure OCR**: `server/lib/azureOCR.ts` → `src/server/services/azure-ocr.ts`
2. **Tesseract**: `server/api/ocr.ts` → `src/server/services/tesseract.ts`
3. **OCR Service**: `server/lib/ocrService.ts` → `src/server/services/ocr-service.ts`
4. **Table Parser**: `server/lib/tableParser.ts` → `src/server/services/table-parser.ts`

### Phase 6: Copy Components (1 hour)

Copy all components from `client/src/components/` to `t3-app/src/components/`:
- All UI components (`ui/*.tsx`)
- All page components
- Update imports to use Next.js conventions

### Phase 7: Update Component Usage (1 hour)

Convert components to use tRPC instead of fetch:

```typescript
// OLD: Using fetch
const response = await fetch("/api/partners");
const partners = await response.json();

// NEW: Using tRPC
const { data: partners } = api.partners.list.useQuery();

// OLD: POST with fetch
await fetch("/api/partners", {
  method: "POST",
  body: JSON.stringify({ name }),
});

// NEW: tRPC mutation
const createPartner = api.partners.create.useMutation();
await createPartner.mutateAsync({ name });
```

### Phase 8: Test & Deploy

```bash
# Test development
cd t3-app
npm run dev

# Test production build
npm run build
npm run start

# Deploy to Vercel
npx vercel deploy
```

## File Upload Handling

For image uploads with tRPC, you have two options:

### Option 1: Base64 (Simpler)
```typescript
// Client
const file = e.target.files[0];
const base64 = await fileToBase64(file);
await api.ocr.analyzeImage.mutate({ imageBase64: base64 });
```

### Option 2: Next.js API Route (Better for large files)
```typescript
// app/api/upload/route.ts
export async function POST(request: Request) {
  const formData = await request.formData();
  const file = formData.get("image") as File;
  // Process file
}
```

## Migration Checklist

- [ ] Phase 1: Set up T3 base structure
- [ ] Phase 2: Create tRPC routers
- [ ] Phase 3: Set up tRPC client
- [ ] Phase 4: Create Next.js pages
- [ ] Phase 5: Copy server services
- [ ] Phase 6: Copy UI components
- [ ] Phase 7: Update component API calls
- [ ] Phase 8: Test functionality
- [ ] Phase 9: Deploy to production

## Cost Comparison

### Current Hosting Options
- Render/Railway/Fly.io: $5-10/month (need container for Express)
- Separate static hosting for frontend

### T3 Hosting (Vercel)
- Free tier: Perfect for this app
- Hobby tier: $20/month (if needed)
- No container needed
- Automatic deployments from Git
- Edge caching included

## Learning Resources

- T3 Stack Docs: https://create.t3.gg/
- tRPC Docs: https://trpc.io/
- Next.js Docs: https://nextjs.org/docs
- Prisma Docs: https://www.prisma.io/docs

## Need Help?

The T3 stack has excellent documentation and a supportive community:
- Discord: https://t3.gg/discord
- GitHub: https://github.com/t3-oss/create-t3-app

## Conclusion

Converting to T3 will give you:
- ✅ Better type safety
- ✅ Simpler codebase
- ✅ Easier deployment
- ✅ Better developer experience
- ✅ Modern best practices

The initial setup takes ~5-8 hours, but the long-term benefits are significant!

---

**Ready to start?** Begin with Phase 1 and work through each phase systematically.
