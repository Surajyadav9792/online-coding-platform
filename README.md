# 💻 CodeJudge — High-Performance Online Judge & Distributed Code Execution Platform

[![React](https://img.shields.io/badge/React-18-blue.svg?logo=react&logoColor=white)](https://react.dev)
[![Node.js](https://img.shields.io/badge/Node.js-Express-green.svg?logo=node.js&logoColor=white)](https://nodejs.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-Atlas-brightgreen.svg?logo=mongodb&logoColor=white)](https://mongodb.com)
[![Redis](https://img.shields.io/badge/Redis-Cloud-red.svg?logo=redis&logoColor=white)](https://redis.io)
[![BullMQ](https://img.shields.io/badge/Queue-BullMQ-orange.svg?logo=redis&logoColor=white)](https://bullmq.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

An advanced, high-throughput, full-stack online judge platform that enables users to compile and evaluate code in real-time. Engineered with a highly scalable, asynchronous queue architecture, sandboxed execution processes, and advanced memory/query caching layers.

---

## 🔗 Live System Endpoints

* **Production Frontend Client (Vercel):** [https://frontend-peach-ten-81.vercel.app](https://frontend-peach-ten-81.vercel.app)
* **Production Backend API Server (Render):** [https://leetcode-backend-ml5e.onrender.com](https://leetcode-backend-ml5e.onrender.com)
* **System Keep-Alive Engine:** 🟢 Active (Utilizing automated cron-pings every 13 minutes to eliminate cold start delays)

---

## 📖 Deep-Dive Project Overview

**CodeJudge** is an interactive platform built for developers to practice Data Structures and Algorithms (DSA) and receive instant, automated feedback on their coding solutions. It simulates real-world online judge environments by compiling and executing user-submitted source code against target test suites, evaluating correctness, runtime efficiency, and peak memory usage.

### 🔄 The User Journey & Core Workflows

```text
[User Logged In] 
       │
       ▼
 1. Browse Homepage ────► Filters problems by tags (Array, DP, LinkedLists, Graph) & Difficulty
       │
       ▼
 2. Coding Workspace ───► Interactive code editor with sample test inputs
       │
       ├─► Click "Run Code" ───► Executes code against visible test cases for quick checks
       └─► Click "Submit" ─────► Evaluates code against hidden test suite for problem completion
```

#### 1. The User Experience Flow
1. **Registration & Session Security:** Users create an account or sign in. The system delivers a signed JSON Web Token (JWT) over secure HttpOnly cookies, protecting the session from XSS/CSRF scripting attacks.
2. **Problem Discovery:** The user is greeted with a dynamic dashboard displaying curated problems. Users can filter questions by tags (e.g., Dynamic Programming, Graph) or difficulty (Easy, Medium, Hard).
3. **Solving & Execution:** Selecting a problem opens an interactive editor. Users write their solution in **JavaScript, C++, or Java**.
   * **Run Code:** Compiles and runs the code using the visible test cases. Instantly shows output matches and error streams.
   * **Submit Code:** Submits the solution for final verification. The system executes the source code against a comprehensive suite of hidden test cases, registers the solution under their profile on success, and logs the metrics.

#### 2. The Administrator Workflow (Problem Creation & Validation)
1. **Problem Creator Panel:** Authorized administrators (`role: 'admin'`) can access a secure interface to create, update, or remove coding problems.
2. **Asynchronous Reference Validation:** To prevent invalid problems, when an admin adds a new question with a set of test cases:
   * The admin must provide a *Reference Solution* (correct solution code).
   * Before saving to MongoDB, the backend automatically runs the admin's reference code against the newly added test cases.
   * Only if the reference solution passes 100% of the test cases (Status: *Accepted*) does the backend write the problem to the database.

---

## ⚙️ How the Code Execution Sandbox Works (Under the Hood)

Security and isolation are critical when compiling and executing arbitrary user code. CodeJudge isolates execution processes to keep the host server secure:

```text
[Submissions API] ──► Push Job to Queue ──► [BullMQ Worker]
                                                    │
                                                    ▼
                                           1. Generate MD5 Hash
                                           2. Write code to /tmp/sub_hash.cpp
                                           3. Compile: g++ -O2 /tmp/sub_hash.cpp
                                           4. Execute binary in Sandbox
                                           5. Capture stdout/stderr + check timeouts
                                           6. Clean up temporary files
```

1. **Compilation Sandboxing:**
   * **Isolation:** When the BullMQ Worker picks up a compilation job, it writes the user's code to a temporary file in the operating system's temp directory named with a unique MD5 hash.
   * **Sub-process Spawn:** It spawns compilation tools (like `g++` for C++ or `javac` for Java) as low-privilege child processes.
2. **Sandboxed Execution & Guardrails:**
   * **Execution Limits:** The compiled executable is run inside a restricted environment with a strict **timeout ceiling of 5000ms**. If an infinite loop is detected, the sub-process is terminated and a `Time Limit Exceeded` (TLE) verdict is returned.
   * **Buffer Caps:** Captures console outputs (`stdout`/`stderr`) up to a maximum buffer of 10MB to prevent heap out-of-memory crashes on infinite print loops.
3. **Evaluation & Cleanup:**
   * Standard output is compared line-by-line with the expected outputs.
   * Temporary execution binaries and source files are unlinked (deleted) instantly to prevent disk space exhaustion.

---

## 🎯 System Engineering & Performance Architecture (For Recruiters)

This project goes beyond a simple CRUD application, solving performance bottlenecks through targeted optimizations:

* **⚡ Concurrency Control (BullMQ + Redis):** Avoids blocking Node's main single-threaded event loop during slow compilations. Express pushes compile tasks to a Redis queue. Isolated workers process these jobs concurrently, supporting up to **25 parallel compilations**.
* **⚡ DB Hydration Bypass (`.select().lean()`):** Replaced heavy Mongoose documents with lightweight, plain JavaScript objects for read-only validation routes, decreasing payload query overhead by **~50-60%**.
* **⚡ Bcrypt Salt Round Tuning:** Reduced salt rounds from 10 to 8. This keeps password storage cryptographically secure while speeding up authentication hashing loops by **~4x**.
* **⚡ Session Token Blocklisting:** Uses Redis storage to revoke active JWT tokens immediately on user logout. This prevents replay attacks without querying MongoDB.

---

## 📌 Distributed System Design

```mermaid
graph TD
    User([User Browser]) -->|HTTP / HTTPS| Vercel[Vercel: React Client]
    Vercel -->|JSON API Requests| Render[Render: Node/Express API Gateway]
    
    subgraph Storage & Caching Layer
        Render -->|User Auth/Metadata| MongoDB[(MongoDB Atlas)]
        Render -->|Blocklist Tokens / Cache| Redis[(Redis Cloud)]
    end
    
    subgraph Code Execution Engine
        Render -->|Push Job| BullMQ[Redis BullMQ Queue]
        BullMQ -->|Fetch Job| Worker[Judge Worker Engine]
        Worker -->|Compile & Run| Sandbox[Compilation Sandbox]
        Sandbox -->|Output / Time / Memory| Worker
        Worker -->|Write Return Value| BullMQ
        Render -->|Wait / Fetch Result| BullMQ
    end
    
    Render -->|Response| Vercel
```

---

## 🚀 Performance Optimization Matrix (Before vs. After)

| Target Area | Bottleneck Identified | Optimization Implemented | Result / Speedup |
|---|---|---|---|
| **API Startup** | 30s-60s Cold Starts | Implemented asynchronous, non-blocking DB & Redis initialization | **Instant Gateway Boot (~1s)** |
| **Auth Hashing** | ~1.5s hashing lag | Tuned Bcrypt salt factor down to `8` for optimal node cycles | **~4x faster hashing** |
| **Database Queries** | Heavy doc hydration | Shifted auth checks and lists to selective `.lean()` queries | **~2x faster query retrieval** |
| **App Initialization** | Sequential boot checks | Refactored initial ping and session validation via `Promise.all` | **~50% reduction in load time** |
| **UX Responsiveness** | Infinite spinners on timeout | Created Axios abort signals, 15s ceilings, and auto-retry blocks | **Zero page lockups** |

---

## 🖥️ Repository Subservices

This platform is divided into microservices hosted across dedicated repositories:

* **🖥️ Frontend React Client:** [Online-Coding-Platform-Frontend-Part](https://github.com/Surajyadav9792/Online-Coding-Platform-Frontend-Part)
* **⚙️ Backend API Server:** [Online-Coding-Platform-Backend-Part](https://github.com/Surajyadav9792/Online-Coding-Platform-Backend-Part)

---

## 📂 Core API Endpoints

### 🔐 Authentication
* `POST /user/register` - Create user profile
* `POST /user/login` - Initiate session (attaches JWT HttpOnly cookie)
* `POST /user/logout` - Revoke token and blocklist in Redis
* `GET /user/check` - Validate active session state

### 🧩 Problem Engine
* `GET /problem/getAllProblem` - Retrieve all available problems
* `GET /problem/problemById/:id` - Fetch problem metadata & reference solutions
* `POST /problem/create` - [Admin] Add new coding challenge
* `PUT /problem/update/:id` - [Admin] Modify challenge parameters

### ⚡ Evaluation
* `POST /submission/run/:problemId` - Execute code against visible test cases
* `POST /submission/submit/:problemId` - Compile and submit against hidden test cases

---

## ⚙️ Local Development Setup

Please refer to the individual **Frontend** and **Backend** repositories for step-by-step guidelines on local environment configuration and dependency setups.

---

## 📝 License

Distributed under the MIT License.
