# AI Production Engineering Handbook
## ML Engineer Perspective — Pre-Docker Foundation

> **Goal:** Understand how production AI/ML applications are served and run, from networking and HTTP up to FastAPI, async execution, processes, threads, GIL, workers, and reverse proxies.
>
> **Depth:** This handbook captures only what we have studied so far. It intentionally avoids hardcore backend/OS internals and does not repeat the same concept in multiple sections.

---

# 1. Production AI Mental Model

We are learning backend/infrastructure **from an ML Engineer perspective**, not to become hardcore backend/DevOps engineers.

The target is to be able to reason about a production AI system:

```text
User
  ↓
UI / Frontend
  ↓
API Gateway / Load Balancer
  ↓
Backend Server : Port
  ↓
Uvicorn
  ↓
FastAPI
  ↓
AI Application
  ↓
RAG / Agent / ML Model / LLM
  ↓
Response
```

Later, this will expand into Docker, Kubernetes, Azure, monitoring, RAG productionization, LLMOps, and system design.

---

# 2. Internet, Network and Server

## 2.1 Network

A **network** is a group of connected computers/devices that can communicate.

Example:

```text
Laptop ── Router ── Phone
```

A home Wi-Fi network is a network.

## 2.2 Internet

The **Internet is a network of networks**.

It connects many independent networks so that machines can communicate globally.

Simplified:

```text
Laptop
  ↓
Router
  ↓
ISP
  ↓
Internet
  ↓
Server
```

## 2.3 Server

"Server" can mean two different things:

### Server as hardware

A computer that provides resources/services.

Examples:

- Azure VM
- AWS EC2 instance
- Your laptop, if it is serving an application

### Server as software

A program that accepts/handles requests.

Examples:

- Uvicorn
- Nginx
- Apache

**Important:**

```text
Azure VM = computer / machine
Uvicorn  = server software
FastAPI  = application/framework
```

A laptop can act as a server if it runs an application that accepts requests.

---

# 3. IP Address

An **IP address identifies a machine/interface on a network** so traffic can be routed to the correct destination.

Examples:

```text
192.168.1.10
43.x.x.x
```

`127.0.0.1` is the loopback/localhost address used to refer back to the local machine.

Think:

```text
IP = Which machine?
```

---

# 4. Port

A single machine can run many applications.

A **port identifies the network endpoint/application service on that machine**.

Example:

```text
192.168.1.10:8000
               ↑
             Port
```

If Uvicorn is listening on port `8000`, requests sent to that port can reach the Uvicorn service.

Think:

```text
IP   = Which machine?
Port = Which service/application endpoint?
```

---

# 5. Socket

A port is just a number. The OS uses **sockets** to provide the actual communication mechanism.

Simplified:

```text
IP
 ↓
Port
 ↓
Socket
 ↓
Application
```

## 5.1 Listening Socket

When Uvicorn starts listening on a port, the OS has a **listening socket** waiting for incoming connections.

Example:

```text
GET /health
        ↓
Server:8000
        ↓
Listening Socket
```

## 5.2 Connected Socket

When clients connect, the OS can create a connection-specific/connected socket for each connection.

Conceptually:

```text
Listening Socket
   ├── Connected Socket → Client A
   ├── Connected Socket → Client B
   └── Connected Socket → Client C
```

**Important:** 100 users do not mean 100 FastAPI copies or 100 Uvicorn processes. A server can maintain many connections.

---

# 6. HTTP

HTTP is the application-level protocol used by clients and servers to communicate.

A simplified request:

```text
GET /chat HTTP/1.1
Host: example.com
Accept: application/json
```

Uvicorn handles the HTTP/ASGI server side and passes the request into the FastAPI application.

FastAPI works with request/application objects rather than manually parsing raw HTTP text.

## GET vs POST

### GET

Generally used to retrieve/read data.

```text
GET /users
```

### POST

Generally used when the client sends data for processing/creation.

AI example:

```text
POST /predict

{
  "prompt": "Explain RAG"
}
```

---

# 7. FastAPI vs Uvicorn

This distinction is fundamental.

## FastAPI

FastAPI is the **web framework/application layer**.

It defines routes and application logic.

Example:

```python
@app.post("/predict")
def predict():
    ...
```

Meaning:

> When a POST request comes to `/predict`, execute this function.

