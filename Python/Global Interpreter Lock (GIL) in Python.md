
## Why GIL Exists — Mechanism & Steps

1. **Python uses Reference Counting for memory management**
    
    - Every Python object tracks how many references point to it.
        
    - When the reference count drops to zero, the object is immediately deallocated.
        
2. **Reference counting requires atomic updates**
    
    - Incrementing/decrementing the count must be thread-safe to avoid corruption or premature deallocation.
        
    - Without atomicity, simultaneous updates by multiple threads can cause race conditions.
        
3. **CPython achieves atomicity using the GIL**
    
    - The GIL is a global mutex that **ensures only one thread runs Python bytecode at a time**.
        
    - This serialization guarantees that reference counts are updated safely without complex locking.
        
4. **Simplifies interpreter internals**
    
    - No need for fine-grained locks on every object access.
        
    - Reduces complexity and overhead, improving single-threaded performance.
        
5. **Historical design context**
    
    - Python was designed before multi-core CPUs were common.
        
    - The GIL was a practical solution balancing simplicity, performance, and safety.
        
6. **Trade-off: Limited CPU-bound multi-threading**
    
    - While simplifying thread safety, the GIL prevents true parallel execution of Python code across multiple cores.
        

---

## How GIL Works — Overview

- The GIL is a **global mutex** controlling thread execution of Python bytecode.
    
- Threads must **acquire the GIL before running Python code**.
    
- After running a few bytecode instructions or a time slice, the thread **releases the GIL**, enabling context switching.
    
- This time-slicing allows multiple threads to take turns executing, preventing starvation.
    
- **C extensions can release the GIL** during long-running native code (via `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS`), enabling parallelism in native code.

# How Multiprocessing Bypasses the GIL in Python

---

## Key Concept

- The **GIL only restricts multiple threads within the same Python process** from executing Python bytecode simultaneously.
    
- **Multiprocessing creates multiple separate processes**, each with its own independent Python interpreter and memory space.
    
- Since each process has its own GIL, they run completely independently on different CPU cores.
    

---

## Mechanism

1. **Separate memory space**
    
    - Each process has its own Python interpreter and memory heap.
        
    - No shared global state or GIL across processes.
        
2. **Independent GIL per process**
    
    - Each process manages its own GIL.
        
    - Threads inside a process are limited by that process’s GIL, but processes do not block each other.
        
3. **True parallel execution**
    
    - Multiple processes can run Python bytecode in parallel on multiple CPU cores.
        
    - This allows full CPU utilization for CPU-bound workloads.
        
4. **Inter-process communication (IPC)**
    
    - Processes communicate via pipes, queues, shared memory, or other IPC methods.
        
    - Data must be serialized/deserialized when exchanged, incurring some overhead.
        

---

## Summary

|Multiprocessing|Multithreading|
|---|---|
|Multiple processes with separate GILs|One process with single GIL|
|True parallelism on multiple CPU cores|Threads execute Python code serially|
|Separate memory spaces|Shared memory space|
|Higher overhead due to IPC and process creation|Lower overhead, but limited by GIL|

---

Multiprocessing is the preferred way to achieve **true CPU-bound parallelism** in Python despite the GIL.