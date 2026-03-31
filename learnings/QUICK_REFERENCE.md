# Quick Reference Card - Cost & Latency Optimization

---

## Cost Savings Cheat Sheet

### 1. Prompt Caching (88% savings)
```
Without: 50 companies × $0.006 = $0.30
With: $0.006 + (49 × $0.0006) = $0.035
Savings: 88%
```

### 2. Fork Agents (90% savings)
```
Parent: $0.006
Children (50): 50 × $0.0006 = $0.03
Total: $0.036 vs $0.30 naive
Savings: 90%
```

### 3. Model Selection
```
Claude: $9/1M tokens (best quality)
GPT-4o: $6.25/1M tokens (balanced)
GPT-4o-mini: $0.38/1M tokens (cheap)
GLM self-hosted: $0/1M tokens (GPU cost)

Strategy: Use cheap for filtering, expensive for finals
Savings: 86% (two-stage: $1.30 vs $9.00)
```

### 4. LRU Caching (67% savings)
```
First read: $0.15 (cache miss)
Repeated reads: $0 (cache hit)
Savings: 67% over 3 reads
```

---

## ⚡ Latency Reduction Cheat Sheet

### 1. Parallel Execution (50x faster)
```
Sequential: 50 companies × 3min = 150min
Parallel: max(3min) = 3min
Speedup: 50x
```

### 2. Streaming (180x perceived)
```
Without: Wait 90s in silence
With: First response in 0.5s
Perceived speedup: 180x
```

### 3. Caching (∞ faster)
```
L1 (memory): 0ms - instant!
L2 (Redis): 5ms - very fast
L3 (DB): 50ms - fast
L4 (LLM): 2000ms - slow
```

### 4. Model Choice (4x faster)
```
GPT-4o-mini: 200ms (fastest)
GPT-4o: 400ms (balanced)
Gemini: 600ms (cheap)
Claude: 800ms (best quality)
Speedup: 4x (mini vs Claude)
```

---

## Decision Framework

### When to use which model?

```python
if decision_value / cost > 1000:
    use "claude"  # Quality matters ($100M decision)

elif decision_value / cost < 100:
    use "gpt-4o-mini"  # Cost matters (screening)

else:
    use "gpt-4o"  # Balanced
```

### Cost vs Quality Spectrum
```
Cheap ←──────────────────────→ Expensive
       GPT-mini   GPT-4o   Claude
Quality:  85%      92%      96%
Cost:   $0.38/1M  $6.25/1M $9/1M
Speed:   200ms    400ms    800ms

Use case:
  Screening  Production  High-stakes
```

---

## Real-World Examples

### Example 1: 50 Company Screen
```
Naive (all Claude):
  Cost: $9
  Time: 150 min

Optimized (two-stage):
  Filter (GPT-mini): $0.40, 10 min
  Deep-dive (Claude): $0.90, 9 min
  Total: $1.30, 19 min

Savings: 86% cost, 87% time
```

### Example 2: Live Dashboard
```
Naive:
  Every query → LLM call
  Latency: 2000ms avg

Optimized:
  L1 cache hits: 0ms (80% of queries)
  L2 cache hits: 5ms (15% of queries)
  LLM calls: 300ms (5% of queries, use GPT-mini)
  Average: 0.8×0 + 0.15×5 + 0.05×300 = 15.75ms

Speedup: 127x faster!
```

---

### "How do you optimize costs?"

**Key points:**
1. **Tiered approach**: Cheap models filter, expensive models finalize
2. **Caching**: LRU cache (25MB) for recent files
3. **Fork agents**: Inherit parent's cached prompt (90% savings)
4. **Smart model selection**: Task-dependent ($0.38/1M to $9/1M)

**Example:**
"For 1000 companies, I use three tiers: GPT-4o-mini filters to top 100 ($8), GPT-4o screens to top 20 ($12), Claude deep-dives on finals ($3.60). Total $23.60 vs $180 naive. That's 87% savings."

### "How do you reduce latency?"

