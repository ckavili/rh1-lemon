# Load Test Results Summary

**Date**: 2026-01-02
**Application**: Lemonade Stand with FMS Guardrails Orchestrator
**Test Tool**: k6
**Test Scenario**: `scale` (14 minute ramp-up)

---

## Infrastructure Configurations

### Configuration A - Initial Optimized (4000 VUs Mixed)

| Component | Replicas | Resources |
|-----------|----------|-----------|
| LLM (vLLM via LLM-D) | 48 | 1 GPU each |
| Guardrails Orchestrator | 50 | Scaled CPU |
| Chunker Service | 30+ | Scaled CPU |
| Application | 20 | - |
| Inference Gateway | 20 | - |
| HAP Detector | 10 | 1 GPU each |
| Prompt Injection Detector | 10 | - |
| Language Detector | 10 | - |

### Configuration B - Scaled Up (6000 VUs Safe)

| Component | Replicas | Resources |
|-----------|----------|-----------|
| LLM (vLLM via LLM-D) | 47 | 1 GPU each |
| Guardrails Orchestrator | 75 | Scaled CPU |
| Chunker Service | 75 | Scaled CPU |
| Application | 30 | - |
| Inference Gateway | 20 | - |
| HAP Detector | 10 | 1 GPU each |
| Prompt Injection Detector | 10 | - |
| Language Detector | 10 | - |

### Configuration C - More LLM (6000 VUs Safe)

| Component | Replicas | Resources |
|-----------|----------|-----------|
| LLM (vLLM via LLM-D) | 51 | 1 GPU each |
| Guardrails Orchestrator | 75 | Scaled CPU |
| Chunker Service | 75 | Scaled CPU |
| Application | 30 | - |
| Inference Gateway | 20 | - |
| HAP Detector | 10 | 1 GPU each |
| Prompt Injection Detector | 10 | - |
| Language Detector | 10 | - |

### vLLM Configuration
```
--max-model-len=512
--gpu-memory-utilization=0.90
--max-num-batched-tokens=8192
--max-num-seqs=160
--enable-chunked-prefill
```

---

## Test Results - Mixed Prompts (Safe + Unsafe)

Mixed prompts: ~40% safe, ~60% various unsafe (blocked at input or output)

| VUs | Throughput | Success Rate | Error Rate | p(95) TTFB | p(95) Total | Blocked Rate | Status |
|-----|------------|--------------|------------|------------|-------------|--------------|--------|
| 1000 | 143 req/s | 99.96% | 0.03% | 671ms | 6.11s | 54.18% | ✅ All pass |
| 2000 | 184 req/s | 99.99% | 0.00% | 2.06s | 16.97s | 54.13% | ✅ All pass |
| 4000 | 211 req/s | 99.98% | 0.01% | 8.35s | 29.66s | 54.24% | ✅ All pass |
| 6000 | 324 req/s | 98.73% | 1.26% | 10.6s | 32.45s | 58.37% | ✅ All pass |

### Observations - Mixed Prompts
- Excellent scaling up to 6000 VUs - all thresholds passed
- ~54-58% of requests blocked at input (guardrails working)
- Very low error rates (<1.5% even at 6000 VUs)
- High throughput because blocked requests return instantly
- **6000 VUs achieved 324 req/s** with only ~42% hitting LLM (~136 req/s effective LLM load)

---

## Test Results - Safe Prompts Only

Safe-only prompts: 100% go through to LLM for inference

| VUs | Config | LLM Replicas | Throughput | Success Rate | Error Rate | p(95) TTFB | p(95) Total | Blocked Rate | Status |
|-----|--------|--------------|------------|--------------|------------|------------|-------------|--------------|--------|
| 4000 | A | 48 | 102 req/s | 99.98% | 0.01% | 4.34s | 50.17s | 26.40% | ✅ All pass |
| 6000 | A | 48 | 164 req/s | 94.11% | 5.88% | 59.28s | 59.32s | 23.49% | ❌ TTFB failed |
| 6000 | B | 47 | 170 req/s | 92.78% | 7.21% | 59.44s | 59.47s | 23.22% | ❌ TTFB failed |
| 6000 | C | 51 | 172 req/s | 92.90% | 7.09% | 59.44s | 59.47s | 23.37% | ❌ TTFB failed |

### Observations - Safe Prompts
- Lower throughput because all requests require LLM inference
- ~23-26% blocked rate from output detection (LLM generates blocked content)
- 4000 VUs is the sweet spot for current LLM capacity (47-51 replicas)
- 6000 VUs overloads the system, hitting 60s timeout
- **Scaling orchestrator/chunker beyond LLM capacity doesn't help** - LLM is the bottleneck
- **Adding 4 more LLM pods (47→51) had minimal impact** - suggests 60s timeout is the real limiter, not LLM capacity
- The 60s timeout cuts off requests before they can complete, regardless of LLM capacity