## Uvicorn

Uvicorn is an **ASGI server**.

It is responsible for things such as:

- Listening on a port
- Working with network connections/sockets
- Handling HTTP/ASGI communication
- Calling the FastAPI application
- Sending the response back

Simplified:

```text
Client
  ↓
Uvicorn
  ↓
FastAPI
  ↓
Python function
  ↓
Response
```

**FastAPI is not the network server itself. Uvicorn serves it.**

---

# 8. Uvicorn Command

Typical command:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

Meaning:

- `uvicorn` → run Uvicorn
- first `app` → `app.py`
- second `app` → FastAPI object, e.g. `app = FastAPI()`
- `--host 0.0.0.0` → listen on all available network interfaces rather than only localhost
- `--port 8000` → listen on port 8000

Flow:

```text
Network
  ↓
0.0.0.0:8000
  ↓
Uvicorn
  ↓
FastAPI
```

---

# 9. Sync vs Async — The Core Distinction

Two different classifications must NOT be mixed.

## Axis 1: What kind of work is the task?

### CPU-bound

Most of the time is spent doing computation.

Examples:

- Heavy Python computation
- CPU-heavy preprocessing
- Some model computations

```text
Request
  ↓
CPU computation
  ↓
CPU computation
  ↓
Done
```

CPU is actively doing work.

### I/O-bound

Most of the time is spent waiting for an external operation.

Examples:

- Database
- Redis
- External API / LLM API
- Vector database
- Azure service
- Network/file I/O

```text
Request
  ↓
External service
  ↓
WAIT
  ↓
Data arrives
  ↓
Small computation
```

During the wait, the CPU may be mostly idle.

---

## Axis 2: How does the program handle the operation?

### Synchronous

The current execution waits for the operation to finish before continuing.

Example:

```python
def get_data():
    data = sync_db_call()
    return data
```

If the DB takes 5 seconds:

```text
Thread
  ↓
DB call
  ↓
WAIT 5 sec
  ↓
Continue
```

The CPU may be idle, but **the thread is blocked**.

### Asynchronous

The operation can be awaited so that the event loop can work on other tasks while the I/O operation is pending.

```python
async def get_data():
    data = await async_db_call()
    return data
```

Conceptually:

```text
Task A
  ↓
await DB
  ↓
waiting

Event Loop
  ↓
Task B
  ↓
Task C

DB response arrives
  ↓
Task A resumes
```

### Golden distinction

> **CPU-bound / I/O-bound describes the nature of the workload.**
>
> **Sync / async describes how the program handles execution and waiting.**

Therefore:

```text
I/O-bound + Sync   → CPU may be idle, but thread is blocked
I/O-bound + Async  → waiting can be yielded to the event loop
CPU-bound + Sync   → CPU does the computation; thread remains occupied
CPU-bound + Async  → async syntax alone does NOT make CPU work non-blocking
```

---

# 10. `async def` and `await`

An `async def` function is a **coroutine function**.

Calling it produces a coroutine object that is normally executed by an event loop.

Example:

```python
async def call_api():
    ...
```

To wait for that coroutine inside another async function:

```python
async def foo():
    result = await call_api()
```

`await` essentially means:

> Suspend this coroutine at this point and allow the async machinery to run other work until the awaited operation is ready.

It does **not** mean:

> "Make any function asynchronous."

---

# 11. Eight Important Function Combinations

Assume:

```python
async def call_openai():
    ...
```

and:

```python
def heavy_cpu_computation():
    ...
```

## 1. Async + await async I/O

```python
async def foo():
    result = await call_openai()
```

**Valid.**

I/O-bound + asynchronous.

The coroutine can yield while waiting for the external API.

---

## 2. Sync function + await

```python
def foo():
    result = await call_openai()
```

**Syntax error.**

`await` cannot normally be used directly inside a normal `def` function.

---

## 3. Async function but no await

```python
async def foo():
    result = call_openai()
```

If `call_openai()` is async, this does **not** properly execute/await the coroutine.

`result` becomes a coroutine object, and if it is never awaited Python can issue a `RuntimeWarning`.

Correct:

```python
async def foo():
    result = await call_openai()
```

---

## 4. Sync + sync I/O

```python
def foo():
    result = sync_call_openai()
```

**Valid.**

I/O-bound + synchronous.

