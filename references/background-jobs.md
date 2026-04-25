# Background Jobs with Celery

Decouple long-running or unreliable work from request/response cycles. Return immediately to the user while Celery workers process jobs asynchronously.

## When to Use

- Processing tasks that take longer than a few seconds
- Sending emails, notifications, or webhooks
- Generating reports or exporting data
- Processing uploads or media transformations
- Integrating with unreliable external services

## Setup

### Celery App

Create `app/worker.py` as the Celery entrypoint:

```python
from celery import Celery
from dishka import make_container
from dishka.integrations.celery import setup_dishka

from app.providers import AppProvider, RequestProvider

celery_app = Celery("worker", broker="redis://localhost:6379")

celery_app.conf.update(
    task_time_limit=3600,              # Hard limit: 1 hour
    task_soft_time_limit=3000,          # Soft limit: 50 minutes
    task_acks_late=True,                # Acknowledge after completion
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,
)

# Same providers as FastAPI — services and repos work identically
container = make_container(AppProvider(), RequestProvider())
setup_dishka(container, celery_app)
```

### Configuration

Add Celery settings to `app/shared/config.py`:

```python
class Settings(BaseSettings):
    DATABASE_URL: str = "sqlite+aiosqlite:///./app.db"
    API_V1_STR: str = "/api/v1"
    CELERY_BROKER_URL: str = "redis://localhost:6379"

    model_config = {"env_file": ".env"}
```

### Running

```bash
celery -A app.worker worker --loglevel=info
```

## Using Services in Tasks

Inject services with `FromDishka` — the same providers and scopes as FastAPI. Each task execution gets its own `Scope.REQUEST`, so sessions, repositories, and services are fresh per task.

```python
from celery import Celery
from dishka.integrations.celery import inject, FromDishka

from app.worker import celery_app
from app.users.service import UserService


@celery_app.task
@inject
def send_welcome_email(user_id: int, user_service: FromDishka[UserService]) -> None:
    """Send welcome email to a newly registered user."""
    user = user_service.get(user_id)
    if not user:
        return

    email_client.send(
        to=user.email,
        subject="Welcome!",
        body=f"Hi {user.name}, thanks for signing up.",
    )
```

Note: Celery tasks are synchronous by default. If your services are async, use `asyncio.run()`:

```python
import asyncio

@celery_app.task
@inject
def deactivate_expired_users(user_service: FromDishka[UserService]) -> None:
    """Deactivate users whose trial has expired."""
    asyncio.run(_deactivate_expired_users(user_service))

async def _deactivate_expired_users(user_service: UserService) -> None:
    users = await user_service.find_all(limit=1000)
    for user in users:
        if user.trial_expired:
            await user_service.update(user.id, UserUpdate(is_active=False))
```

## Triggering Tasks from Endpoints

Enqueue tasks from FastAPI endpoints and return immediately:

```python
from dishka.integrations.fastapi import DishkaRoute, FromDishka
from fastapi import APIRouter, status

from app.users.schemas import UserCreate, UserRead
from app.users.service import UserService
from app.users.tasks import send_welcome_email

router = APIRouter(route_class=DishkaRoute)


@router.post("/", response_model=UserRead, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_in: UserCreate,
    user_service: FromDishka[UserService],
):
    user = await user_service.create(user_in)

    # Fire-and-forget — runs in a Celery worker
    send_welcome_email.delay(user.id)

    return user
```

## Retry and Error Handling

### Automatic Retry on Transient Errors

```python
@celery_app.task(
    bind=True,
    max_retries=3,
    autoretry_for=(ConnectionError, TimeoutError),
    default_retry_delay=60,
)
@inject
def sync_to_external_api(
    self,
    user_id: int,
    user_service: FromDishka[UserService],
) -> None:
    """Sync user data to external API with automatic retry."""
    user = user_service.get(user_id)
    if not user:
        return  # Don't retry — permanent failure

    external_api.sync_user(user)
```

### Exponential Backoff

```python
@celery_app.task(bind=True, max_retries=5)
@inject
def process_payment(
    self,
    payment_id: str,
    payment_service: FromDishka[PaymentService],
) -> dict:
    """Process payment with exponential backoff."""
    try:
        return payment_service.charge(payment_id)
    except TransientError as e:
        raise self.retry(exc=e, countdown=2 ** self.request.retries * 60)
    except PaymentDeclinedError:
        return {"status": "declined"}  # Don't retry permanent failures
```

## Idempotency

Tasks may be retried on crash or timeout. Design for safe re-execution.

```python
@celery_app.task(bind=True)
@inject
def process_order(
    self,
    order_id: int,
    order_service: FromDishka[OrderService],
) -> None:
    """Process order idempotently."""
    order = order_service.get(order_id)

    # Already processed — return early
    if order.status == OrderStatus.COMPLETED:
        return

    # Use idempotency key for external calls
    payment_provider.charge(
        amount=order.total,
        idempotency_key=f"order-{order_id}",
    )

    order_service.update(order_id, OrderUpdate(status=OrderStatus.COMPLETED))
```

**Strategies:**
- **Check-before-write** — verify state before action
- **Idempotency keys** — use unique tokens with external services
- **Upsert patterns** — `INSERT ... ON CONFLICT UPDATE`

## Task Chaining and Workflows

Compose complex workflows from simple tasks:

```python
from celery import chain, group, chord

# Sequential: extract → transform → load
workflow = chain(
    extract_data.s(source_id),
    transform_data.s(),
    load_data.s(destination_id),
)

# Parallel: send all notifications at once
parallel = group(
    send_email.s(user_email),
    send_sms.s(user_phone),
    update_analytics.s(event_data),
)

# Chord: parallel tasks, then a callback when all complete
workflow = chord(
    [process_item.s(item_id) for item_id in item_ids],
    send_completion_notification.s(batch_id),
)

workflow.apply_async()
```

## Job Status Polling

For long-running tasks, return a job ID and provide a polling endpoint:

```python
@router.post("/exports", status_code=status.HTTP_202_ACCEPTED)
async def start_export(
    request: ExportRequest,
    export_service: FromDishka[ExportService],
):
    job = await export_service.create(request)
    generate_export.delay(job.id)
    return {"job_id": job.id, "poll_url": f"/exports/{job.id}/status"}


@router.get("/exports/{job_id}/status")
async def get_export_status(
    job_id: int,
    export_service: FromDishka[ExportService],
):
    job = await export_service.get(job_id)
    if not job:
        raise HTTPException(status_code=404)
    return {
        "status": job.status,
        "result": job.result if job.status == "succeeded" else None,
        "error": job.error if job.status == "failed" else None,
    }
```

## Best Practices

1. **Return immediately** — don't block requests for long operations
2. **Make tasks idempotent** — safe to retry on any failure
3. **Use idempotency keys** — for external service calls
4. **Set timeouts** — both soft and hard limits on every task
5. **Don't retry permanent failures** — validation errors, invalid credentials
6. **Retry with backoff** — exponential backoff for transient errors
7. **Log task transitions** — track state changes for debugging
8. **Monitor queue depth** — alert on backlog growth
