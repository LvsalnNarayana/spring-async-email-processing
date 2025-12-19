
# Spring-Async-Email-Processing

## Overview

This project is a **multi-microservice demonstration** of **asynchronous email processing** in Spring Boot 3.x using **@Async**, **TaskExecutor**, and **queue-based decoupling**. It showcases how to offload time-consuming email sending operations from the main user request thread to prevent blocking and improve responsiveness.

The design emphasizes reliability with retry mechanisms, dead-letter handling, and monitoring — essential for production email systems.

## Real-World Scenario (Simulated)

In email marketing platforms like **Mailchimp** or **SendGrid**:
- Users trigger bulk email campaigns (thousands to millions of recipients).
- Sending emails involves SMTP calls that can take seconds per batch.
- Blocking the user request thread would make the UI unresponsive.
- Failed sends must be retried without losing emails.

We simulate a campaign management system where creating/sending a campaign is instant for the user, while actual email delivery happens asynchronously in background workers with retries and persistence.

## Microservices Involved

| Service                   | Responsibility                                                                 | Port  |
|---------------------------|--------------------------------------------------------------------------------|-------|
| **eureka-server**         | Service discovery (Netflix Eureka)                                             | 8761  |
| **campaign-service**      | User-facing: create campaign, enqueue emails for async processing             | 8081  |
| **email-worker-service**  | Background workers: consume queue, send emails via JavaMailSender             | 8082  |
| **notification-service**  | Sends user notifications on campaign completion/failure                       | 8083  |

Email jobs are persisted in a database queue for durability.

## Tech Stack

- Spring Boot 3.x
- @Async + TaskExecutor (thread pool configuration)
- Spring Data JPA + PostgreSQL (persistent email job queue)
- JavaMailSender (SMTP integration - Gmail/SendGrid mockable)
- Spring Retry (for failed sends)
- Spring Cloud Netflix Eureka
- Micrometer + Actuator (async metrics)
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: emaildb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  campaign-service:
    build: ./campaign-service
    depends_on:
      - postgres
      - eureka-server
    ports:
      - "8081:8081"

  email-worker-service:
    build: ./email-worker-service
    depends_on:
      - postgres
      - eureka-server
    ports:
      - "8082:8082"

  notification-service:
    build: ./notification-service
    depends_on:
      - eureka-server
    ports:
      - "8083:8083"
```

Run with: `docker-compose up --build`

## Async Processing Strategy

| Component                  | Implementation Details                                                  |
|----------------------------|-------------------------------------------------------------------------|
| **Job Queue**              | `email_jobs` table with status (PENDING, PROCESSING, SENT, FAILED)     |
| **Enqueue**                | Campaign creation → insert jobs → return immediate response            |
| **Worker**                 | @Scheduled task polls for PENDING jobs → @Async processing              |
| **Thread Pool**            | Custom TaskExecutor with corePoolSize, maxPoolSize, queueCapacity      |
| **Retry**                  | @Retryable on send method (exponential backoff, max attempts)          |
| **Dead Letter**            | After max retries → status FAILED → notify admin                       |
| **Idempotency**            | Job ID prevents duplicate processing                                   |

## Key Features

- Immediate response to user on campaign creation
- Configurable thread pool for concurrent sending
- Persistent queue survives restarts
- Automatic retry with exponential backoff
- Metrics: queued, processed, failed emails
- Dead-letter handling and alerting
- Progress tracking (percentage complete)
- Template-based emails (Thymeleaf optional)

## Expected Endpoints

### Campaign Service (`http://localhost:8081`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| POST   | `/api/campaigns`                | Create campaign → enqueue emails (immediate response) |
| GET    | `/api/campaigns/{id}`           | Get campaign status + progress                   |
| GET    | `/api/campaigns/{id}/jobs`      | List email jobs with status                      |

### Email Worker Service (`http://localhost:8082`)

| Method | Endpoint                  | Description                                      |
|--------|---------------------------|--------------------------------------------------|
| GET    | `/actuator/metrics`       | View async pool metrics (active threads, queue size) |

### Notification Service (`http://localhost:8083`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/notifications`            | Recent campaign completion/failure alerts        |

## Architecture Overview

```
User / Admin
   ↓
Campaign Service → immediate response
   ↓
Persist Email Jobs (PostgreSQL queue)
   ↓ (@Scheduled poller)
Email Worker Service → @Async send
   ↓ (on success/failure)
Update job status → notify via Notification Service
```

**Async Flow**:
1. User creates campaign → jobs inserted
2. Response returned instantly
3. Worker polls → picks jobs → sends @Async
4. On exception → retry → final failure → DLQ handling
5. Progress updated → user can poll status

## How to Run

1. Clone repository
2. Configure SMTP in `application.yml` (or use mock)
3. Start Docker: `docker-compose up --build`
4. Create campaign: POST `/api/campaigns` (with recipient list)
5. Response immediate → check status endpoint
6. Watch worker logs → emails processed in background
7. Simulate failures → observe retries

## Testing Asynchrony

1. Create campaign with 1000 emails → response < 1s
2. Worker processes in background → UI remains responsive
3. Check thread pool metrics → active threads increase
4. Force SMTP failure → retries triggered
5. After max retries → job marked FAILED → notification sent

## Skills Demonstrated

- Proper use of @Async with custom TaskExecutor
- Persistent queue for durable async processing
- Retry and dead-letter patterns
- Thread pool tuning and monitoring
- Non-blocking user experience design
- Background job processing architecture
- Production-ready email sending concerns

## Future Extensions

- RabbitMQ/SQS for distributed queue
- Rate limiting per SMTP provider
- Email templates with Thymeleaf
- WebSocket for real-time progress
- Distributed workers (multiple instances)
- Integration with SendGrid/Mailgun API