The calling thread blocks while waiting.

---

## 5. Async + await normal CPU function

```python
async def foo():
    result = await heavy_cpu_computation()
```

**Type error at runtime.**

`heavy_cpu_computation()` is a normal synchronous function, so its result is not awaitable.

The problem is not simply that it is CPU-bound; the immediate issue is that a non-awaitable result cannot be awaited.

---

## 6. Sync + await CPU function

```python
def foo():
    result = await heavy_cpu_computation()
```

**Syntax error.**

Again, `await` cannot normally appear directly inside `def`.

---

## 7. Async syntax + synchronous CPU work

```python
async def foo():
    result = heavy_cpu_computation()
```

**Syntactically valid, but the CPU work still blocks the event-loop thread.**

Writing `async def` does not magically make the computation non-blocking.

---

## 8. Normal synchronous CPU work

```python
def foo():
    result = heavy_cpu_computation()
```

**Valid.**

Normal CPU-bound synchronous code.

---

# 12. Event Loop

An **event loop is a software mechanism**, not hardware.

Python's `asyncio` ecosystem provides event-loop infrastructure. In a normal FastAPI/Uvicorn async setup, we do not implement an event loop ourselves.

Its job is roughly:

- Track async tasks
- Run tasks that are ready
- Let a coroutine yield at `await`
- Wait for I/O readiness
- Resume the appropriate coroutine when its awaited operation is ready

Simplified:

```text
Event Loop
 ├── Coroutine A → waiting for OpenAI
 ├── Coroutine B → waiting for DB
 ├── Coroutine C → ready
 └── Coroutine D → running
```

### Important mental model

Normally:

```text
Worker Process
  ↓
Main Thread
  ↓
Event Loop
  ↓
Many async tasks/coroutines
```

An event loop does **not** normally mean "one loop managing many threads."

For the learning model used here:

> **One event loop runs on one thread and manages many async tasks/coroutines.**

Advanced Python programs can use multiple event loops across different threads, but that is outside our current scope.

---

# 13. Coroutine vs Thread

These are different levels.

### Thread

An execution unit inside a process, managed by the OS/runtime.

### Coroutine

An application-level async execution unit managed by the event loop.

Typical async FastAPI picture:

```text
Process
│
└── Main Thread
      │
      └── Event Loop
            ├── Coroutine A
            ├── Coroutine B
            └── Coroutine C
```

The three coroutines are **not three threads**.

---

# 14. Process and Thread

## Process

A process is an independently managed running program/execution context.

A Python process contains things such as:

- Python interpreter
- Memory/address space
- Threads
- Runtime state

A process normally has at least one thread.

## Thread

A thread is an execution path inside a process.

```text
Process
├── Thread 1
├── Thread 2
└── Thread 3
```

Threads within the same process can share the process's memory/resources.

Processes normally have isolated memory spaces.

### Important

> **The process is not "inside the CPU."**

The process's memory/resources are associated with RAM and OS-managed resources. The CPU executes instructions belonging to a thread.

Simplified:

```text
SSD
 ↓
Program loaded
 ↓
RAM
 ↓
Process
 ↓
Thread
 ↓
CPU executes instructions
```

---

# 15. CPU, RAM and SSD/HDD

### CPU

Executes instructions and performs computation.

### RAM

Working memory used by running programs/processes.

### SSD/HDD

Persistent storage where programs/files/data remain when the machine is powered off.

Simplified:

```text
SSD/HDD
   ↓
Program/data
   ↓
RAM
   ↓
Process + Threads
   ↓
CPU executes
```

---

# 16. GIL

**GIL = Global Interpreter Lock.**

For CPython, the GIL is an interpreter/runtime mechanism, not a CPU or OS feature.

Simplified location:

```text
Python Process
│
└── Python Interpreter
      │
      └── GIL
```

Its key effect for traditional CPython execution:

> At a given time, only one thread in a CPython process can execute Python bytecode while holding the GIL.

Therefore:

```text
One Process
├── Thread 1 → Python bytecode
├── Thread 2 → waits for GIL
└── Thread 3 → waits for GIL
```

for CPU-bound Python-bytecode execution.

### Separate processes

Each process has its own interpreter/GIL:

```text
Process 1 → Interpreter → GIL 1
Process 2 → Interpreter → GIL 2
Process 3 → Interpreter → GIL 3
```

