# k6 Performance Testing — Summary & Guide

This document summarizes the video content about k6 and performance testing. It explains the different types of performance tests (load, stress, spike, soak), how to get started with k6, strategies for realistic tests (including running tests on a separate machine like a Raspberry Pi), and practical examples and configurations.

<br>
<img src="./img/k6.png" width="800">
<br>

---

## 🧭 Why Performance Testing with k6?
Performance testing helps you evaluate how an application behaves under realistic and extreme conditions — beyond unit and integration tests. k6 is a free, developer-friendly tool that lets you script tests in JavaScript and simulate many concurrent virtual users (VUs) to measure latency, throughput, and reliability.

Key reasons to include performance testing:
- Validate response time and throughput under load
- Find stability issues (memory leaks, resource exhaustion)
- Measure how gracefully the system degrades under stress
- Ensure performance-related SLAs are met

---

## 🔎 Types of Performance Tests (Overview)
Use different test types to answer different questions about your system.

| Test Type | Purpose | Ramp Pattern | Typical Duration | What it reveals |
|---|---:|---|---:|---|
| Load Testing | Measure system under expected traffic | Steady or staged ramp to target VUs | Minutes to hours | Baseline performance, throughput, latency |
| Stress Testing | Find breaking point | Increase until failure | Minutes to hours | System limits and failure modes |
| Spike Testing | Simulate sudden traffic bursts | Very quick ramp-up, short peak | Seconds to minutes | Autoscaling, throttling behavior |
| Soak (Endurance) Testing | Long-term stability | Sustained load over long time | Hours (e.g., ~8 hours) | Memory leaks, resource exhaustion, slow degradation |

---

## ⚙️ Installing k6
k6 can be installed on macOS, Windows, Linux, or run as a Docker image.

Examples:

- macOS (Homebrew):
```bash
brew install k6
```

- Linux (Debian/Ubuntu):
```bash
sudo apt install -y gnupg software-properties-common
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 9E1F4E7D
sudo add-apt-repository https://dl.k6.io/deb
sudo apt update
sudo apt install k6
```

- Docker:
```bash
docker run --rm -i loadimpact/k6 run - <script.js
```