**Key points:**
1. **Parallel execution**: 10 companies simultaneously (50x speedup)
2. **Streaming**: TTFB 0.5s vs 90s (180x perceived)
3. **Multi-tier caching**: L1 memory (0ms) → L2 Redis (5ms) → L3 LLM (300ms)
4. **Fast models**: GPT-4o-mini (200ms) for real-time

**Example:**
"For live dashboard, 80% queries hit L1 cache (instant), 15% hit L2 (5ms), only 5% call LLM with fast model (300ms). Average latency: 15ms vs 2000ms naive. That's 127x faster."

### "What are the trade-offs?"

**Key points:**
1. **Cost vs Quality**: Claude 96% quality at $9/1M vs GPT-mini 85% at $0.38/1M
2. **Speed vs Quality**: GPT-mini 200ms vs Claude 800ms (4x faster, 11% less accurate)
3. **Decision framework**: Use quality when decision value > 1000× cost

**Example:**
"For $100M investment decision, Claude's $0.18 cost is irrelevant - 96% quality worth it. For screening 1000 companies, GPT-mini's $8 total vs Claude's $180 makes sense - 85% quality good enough for filtering, catch false negatives in next stage."

---

## Key Numbers to Memorize

### Cost Metrics
- Prompt caching: **88% savings**
- Fork agents: **90% savings**
- Two-stage pipeline: **86% savings**
- LRU cache: **67% savings**

### Latency Metrics
- Parallel execution: **50x faster**
- Streaming (perceived): **180x faster**
- Cache hits: **∞ faster** (instant!)
- Model choice: **4x faster** (mini vs Claude)

### Model Pricing (per 1M tokens)
- Claude: **$9** (96% quality, 800ms)
- GPT-4o: **$6.25** (92% quality, 400ms)
- GPT-4o-mini: **$0.38** (85% quality, 200ms)
- GLM self-hosted: **$0** API (75% quality, 500ms + GPU cost)

### Trade-Off Formula
```
Quality Premium = Decision Value / Analysis Cost

> 1000: Use Claude (quality matters)
< 100: Use GPT-mini (cost matters)
100-1000: Use GPT-4o (balanced)
```

---

## Pre-Interview Checklist

**Can you explain:**
- [ ] How prompt caching works? (Byte-exact prompts, 90% savings)
- [ ] How fork agents save money? (Inherit parent's cached prompt)
- [ ] When to use which model? (Task-dependent: screening vs high-stakes)
- [ ] How to reduce latency? (Parallel + streaming + caching + fast models)
- [ ] What are the trade-offs? (Cost vs quality vs speed)
- [ ] Real example with numbers? (50 companies: $9 → $1.30, 150min → 19min)

**Can you code:**
- [ ] Parallel execution with asyncio? (See rag_implementation.py)
- [ ] LRU cache implementation? (See FileStateCache class)
- [ ] Multi-model wrapper? (See model_benchmark.py)
- [ ] Streaming responses? (async generators)

**Can you benchmark:**
- [ ] Latency comparison? (200ms mini vs 800ms Claude)
- [ ] Cost comparison? ($0.38/1M vs $9/1M)
- [ ] Quality metrics? (Accuracy, hallucination rate)
- [ ] Generate report? (See benchmark.generate_report())

---

## One-Minute Pitch

"I built a hedge fund RAG system optimized for cost and latency.

**Cost optimization:** Three-tier approach - GPT-4o-mini filters 1000 companies to 100 ($8), GPT-4o screens to top 20 ($12), Claude deep-dives on finals ($3.60). That's $23.60 vs $180 naive, **87% savings**.

**Latency optimization:** Parallel execution (10 simultaneous analyses), multi-tier caching (L1 memory 0ms, L2 Redis 5ms, L3 LLM 300ms with fast model), streaming for 0.5s TTFB. Average latency 15ms vs 2000ms naive, **127x faster**.

**Trade-offs:** I use decision value framework - Claude for $100M decisions (quality worth it), GPT-mini for screening (85% accuracy good enough), GPT-4o for production balance. This gives optimal quality-adjusted ROI.

**Implementation:** 450 lines of Python with LRU cache, knowledge graph, multi-model support, and benchmarking. All learned from Claude Code's production architecture."