Therefore processes can provide true parallel Python execution across CPU cores, subject to the usual OS/resource constraints.

---

# 17. Why GIL Does Not Make Threads Useless

For I/O-bound work, a thread can spend time waiting for external I/O.

The important point is:

```text
CPU may be idle
BUT
thread can be blocked
```

Async programming avoids blocking the event-loop thread when the I/O operation is implemented in an awaitable/non-blocking way.

Threads are also useful when you need to run a **blocking synchronous I/O library** without blocking the event-loop thread.

---

# 18. Multithreading

Multithreading means using multiple threads within one process.

Example:

```text
Process
├── Thread 1 → DB task
├── Thread 2 → API task
└── Thread 3 → File task
```

For CPU-bound Python bytecode in CPython, the GIL limits simultaneous Python-bytecode execution within that process.

For I/O-bound workloads, multiple threads can still be useful because threads spend time waiting on I/O.

---

# 19. Thread Pool

A **thread pool is a reusable collection of threads**.

```text
Process
│
├── Main Thread
│     └── Event Loop
│
└── Thread Pool
      ├── Thread 1
      ├── Thread 2
      └── Thread 3
```

It is especially useful for integrating **blocking synchronous I/O** into an async application.

Example concept:

```python
async def endpoint():
    result = await run_in_threadpool(sync_db_call)
    return result
```

Conceptually:

```text
Main Thread
  ↓
Event Loop
  ↓
await
  ↓
Thread Pool
  ↓
Thread 1
  ↓
Blocking sync DB call
```

While the thread performs the blocking call, the event-loop thread can handle other async tasks.

### Important

A thread pool does **not** mean one thread per request. Threads are reusable and the pool has a finite size.

### CPU-bound warning

A thread pool is generally not the default way to obtain parallelism for CPU-heavy Python code because of the CPython GIL.

---

# 20. Process Pool

A process pool is a reusable collection of processes.

```text
Main Process
│
└── Process Pool
      ├── Process 1
      ├── Process 2
      └── Process 3
```

Each process has:

- Its own interpreter
- Its own GIL
- Separate process memory

Therefore a process pool is a common approach for CPU-heavy Python work when parallel CPU execution is needed.

It has more memory/process overhead than a thread pool.

---

# 21. Thread Pool vs Process Pool

| | Thread Pool | Process Pool |
|---|---|---|
| Unit | Thread | Process |
| Memory | Shared within process | Separate |
| GIL | Same process GIL | Each process has its own GIL |
| Overhead | Lower | Higher |
| Good for | Blocking I/O | CPU-heavy Python work |
| Parallel Python bytecode | Limited by GIL | Can run across processes/cores |

For ML workloads, CPU-heavy work can also be handled through native libraries, GPU execution, specialized inference servers, batching, or external workers. Process pools are a common concept, not a universal rule.

---

# 22. Uvicorn Workers

Uvicorn can run multiple worker processes.

Example:

```bash
uvicorn app:app --workers 4
```

Simplified:

```text
Uvicorn
├── Worker 1 → Process 1
├── Worker 2 → Process 2
├── Worker 3 → Process 3
└── Worker 4 → Process 4
```

Each worker is a separate process with its own:

- Python interpreter
- GIL
- Memory space
- Application runtime

A worker process normally has its own main thread/event-loop setup.

Simplified:

```text
Worker Process
│
├── Python Interpreter
├── GIL
└── Main Thread
      └── Event Loop
            ├── Request A
            ├── Request B
            └── Request C
```

## Why multiple workers?

Multiple workers can provide:

- Better CPU utilization for appropriate workloads
- Process-level isolation
- Better resilience if one worker crashes
- More independent application execution contexts

### Important for ML

Do **not** blindly set:

```text
workers = CPU cores
```

because every process can consume additional RAM and may load its own ML model.

Example:

```text
Model = 4 GB

Worker 1 → potentially 4 GB
Worker 2 → potentially 4 GB
Worker 3 → potentially 4 GB
Worker 4 → potentially 4 GB
```

Actual memory behavior depends on model/framework/OS loading details, but multiple workers can significantly increase memory usage.

GPU serving is even more sensitive:

```text
GPU VRAM = 8 GB
Model = 6 GB
```

Multiple processes trying to load separate copies can exceed VRAM.

