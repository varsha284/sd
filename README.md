# TicketLane

TicketLane is a full-stack event ticket booking and system-design demonstration for a local college practical. It uses a React + Vite frontend and an Express API gateway backed by PostgreSQL. The demo includes event discovery, seat holds, mock payments, QR entry passes, and an interactive system dashboard.

> **Demo boundaries:** PostgreSQL is required and stores the application data. Redis and RabbitMQ are optional local services; the app reports their actual connection state and explains its fallback when they are absent. Payments never contact a payment processor or collect card details. Confirmation messages are simulated records, not sent emails or SMS.

## Architecture

```text
React browser (Vite :5173 in development)
        │ /api (Vite proxy)
        ▼
Express API gateway (:3000)
  ├── PostgreSQL (:5432) — accounts, events, seats, bookings, payments, tickets
  ├── Redis (:6379, optional) — cache-aside event details
  └── RabbitMQ (:5672, optional)
        └── booking worker — ticket QR data + simulated confirmation record
```

The Express API is stateless between requests; signed JWTs carry the authenticated user ID. PostgreSQL is the source of truth. Reservation transactions lock the requested seat rows, verify all are available, create an expiring hold, and create a pending booking atomically. This ensures concurrent requests cannot both reserve the same seat. A hold lasts five minutes by default and is safely reclaimed when availability is read or another reservation is attempted.

The mock payment endpoint accepts a success or failure outcome. A unique idempotency key makes payment retries safe. A confirmed booking creates a single ticket; the QR code contains the ticket identifier only. Cancelling a confirmed demo booking marks its mock payment refunded and releases its seats; no real payment/refund is performed. The optional RabbitMQ consumer is idempotent and retries failures up to three times.

## Requirements

- Node.js 20 or newer and npm
- Docker Desktop with Docker Compose (for the required PostgreSQL database; Redis and RabbitMQ are optional)
- VS Code (recommended)

## Run locally on Windows

Use a separate VS Code terminal for each long-running command.

1. Open this folder in VS Code and install the root tooling and both apps:

   ```powershell
   npm install
   npm run install:all
   ```

2. Create the local API environment file and set a private development JWT secret:

   ```powershell
   Copy-Item backend\.env.example backend\.env
   notepad backend\.env
   ```

   Change `JWT_SECRET` to a long, random value (at least 24 characters). The provided database URL matches the local Compose configuration. Never commit `backend\.env`.

3. Start PostgreSQL and the optional demonstration integrations:

   ```powershell
   docker compose up -d postgres redis rabbitmq
   docker compose ps
   ```

   PostgreSQL is required. If Redis or RabbitMQ is not needed, start only `postgres`; the web app identifies the unavailable integrations and uses the described fallback.

4. Create the database tables and add sample events and seats:

   ```powershell
   npm run db:setup
   ```

5. Start the Express API gateway (port **3000**) and React development server (port **5173**):

   ```powershell
   npm run dev
   ```

   Open **http://localhost:5173/**. Vite forwards `/api` requests to the real Express gateway at port 3000. For a single production-style origin at **http://localhost:3000/**, stop the dev servers, build the frontend, and run:

   ```powershell
   npm run build
   $env:NODE_ENV = "production"
   npm start
   ```

   Then open **http://localhost:3000/**. Express serves the compiled frontend and the REST API on the same port.

6. To run the optional RabbitMQ consumer, start it in another terminal after the API and RabbitMQ are up:

   ```powershell
   npm run worker --prefix backend
   ```

   Check the System Design page for live connection and worker-heartbeat status. Without the worker, the app creates the ticket synchronously and marks the message handling as simulated. RabbitMQ does not send real notifications.

## Try the application

1. On **Discover**, search/filter by event name, category, city, or date. The event list is loaded from PostgreSQL.
2. Open an event, refresh seat availability, and select up to eight available seats. A reservation requires an account; create one using **Sign in → Create account**.
3. Choose **Continue to checkout**. Select either **Simulate successful payment** or **Simulate declined payment**. No card fields or real charge are involved.
4. A successful mock payment confirms the booking and produces an entry QR code. The booking appears under **My bookings**; canceling releases the seats and records a simulated refund if it had been confirmed.
5. In **System design**, inspect the live health checks, relational entity overview, API contract, Redis keys/hit metrics, RabbitMQ queue/worker metrics, and PostgreSQL-backed booking chart.
6. Sign in and choose **Run concurrency test** to send six simultaneous reservation requests for one real seat. One should win and the others should be rejected with a conflict. The winning hold is real and expires in five minutes; cancel its pending booking under **My bookings** to release it sooner.

## Tests

The cache unit test runs without external services:

```powershell
npm test
```

Seat-lock concurrency, payment idempotency/retry, and cancellation/refund tests use an isolated PostgreSQL database. Create a separate empty database (do not point tests at data you care about), initialize it, then run:

```powershell
$env:DATABASE_URL = "postgres://ticketlane:ticketlane_dev@localhost:5432/ticketlane_test"
npm run db:setup --prefix backend
$env:TEST_DATABASE_URL = $env:DATABASE_URL
npm test
Remove-Item Env:DATABASE_URL
Remove-Item Env:TEST_DATABASE_URL
```

Those integration tests create and remove only their uniquely named fixture rows. They are skipped when `TEST_DATABASE_URL` is unset. Do not use a production or shared database for tests.

Frontend production compilation:

```powershell
npm run build
```

## Troubleshooting

- **API says database unavailable:** confirm `docker compose ps`, check `DATABASE_URL` in `backend\.env`, then rerun `npm run db:setup`.
- **No events appear:** ensure database setup completed successfully; seed data is inserted idempotently.
- **Redis or RabbitMQ says unavailable:** this is an honest optional-service status. Check the Compose service and ports `6379` / `5672`; booking and payment demonstrations continue with explicit fallbacks.
- **No async tickets:** start the worker command in its own terminal. The UI will display consumer heartbeat and queue metrics.
- **Port already in use:** change `PORT` in `backend\.env` or the Vite `server.port` in `frontend\vite.config.js`; update the Vite proxy target if you change the API port.

See [backend/API.md](./backend/API.md) for REST request and response details and [docker-compose.yml](./docker-compose.yml) for local service configuration.
