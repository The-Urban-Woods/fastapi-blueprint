# Background Jobs with Celery

Decouple long-running or unreliable work from request/response cycles. Return immediately to the user while Celery workers process jobs asynchronously.

## When to Use

- Processing tasks that take longer than a few seconds
- Sending emails, notifications, or webhooks
- Generating reports or exporting data
- Processing uploads or media transformations
- Integrating with unreliable external services

## Setup

### Celery App with Dishka

Create `app/worker.py` as the Celery entrypoint. Use `DishkaTask` as the base task class so every task gets automatic dependency injection via Dishka — the same providers and scopes as FastAPI.

```python
from celery import Celery
from dishka import make_container
from dishka.integrations.celery import DishkaTask, setup_dishka

from app.providers import AppProvider, RequestProvider
from app.shared.config import get_settings

settings = get_settings()

celery_app = Celery("worker", broker=settings.CELERY_BROKER_URL, task_cls=DishkaTask)

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

Key points:
- `task_cls=DishkaTask` — sets the default base class for all tasks, enabling automatic `FromDishka` injection without needing `@inject` on each task
- `make_container` (not `make_async_container`) — Celery tasks are synchronous. Dishka handles async providers (like the session generator) internally.
- Same `AppProvider` and `RequestProvider` as FastAPI — services, repositories, and sessions are shared. Each task execution gets its own `Scope.REQUEST`, so sessions and services are fresh per task.

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

Inject services with `FromDishka` — identical to how endpoints use `FromDishka`. `DishkaTask` handles the injection automatically. Each task execution gets its own `Scope.REQUEST`, so sessions, repositories, and services are fresh per task.

```python
from dishka.integrations.celery import FromDishka

from app.worker import celery_app
from app.users.service import UserService


@celery_app.task
def send_welcome_email(user_id: int, user_service: FromDishka[UserService]) -> None:
    """Send welcome email to a newly registered user."""
    import asyncio
    asyncio.run(_send_welcome(user_id, user_service))


async def _send_welcome(user_id: int, user_service: UserService) -> None:
    user = await user_service.get(user_id)
    if not user:
        return

    # email_client.send(to=user.email, subject="Welcome!", body=f"Hi {user.name}")
```

### Async Services in Sync Tasks

Celery tasks are synchronous by default. Since our services are async (they use `AsyncSession`), wrap the async call with `asyncio.run()`:

```python
@celery_app.task
def process_report(
    report_id: str,
    report_service: FromDishka[ReportService],
) -> dict:
    return asyncio.run(_process(report_id, report_service))


async def _process(report_id: str, report_service: ReportService) -> dict:
    report = await report_service.get(report_id)
    # ... async processing ...
    return result
```

The `FromDishka` parameters are resolved by Dishka before the task body runs. The injected service is a fully wired instance with its session and dependencies already set up. You pass it into the async helper as a regular argument.

### Without DishkaTask

If `DishkaTask` is not set as the global task class, use `@inject` or `base=DishkaTask` per task:

```python
from dishka.integrations.celery import DishkaTask, FromDishka, inject

# Option A: per-task base class
@celery_app.task(base=DishkaTask)
def my_task(service: FromDishka[MyService]) -> None: ...

# Option B: inject decorator
@celery_app.task
@inject
def my_task(service: FromDishka[MyService]) -> None: ...
```

Prefer setting `task_cls=DishkaTask` globally in the Celery app to avoid decorating every task.

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
def sync_to_external_api(
    self,
    user_id: int,
    user_service: FromDishka[UserService],
) -> None:
    """Sync user data to external API with automatic retry."""
    import asyncio
    asyncio.run(_sync(user_id, user_service))


async def _sync(user_id: int, user_service: UserService) -> None:
    user = await user_service.get(user_id)
    if not user:
        return  # Don't retry — permanent failure

    # external_api.sync_user(user)
```

### Exponential Backoff

```python
@celery_app.task(bind=True, max_retries=5)
def process_payment(
    self,
    payment_id: str,
    payment_service: FromDishka[PaymentService],
) -> dict:
    """Process payment with exponential backoff."""
    import asyncio
    try:
        return asyncio.run(payment_service.charge(payment_id))
    except TransientError as e:
        raise self.retry(exc=e, countdown=2 ** self.request.retries * 60)
    except PaymentDeclinedError:
        return {"status": "declined"}  # Don't retry permanent failures
```

## Idempotency

Tasks may be retried on crash or timeout. Design for safe re-execution.

```python
@celery_app.task(bind=True)
def process_order(
    self,
    order_id: int,
    order_service: FromDishka[OrderService],
) -> None:
    """Process order idempotently."""
    import asyncio
    asyncio.run(_process_order(order_id, order_service))


async def _process_order(order_id: int, order_service: OrderService) -> None:
    order = await order_service.get(order_id)

    # Already processed — return early
    if order.status == OrderStatus.COMPLETED:
        return

    # Use idempotency key for external calls
    # payment_provider.charge(amount=order.total, idempotency_key=f"order-{order_id}")

    await order_service.update(order_id, OrderUpdate(status=OrderStatus.COMPLETED))
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
    job_service: FromDishka[BackgroundJobService],
):
    from uuid import uuid4

    task_id = str(uuid4())
    job = await job_service.create_job(task_name="generate_export", task_id=task_id)

    generate_export.delay(job.id, task_id)

    return {"job_id": job.id, "poll_url": f"/exports/{job.id}/status"}


@router.get("/exports/{job_id}/status")
async def get_export_status(
    job_id: int,
    job_service: FromDishka[BackgroundJobService],
):
    job = await job_service.get(job_id)
    if not job:
        raise HTTPException(status_code=404)
    return {
        "status": job.status,
        "result": job.result if job.status == "succeeded" else None,
        "error": job.error if job.status == "failed" else None,
    }
```

The endpoint creates a `BackgroundJob` record first (to get a pollable ID), then enqueues the Celery task with the `task_id` so the task can update the job record on completion or failure.

## Worker Shutdown

Optionally close the Dishka container on worker shutdown to release connections:

```python
from celery import current_app
from celery.signals import worker_process_shutdown
from dishka import Container


@worker_process_shutdown.connect()
def close_dishka(*args, **kwargs):
    container: Container = current_app.conf["dishka_container"]
    container.close()
```

## Best Practices

1. **Use DishkaTask globally** — set `task_cls=DishkaTask` on the Celery app, not per task
2. **Same providers as FastAPI** — reuse `AppProvider` and `RequestProvider` so services behave identically
3. **Return immediately** — don't block requests for long operations
4. **Make tasks idempotent** — safe to retry on any failure
5. **Use idempotency keys** — for external service calls
6. **Set timeouts** — both soft and hard limits on every task
7. **Don't retry permanent failures** — validation errors, invalid credentials
8. **Retry with backoff** — exponential backoff for transient errors
9. **Wrap async with asyncio.run()** — Celery tasks are sync, but injected services work with async sessions
10. **Log task transitions** — track state changes for debugging
11. **Monitor queue depth** — alert on backlog growth
