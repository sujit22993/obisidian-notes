
tags: #Celery #Async #Django #Redis #RabbitMQ #apply_async 

related: [[Redis - Django + Celery + RabbitMQ + Postgres Setup]]


```mermaid
flowchart LR
    A[Django View: start_task] -->|send task| B[RabbitMQ Broker]
    B -->|deliver task| C[Celery Worker]
    C -->|process task| D[Redis Result Backend]
    D -->|store result| D
    E[Django View: get_result] -->|fetch result| D

```


## 🔧 Architecture

1. **Django View** → sends a task to **Celery** via **RabbitMQ** (message broker).
2. **Celery Worker** → fetches the task from **RabbitMQ**, processes it, and stores result in **Redis** (result backend).
3. **Django View (later)** → queries **Redis** for the result.

---

## ⚙️ Setup

##### 1. `celery.py` (Django project root, e.g., `myproject/celery.py`)

```python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

app = Celery("myproject")

# Configure broker (RabbitMQ) and result backend (Redis)
app.conf.broker_url = "amqp://guest:guest@localhost:5672//"  # RabbitMQ
app.conf.result_backend = "redis://localhost:6379/0"         # Redis

# Auto-discover tasks in all apps
app.autodiscover_tasks()

```

#### 2. `tasks.py` (inside a Django app, e.g., `app/tasks.py`)

```python
from celery import shared_task
import time

@shared_task
def add(x, y):
    time.sleep(5)  # simulate long running job
    return x + y

```

#### 3. Django `views.py`

```python
from django.http import JsonResponse
from app.tasks import add
from celery.result import AsyncResult
from myproject.celery import app

# Start a task
def start_task(request):
    task = add.delay(10, 20)  # Send to RabbitMQ
    return JsonResponse({"task_id": task.id})

# Get task result later
def get_result(request, task_id):
    result = AsyncResult(task_id, app=app)
    if result.ready():
        return JsonResponse({"task_id": task_id, "status": result.status, "result": result.result})
    else:
        return JsonResponse({"task_id": task_id, "status": result.status})

```


> [! ✅ Flow in Action]
> 
>
>
> 1. **Client calls** `http://127.0.0.1:8000/start_task/`
>     - Django sends task to RabbitMQ.
>     - Response: `{"task_id": "f3e9b9b2-..."}`
>         
> 2. **Celery worker fetches from RabbitMQ**
>     - Executes `add(10, 20)`.
>     - Stores result `30` into Redis with key = `task_id`.
>     
> 3. **Client polls** `http://127.0.0.1:8000/get_result/f3e9b9b2-.../`
>     - Django fetches result from Redis via `AsyncResult`.
>     - Returns `{"task_id": "...", "status": "SUCCESS", "result": 30}`.


