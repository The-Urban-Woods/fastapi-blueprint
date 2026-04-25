# Application Entrypoint

## `app/main.py`

The FastAPI application factory. Responsibilities: create the app, set up DI, register middleware, include routers.

```python
from contextlib import asynccontextmanager

from dishka import make_async_container
from dishka.integrations.fastapi import setup_dishka
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.ext.asyncio import AsyncEngine

from app.providers import AppProvider, RequestProvider
from app.shared.config import get_settings
from app.shared.database import Base
from app.users.api import router as users_router

settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = await app.state.dishka_container.get(AsyncEngine)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    await app.state.dishka_container.close()


app = FastAPI(title="My API", version="0.1.0", lifespan=lifespan)

container = make_async_container(AppProvider(), RequestProvider())
setup_dishka(container, app)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(users_router, prefix=f"{settings.API_V1_STR}/users", tags=["users"])
```

## `main.py` (project root)

The uvicorn runner for local development:

```python
import uvicorn

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, reload=True)
```

## Lifespan Events

Use the `lifespan` async context manager for startup/shutdown logic:

- **Startup** (before `yield`): create database tables, warm caches, connect to external services
- **Shutdown** (after `yield`): close the DI container (which disposes the engine and any other managed resources)

Do NOT use the deprecated `@app.on_event("startup")` / `@app.on_event("shutdown")` handlers.