(Choose the appropriate install for your environment. Docker is handy when you don't want local installation.)

---

## 🧩 k6 Test Basics — Script Structure & Options
k6 tests are written in JavaScript. Core concepts:
- VUs (virtual users) simulate concurrent clients
- Stages allow ramping users up/down over time
- Thresholds let you enforce performance requirements
- Checks provide pass/fail assertions on responses

Example: simple load test script (k6):
```javascript
import http from 'k6/http';
import { sleep, check } from 'k6';

export let options = {
  stages: [
    { duration: '30s', target: 50 },   // ramp-up to 50 VUs
    { duration: '2m',  target: 50 },   // stay at 50 VUs
    { duration: '30s', target: 0 },    // ramp-down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'],  // 95% of requests < 500ms
    'http_req_failed': ['rate<0.01'],    // <1% errors
  },
};

export default function () {
  let res = http.get('https://api.example.com/endpoint');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'body not empty': (r) => r.body.length > 0,
  });
  sleep(1);
}
```

Example explanation:
- stages: simulate ramp-up, steady state, ramp-down
- thresholds: enforce performance gates
- checks: validate correctness under load

---

## 🖥️ Running Tests on a Separate Machine (e.g., Raspberry Pi)
Running load generators on a separate host is helpful to avoid saturating the system under test network or compute resources and to simulate distributed clients.

Simple architecture:
- Local developer machine: prepare scripts
- Raspberry Pi (or cloud instance): run k6 to generate load
- System under test: receives traffic and is monitored

ASCII schema:
App Clients (k6 on Raspberry Pi) --> Network --> System Under Test (API / App Servers)
Local Dev ──> prepare scripts, push to Pi
Monitoring metrics (CPU, memory, network, app metrics) collected on SUT

Tips:
- Use a machine with enough network bandwidth for target load.
- Synchronize clocks for logs/metrics correlation.
- Avoid running test generator on the same host as the SUT.

---

## 🔁 Stages and Test Data — Making Tests Realistic
- Use staged ramps to emulate traffic growth and decay.
- Vary test data and endpoints to mimic real user behavior.
- Use randomized payloads, realistic think times (sleep), and sequence of API calls.

Example staging patterns:

Load (steady):
- 0 → 100 users over 2 minutes
- stay at 100 for 10 minutes
- ramp down to 0 over 1 minute

Spike:
- { duration: '10s', target: 0 }
- { duration: '30s', target: 1000 }  // sudden spike
- { duration: '2m', target: 0 }

Soak:
- { duration: '30m', target: 200 }   // long sustained load
- repeat or extend to 8 hours for real soak tests

Table: Example stage configs
| Test type | Example stages |
|---|---|
| Load | [ {30s, 50}, {10m, 50}, {30s,0} ] |
| Spike | [ {10s,0}, {30s,1000}, {2m,0} ] |
| Soak | [ {1h,200} ] (run for 8 hours by repeating/long duration) |

---

## 📈 Monitoring & Key Metrics to Collect
Collect both k6 metrics and system metrics (on the SUT).

k6 metrics:
- http_req_duration (latency) — use p(90), p(95), p(99)
- http_reqs (throughput / requests count)
- http_req_failed (error rate)
- vus, vus_max (virtual user counts)

System metrics (SUT):
- CPU usage
- Memory usage (RSS/heap)
- Open file descriptors
- GC metrics for managed runtimes
- Network bandwidth, connections
- Disk I/O (if applicable)

Recommended thresholds to track (example):
- 95th percentile latency < X ms
- Error rate < 1%
- CPU < 80% sustained (or tuned per infra)
- Memory stable over soak test (no steady growth)

---

## 🧪 Example Test Scenarios

1) Load test to validate baseline
- Goal: Validate 200 concurrent users with 95th percentile latency < 400ms.
- k6 options: stages ramp to 200, sustain 10m.

2) Spike test to validate autoscaling
- Goal: System scales or sheds load gracefully when a 1000-user spike occurs for 30s.
- k6 stages: quick ramp to 1000 for 30s, then ramp down.

3) Soak test to reveal resource leaks
- Goal: Run 8-hour test at 150 VUs to detect memory leaks.
- Observe memory/CPU, GC, DB connections over time.

Example spike config snippet:
```javascript
export let options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '30s', target: 1000 },  // spike
    { duration: '2m', target: 0 },
  ],
};
```

---

## ✅ Practical Tips & Best Practices
- Run load generators on separate machines to emulate distributed clients.
- Use staging environment or separate test infrastructure to avoid affecting production.
- Vary test data and user journeys — don't hit the exact same URL with identical payloads every time.
- Use thresholds in k6 to fail tests automatically when performance regresses.
- Correlate k6 results with SUT logs/metrics (use timestamps).
- Start small and iterate — discover bottlenecks progressively.
- For soak tests, ensure you have alerting and logs to capture long-term issues.

---

## 🧾 Quick Cheatsheet (Commands & Snippets)
- Run a script locally:
```bash
k6 run script.js
```

- Run via Docker:
```bash
docker run --rm -i loadimpact/k6 run - < script.js
```

- Example basic options:
```javascript
export let options = {
  vus: 50,
  duration: '5m',
};
```

---

## 📌 Concise Resume
k6 is a simple, scriptable tool for performance testing using JavaScript. Use it to run load, stress, spike, and soak tests to evaluate latency, throughput, and stability. Install k6 locally or run it in Docker; run load generators on separate machines (e.g., Raspberry Pi) to simulate real-world clients. Design realistic stage profiles and test data, monitor both k6 and system metrics, and use thresholds to enforce performance requirements. Spike tests reveal autoscaling/behavior under sudden bursts, while soak tests (multi-hour) are vital to find memory leaks and long-term degradation.
