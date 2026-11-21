# E-Commerce REST API

A backend API for an online store, featuring JWT authentication with role-based access control, a cached and searchable product catalog, and transactional order checkout. Built with Node.js, Express, TypeScript, PostgreSQL, Prisma and Redis.

## Features

- **Authentication:** registration and login with bcrypt-hashed passwords and signed JWTs (15-minute expiry)
- **Role-based access control:** `USER` and `ADMIN` roles, with admin-only routes
- **Security:** Helmet security headers, CORS restricted to a trusted origin, and login rate limiting (5 attempts per 15 minutes) backed by Redis
- **Products:** paginated listing with case-insensitive name search; admin-only create, update and delete
- **Redis caching:** the product listing uses the Cache-Aside pattern with a 1-hour TTL
- **Cache invalidation:** creating, updating or deleting a product, and placing an order (stock changes), purges the cached lists immediately
- **Orders:** checkout runs inside a single Prisma `$transaction` that validates products, deducts stock atomically, calculates the total on the server and rolls everything back on failure
- **Relational data model:** `User`, `Product`, `Order` and `OrderItem` with one-to-many and many-to-many relations

## Tech Stack

| Layer           | Technology                                  |
| --------------- | ------------------------------------------- |
| Runtime         | Node.js                                     |
| Framework       | Express 5                                   |
| Language        | TypeScript                                  |
| Database        | PostgreSQL                                  |
| ORM             | Prisma (with `@prisma/adapter-pg`)          |
| Cache           | Redis (node-redis), run locally with Docker |
| Auth & Security | jsonwebtoken, bcrypt, helmet, cors          |
| Rate limiting   | express-rate-limit + rate-limit-redis       |

## Project Structure

```
prisma/
  schema.prisma        # Data models and relations
  migrations/          # SQL migrations
src/
  config/              # Redis client and connection
  controllers/         # Request validation and HTTP responses
  middleware/          # authenticateToken, authorizeRole, rate limiter
  routes/              # Route definitions
  services/            # Business logic, database access and caching
  prisma.ts            # Prisma client instance
  index.ts             # App entry point and global error handler
```

The code follows a **Routes → Controllers → Services** separation of concerns.

## API Endpoints

| Method | Endpoint                   | Access        | Description                                     |
| ------ | -------------------------- | ------------- | ----------------------------------------------- |
| POST   | `/api/auth/register`       | Public        | Create a new account                            |
| POST   | `/api/auth/login`          | Public        | Log in and receive a JWT (rate limited)         |
| GET    | `/api/auth/profile`        | Authenticated | Get the logged-in user's data                   |
| GET    | `/api/auth/admindashboard` | Admin         | Admin-only test route                           |
| GET    | `/api/products`            | Public        | List products (`?page=1&limit=10&search=shirt`) |
| POST   | `/api/products`            | Admin         | Create a product                                |
| PUT    | `/api/products/:id`        | Admin         | Update a product                                |
| DELETE | `/api/products/:id`        | Admin         | Delete a product                                |
| POST   | `/api/orders`              | Authenticated | Place an order                                  |

Protected routes expect the header `Authorization: Bearer <token>`.

Status codes: `PUT`/`DELETE` return `404` if the product does not exist, `DELETE` returns `409` if the product is part of an existing order, and invalid JSON bodies return `400`.

### Example: place an order

Request:

```http
POST /api/orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "items": [{ "productId": 1, "quantity": 2 }]
}
```

Response `201 Created`:

```json
{
  "id": 1,
  "total": "200",
  "userId": 5,
  "orderItems": [{ "id": 1, "orderId": 1, "productId": 1, "quantity": 2 }]
}
```

If the stock is not enough, or a product does not exist, the API responds with `400` and the whole operation is rolled back.

## Caching with Redis

`GET /api/products` follows the **Cache-Aside** pattern:

1. Build a key from the query: `products:list:<page>:<limit>:<search>`
2. **Cache hit:** return the cached JSON immediately
3. **Cache miss:** query PostgreSQL, store the result in Redis with a 1-hour TTL, and return it

Because one product can appear in many cached pages, every write (`POST`, `PUT`, `DELETE` on products, and order checkout) deletes all keys matching `products:list:*` using `SCAN`. The database write happens first and the purge after it, so a concurrent read cannot re-cache stale data.

If Redis is unavailable, the API keeps working without a cache, and the login rate limiter lets requests through instead of failing.

## Getting Started

### Prerequisites

- Node.js
- A running PostgreSQL instance
- Docker (for Redis)

### Installation

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
npm install
```

### Start Redis

```bash
docker run -d --name redis-task6 -p 6379:6379 redis:7-alpine
```

### Environment variables

Copy `.env.example` to `.env` and fill in the values:

| Variable       | Description                                         |
| -------------- | --------------------------------------------------- |
| `DATABASE_URL` | PostgreSQL connection string                        |
| `JWT_SECRET`   | Secret used to sign JWTs (use a long random string) |
| `REDIS_URL`    | Redis connection string (`redis://localhost:6379`)  |

### Database setup

```bash
npx prisma migrate dev
npx prisma generate
```

### Run the server

```bash
npm run dev
```

The API runs on `http://localhost:3000`.

### Creating an admin

New accounts are created with the `USER` role. To promote an account, open Prisma Studio, find the user and change `role` to `ADMIN`:

```bash
npx prisma studio
```

## Scripts

| Script          | Description                    |
| --------------- | ------------------------------ |
| `npm run dev`   | Start the server in watch mode |
| `npm run build` | Compile TypeScript to `dist/`  |
| `npm start`     | Run the compiled build         |

## Design Decisions

- **Prices come from the database, never from the client.** The order total is calculated on the server with `Decimal` to avoid floating-point errors.
- **Stock is deducted with a conditional update** (`stock >= quantity`), so two customers buying the last item at the same time cannot oversell it.
- **Rate limit counters live in Redis**, not in server memory, so the limit is shared by every server instance and survives restarts.
- **Cache is purged after the database write**, never before, to avoid re-caching old data.
- **Errors are split by cause:** client mistakes return `400`, missing or invalid tokens return `401`/`403`, and unexpected failures return `500`.

## Roadmap

- Docker and Docker Compose setup for the API, PostgreSQL and Redis
- Automated tests and a CI pipeline with GitHub Actions
