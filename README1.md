# Allo Inventory Reservation System

A Next.js application implementing a race-condition-safe inventory reservation system for multi-warehouse retail operations.

## Live Demo

🔗 https://drive.google.com/file/d/1WdXvgv9wSikRw1g73SYyup8DUJuf-Nk7/view?usp=sharing 

## Features

- ✅ Product listing with real-time stock per warehouse
- ✅ 10-minute reservation hold during checkout
- ✅ Race-condition-safe concurrent reservations (Redis distributed locking)
- ✅ Live countdown timer on checkout page
- ✅ Automatic expiry of abandoned reservations
- ✅ 409 (insufficient stock) and 410 (expired) error handling

## Tech Stack

- **Framework:** Next.js 14 (App Router)
- **Language:** TypeScript
- **Database:** PostgreSQL (Supabase)
- **Cache/Locking:** Redis (Upstash)
- **ORM:** Prisma
- **Validation:** Zod
- **UI:** Tailwind CSS + shadcn/ui

## Getting Started

### Prerequisites

- Node.js 18+
- Supabase account (free tier)
- Upstash account (free tier)

### Environment Variables

Create a `.env` file:

\`\`\`env
DATABASE_URL="postgresql://..."
DIRECT_URL="postgresql://..."
UPSTASH_REDIS_REST_URL="https://..."
UPSTASH_REDIS_REST_TOKEN="..."
CRON_SECRET="your-secret-key"
\`\`\`

### Installation

\`\`\`bash
npm install
npx prisma migrate deploy
npx prisma db seed
npm run dev
\`\`\`

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/products | List products with available stock |
| GET | /api/warehouses | List warehouses |
| POST | /api/reservations | Create a reservation (returns 409 if insufficient stock) |
| POST | /api/reservations/:id/confirm | Confirm reservation (returns 410 if expired) |
| POST | /api/reservations/:id/release | Release reservation early |

## Concurrency Solution

The reservation system uses a two-layer approach:

1. **Redis Distributed Lock:** Before checking/modifying stock, we acquire a lock on the `stock:{productId}:{warehouseId}` key using Redis `SET NX` with a TTL. Only one request can hold the lock at a time.

2. **Prisma Transaction:** Within the lock, we use a database transaction to atomically read stock, validate availability, and update both the stock level and create the reservation.

The lock release uses a Lua script to ensure we only delete our own lock (prevents the bug where a slow process deletes a lock acquired by another process after the original lock expired).

## Expiry Mechanism

**Lazy Cleanup (Primary):** The `releaseExpiredReservations()` function runs before every stock query. This ensures accurate stock levels without a background worker.

**Vercel Cron (Secondary):** A cron job at `/api/cron/cleanup` runs every minute to catch any expired reservations that weren't cleaned up lazily.

## Trade-offs & Future Improvements

### Trade-offs Made
- **Single-unit reservations:** Currently only reserves 1 unit at a time for simplicity
- **No authentication:** Real system would have user accounts
- **No actual payment integration:** Confirm button simulates payment success

### With More Time
- Add WebSocket/SSE for real-time stock updates
- Implement cart functionality (multi-product reservations)
- Add proper payment gateway integration
- Implement the idempotency bonus feature
- Add comprehensive test coverage
- Add rate limiting
\`\`\`

### Update `tsconfig.json` for strict mode:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    // ... rest of config
  }
}