---

## Performance Evolution (Scaling Journey)

### Initial State (Before Optimization)
| Metric | Value |
|--------|-------|
| Throughput | ~34 req/s |
| p(95) TTFB | 33.94s |
| Error Rate | ~10% |
| Status | ❌ Multiple thresholds failed |

### After Scaling Orchestrator + Detectors
| Metric | Value |
|--------|-------|
| Throughput | ~57 req/s |
| p(95) TTFB | 6.7s |
| Error Rate | ~3% |
| Status | ✅ All pass |

### After Full Scaling (Final - Mixed Prompts)
| Metric | Value |
|--------|-------|
| Throughput | 211 req/s (mixed) / 102 req/s (safe) |
| p(95) TTFB | 8.35s (mixed) / 4.34s (safe) |
| Error Rate | 0.01% |
| Status | ✅ All pass at 4000 VUs |

---

## Key Findings

### Bottlenecks Identified (in order)
1. **Chunker Service** - Initial bottleneck, resolved with more replicas + CPU
2. **Guardrails Orchestrator** - Second bottleneck, resolved with more replicas + CPU
3. **Detectors** - Minor bottleneck, resolved with more replicas
4. **LLM (vLLM)** - Final bottleneck for safe-only prompts at high VUs

### Performance Characteristics
- **Blocked requests are fast**: ~54% of mixed prompts blocked at input = instant response
- **Safe requests are slow**: Full LLM inference takes ~20s average
- **60s timeout exists**: Somewhere in the stack (not identified), affects ~5-7% at high load
- **LLM is the ceiling**: With 47-48 replicas, max ~4000 safe VUs before timeout issues

### Scaling Limits Discovered
| Component | Scaling Impact |
|-----------|----------------|
| Orchestrator 50→75 | No improvement (not bottleneck) |
| Chunker 30→75 | No improvement (not bottleneck) |
| LLM 48→47 | Slight degradation |
| LLM 47→51 | Minimal improvement (+1% throughput) - 60s timeout is the real limiter |

### Thresholds Used
```javascript
thresholds: {
    'http_req_duration': ['p(95)<60000'],      // 60s
    'sse_ttfb_ms': ['p(50)<5000', 'p(95)<15000'], // 5s median, 15s p95
    'sse_total_time_ms': ['p(95)<60000'],      // 60s
    'sse_success_rate': ['rate>0.80'],         // 80%
    'sse_error_rate': ['rate<0.20'],           // 20%
}
```

---

## Recommendations

### For Higher Safe-Prompt Throughput (Beyond 4000 VUs)
- **Find and increase the 60s timeout** - this is the primary limiter at 6000 VUs
  - Check: Orchestrator HTTP client, Gateway, Istio sidecar, vLLM server
- Adding more LLM replicas has diminishing returns until timeout is fixed
- Reduce `--max-model-len` for shorter responses (faster completion)
- Adding more orchestrator/chunker won't help (already scaled past bottleneck)

### For Higher Mixed-Prompt Throughput
- Current setup handles **6000 VUs** successfully (324 req/s)
- Could potentially push to 8000+ VUs since ~58% are blocked instantly
- Mixed workloads scale better because guardrails reduce effective LLM load

### Monitoring Metrics
- `kserve_vllm:num_requests_waiting` - vLLM queue depth (was 0, meaning LLM can accept requests but inference is slow)
- Orchestrator CPU/memory
- Chunker CPU/memory
- Detector response times

---

## Test Commands

```bash
# Mixed prompts (default)
./k6 run k6-load-test.js \
  --env BASE_URL=https://lemonade-stand-lemonade-stand.apps.cluster-bjggc.bjggc.sandbox3262.opentlc.com \
  --env SCENARIO=scale \
  --env VUS=4000

# Safe prompts only
./k6 run k6-load-test.js \
  --env BASE_URL=https://lemonade-stand-lemonade-stand.apps.cluster-bjggc.bjggc.sandbox3262.opentlc.com \
  --env SCENARIO=scale \
  --env VUS=4000 \
  --env SAFE_ONLY=true

# Safe prompts at 6000 VUs (exceeds LLM capacity)
./k6 run k6-load-test.js \
  --env BASE_URL=https://lemonade-stand-lemonade-stand.apps.cluster-bjggc.bjggc.sandbox3262.opentlc.com \
  --env SCENARIO=scale \
  --env VUS=6000 \
  --env SAFE_ONLY=true
```
