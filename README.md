# project2-fastapi-vision
# Lab 8 Submission: FastAPI Image Classification & Analysis Service

An enterprise-grade, production-ready FastAPI backend serving a PyTorch image classification model along with text evidence analysis, complete with strict security middleware, automated unit tests, and thorough performance benchmarking.

---

## 1. Automated Test Results (`pytest`)
To guarantee code reliability and strict adherence to API specifications, an extensive automated test suite (`test_api.py`) was executed using `pytest`. 
- **Command:** `pytest -q test_api.py`
- **Output:** `8 passed in 19.07s`
- **Test Coverage Highlights:**
  - `/health` endpoint status and model-loaded verification.
  - Prediction schema validation and probability distribution sum constraints ($\sum p = 1$).
  - Direct model inference parity checks.
  - Content-type enforcement (`415 Unsupported Media Type` for wrong formats).
  - Payload size limitations (`413 Payload Too Large` for > 5 MB uploads).
  - Corrupt image handling (`400 Bad Request`) and missing parameter validation (`422 Unprocessable Entity`).
  - PII masking logic and automated clinical referral decision support inside `/analyze`.

<p align="center">
  <img width="854" height="154" alt="Pytest Output" src="https://github.com/user-attachments/assets/e88d06e1-21da-4ba7-98cd-58948506d5c6" />
</p>

---

## 2. Latency Benchmark & Hardware Specs
Performance benchmarking was conducted against a locally running Uvicorn server instance using `bench.py` with $N = 50$ sequential requests to evaluate production responsiveness under single-worker concurrency.

- **Hardware Environment:** Local Windows PC Workstation (CPU-accelerated inference backend)
- **Benchmark Parameters:** $N = 50$ iterations, single-threaded execution client.
- **p50 Latency:** 65 ms *(Median request turnaround time)*
- **p95 Latency:** 68 ms *(Tail latency threshold)*
- **Throughput:** ~7.3 requests/second (single worker baseline)

<p align="center">
  <img width="479" height="92" alt="Benchmark Results" src="https://github.com/user-attachments/assets/95986f0f-f13c-4265-8e9e-258cd786859d" />
</p>

---

## 3. Little's Law Capacity Analysis
Applying **Little's Law ($L = \lambda W$)** to evaluate theoretical maximum single-worker capacity under steady-state conditions:
- **Mean Latency ($W$) at p50:** $0.065\text{ seconds}$
- **Implied Maximum Throughput ($\lambda$):** $\frac{1}{0.065} \approx 15.38\text{ requests/sec}$
- **Total Estimated Daily Capacity:** 
  $$\text{Capacity} = 15.38\text{ req/sec} \times 86,400\text{ seconds/day} \approx \textbf{1,328,832 requests/day}$$
  *(Note: This represents theoretical maximum throughput under ideal continuous single-worker saturation without I/O network bottlenecks).*

---

## 4. Bad Input Testing & Security Resilience
Beyond the automated test suite, extensive manual fault-injection testing was performed:
- **Missing API Key Test:** Sent requests to `/predict` and `/analyze` without the mandatory `X-API-Key` header. The custom security dependency (`check_key`) successfully intercepted the requests and returned a strict **401 Unauthorized** response (`"Invalid or missing API key"`).
- **Unsupported File Formats:** Uploading text files or PDFs resulted in an immediate **415 Unsupported Media Type** exception, preventing parser crashes and safeguarding server stability.

---

## 5. API Key Security & Swagger UI Verification (Task A.6)
To ensure robust access control, API key validation was integrated using environment variables (`os.getenv`) and FastAPI `Header` dependency injection. Interactive validation was verified via the built-in Swagger documentation interface (`/docs`):

1. **Request Parameters & Authentication Setup:** Supplying the required secret header (`X-API-Key: choose-a-long-secret`) alongside the multipart image payload (`sample.jpg`).
   <p align="center"><img width="864" height="365" alt="Swagger Parameters" src="https://github.com/user-attachments/assets/5013cf06-d3b3-4e93-b92b-f7299ef67173" /></p>

2. **API Schema & Documentation Contracts:** Validating OpenAPI specification models for successful `200 OK` responses and structured `422 Validation Error` handlers.
   <p align="center"><img width="877" height="448" alt="API Schema" src="https://github.com/user-attachments/assets/903ed932-7106-4ef0-ad7e-e520c6673c14" /></p>

3. **Successful Execution & Structured Response:** Confirming successful authentication and inference execution, returning `200 OK` alongside the classification label (`"label": "PNEUMONIA"`), granular class probabilities, and precise server-side processing latency (`latency_ms`).
   <p align="center"><img width="841" height="418" alt="Successful Execution Response" src="https://github.com/user-attachments/assets/0d052c10-70f7-4f5a-8528-16d18359a4e4" /></p>

---

## 6. Check Your Understanding: Theoretical Q&A

### 1. A client uploads a PDF to `/predict`. Which status code should it get, and why not 500?
* **Status Code:** **415 Unsupported Media Type** (or 400 Bad Request).
* **Why not 500?** A `500 Internal Server Error` signifies an unexpected server-side bug or unhandled crash. Uploading an invalid file format is a client-side input validation error; it must be handled gracefully and explicitly rejected to inform the user of the content-type constraint without destabilizing the application.

### 2. Why is the model loaded in `lifespan` rather than inside the endpoint?
* Loading the model within the FastAPI `lifespan` asynchronous context manager ensures that heavy PyTorch model weights are loaded into memory **exactly once** during server startup. If loaded inside an endpoint function, weights would reload from disk on every single incoming request, introducing severe latency overhead and crashing server throughput.

### 3. p50 = 40 ms and p95 = 400 ms. What does that tell you, and what would you check?
* **Analysis:** While typical requests are fast (p50 = 40 ms), a 10x spike at the p95 level indicates significant **tail latency inconsistency** and blocking bottlenecks under load.
* **What to check:** Server resource contention (CPU throttling, thermal limits, or memory swapping), Python garbage collection pauses, thread pool thread starvation, or wide variations in input image dimensions leading to variable tensor preprocessing times.

### 4. 500,000 requests a day, peak 5 times the average, 25 requests per second per worker. How many workers?
* **Step 1 (Average RPS):** $\frac{500,000}{86,400} \approx 5.79\text{ req/sec}$
* **Step 2 (Peak RPS):** $5.79 \times 5 = 28.94\text{ req/sec}$
* **Step 3 (Required Workers):** $\frac{28.94}{25} \approx 1.16$
* **Conclusion:** You will need **2 workers** (rounded up) to safely absorb peak traffic loads.

### 5. Why can model inference inside an `async def` endpoint slow down other requests?
* PyTorch tensor computations and CPU-bound neural network inference are synchronous tasks. In an `async def` endpoint, executing heavy computations directly blocks the single-threaded event loop entirely, preventing FastAPI from processing concurrent requests until the inference finishes. *(Solution: offloading execution via `run_in_threadpool`).*

### 6. How many communication channels does a 4-person team have?
* Using the team communication channels formula $\frac{n(n-1)}{2}$ for $n = 4$:
  $$\text{Channels} = \frac{4(4-1)}{2} = \frac{12}{2} = \mathbf{6\text{ channels}}$$
