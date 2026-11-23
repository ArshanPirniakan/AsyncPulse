AsyncPulse
An asynchronous URL monitoring and job processing system built with FastAPI, asyncio, HTTPX and async SQLAlchemy (SQLite).
Submit a single URL or a batch of URLs and get an ID back immediately. A pool of asyncio workers checks each URL in the background, measures response time, retries transient failures with exponential backoff, and stores the results in SQLite so they survive restarts.
Features
Single and batch URL submission with strict URL validation (http/https only)
asyncio.Queue feeding a configurable pool of async workers
One shared httpx.AsyncClient, with asyncio.Semaphore limiting concurrent requests
Configurable timeout, retry count and exponential backoff
Retryable vs non-retryable error classification
Job states PENDING, RUNNING, COMPLETED, PARTIALLY_FAILED, FAILED; batch status is derived from its jobs
Persistence of jobs, batches and results in SQLite (via aiosqlite)
Recovery on startup: unfinished jobs are re-queued after a restart or crash
Graceful shutdown: in-flight jobs finish, or return to PENDING if the shutdown timeout expires
Logging, type hints, dependency injection through FastAPI Depends
Requirements
Python 3.11 or newer.
Installation
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
Running
uvicorn app.main:create_app --factory --host 127.0.0.1 --port 8000
Interactive API docs are served at http://127.0.0.1:8000/docs. The SQLite database file (asyncpulse.db by default) is created on first start.
Running the tests
python -m pytest
The suite needs no internet access. External HTTP calls are mocked with httpx.MockTransport; the one real-network test talks to a throwaway server bound to 127.0.0.1.
Configuration
Settings are read from environment variables prefixed with ASYNCPULSE_ or from a .env file (see .env.example).
Variable
Default
Description
ASYNCPULSE_DATABASE_URL
sqlite+aiosqlite:///./asyncpulse.db
SQLAlchemy async database URL
ASYNCPULSE_WORKER_COUNT
4
Number of queue workers (jobs processed in parallel)
ASYNCPULSE_MAX_CONCURRENCY
10
Maximum simultaneous outgoing HTTP requests (semaphore size)
ASYNCPULSE_REQUEST_TIMEOUT
10
Per-request timeout in seconds
ASYNCPULSE_MAX_RETRIES
3
Retries after the first attempt (so up to 4 attempts)
ASYNCPULSE_BACKOFF_BASE
0.5
First retry delay in seconds; doubles on each retry
ASYNCPULSE_BACKOFF_MAX
30
Upper bound for a single retry delay in seconds
ASYNCPULSE_QUEUE_MAXSIZE
1000
Queue capacity; 0 means unbounded
ASYNCPULSE_SHUTDOWN_TIMEOUT
10
Seconds to let in-flight jobs finish on shutdown
ASYNCPULSE_USER_AGENT
AsyncPulse/1.0
User-Agent header sent with checks
ASYNCPULSE_LOG_LEVEL
INFO
Python logging level
WORKER_COUNT and MAX_CONCURRENCY are independent: workers decide how many jobs are in progress, and the semaphore caps how many requests are actually on the wire at once.
API
Method
Endpoint
Purpose
Success
POST
/api/v1/jobs
Submit a single URL
202
POST
/api/v1/batches
Submit multiple URLs (1 to 100)
202
GET
/api/v1/jobs/{job_id}
Retrieve job status and result
200
GET
/api/v1/batches/{batch_id}
Retrieve batch status and counts
200
GET
/api/v1/batches/{batch_id}/results
Retrieve results of finished checks
200
GET
/health
Health check
200 / 503
Error codes: 404 unknown job or batch, 422 invalid body or malformed ID, 503 queue full (with Retry-After) or unhealthy service.
Submit a single URL
curl -X POST http://127.0.0.1:8000/api/v1/jobs \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'
{
  "job_id": "0da9dc30-fbc0-4afd-908a-673f5669e944",
  "url": "https://example.com/",
  "status": "PENDING",
  "created_at": "2026-11-23T07:49:18.749702Z"
}
URLs are normalised by Pydantic (note the trailing /).
Submit a batch
curl -X POST http://127.0.0.1:8000/api/v1/batches \
  -H "Content-Type: application/json" \
  -d '{"urls": ["https://example.com", "https://example.com/missing", "http://localhost:1/"]}'
{
  "batch_id": "5112179d-7aaa-4ed3-9553-1c876615972b",
  "status": "PENDING",
  "total": 3,
  "job_ids": [
    "a60fe07a-35c8-4ec9-9320-be45f03cdc14",
    "85ee6e1b-a606-409f-ac7a-c59df1667b3d",
    "b5f3c95f-07dc-43c0-bc41-315e0dfe7df5"
  ],
  "created_at": "2026-11-23T07:49:18.772810Z"
}
Get a job
curl http://127.0.0.1:8000/api/v1/jobs/a60fe07a-35c8-4ec9-9320-be45f03cdc14
{
  "job_id": "a60fe07a-35c8-4ec9-9320-be45f03cdc14",
  "batch_id": "5112179d-7aaa-4ed3-9553-1c876615972b",
  "url": "http://127.0.0.1:9099/",
  "status": "COMPLETED",
  "created_at": "2026-11-23T07:49:18.772810Z",
  "updated_at": "2026-11-23T07:49:18.794603Z",
  "result": {
    "job_id": "a60fe07a-35c8-4ec9-9320-be45f03cdc14",
    "url": "http://127.0.0.1:9099/",
    "status_code": 200,
    "response_time_ms": 4.5,
    "success": true,
    "error_message": null,
    "attempts": 1,
    "checked_at": "2026-11-23T07:49:18.794603Z"
  }
}
result is null until the check finishes.
Get batch status
curl http://127.0.0.1:8000/api/v1/batches/5112179d-7aaa-4ed3-9553-1c876615972b
{
  "batch_id": "5112179d-7aaa-4ed3-9553-1c876615972b",
  "status": "PARTIALLY_FAILED",
  "total": 3,
  "counts": {"pending": 0, "running": 0, "completed": 1, "failed": 2},
  "created_at": "2026-11-23T07:49:18.772810Z",
  "updated_at": "2026-11-23T07:49:18.896885Z"
}
Get batch results
curl http://127.0.0.1:8000/api/v1/batches/5112179d-7aaa-4ed3-9553-1c876615972b/results
{
  "batch_id": "5112179d-7aaa-4ed3-9553-1c876615972b",
  "status": "PARTIALLY_FAILED",
  "total": 3,
  "completed_checks": 3,
  "results": [
    {
      "job_id": "85ee6e1b-a606-409f-ac7a-c59df1667b3d",
      "url": "http://127.0.0.1:9099/missing",
      "status_code": 404,
      "response_time_ms": 7.72,
      "success": false,
      "error_message": "HTTP 404",
      "attempts": 1,
      "checked_at": "2026-11-23T07:49:18.793913Z"
    },
    {
      "job_id": "b5f3c95f-07dc-43c0-bc41-315e0dfe7df5",
      "url": "http://127.0.0.1:1/",
      "status_code": null,
      "response_time_ms": null,
      "success": false,
      "error_message": "Connection failed (ConnectError): All connection attempts failed",
      "attempts": 2,
      "checked_at": "2026-11-23T07:49:18.896305Z"
    }
  ]
}
Only finished checks appear in results; poll the batch status until it is COMPLETED, PARTIALLY_FAILED or FAILED.
Health
curl http://127.0.0.1:8000/health
{"status": "ok", "database": "ok", "workers_running": 4, "workers_busy": 0, "queue_size": 0}
Returns 503 with "status": "unhealthy" if the database is unreachable or no workers are running.
Status model
Individual jobs move through PENDING → RUNNING → COMPLETED or FAILED. Batch status is computed from the current state of its jobs, so it can never drift from them:
Jobs in the batch
Batch status
All PENDING
PENDING
Any PENDING or RUNNING (and not all pending)
RUNNING
All finished, all succeeded
COMPLETED
All finished, all failed
FAILED
All finished, mixed
PARTIALLY_FAILED
PARTIALLY_FAILED only ever applies to batches. A single job is COMPLETED when the final response status is below 400 and FAILED otherwise.
Reliability rules
Success: final HTTP status below 400 (redirects are followed).
Retryable: timeouts, connection and other transport errors, HTTP 408, 425, 429 and any 5xx.
Not retryable: other 4xx, too many redirects, invalid URLs.
Backoff: delay = min(BACKOFF_MAX, BACKOFF_BASE * 2^(attempt-1)), no jitter. With the defaults the waits are 0.5s, 1s and 2s.
Response time: time until response headers arrive for the last attempt, in milliseconds. It is null when no response was received.
Attempts: number of requests made. It is 0 for a URL rejected by validation before any request was sent.
The semaphore is held only while a request is in flight, not during backoff sleeps.
Architecture
app/
  main.py              create_app(): builds the FastAPI app and lifespan
  schemas.py           Pydantic request/response models
  api/
    routes.py          REST endpoints
    deps.py            FastAPI dependencies (service, queue, container)
  core/
    config.py          Settings (pydantic-settings, env + .env)
    logging.py         Logging setup
    lifecycle.py       AppContainer: wires components, startup/shutdown
  db/
    models.py          SQLAlchemy models: Batch, Job, CheckResult
    session.py         Async engine, session factory, SQLite pragmas
  services/
    monitor.py         UrlChecker: HTTPX, semaphore, retries, backoff
    queue.py           JobQueue: asyncio.Queue wrapper with capacity checks
    jobs.py            JobService: persistence and batch status derivation
    workers.py         WorkerPool: worker tasks, graceful shutdown
