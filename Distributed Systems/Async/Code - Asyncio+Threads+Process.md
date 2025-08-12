
```
	import asyncio
	from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
	import time

	# CPU-bound function
	def heavy_math(x):
		total = 0
		for _ in range(10_000_000):
			total += x
		
		return total
	
	# Simulate blocking I/O
	def blocking_io():
		time.sleep(2)
		return "I/O done"

	async def main():
	
		loop = asyncio.get_running_loop()
		
		# Use ThreadPoolExecutor for I/O-bound blocking
		with ThreadPoolExecutor() as pool:
			io_future = loop.run_in_executor(pool, blocking_io)
	
		# Use ProcessPoolExecutor for CPU-bound work
		with ProcessPoolExecutor() as pool:
			cpu_future = loop.run_in_executor(pool, heavy_math, 5)
	
		result_io = await io_future
		result_cpu = await cpu_future

		print(result_io)		
		print(f"CPU result: {result_cpu}")

	if __name__ == "__main__": # <-- IMPORTANT on Windows!
		asyncio.run(main())
```


## **Step-by-step Breakdown**

### **1. The Problem**

- **`asyncio`** works best for **non-blocking** I/O (like network calls).
    
- But sometimes:
    
    - You have **blocking I/O** (e.g., `time.sleep`, file reads/writes) → it stalls the event loop.
        
    - You have **CPU-bound work** (heavy computation) → it hogs the GIL and blocks _everything_.
        

If either happens inside an `async` function **without offloading it**, the entire async app freezes.

---

### **2. How This Code Solves It**

- The event loop delegates blocking work to **executors** (thread/process pools), freeing the loop to keep running other coroutines.
    
- Two executors are used:
    
    - **ThreadPoolExecutor** → good for _I/O-bound blocking_ work  
        (Threads can wait on I/O without blocking the main async loop.)
        
    - **ProcessPoolExecutor** → good for _CPU-bound_ work  
        (Bypasses the GIL because each process has its own interpreter and memory space.)
        

---

### **3. Core Concepts in Play**

#### **a) The Event Loop**

- `asyncio.get_running_loop()` → gets the currently running loop.
    
- The loop is the **traffic controller**:
    
    - Knows about all pending coroutines.
        
    - Decides who runs next.
        
    - Delegates blocking functions to background workers via `run_in_executor`.
        

---

#### **b) `run_in_executor`**

python

CopyEdit

`loop.run_in_executor(executor, func, *args)`

- Submits `func(*args)` to an executor (thread/process).
    
- Returns an **awaitable Future** (integrates seamlessly with `await`).
    
- Key benefit → Offloads blocking calls so the main loop stays free.
    

---

#### **c) Executors**

- **ThreadPoolExecutor**:
    
    - Shares memory with the main process.
        
    - Still subject to GIL, so not ideal for CPU-heavy loops.
        
    - But perfect for blocking I/O (because the GIL is released during I/O waits).
        
- **ProcessPoolExecutor**:
    
    - Each task runs in its own process → avoids GIL.
        
    - Better for CPU-bound work (like your `heavy_math`).
        
    - Has overhead from process startup and inter-process communication.
        

---

#### **d) CPU-bound vs I/O-bound**

- `blocking_io()` → simulates slow I/O with `time.sleep(2)` (blocking call).  
    Placed in **ThreadPoolExecutor** because threads can wait without GIL contention.
    
- `heavy_math()` → CPU-intensive loop over 10 million iterations.  
    Placed in **ProcessPoolExecutor** because CPU-bound code in threads would still block due to GIL.
    

---

### **4. Execution Flow**

1. **Event loop starts** via `asyncio.run(main())`.
    
2. Inside `main()`:
    
    - Gets running loop.
        
    - Submits `blocking_io()` to a **thread pool**.
        
    - Submits `heavy_math(5)` to a **process pool**.
        
3. While those run in parallel workers:
    
    - The main loop is _not blocked_ — it could handle other coroutines.
        
4. `await io_future` and `await cpu_future` pause until each finishes.
    
5. Results are printed.
    

---

### **5. Subtleties and Nuances**

#### **a) Why `if __name__ == "__main__":` matters**

- On Windows, `ProcessPoolExecutor` uses **spawn** instead of fork.
    
- Without this guard, the module will be re-imported in child processes → could cause infinite process spawning.
    

---

#### **b) ThreadPoolExecutor vs ProcessPoolExecutor trade-offs**

|Criteria|ThreadPoolExecutor|ProcessPoolExecutor|
|---|---|---|
|Memory overhead|Low|High (separate memory)|
|Startup overhead|Low|High|
|GIL impact|GIL-bound|GIL-free|
|Best for|Blocking I/O|CPU-bound|

---

#### **c) Event loop fairness**

- While the thread/process runs, the event loop schedules other coroutines.
    
- If you put blocking code directly in the coroutine without `run_in_executor`, the loop would **freeze** until that function finishes.
    

---

#### **d) `loop.run_in_executor(None, func, *args)`**

- Passing `None` uses the **default ThreadPoolExecutor** created by asyncio.
    
- In this code, custom pools are explicitly created for better resource control.
    

---

#### **e) Heavy CPU-bound work in threads**

- Even if you use threads, CPU-bound Python code won’t run in true parallel because of the GIL.  
    Processes bypass this limitation.
    

---

#### **f) Clean resource management**

- Executors are created with `with ...` context managers here, ensuring they’re properly shut down after use.
    

---

### **6. Bigger Picture**

This pattern is **critical** in real-world async apps:

- **Web frameworks** like FastAPI or aiohttp use exactly this trick to run database drivers or legacy sync code without blocking async endpoints.
    
- **Hybrid workloads** (network + CPU processing) often mix both executors.