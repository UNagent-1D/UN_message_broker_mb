# AGENTS.md — UN_message_broker_mb

Infrastructure-only component. No business logic lives here.

---

## Role

`UN_message_broker_mb` is the RabbitMQ message broker that decouples
`UN_agent_runtime_ms` (producer) from `UN_conversation_chat_ms` (consumer).
It stores pending chat jobs and delivers results, enabling asynchronous LLM
processing without blocking the Telegram ingress path.

---

## Queues

| Queue | Direction | Producer | Consumer |
|---|---|---|---|
| `chat_requests` | job submission | `agent-runtime` | `conversation-chat` |
| `chat_results` | job results | `conversation-chat` | `agent-runtime` |

Both queues are **durable** — they survive broker restarts. They are
pre-declared via `definitions.json` at container startup so services can
connect in any order.

---

## Message shapes

### `chat_requests` (published by agent-runtime)
```json
{
  "job_id":    "b3d1a4c2-...",
  "chat_id":   123456789,
  "tenant_id": "demo-tenant",
  "session_id": "sess_abc...",
  "message":   "user text"
}
```

### `chat_results` (published by conversation-chat)
```json
{
  "job_id":    "b3d1a4c2-...",
  "chat_id":   123456789,
  "session_id": "sess_abc...",
  "text":      "LLM reply text"
}
```

---

## Connection

| Setting | Value |
|---|---|
| AMQP URL | `amqp://rabbitmq:5672` (inside compose network) |
| Management UI | `http://localhost:15672` (guest / guest) |

---

## Running standalone

```bash
cd UN_message_broker_mb
docker compose up -d
# Management UI available at http://localhost:15672
```

## Running as part of the full stack

The umbrella `dev-runner/docker-compose.yml` builds this image via
`build: context: ./UN_message_broker_mb` and attaches it to the
`archsoft` network. All other services that need the broker declare
`depends_on: rabbitmq: condition: service_healthy`.

---

## Files

| File | Purpose |
|---|---|
| `Dockerfile` | Extends `rabbitmq:3-management`, copies config |
| `rabbitmq.conf` | Instructs RabbitMQ to load `definitions.json` at boot |
| `definitions.json` | Pre-declares both queues + default user/permissions |
| `docker-compose.yml` | Standalone or reference compose definition |
| `.env.example` | Example env vars consumed by producer/consumer services |
