# project2-fastapi-vision
# Lab 8 Submission: FastAPI Image Classification & Analysis Service

## 1. Automated Test Results
- Ran `pytest -q test_api.py`
- **Output:** `8 passed in 19.07s` (Covering health check, prediction schema, parity, content-type checks, payload size limits, and PII masking).
- <img width="854" height="154" alt="Screenshot 2026-11-06 112704" src="https://github.com/user-attachments/assets/e88d06e1-21da-4ba7-98cd-58948506d5c6" />


## 2. Latency Benchmark & Hardware Specs
- **Hardware:** Local Windows PC Workstation
- **Benchmark Parameters:** N = 50 requests
- **p50 Latency:** 65 ms
- **p95 Latency:** 68 ms
- **Throughput:** ~7.3 requests/s (single worker)
- <img width="479" height="92" alt="Screenshot 2026-11-05 163559" src="https://github.com/user-attachments/assets/95986f0f-f13c-4265-8e9e-258cd786859d" />


## 3. Little's Law Capacity Analysis
Using Little's Law ($L = \lambda W$):
- Latency ($W$) at p50 = $0.065\text{ seconds}$
- Throughput ($\lambda$) = $\frac{1}{0.065} \approx 15.38\text{ requests/sec}$
- **Total Estimated Capacity:** $15.38 \times 86,400\text{ seconds} \approx \textbf{1,328,832 requests/day}$ per worker under ideal continuous load.

## 4. Bad Input Testing & API Response
- **Tested Input:** Sent a request to the `/predict` endpoint without providing the required `X-API-Key` header.
- **API Response:** The security dependency successfully intercepted the request and returned a **401 Unauthorized** ("Invalid or missing API key") status code.

- ## 5. API Key Security & Swagger UI Verification (Task A.6)

To test and verify API key protection and endpoint execution interactively, the FastAPI Swagger UI (`/docs`) was utilized:

- **1. Request Parameters Setup:** Providing the required `X-API-Key` header (`choose-a-long-secret`) and uploading the sample image (`sample.jpg`).
  <img width="864" height="365" alt="Screenshot 2026-11-06 111705" src="https://github.com/user-attachments/assets/5013cf06-d3b3-4e93-b92b-f7299ef67173" />


- **2. API Schema & Documentation:** Verified OpenAPI schema definitions for successful `200 OK` responses and `422 Validation Error` handling.
  <img width="877" height="448" alt="Screenshot 2026-11-06 111723" src="https://github.com/user-attachments/assets/903ed932-7106-4ef0-ad7e-e520c6673c14" />


- **3. Successful Execution & Prediction Response:** Receiving a `200 OK` status code along with the model prediction (`"label": "PNEUMONIA"`), probability breakdown, and execution latency.
  <img width="841" height="418" alt="Screenshot 2026-11-06 111737" src="https://github.com/user-attachments/assets/0d052c10-70f7-4f5a-8528-16d18359a4e4" />


- # check understanding questions answers
- 1. A client uploads a PDF to /predict. Which status code should it get, and why not 500?Status Code: 415 Unsupported Media Type (or 400 Bad Request depending on validation).Why not 500?
      A 500 Internal Server Error represents an unexpected server-side crash or bug. Uploading an unsupported file format like a PDF is a client-side error (invalid input type), which should be handled gracefully and explicitly rejected with a 415 error instead of crashing the server.
 2. Why is the model loaded in lifespan rather than inside the endpoint?
        Loading the model in the FastAPI lifespan context ensures it is loaded only once when the server starts up.If it were loaded inside the endpoint function, the heavy PyTorch model weights would have to be reloaded from disk into memory on every single incoming request, causing massive latency spikes and destroying server throughput.
 3. p50 = 40 ms and p95 = 400 ms. What does that tell you, and what would you check?
    What it tells you: While half of the requests are fast (p50 = 40 ms), a small tail of requests (5%) experiences a huge latency spike up to 400 ms. This indicates significant latency inconsistency or tail latency issues.
    What you would check: Server resource contention (CPU/GPU throttling or memory swapping).Garbage collection pauses or cold-start thread pool delays.Varying input image sizes causing longer preprocessing or variable tensor inference times.
 4. 500,000 requests a day, peak 5 times the average, 25 requests per second per worker. How many workers?Step 1: Calculate average requests per second (RPS):$$\text{Average RPS} = \frac{500,000 \text{ requests}}{86,400 \text{ seconds}} \approx 5.79 \text{ req/sec}$$
Step 2: Calculate peak RPS:$$\text{Peak RPS} = 5.79 \times 5 = 28.94 \text{ req/sec}$$
Step 3: Calculate required workers:$$\text{Workers} = \frac{\text{Peak RPS}}{\text{Capacity per worker}} = \frac{28.94}{25} \approx 1.16$$
Answer: You will need 2 workers (rounded up to handle the peak load safely).
 5. Why can model inference inside an async def endpoint slow down other requests?
   PyTorch tensor operations and CPU-bound model inference are synchronous and block the main event loop thread.In an async def function, if heavy computation is run directly without offloading (like using run_in_threadpool), it blocks the single-threaded event loop entirely, preventing FastAPI from handling concurrent requests until the inference finishes.
 6. How many communication channels does a 4-person team have?
    Using the communication channels formula for a team of size $n$:$$\text{Channels} = \frac{n(n - 1)}{2}$$For $n = 4$:$$\text{Channels} = \frac{4(4 - 1)}{2} = \frac{4 \times 3}{2} = \mathbf{6 \text{ channels}} 