### Important distinction

```text
4 Uvicorn workers
```

means roughly:

```text
4 processes
```

It does **not** mean only 4 concurrent requests. One async worker can handle many concurrent I/O-bound operations.

---

# 23. Load Balancer

A load balancer distributes incoming traffic across healthy backend instances.

```text
                Load Balancer
                /     |     \
               ↓      ↓      ↓
           Worker 1 Worker 2 Worker 3
```

It can also perform health checks.

Example:

```text
GET /health
```

If an instance:

- Times out
- Does not respond
- Returns an unhealthy response

the load balancer can stop routing traffic to it.

### Interview statement

> A load balancer performs health checks and routes traffic only to healthy instances.

---

# 24. Worker vs Load Balancer

Do not mix them.

### Worker

An application execution process.

```text
Worker → Process → FastAPI
```

### Load Balancer

A traffic-distribution component.

```text
Clients
  ↓
Load Balancer
  ↓
Multiple workers/instances
```

---

# 25. Reverse Proxy / Nginx

A **reverse proxy** sits in front of backend applications and receives client requests on their behalf.

Example:

```text
User
 ↓
Reverse Proxy
 ↓
Uvicorn
 ↓
FastAPI
```

Nginx is a common reverse proxy.

It can handle responsibilities such as:

- Request routing
- TLS/HTTPS termination
- Connection handling
- Potential load balancing

Nginx does **not** replace FastAPI or Uvicorn.

### Roles

```text
Nginx / Reverse Proxy
→ Traffic management / routing / TLS

Uvicorn
→ ASGI server, serves FastAPI

FastAPI
→ Application/API logic
```

---

# 26. API Gateway vs Reverse Proxy vs Load Balancer

These roles can overlap in real architectures, and they do not have to be separate boxes.

A possible architecture:

```text
User
 ↓
UI
 ↓
API Gateway / Load Balancer
 ↓
Reverse Proxy (optional)
 ↓
Uvicorn
 ↓
FastAPI
```

In a cloud/Kubernetes environment, some of these responsibilities may be provided by managed cloud services, ingress/gateway components, or Kubernetes networking rather than a manually deployed Nginx server.

Do not assume every production system has all three as separate layers.

---

# 27. Full Request Journey

A useful conceptual flow:

```text
User
 ↓
UI / Frontend
 ↓
API Gateway / Load Balancer
 ↓
Backend IP : Port
 ↓
Listening Socket
 ↓
Connection Socket
 ↓
Uvicorn
 ↓
FastAPI
 ↓
AI / Business Logic
 ↓
LLM / DB / Vector DB / Redis
 ↓
Response
```

The response travels back through the network/proxy path to the client.

Socket terminology is mainly useful for understanding how the network connection reaches Uvicorn; application developers normally work at higher abstractions.

---

# 28. Cloud Fundamentals Already Covered

## On-Premises

Company owns and manages:

- Physical servers
- Electricity
- Cooling
- Networking
- Physical security
- Hardware maintenance

Scaling requires acquiring/configuring more hardware.

## Cloud

Instead of owning all infrastructure, organizations use infrastructure/services provided by cloud providers.

Examples:

- Azure
- AWS
- GCP

Cloud can provide:

- CPU
- RAM
- Storage
- GPU
- Networking
- Managed services

---

# 29. Azure Datacenter and VM

A cloud provider operates large datacenters containing physical servers.

Simplified:

```text
Azure Datacenter
 ↓
Physical Servers
 ↓
Virtualization
 ↓
Virtual Machines
```

A **VM is a virtual computer** with allocated resources such as CPU, RAM, storage, and an operating system.

Example:

```text
Azure VM
 ↓
Ubuntu
 ↓
Python
 ↓
Uvicorn
 ↓
FastAPI
```

The cloud provider supplies/manages the underlying infrastructure; your application still needs to be served by something like Uvicorn.

---

# 30. Hypervisor

A **hypervisor** is software that enables multiple virtual machines to run on a physical server.

```text
Physical Server
 ↓
Hypervisor
 ↓
VM 1
VM 2
VM 3
```

Examples discussed:

- VMware ESXi
- Hyper-V
- KVM

---

# 31. VM vs Container — Conceptual Difference

VM:

```text
Physical Server
 ↓
Hypervisor
 ↓
VM
 ↓
Guest OS
 ↓
Application
```

