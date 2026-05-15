# Cal.diy Application Flow - Deep Dive

This document provides an in-depth explanation of the architecture and data flow within the Cal.diy (Cal.com fork) codebase.

---

## 1. Monorepo Architecture
Cal.diy is built as a **Yarn/Turbo Monorepo**. This structure allows for code sharing across different applications while maintaining clear boundaries.

### Key Packages & Applications
- **`apps/web`**: The main Next.js 13+ application. It handles both the authenticated dashboard and public-facing booking pages.
- **`packages/trpc`**: The type-safe API layer. It defines how the frontend communicates with the backend.
- **`packages/prisma`**: The database layer, containing the schema and generated client.
- **`packages/features`**: Domain-specific logic (e.g., `bookings`, `availability`, `eventtypes`). This is where the "business logic" lives.
- **`packages/ui`**: Shared React components and design system.
- **`packages/app-store`**: Framework for 3rd-party integrations (Calendars, Conferencing, Payments).

---

## 2. Request & Data Flow

### A. Authentication Flow (NextAuth.js)
1. **Request**: A user visits `cal.diy`.
2. **Middleware/Layout**: The app checks for a session cookie.
3. **Session Verification**: NextAuth validates the JWT or database session.
4. **Context**: The `getServerSession` utility provides the `user` object to Server Components.
5. **Redirection**:
   - Authenticated: Redirected to `/event-types`.
   - Unauthenticated: Redirected to `/auth/login`.

### B. API Flow (tRPC)
Instead of standard REST endpoints, Cal.diy uses **tRPC**.
1. **Frontend**: A component calls a hook: `trpc.viewer.me.useQuery()`.
2. **Routing**: The request is routed through `apps/web/app/api/trpc/[trpc]/route.ts` to the `packages/trpc/server/routers/_app.ts`.
3. **Context Creation**: `createContext.ts` injects the database instance (`prisma`) and user session into the procedure.
4. **Procedure Execution**: The backend logic runs (often calling a service in `packages/features`).
5. **Response**: The data is returned with full TypeScript types, ensuring the frontend knows exactly what it's receiving.

### C. Booking Flow (The Core Engine)
The most complex part of the app is calculating availability and creating bookings.

1. **Availability Query**:
   - The app fetches the host's `Availability` records (e.g., Mon-Fri, 9-5).
   - It identifies all connected calendars (Google, Outlook) from the `Credential` table.
   - It calls the **`busyTimes`** service in `packages/features` to fetch real-time events from these calendars.
   - The **Slots Service** subtracts "busy" blocks from "available" blocks to generate the free slots shown to the booker.

2. **Creating a Booking**:
   - The booker submits the form.
   - A tRPC mutation validates the slot is still free.
   - A `Booking` record is created in Prisma.
   - **Background Tasks**:
     - **Calendar Sync**: A new event is pushed to the host's calendar.
     - **Meeting Link**: If applicable, a conferencing link is generated (Zoom, Cal Video, etc.).
     - **Notifications**: Emails are sent using the `NotificationService`.

---

## 3. Data Modeling (Prisma)
The database is the source of truth. Key models include:
- **`User`**: Profiles, settings, and credentials.
- **`EventType`**: Templates for meetings (duration, location, hidden/public).
- **`Booking`**: A specific instance of a scheduled meeting.
- **`Credential`**: OAuth tokens for integrations (NEVER expose the `key` field).
- **`Schedule`**: Reusable sets of availability rules.

---

## 4. Developer Workflows & Rules

### Type Safety
- Always import types using `import type { ... }`.
- Never use `as any`.
- Prisma types are generated automatically after schema changes (`yarn prisma generate`).

### Performance & Security
- **Prisma Queries**: Use `select` instead of `include` to fetch only necessary fields and prevent leaking sensitive data.
- **Early Returns**: Use early returns to keep logic flat and readable.

### UI Development
- Components should be built in `packages/ui`.
- Use **Tailwind CSS** for styling.
- All strings must be internationalized via `packages/i18n`.

---

## 5. Summary of Key Files
- `packages/prisma/schema.prisma`: The database definition.
- `packages/trpc/server/routers/viewer/_router.ts`: The main entry point for authenticated API calls.
- `apps/web/app/layout.tsx`: The root layout for the web app.
- `packages/features/bookings/lib/createBooking.ts`: Core logic for creating meetings.
