AsyncPulse


An asynchronous URL monitoring and job processing system built with FastAPI, asyncio, HTTPX and async SQLAlchemy (SQLite).


Submit a single URL or a batch of URLs and get an ID back immediately. A pool of asyncio workers checks each URL in the background, measures response time, retries transient failures with exponential backoff, and stores the results in SQLite so they survive restarts.


Features




Single and batch URL submission with strict URL validation (http/https only).


asyncio.Queue feeding a configurable pool of async workers.


One shared httpx.AsyncClient, with asyncio.Semaphore limiting concurrent requests.


Configurable timeout, retry count and exponential backoff.


Retryable vs non-retryable error classification.


Job states: PENDING, RUNNING, COMPLETED, PARTIALLY_FAILED, FAILED.


Persistence of jobs, batches and results in SQLite via aiosqlite.


Recovery on startup: unfinished jobs are re-queued after a restart or crash.


Graceful shutdown.


Logging, type hints and dependency injection through FastAPI Depends.




Requirements


Python 3.11 or newer.


Installation


python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env



Running


uvicorn app.main:create_app --factory --host 127.0.0.1 --port 8000



Interactive API documentation:


http://127.0.0.1:8000/docs


The SQLite database file (asyncpulse.db by default) is created on first start.


Running the tests


python -m pytest



The test suite uses mocked HTTP responses and does not require external websites.


Configuration


Settings are read from environment variables prefixed with ASYNCPULSE_ or from a .env file.




Variable
Default
Description




ASYNCPULSE_DATABASE_URL
sqlite+aiosqlite:///./asyncpulse.db
SQLAlchemy async database URL


ASYNCPULSE_WORKER_COUNT
4
Number of queue workers


ASYNCPULSE_MAX_CONCURRENCY
10
Maximum simultaneous outgoing HTTP requests


ASYNCPULSE_REQUEST_TIMEOUT
10
Per-request timeout in seconds


ASYNCPULSE_MAX_RETRIES
3
Retries after the first attempt


ASYNCPULSE_BACKOFF_BASE
0.5
First retry delay in seconds


ASYNCPULSE_BACKOFF_MAX
30
Maximum retry delay in seconds


ASYNCPULSE_QUEUE_MAXSIZE
1000
Queue capacity; 0 means unbounded


ASYNCPULSE_SHUTDOWN_TIMEOUT
10
Seconds allowed for in-flight jobs to finish


ASYNCPULSE_USER_AGENT
AsyncPulse/1.0
User-Agent header


ASYNCPULSE_LOG_LEVEL
INFO
Python logging level




Worker count and maximum concurrency are independent settings. Workers determine how many jobs can be processed simultaneously, while the semaphore limits outgoing HTTP requests.


API Reference




Method
Endpoint
Purpose




POST
/api/v1/jobs
Submit a single URL


POST
/api/v1/batches
Submit multiple URLs


GET
/api/v1/jobs/{job_id}
Retrieve job status and result


GET
/api/v1/batches/{batch_id}
Retrieve batch status


GET
/api/v1/batches/{batch_id}/results
Retrieve batch results


GET
/health
Application health check




Submit a single URL


curl -X POST http://127.0.0.1:8000/api/v1/jobs \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'



Submit a batch


curl -X POST http://127.0.0.1:8000/api/v1/batches \
  -H "Content-Type: application/json" \
  -d '{"urls": ["https://example.com", "https://google.com", "https://github.com"]}'



Retrieve job status


curl http://127.0.0.1:8000/api/v1/jobs/{job_id}



Retrieve batch status


curl http://127.0.0.1:8000/api/v1/batches/{batch_id}



Retrieve batch results


curl http://127.0.0.1:8000/api/v1/batches/{batch_id}/results



Health check


curl http://127.0.0.1:8000/health



Architecture


app/
├── main.py
├── schemas.py
├── api/
│   ├── routes.py
│   └── deps.py
├── core/
│   ├── config.py
│   ├── logging.py
│   └── lifecycle.py
├── db/
│   ├── models.py
│   └── session.py
└── services/
    ├── monitor.py
    ├── queue.py
    ├── jobs.py
    └── workers.py

tests/



Request processing flow




The API validates the submitted URLs.


Jobs are stored in the database with PENDING status.


Job IDs are added to the asynchronous queue.


The API immediately returns a job or batch ID.


Workers consume jobs from the queue.


Each worker executes the URL monitoring operation asynchronously.


Concurrency limits prevent excessive simultaneous HTTP requests.


Failed requests are retried when appropriate.


Results are stored in SQLite.


Job and batch statuses are updated.




Technical Limitations




Single-process architecture: The in-memory queue is designed for one Uvicorn process. Multiple application instances are not supported.


SQLite write concurrency: SQLite is suitable for this project's workload but is not designed for high-throughput distributed processing.


Crash recovery: Unfinished jobs are recovered after restarting the application. A job may be checked more than once following an unexpected shutdown.


Queue capacity: When the queue is full, the API returns HTTP 503.


HTTP method: Monitoring uses GET requests and does not download response bodies.


One-time monitoring: Recurring scheduled checks are outside the project's scope.


No authentication or frontend dashboard: The application focuses on asynchronous backend processing.




Copyright & Licensing


Copyright © 2026 Arshan Pirniakan. All Rights Reserved.


AsyncPulse is proprietary software.


No permission is granted to use, reproduce, modify, distribute, sublicense, or commercially exploit this project without prior written authorization from the copyright holder.


Commercial licensing is available upon request and may require payment under a separate written agreement.


Any authorized use must comply with the terms established by the copyright holder.


Attribution alone does not constitute permission to use this software.


Third-party dependencies are governed by their respective licenses.


For licensing inquiries, please contact the copyright holder.