Container:

```text
Host OS
 ↓
Container Runtime
 ↓
Container
 ↓
Application + Dependencies
```

Containers share the host OS kernel, whereas VMs provide a separate guest OS environment.

Containers are generally lighter and faster to start than full VMs.

---

# 32. Docker — Why We Need It

Before Docker:

```text
Developer Machine
→ Python version
→ Library versions
→ OS/runtime assumptions

Production Machine
→ Different versions
→ Different environment
```

This creates:

> "Works on my machine."

Docker packages an application with its required runtime/dependencies into a reproducible image.

### Docker Image

A read-only blueprint/template used to create containers.

```text
Dockerfile
 ↓
Docker Image
```

### Docker Container

A running instance of an image.

```text
Docker Image
 ↓
Container
 ↓
Application
```

One image can create multiple containers.

### Dockerfile

Defines image-building instructions such as:

```text
FROM
WORKDIR
COPY
RUN
CMD
```

### Image layers

Images are built in layers. With appropriate caching, unchanged layers can be reused so builds do not always redo every step.

---

# 33. Kubernetes — Basic Intuition

Docker makes containers manageable, but manually managing many containers becomes difficult.

Kubernetes helps manage containers at scale.

Responsibilities introduced at a high level:

- Deployment
- Restart/recovery
- Scaling
- Load balancing
- Self-healing

Example:

```text
Need 5 application instances
        ↓
Kubernetes
        ↓
5 Pods
```

If one fails:

```text
Pod ❌
 ↓
Kubernetes
 ↓
Replacement Pod
```

### Azure example

Azure Kubernetes Service = **AKS**.

Conceptual architecture:

```text
Azure Datacenter
 ↓
Infrastructure
 ↓
AKS
 ↓
Pods / Containers
 ↓
FastAPI
```

---

# 34. Load Balancer + Kubernetes

A Kubernetes/cloud deployment may look like:

```text
Internet
 ↓
Cloud Load Balancer
 ↓
Ingress / Gateway
 ↓
Kubernetes Service
 ↓
Pods
 ↓
Container
 ↓
Uvicorn
 ↓
FastAPI
```

Kubernetes Pods/replicas and Uvicorn workers are different scaling layers.

For example, a deployment could use:

```text
Pod 1 → Container → Uvicorn
Pod 2 → Container → Uvicorn
Pod 3 → Container → Uvicorn
```

Kubernetes can scale Pods while each container runs its application server.

---

# 35. Queues — Why They Exist

A queue does **not** make a heavy task itself faster.

Suppose model training takes 15 minutes.

Without a queue:

```text
User
 ↓
FastAPI worker
 ↓
15-minute training
 ↓
Response
```

The request-handling worker remains occupied.

With a background task queue:

```text
User
 ↓
FastAPI
 ↓
Queue
 ↓
"Training started"
```

Then:

```text
Queue
 ↓
Background Worker
 ↓
15-minute training
```

The important benefit is:

> **The API/request-handling layer is freed quickly; the heavy task continues independently.**

This is useful for long-running work such as training, document processing, or other jobs that do not need to complete inside the request-response cycle.

---

# 36. Production AI Mental Model — Current Level

At the end of the pre-Docker foundation, the architecture we understand is:

```text
                         User
                           │
                           ▼
                     UI / Frontend
                           │
                           ▼
               API Gateway / Load Balancer
                           │
                           ▼
                  Backend IP : Port
                           │
                           ▼
                   Listening Socket
                           │
                           ▼
                        Uvicorn
                           │
                           ▼
                       FastAPI
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 Async          Blocking
                I/O work        sync work
                    │             │
                Event Loop    Thread Pool
                    │
                    ▼
             AI Application
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Vector DB  Redis      LLM
```

For CPU-heavy work:

```text
FastAPI
  ↓
Process Pool / Background Worker / Specialized compute
```

For large-scale deployment:

```text
Cloud
 ↓
Kubernetes / AKS
 ↓
Pods / Containers
 ↓
FastAPI + Uvicorn
```

---

# 37. Key Interview Takeaways

## Networking

- IP identifies/routs to a machine/interface.
- Port identifies a service endpoint on that machine.
- Socket provides the actual communication mechanism.
- A listening socket waits for incoming connections.
- Connected sockets represent established client connections.

