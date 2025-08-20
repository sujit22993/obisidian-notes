
Read more : [[Additional Notes - Visualizing Healthchecks (Compose vs Kubernetes)]]
## 📌 Introduction
In Docker Compose, services often rely on each other.  
For example:  
- A **Python backend** depends on a **Postgres database**.  
- If the backend starts before the DB is ready, it may fail to connect.  

To manage this:  
- **`depends_on`** → Controls container startup order.  
- **`healthcheck`** → Ensures a service is actually *ready*, not just “running.”

---

## ⚠️ The Problem
By default:  
- `depends_on` only guarantees **container startup order**.  
- It does **not** wait for readiness (e.g., DB accepting connections).  

👉 Example: Backend may try connecting to DB while it’s still booting.

---

## ✅ The Solution
Use **`healthcheck`** with **`depends_on: condition: service_healthy`**.  
This ensures dependent services only start once their dependencies are truly healthy.

---

## 📝 Example: Python Backend + PostgreSQL


```yaml
version: "3.8"

services:
  db:
    image: postgres:14
    container_name: postgres_db
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend
    container_name: python_backend
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8000:8000"
```

## 💡 Best Practices

- Always add **healthchecks** for critical services (DBs, queues, etc.).
    
- Keep healthchecks **lightweight** (avoid heavy queries).
    

> [!NOTE]
> - Use **app-level retry logic** in case services fail at runtime.

    
- ✅ Correct usage prevents **race conditions** and makes services start in the right order.
- Prefer `condition: service_healthy` over plain `depends_on`.

---

## 🚫 Common Pitfalls

- Using `depends_on` without healthchecks → backend may still fail.
    
- Writing a healthcheck that always succeeds → false readiness.
    
- Forgetting retries in application code → brittle systems.