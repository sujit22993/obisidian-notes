
#multithreading #multiprocessing #asyncio #async #await #event_loop


## 📝 **Concurrency & Parallelism in Python – Quick Revision Notes**

---

### **Core Concepts**

1. **Multitasking** → Running multiple tasks “at the same time” from a user’s perspective.
    
2. **Concurrency** → Structuring tasks so they can make progress together (may be parallel, may not).
    
3. **Parallelism** → Actually running tasks at the same time on different CPU cores.
    
4. **GIL (Global Interpreter Lock)** → In CPython, only one thread can execute Python bytecode at once.
    

---

### **Execution Models**

|Model|Use Case|Parallel?|Notes|
|---|---|---|---|
|**Threads**|I/O-bound (blocking libs)|❌ (in Python)|OS threads, GIL prevents CPU parallelism|
|**Processes**|CPU-bound|✅|Separate memory space, no GIL interference|
|**Asyncio**|I/O-bound (async libs)|❌|Single-threaded, cooperative multitasking|

---

### **Scheduling Types**

- **Preemptive** → OS decides when to switch tasks (threads, processes).
    
- **Cooperative** → Tasks yield control voluntarily (`await` in asyncio).
    

---

### **Context Switching**

- **Threads/Processes** → OS saves/restores CPU registers, memory maps (expensive).
    
- **Asyncio** → Just saves/restores Python stack frames & variables (cheap).
    

---

### **Choosing the Right Tool**

1. **Is task CPU-bound?**
    
    - ✅ → Use **multiprocessing**
        
    - ❌ → Continue
        
2. **Is it I/O-bound with async-friendly libs?**
    
    - ✅ → Use **asyncio**
        
    - ❌ → Use **threads**
        

---

### **Bridging Blocking and Async**

- `loop.run_in_executor()` → Runs blocking code in a thread or process without freezing the async loop.
    
- Good for combining async I/O with existing synchronous libraries.
    

---

### **Best Practices (Modern)**

- Prefer **asyncio** for high-concurrency I/O workloads.
    
- Use **processes** for CPU-bound workloads.
    
- Only use **threads** when stuck with blocking I/O libraries.
    
- Avoid mixing models unless necessary — complexity increases quickly.
    

---

### **Quick Analogies**

- **Threads** → Multiple people sharing one kitchen, waiting turns for the stove.
    
- **Processes** → Each person has their own kitchen.
    
- **Asyncio** → One cook juggling many dishes, only working on one at a time while others are waiting.
    

---
## When to Choose Threads, Processes, or Asyncio in Real-World Systems**

Think of this step as “the decision-making framework” — how modern Python developers choose the right concurrency tool.

---

### **1️⃣ Threads**

- **Best for:** I/O-bound work with blocking libraries (e.g., `requests`, file reads, DB drivers without async support).
    
- **Why:** While GIL prevents true parallel CPU work, threads can run I/O-bound code concurrently by releasing the GIL during I/O waits.
    
- **Pros:**
    
    - Easy to add to existing code
        
    - Works with synchronous libraries
        
- **Cons:**
    
    - Overhead from OS threads
        
    - Shared memory risks (race conditions)
        
    - Still limited to 1 CPU core for Python bytecode execution (due to GIL)
        

---

### **2️⃣ Processes**

- **Best for:** CPU-bound work that needs multiple cores (e.g., image processing, machine learning preprocessing, simulations).
    
- **Why:** Each process has its own Python interpreter & GIL, so they run truly in parallel.
    
- **Pros:**
    
    - Bypasses GIL
        
    - Scales with CPU cores
        
- **Cons:**
    
    - Higher startup cost
        
    - Data must be serialized between processes (pickle overhead)
        
    - More memory usage
        

---

### **3️⃣ Asyncio**

- **Best for:** High-concurrency I/O-bound tasks with async-friendly libraries (e.g., web scraping with `aiohttp`, async DB with `asyncpg`).
    
- **Why:** Single-threaded, cooperative multitasking using an event loop — avoids thread/process overhead for massive numbers of concurrent I/O tasks.
    
- **Pros:**
    
    - Extremely lightweight concurrency
        
    - Scales to thousands of tasks
        
    - Deterministic (no race conditions unless you use shared state badly)
        
- **Cons:**
    
    - Blocking calls freeze the loop unless offloaded (`run_in_executor`)
        
    - Requires async-compatible libraries
        
    - Steeper learning curve
        

---

### **4️⃣ Modern Best Practice Flowchart**
#### Modern Best Practice Flowchart

```
	Is it CPU-bound?
	  ├── Yes → multiprocessing
	  └── No → Is it I/O-bound?
	        ├── Yes, async-friendly library available → asyncio
	        ├── Yes, blocking library only → threads
	        └── No, minimal concurrency needs → keep synchronous


```


### Example Code : Mix of all

[[Code - Asyncio+Threads+Process]]