tests/                 pytest suite
Request flow: the API validates the payload, stores PENDING jobs, puts their IDs on the queue and returns 202. A worker takes an ID, atomically claims the job (PENDING → RUNNING), runs the check through UrlChecker, and stores the result and final status.
There is no module-level mutable state. create_app() builds an AppContainer that owns the database, queue, HTTP client, checker and worker pool; routes receive pieces of it through Depends. Tests call create_app(settings, transport=httpx.MockTransport(...)) to swap the network layer.
Startup and shutdown
Startup: create tables, reset any RUNNING jobs left by a crash to PENDING, start the workers and re-queue every PENDING job in the background.
Shutdown: idle workers are cancelled immediately, busy workers get SHUTDOWN_TIMEOUT seconds to finish their current job, and any still running after that are cancelled and their job is returned to PENDING. Jobs still waiting in the queue stay PENDING in the database and are picked up on the next start.
Limitations
Single process only. The queue lives in memory, so run one Uvicorn process (no --workers N). Persistence and startup recovery cover restarts, not multiple instances.
SQLite write concurrency. SQLite allows one writer at a time. WAL mode and a 30 second busy timeout handle the small writes this app makes, but it is not meant for very high throughput.
A hard kill is covered by recovery, not by shutdown logic. If the process is killed (kill -9), jobs that were RUNNING are reset and re-run on the next start, so such a URL may be checked twice.
Queue backpressure. When the queue is full the API returns 503 instead of accepting more work. Recovered jobs wait for free slots without blocking startup.
Checks are GET requests that read only the response headers; the body is not downloaded.
Results are one record per job. This is a one-shot checker; scheduling recurring checks is out of scope.
No authentication, dashboard or schema migrations (tables are created with create_all).
Copyright & Licensing


Copyright © 2026 Arshan Pirniakan. All Rights Reserved.


AsyncPulse is proprietary software.


No permission is granted to use, reproduce, modify, distribute, sublicense, or commercially exploit this project without prior written authorization from the copyright holder.


Commercial licensing is available upon request and may require payment under a separate agreement.


For licensing inquiries, please contact the copyright holder.


Third-party dependencies are governed by their respective licenses.

