
related: [[Celery + RabbitMQ + Redis + Django + Integration]]


```mermaid
flowchart LR
    subgraph Django_App["Django Application"]
        direction TB
        ORM["Django ORM (Models)"] -->|SQL Queries| Postgres
        Views["Django Views"] -->|Send Task| RabbitMQ
        Views -->|Cache/Session| Redis
    end

    subgraph Broker["Message Broker"]
        RabbitMQ["RabbitMQ"]
    end

    subgraph Worker["Celery Worker"]
        Task["Celery Worker Process"] -->|Store Results| Redis
    end

    Postgres[("PostgreSQL DB")]
    Redis[("Redis Cache/Backend")]

    Django_App --> RabbitMQ
    RabbitMQ --> Task
    Task --> Postgres

```


> [! ✅ With this setup:]
>
> - **Postgres** = persistent data
> - **RabbitMQ** = broker for async tasks
> - **Redis** = cache + result backend
> - **Celery Worker** = async task execution
> - **Django** = main app (views, APIs, task producer)
> 

## 📌 Components

#### 1. **Django**

- Handles web requests (views, APIs).
- Uses **Postgres** for relational data.
- Sends async tasks to **RabbitMQ** (via Celery).
- Uses **Redis** for caching and sessions.

Config (`settings.py`)
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "myuser",
        "PASSWORD": "mypassword",
        "HOST": "db",
        "PORT": "5432",
    }
}

CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://redis:6379/1",
        "OPTIONS": {"CLIENT_CLASS": "django_redis.client.DefaultClient"},
    }
}

```


#### 2. Postgresql

- Main relational database.
- Stores models, transactions, persistent data.

```yaml
db:
  image: postgres:17
  environment:
    POSTGRES_USER: myuser
    POSTGRES_PASSWORD: mypassword
    POSTGRES_DB: mydb
  ports:
    - "5432:5432"
  volumes:
    - pgdata:/var/lib/postgresql/data

```

#### 3. **Redis**

- Used as **cache** + **Celery result backend**.
- Stores short-lived data and task results.

**Docker Compose snippet**

```yaml
redis:
  image: redis:7
  ports:
    - "6379:6379"

```

#### 4. RabbitMQ

- Message broker for Celery.
- Queues tasks sent by Django and delivers them to Celery workers.

**Docker Compose snippet**

```yaml
rabbitmq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"
    - "15672:15672" # Web UI (guest/guest)

```

#### 5. **Celery Worker**

- Executes tasks from the RabbitMQ queue.
- Stores results in Redis.
- Can also update Postgres if tasks involve DB writes.

**Celery Config (`celery.py`)**
```python
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

app = Celery("myproject")

app.conf.broker_url = "amqp://guest:guest@rabbitmq:5672//"
app.conf.result_backend = "redis://redis:6379/0"

app.autodiscover_tasks()

```

**Example Task(`tasks.py`)**
```python
from celery import shared_task
import time

@shared_task
def add(x, y):
    time.sleep(5)
    return x + y

```


**Worker Service (docker-compose)**
```yaml
worker:
  build: .
  command: celery -A myproject worker -l info
  depends_on:
    - db
    - redis
    - rabbitmq

```

