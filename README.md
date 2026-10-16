# Webhook Delivery Engine

A lightweight, reliable webhook delivery system that never drops an event. Built with Node.js, Express, and SQLite — zero queue dependencies, just dependable delivery with full audit logs.

---

## What It Does
When a backend service triggers a webhook, this engine receives and delivers it to the target endpoint. If the destination server is down or returns errors, it automatically retries with exponential backoff (10s, 30s, 2m, 10m). If the endpoint fails after 5 attempts, the event moves to a Dead Letter Queue (DLQ) where it can be inspected and manually replayed. Every delivery attempt is timestamped and logged.

---

## Features
- **Reliable At-Least-Once Delivery**: Guaranteed delivery tracking across backend services.
- **Exponential Backoff**: Automatic retries to prevent overwhelming destination servers during outages.
- **Dead Letter Queue (DLQ)**: Isolates permanently failed events for inspection.
- **Manual Event Replay**: Re-trigger failed deliveries once the destination endpoint is fixed.
- **SQLite Persistence**: File-based durability with zero external message broker dependencies.
- **Crash Recovery**: Automatically reclaims pending tasks upon server restart.
- **Full Audit Logging**: Records HTTP status codes, latency, and response bodies for every attempt.

---

## Architecture at a Glance

```text
POST /webhooks/send
        │
        ▼
     PENDING
        │
        ▼
  Retry Scheduler (polls SQLite every 2 seconds)
        │
        ▼
   HTTP Dispatcher
        │
 ┌──────┼──────────┐
 │      │          │
 ▼      ▼          ▼
2xx    5xx      Timeout
 │      │          │
 ▼      ▼          ▼
DELIVERED    RETRYING
                │
         After 5 Attempts
                │
                ▼
              DEAD (DLQ)
                │
                ▼
        POST /webhooks/:id/replay
```

---

## Project Structure

```bash
src/
  app.js                            Express setup and route configuration
  server.js                         Database boot, worker initialization, graceful shutdown
  config/database.js                SQLite schema and connection setup
  controllers/webhookController.js  Input validation and API responses
  routes/webhookRoutes.js           REST endpoint mapping
  storage/webhookRepository.js      Database queries and status updates
  services/backoffService.js        Exponential backoff calculation
  workers/retryScheduler.js         Background polling worker
  workers/webhookDispatcher.js      HTTP request execution and state transitions
```

---

## API Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/webhooks/send` | Queue a new webhook delivery |
| `GET` | `/webhooks/:id` | Get delivery status and attempt history |
| `GET` | `/webhooks/dead` | List all Dead Letter Queue (failed) events |
| `POST` | `/webhooks/:id/replay` | Manually re-trigger a dead event |
| `GET` | `/health` | Check engine health and worker status |

---

## Key Design Decisions

- **Why SQLite instead of Redis/RabbitMQ?**  
  SQLite is self-contained, requires zero external services, and makes state persistence crash-safe directly on disk without complex infrastructure setup.
- **Concurrency & Deduplication:**  
  Background workers claim jobs using atomic SQL updates, preventing multiple workers from dispatching the same event simultaneously.
- **At-Least-Once Delivery:**  
  Uses an `x-delivery-id` header so destination receivers can detect and handle duplicate deliveries idempotently.

---

## Getting Started

```bash
# Clone the repository
git clone https://github.com/enoshkumar17/Webhook-Delivery-Engine.git
cd Webhook-Delivery-Engine

# Install dependencies
npm install

# Run the development server
npm run dev
```

The server starts on `http://localhost:3000`. You can test requests using Postman or `test.http`.