## FastAPI stack

- FastAPI = application/framework.
- Uvicorn = ASGI server.
- Uvicorn receives/handles HTTP/ASGI traffic and invokes FastAPI.
- FastAPI defines routes and application logic.

## Async

- I/O-bound ≠ automatically async.
- CPU-bound ≠ automatically sync.
- Sync I/O can leave CPU idle while blocking the current thread.
- Async I/O allows the event loop to handle other tasks during awaitable I/O waits.
- `async def` alone does not make CPU-heavy work non-blocking.
- `await` is for awaitable operations/coroutines.

## Process/Thread/GIL

- Process = independent execution context.
- Thread = execution unit inside a process.
- Threads in one process can share memory.
- CPython GIL limits simultaneous Python-bytecode execution within one process.
- Separate processes have separate interpreters/GILs.
- CPU-heavy Python parallelism commonly uses multiple processes.
- I/O-heavy work can benefit from async or threads depending on the library/execution model.

## Workers

- Uvicorn workers are separate processes.
- More workers ≠ simply more concurrent requests.
- Async workers can handle many concurrent I/O-bound operations.
- More workers can increase RAM/VRAM usage, especially when each process loads an ML model.

## Reverse proxy

- Reverse proxy sits in front of backend services.
- Nginx is one possible reverse proxy.
- It can handle routing, TLS termination, and potentially load balancing.
- In cloud/Kubernetes, managed gateways/ingress/load balancers can provide overlapping functionality.

## Cloud

- On-prem = company owns/manages infrastructure.
- Cloud = infrastructure/services provided by cloud provider.
- VM = virtual computer.
- Hypervisor enables multiple VMs on physical infrastructure.

---

# 38. What Comes Next

We are **not starting Docker from this handbook**; Docker is the next chapter.

Immediate next sequence:

```text
Docker + FastAPI
 ↓
Docker Networking
 ↓
Host Port vs Container Port
 ↓
Container-to-Container Communication
 ↓
Docker Volumes
 ↓
Docker Compose
 ↓
Git / GitHub
 ↓
CI/CD
 ↓
MLflow
 ↓
Kubernetes
 ↓
Azure Services
 ↓
Monitoring / Observability
 ↓
Production ML
 ↓
Production RAG
 ↓
Agentic AI Production
 ↓
LLMOps
 ↓
AI System Design
```

---

# 39. Ultra-Quick Revision Sheet

If you only have 5 minutes:

```text
Server
→ Computer/service provider

IP
→ Which machine/interface?

Port
→ Which service endpoint?

Socket
→ Actual communication endpoint

HTTP
→ Request/response protocol

Uvicorn
→ ASGI server

FastAPI
→ API/application framework

Process
→ Independent execution context

Thread
→ Execution unit inside process

GIL
→ CPython mechanism limiting Python-bytecode execution to one thread at a time per process

Event Loop
→ Runs/manages async tasks on a thread

Coroutine
→ Async execution unit managed by event loop

I/O-bound
→ Mostly waiting for external operation

CPU-bound
→ Mostly doing computation

Sync
→ Current execution waits/blocking

Async
→ Awaitable I/O can yield control while waiting

Thread Pool
→ Reusable threads, useful for blocking sync I/O

Process Pool
→ Reusable processes, useful for CPU-heavy Python work

Uvicorn Workers
→ Multiple Uvicorn processes

Load Balancer
→ Distributes traffic across healthy instances

Reverse Proxy
→ Front-facing proxy that routes/manages backend traffic

VM
→ Virtual computer

Docker
→ Package application/runtime/dependencies consistently

Kubernetes
→ Manage/scale/recover containers
```

---

## Current Study Boundary

**Completed before Docker:**

- Networking fundamentals
- IP / Port / Socket
- HTTP
- FastAPI
- Uvicorn
- Sync vs Async
- Event Loop
- Coroutine / `await`
- CPU-bound vs I/O-bound
- Process vs Thread
- RAM / CPU / SSD basics
- GIL
- Multithreading
- Thread Pool
- Multiprocessing / Process Pool
- Uvicorn Workers
- Load Balancer
- Reverse Proxy / Nginx
- Basic cloud / VM / hypervisor / container / Kubernetes intuition

**Next chapter: Docker + FastAPI.**
