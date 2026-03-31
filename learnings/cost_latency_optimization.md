# Cost Savings & Latency Reduction - Deep Dive

## Why This Matters for Hedge Funds

**Scenario**: PM asks you to screen 50 biotech companies for FDA pipeline risk by EOD.

**Without optimization:**
- Cost: $500 (each company = $10)
- Time: 2.5 hours (50 × 3 min/company)
- **Result**: Too slow and expensive to be useful

**With optimization:**
- Cost: $25 (95% reduction!)
- Time: 15 minutes (97% reduction!)
- **Result**: Actionable insights before lunch

Let's understand HOW to achieve this.

---

## Part 1: Cost Savings Mechanisms

### 1.1 Prompt Caching (The Big Win!)

**How Claude API Caching Works:**

Claude caches the **beginning** of your prompt. If multiple requests share the same prefix, you only pay once.

**Example without caching:**
```
Request 1 (AAPL analysis):
  System: "You are a hedge fund analyst specializing in..." [2000 tokens]
  User: "Analyze AAPL" [5 tokens]
  Cost: 2005 tokens × $3/1M = $0.006015

Request 2 (MSFT analysis):
  System: "You are a hedge fund analyst specializing in..." [2000 tokens]
  User: "Analyze MSFT" [5 tokens]
  Cost: 2005 tokens × $3/1M = $0.006015

50 companies: 50 × $0.006 = $0.30
```

**With caching:**
```
Request 1 (AAPL):
  System: [2000 tokens] - CACHE MISS
  Cost: 2005 tokens × $3/1M = $0.006015

Request 2 (MSFT):
  System: [2000 tokens] - CACHE HIT! (only pay $0.30/1M for cached)
  New: [5 tokens]
  Cost: (2000 × $0.30/1M) + (5 × $3/1M) = $0.00060 + $0.000015 = $0.000615

50 companies: $0.006 + (49 × $0.0006) = $0.0354

Savings: $0.30 → $0.035 = 88% reduction!
```

**Key insight**: The more repetitive your prompts, the bigger the savings!

---

### 1.2 Fork Agents (Claude Code's Innovation)

**The Problem:**
When you spawn a new agent, you need to re-send the system prompt. But if each agent renders the prompt slightly differently (timestamps, random IDs, etc.), cache misses!

**Claude Code's Solution: Cache-Safe Parameters**

From `forkedAgent.ts`:
```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt,  // Byte-exact copy from parent
  userContext: { ... },
  systemContext: { ... },
  toolUseContext: { ... }
}
```

**How it works:**

```
┌─────────────────────────────────────────────┐
│ Parent Agent (Main)                         │
│                                             │
│ System Prompt: [Rendered ONCE]             │
│ "You are a hedge fund analyst..."           │
│ [2000 tokens]                               │
│                                             │
│ Cost: $0.006 (cache miss on first call)    │
└─────────────────────────────────────────────┘
           │
           ├── Fork Child 1 (AAPL)
           │   ↓
           │   Inherits: EXACT byte-for-byte prompt
           │   New: "Analyze AAPL 10-K"
           │   Cost: $0.0006 (cache hit!)
           │
           ├── Fork Child 2 (MSFT)
           │   ↓
           │   Inherits: EXACT same prompt
           │   New: "Analyze MSFT 10-K"
           │   Cost: $0.0006 (cache hit!)
           │
           └── Fork Child 3 (GOOGL)
               ↓
               Cost: $0.0006 (cache hit!)

Total: $0.006 + (3 × $0.0006) = $0.0078
```

**Why "byte-exact" matters:**

```python
# BAD: Each child re-renders prompt (cache miss!)
for company in companies:
    prompt = f"You are an analyst. Today is {datetime.now()}..."
    # ❌ Timestamp changes! Cache miss every time!

# GOOD: Parent renders once, children inherit
parent_prompt = render_once("You are an analyst...")
for company in companies:
    spawn_fork_child(
        inherited_prompt=parent_prompt,  # ✅ Byte-exact match!
        new_task=f"Analyze {company}"
    )
```

---

### 1.3 Model Selection (Right Tool for Right Job)

**Cost comparison (per 1M tokens):**

| Model | Input | Output | Avg* | Speed |
|-------|-------|--------|------|-------|
| Claude Sonnet 4.5 | $3.00 | $15.00 | $9.00 | 800ms |
| GPT-4o | $2.50 | $10.00 | $6.25 | 400ms |
| GPT-4o-mini | $0.15 | $0.60 | $0.38 | 200ms |
| Gemini 1.5 Pro | $1.25 | $5.00 | $3.13 | 600ms |
| GLM-4 (self-hosted) | $0 | $0 | $0** | 500ms |

*Avg = (Input + Output) / 2 for estimation
**GPU cost instead: AWS p3.2xlarge = $3.06/hour ≈ $0.001/second

**Strategic model selection:**

```python
class SmartRAG:
    def analyze_company(self, company, priority):
        if priority == "high_stakes":
            # Investment memo for PM
            model = "claude"  # Best quality, worth $9/1M

        elif priority == "screening":
            # Initial pass on 50 companies
            model = "gpt-4o-mini"  # 24x cheaper than Claude!

        elif priority == "bulk":
            # Screen 500+ companies daily
            model = "glm_selfhosted"  # $0 API cost

        return self.extract(company, model=model)
```

**Real example:**

```
Task: Screen 50 companies, deep-dive on top 5

Without optimization:
  All with Claude: 50 × $0.18 = $9.00

With optimization:
  Initial screen (GPT-4o-mini): 50 × $0.008 = $0.40
  Deep dive (Claude): 5 × $0.18 = $0.90
  Total: $1.30

Savings: $9.00 → $1.30 = 86% reduction!
Quality: Still excellent on the 5 that matter!
```

---

### 1.4 Token Efficiency (Compaction)

**The Problem: Context Window Bloat**

```
Turn 1: Read 10-K (50k tokens)
Turn 2: Extract revenue (60k tokens total)
Turn 3: Extract margins (70k tokens total)
...
Turn 50: Context full! (200k tokens)
```

**Cost impact:**
```
Without compaction:
  Turn 1: 50k tokens = $0.15
  Turn 2: 60k tokens = $0.18
  Turn 3: 70k tokens = $0.21
  ...
  Turn 50: 200k tokens = $0.60

  Total: ~$15.00 for one conversation!
```

**Claude Code's solution: Auto-Compaction**

From `autoCompact.ts:75-90`:
```typescript
const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

if (tokenUsage > autocompactThreshold) {
  // Summarize old messages aggressively
  compactedMessages = await summarizeOldContext(messages)
}
```

**How it works:**

```
Turn 1-30: Detailed messages (150k tokens)
   ↓
[Auto-compact triggered at 80% threshold]
   ↓
Summarized: "Previous analysis covered AAPL revenue trends..." (5k tokens)
   ↓
Turn 31-60: Fresh context space!
```

**Cost impact:**
```
With compaction:
  Turn 1-30: Build up to 160k tokens = $4.80
  [Compact to 10k tokens]
  Turn 31-60: Build up to 160k tokens = $4.80
  [Compact to 10k tokens]

  Total: ~$10.00 (33% savings)
  Plus: Can continue indefinitely!
```

---

### 1.5 LRU Cache (Avoid Re-reading Files)

**The Problem:**

```
Turn 5: "What's AAPL's revenue?"
  → Read 10-K (50k tokens = $0.15)

Turn 25: "Remind me of AAPL's revenue?"
  → Read 10-K AGAIN (50k tokens = $0.15)

Turn 45: "Compare AAPL revenue to earlier..."
  → Read 10-K AGAIN (50k tokens = $0.15)

Wasted: $0.30 on redundant reads!
```

**Solution: LRU Cache**

From `fileStateCache.ts:17-22`:
```typescript
export const READ_FILE_STATE_CACHE_SIZE = 100
const DEFAULT_MAX_CACHE_SIZE_BYTES = 25 * 1024 * 1024  // 25MB
```

**How it saves money:**

```python
class HedgeFundRAG:
    def __init__(self):
        self.cache = FileStateCache(100, 25)  # 100 files, 25MB

    def get_revenue(self, company):
        pdf_path = f"{company}_10k.pdf"

        # Check cache first (FREE!)
        if self.cache.has(pdf_path):
            cached = self.cache.get(pdf_path)
            if not cached.isPartialView:
                return cached.content  # $0 cost!

        # Cache miss: read from disk (costs tokens)
        content = read_pdf(pdf_path)  # $0.15
        self.cache.set(pdf_path, content)
        return content
```

**Cost impact:**

```
Without cache:
  Turn 5: Read AAPL 10-K = $0.15
  Turn 25: Read AAPL 10-K = $0.15
  Turn 45: Read AAPL 10-K = $0.15
  Total: $0.45

With cache:
  Turn 5: Read AAPL 10-K = $0.15 (cache miss)
  Turn 25: Get from cache = $0.00 (cache hit!)
  Turn 45: Get from cache = $0.00 (cache hit!)
  Total: $0.15

Savings: 67% on repeated reads!
```

---

## Part 2: Latency Reduction Techniques

### 2.1 Parallel Execution (The Biggest Win!)

**Sequential (slow):**
```python
results = []
for company in companies:
    result = analyze(company)  # 3 minutes each
    results.append(result)

# Total: 50 × 3 min = 150 minutes (2.5 hours!)
```

**Parallel (fast):**
```python
import asyncio

async def analyze_all(companies):
    tasks = [analyze_async(company) for company in companies]
    results = await asyncio.gather(*tasks)
    return results

# Total: max(3 min) = 3 minutes!
# Speedup: 50x faster!
```

**Why this works:**

```
Sequential:
[Company 1] → [Company 2] → [Company 3] → ...
   3 min        3 min        3 min

Parallel:
[Company 1]
[Company 2]  ← All running simultaneously
[Company 3]
[Company 4]
...
[Company 50]
   ↓
All finish in ~3 minutes (limited by slowest one)
```

**Real implementation:**

```python
class HedgeFundRAG:
    async def analyze_portfolio(self, tickers: List[str]):
        # Create tasks
        tasks = [
            self.analyze_10k_async(ticker)
            for ticker in tickers
        ]

        # Execute in parallel (up to 10 concurrent to avoid rate limits)
        results = []
        for i in range(0, len(tasks), 10):
            batch = tasks[i:i+10]
            batch_results = await asyncio.gather(*batch)
            results.extend(batch_results)

        return results

# Usage
rag = HedgeFundRAG()
results = await rag.analyze_portfolio([
    "AAPL", "MSFT", "GOOGL", ...  # 50 companies
])

# Time: ~15 minutes (5 batches × 3 min/batch)
# vs Sequential: 150 minutes
# Speedup: 10x faster!
```

---

### 2.2 Streaming (Perceived Latency)

**Without streaming (feels slow):**
```
User: "Analyze AAPL"
  [Wait 30 seconds...]
  [Wait 30 seconds...]
  [Wait 30 seconds...]
Assistant: "Here's the full analysis..." [90 seconds later]

Perceived latency: 90 seconds of silence!
```

**With streaming (feels fast):**
```
User: "Analyze AAPL"
  [0.5s] Assistant: "Analyzing AAPL 10-K..."
  [1.0s] Assistant: "Revenue for 2024 was..."
  [2.0s] Assistant: "Operating margins improved..."
  [3.0s] Assistant: "Key risks include..."
  ...

Perceived latency: 0.5 seconds to first response!
Actual latency: Still 90 seconds, but user sees progress
```

**Implementation:**

```python
async def analyze_with_streaming(self, pdf_path):
    # Start streaming immediately
    yield "Reading 10-K filing..."

    content = await self.read_pdf_async(pdf_path)
    yield f"Processing {len(content)} pages..."

    # Stream extraction results as they come
    async for chunk in self.extract_streaming(content):
        yield chunk

    yield "Analysis complete!"

# Usage
async for update in rag.analyze_with_streaming("aapl_10k.pdf"):
    print(update)  # User sees progress in real-time
```

**Latency impact:**

```
Without streaming:
  Time to first byte (TTFB): 90 seconds
  User experience: "Is this thing working?"

With streaming:
  TTFB: 0.5 seconds
  User experience: "Great, it's working!"

Perceived speedup: 180x faster!
```

---

### 2.3 Model Selection (Speed vs Quality)

**Latency benchmarks (for same task):**

| Model | Latency | Quality | Use Case |
|-------|---------|---------|----------|
| GPT-4o-mini | 200ms | 85% | Real-time screening |
| GPT-4o | 400ms | 92% | Production balance |
| Gemini 1.5 Pro | 600ms | 90% | Cost-sensitive |
| Claude Sonnet 4.5 | 800ms | 96% | High-stakes analysis |

**Strategic model choice:**

```python
class SmartRAG:
    def analyze(self, company, deadline_minutes):
        if deadline_minutes < 5:
            # Need results NOW
            model = "gpt-4o-mini"  # 200ms latency
            # Trade-off: 85% quality, but 4x faster

        elif deadline_minutes < 30:
            # Balanced deadline
            model = "gpt-4o"  # 400ms latency
            # Trade-off: 92% quality, good speed

        else:
            # Quality over speed
            model = "claude"  # 800ms latency
            # Trade-off: 96% quality, worth the wait

        return self.extract(company, model=model)
```

**Real example:**

```
Scenario: PM asks "Quick thoughts on AAPL?" during meeting

Without optimization:
  Use Claude (best quality)
  Wait: 90 seconds
  PM: "Never mind, meeting moved on..."

With optimization:
  Use GPT-4o-mini (fast)
  Wait: 15 seconds
  PM: "Great, thanks!"

Trade-off: 96% → 85% quality
Benefit: Actually useful! (speed matters)
```

---

### 2.4 Caching (Latency + Cost)

**Cache hierarchy (fastest to slowest):**

```
1. In-memory cache (LRU)     → 0ms    | $0
2. Redis (shared team cache) → 5ms    | ~$0.001
3. Database (PostgreSQL)     → 50ms   | ~$0.01
4. Re-compute with LLM       → 2000ms | $0.15
```

**Smart caching strategy:**

```python
class MultiTierCache:
    def get_revenue(self, ticker):
        # L1: In-memory (fastest)
        if ticker in self.memory_cache:
            return self.memory_cache[ticker]  # 0ms

        # L2: Redis (fast)
        if redis.exists(ticker):
            result = redis.get(ticker)  # 5ms
            self.memory_cache[ticker] = result
            return result

        # L3: Database (slow)
        result = db.query(ticker)  # 50ms
        if result:
            redis.set(ticker, result, expire=3600)
            self.memory_cache[ticker] = result
            return result

        # L4: Compute (slowest)
        result = self.analyze_with_llm(ticker)  # 2000ms
        db.save(ticker, result)
        redis.set(ticker, result, expire=3600)
        self.memory_cache[ticker] = result
        return result
```

**Latency impact:**

```
First request (AAPL):
  L1 miss → L2 miss → L3 miss → L4 compute
  Latency: 2000ms

Second request (AAPL, same session):
  L1 hit!
  Latency: 0ms
  Speedup: ∞ (instant!)

Third request (AAPL, new session):
  L1 miss → L2 hit
  Latency: 5ms
  Speedup: 400x faster!
```

---

## Part 3: Trade-Offs Framework

### 3.1 Cost vs Quality

**The Spectrum:**

```
Low Cost ←────────────────────→ High Cost
Low Quality                   High Quality

GLM          GPT-4o-mini    GPT-4o      Claude
(self-hosted)
$0 API       $0.38/1M       $6.25/1M    $9.00/1M
75% quality  85% quality    92% quality 96% quality
```

**Decision framework:**

```python
def choose_model(task_importance, budget_constraint):
    if task_importance == "critical":
        # Investment memo for $100M decision
        # Cost: $0.18 per analysis
        # Trade-off: Worth it! 96% quality matters
        return "claude"

    elif task_importance == "high":
        # Due diligence for potential investments
        # Cost: $0.12 per analysis
        # Trade-off: Good balance (92% quality)
        return "gpt-4o"

    elif task_importance == "screening":
        # Initial pass, just need basic numbers
        # Cost: $0.008 per analysis
        # Trade-off: 85% good enough for filtering
        return "gpt-4o-mini"

    elif budget_constraint == "very_tight":
        # Analyzing 10,000 companies
        # Cost: $0 API (just GPU)
        # Trade-off: 75% quality, but volume matters
        return "glm_selfhosted"
```

**Real example:**

```
Task: Screen 1000 biotech companies for FDA risk

Option 1 (All Claude):
  Cost: 1000 × $0.18 = $180
  Time: 1000 × 2 min = 2000 min (33 hours!)
  Quality: 96%

Option 2 (Two-stage):
  Stage 1 (GPT-4o-mini): 1000 × $0.008 = $8
    → Filter to top 100 (95% recall)
    Time: 1000 × 0.3 min = 300 min (5 hours)

  Stage 2 (Claude): 100 × $0.18 = $18
    → Deep dive on filtered list
    Time: 100 × 2 min = 200 min (3.3 hours)

  Total Cost: $26 (86% savings!)
  Total Time: 8.3 hours (75% faster)
  Quality: 96% on the 100 that matter!

Winner: Option 2 (smarter, not harder)
```

---

### 3.2 Speed vs Quality

**The Spectrum:**

```
Fast ←────────────────────→ Slow
Low Quality              High Quality

GPT-4o-mini  GPT-4o    Gemini   Claude
200ms        400ms     600ms    800ms
85% quality  92%       90%      96%
```

**Decision framework:**

```python
def choose_model_by_deadline(deadline_minutes, min_quality):
    options = [
        ("gpt-4o-mini", 200, 85),  # (model, latency_ms, quality%)
        ("gpt-4o", 400, 92),
        ("gemini", 600, 90),
        ("claude", 800, 96),
    ]

    # Filter by quality requirement
    viable = [o for o in options if o[2] >= min_quality]

    # Choose fastest that meets quality bar
    return min(viable, key=lambda x: x[1])[0]

# Examples:
choose_model_by_deadline(deadline=5, min_quality=80)
# → gpt-4o-mini (fast enough, meets quality)

choose_model_by_deadline(deadline=60, min_quality=95)
# → claude (have time, need quality)
```

**Real example:**

```
Scenario: Live earnings call, PM wants quick take

Without optimization:
  Use Claude (96% quality)
  Wait: 60 seconds for analysis
  PM: "Too late, call already moved on"

With optimization:
  Use GPT-4o-mini (85% quality)
  Wait: 15 seconds
  PM: "Good enough, let's discuss"

Trade-off: 96% → 85% quality
Benefit: Actually useful in real-time!
```

---

### 3.3 Cost vs Speed (The Big Trade-Off)

**Four quadrants:**

```
        Cheap
          │
          │   Q1: Cheap & Fast          Q2: Expensive & Fast
          │   (Best for volume)         (Best for urgent + quality)
          │   • GPT-4o-mini             • GPT-4o + parallel
          │   • Parallel execution      • Streaming
Fast ─────┼──────────────────────────────────────────── Slow
          │   Q3: Cheap & Slow          Q4: Expensive & Slow
          │   (Best for batch)          (Worst - avoid!)
          │   • GLM self-hosted         • Claude sequential
          │   • Overnight jobs
          │
      Expensive
```

**Strategic choices:**

```python
class StrategicRAG:
    def analyze_portfolio(self, companies, deadline_hours, budget_usd):
        if deadline_hours < 1 and budget_usd > 50:
            # Q2: Urgent + have budget
            return self.fast_expensive(companies)
            # Strategy: GPT-4o + parallel + streaming
            # Cost: $50
            # Time: 15 min

        elif deadline_hours < 1 and budget_usd < 50:
            # Q1: Urgent + tight budget
            return self.fast_cheap(companies)
            # Strategy: GPT-4o-mini + parallel
            # Cost: $10
            # Time: 20 min
            # Trade-off: Lower quality

        elif deadline_hours > 24 and budget_usd < 50:
            # Q3: Batch job + tight budget
            return self.slow_cheap(companies)
            # Strategy: GLM self-hosted + overnight
            # Cost: $5 (GPU cost)
            # Time: 24 hours
            # Trade-off: Slow but cheap

        else:
            # Q4: Avoid this quadrant!
            # If slow, should be cheap
            # If expensive, should be fast
            raise ValueError("Bad trade-off!")
```

---

## Part 4: Real-World Case Studies

### Case Study 1: Biotech FDA Screening

**Requirements:**
- Analyze 50 biotech companies
- Deadline: 4 hours
- Budget: $50
- Quality: Need 90%+ accuracy

**Optimization strategy:**

```python
# Stage 1: Quick filter (GPT-4o-mini)
candidates = await parallel_analyze(
    companies=all_50,
    model="gpt-4o-mini",
    batch_size=10
)
# Cost: 50 × $0.008 = $0.40
# Time: 5 batches × 2 min = 10 min
# Output: 15 high-potential companies

# Stage 2: Deep dive (Claude)
final_analysis = await parallel_analyze(
    companies=top_15,
    model="claude",
    batch_size=5
)
# Cost: 15 × $0.18 = $2.70
# Time: 3 batches × 3 min = 9 min

# Stage 3: Synthesis (GPT-4o)
report = await synthesize(
    analyses=final_analysis,
    model="gpt-4o"
)
# Cost: $0.50
# Time: 5 min

# Total: $3.60, 24 minutes
# Under budget: ✓ ($50)
# Under deadline: ✓ (4 hours)
# Quality: 96% on top 15 (what matters)
```

---

### Case Study 2: Real-Time Earnings Monitoring

**Requirements:**
- Monitor 20 companies during earnings season
- Deadline: < 5 minutes per company
- Budget: $100/day
- Quality: Good enough for initial take

**Optimization strategy:**

```python
class EarningsMonitor:
    def __init__(self):
        self.cache = FileStateCache(100, 25)

    async def quick_analysis(self, company):
        # Check cache first (instant!)
        cached = self.cache.get(f"{company}_earning")
        if cached and self.is_recent(cached, hours=4):
            return cached  # 0ms, $0

        # Use fast model (GPT-4o-mini)
        result = await analyze_streaming(
            company=company,
            model="gpt-4o-mini"  # 200ms latency
        )

        self.cache.set(f"{company}_earning", result)
        return result

# Cost per company: $0.008
# Time per company: 15 seconds
# Quality: 85% (good enough for screening)
# Daily cost: 20 × $0.008 × 4 (4 times/day) = $0.64

# Well under budget! Can even upgrade to GPT-4o:
# Daily cost: 20 × $0.12 × 4 = $9.60 (still under $100)
```

---

## Part 5: Interview Talking Points

### Question: "How would you optimize costs for analyzing 1000 10-Ks?"

**Good answer:**

"I'd use a three-tier strategy:

**Tier 1 - Cheap Filter (GPT-4o-mini)**
- Analyze all 1000 companies
- Cost: 1000 × $0.008 = $8
- Time: ~2 hours (parallel batches)
- Goal: Filter to top 100 (10%)

**Tier 2 - Quality Screen (GPT-4o)**
- Deep dive on 100 candidates
- Cost: 100 × $0.12 = $12
- Time: ~30 minutes
- Goal: Filter to top 20 (2%)

**Tier 3 - Final Analysis (Claude)**
- Investment-grade analysis on 20 finalists
- Cost: 20 × $0.18 = $3.60
- Time: ~10 minutes
- Goal: Final recommendations

**Total: $23.60, 2.5 hours**
vs naive Claude-only: $180, 33 hours

**That's 87% cost savings and 13x faster!**

The key insight: Use cheaper models for filtering, expensive models only where quality matters."

---

### Question: "How would you reduce latency for a live dashboard?"

**Good answer:**

"I'd implement a five-layer strategy:

**1. Aggressive caching (L1: Memory, L2: Redis)**
- Most queries hit cache (< 5ms)
- Only cache misses call LLM

**2. Parallel execution**
- Analyze 10 companies simultaneously
- Time = max(individual) not sum(all)

**3. Streaming responses**
- User sees first results in 0.5s
- Perceived latency <<< actual latency

**4. Smart model selection**
- Real-time queries: GPT-4o-mini (200ms)
- Background jobs: Claude (800ms, better quality)

**5. Pre-computation**
- Analyze all S&P 500 overnight
- Dashboard queries = cache hits (instant!)

**Result:**
- P50 latency: 5ms (cache hit)
- P95 latency: 300ms (cache miss + GPT-4o-mini)
- P99 latency: 1000ms (cache miss + Claude)

Much better than naive 2000ms average!"

---

### Question: "What's the trade-off between cost and quality?"

**Good answer:**

"It's not binary - it's a spectrum with task-dependent optima:

**For $100M investment decision:**
- Use Claude ($0.18/analysis)
- Cost is irrelevant compared to decision value
- 96% quality worth 10x cost vs cheaper model

**For screening 1000 companies:**
- Use GPT-4o-mini ($0.008/analysis)
- Total cost $8 vs $180 for Claude
- 85% quality good enough for filtering
- False negatives caught in next stage

**The framework:**
```
Decision Value / Analysis Cost = Quality Premium Worth
```

**Examples:**
- $100M decision / $0.18 = $555M per $1 → Quality matters!
- $0 decision / $0.18 = $0 per $1 → Cost matters!

**My rule:**
- Decision value > 1000× cost: Use best model (Claude)
- Decision value < 100× cost: Use cheap model (GPT-4o-mini)
- In between: Use balanced model (GPT-4o)

This gives optimal quality-adjusted ROI."

---

## Summary: Key Numbers to Remember

### Cost Metrics
- **Prompt caching**: 88% savings on repetitive prompts
- **Fork agents**: 90% savings on parallel tasks
- **Two-stage pipeline**: 86% savings on screening tasks
- **LRU cache**: 67% savings on repeated reads

### Latency Metrics
- **Parallel execution**: 50x faster (50 companies: 150min → 3min)
- **Streaming**: 180x perceived speedup (TTFB: 90s → 0.5s)
- **Cache hits**: ∞ faster (2000ms → 0ms)
- **Model choice**: 4x faster (Claude 800ms → GPT-4o-mini 200ms)

### Trade-Off Framework
```
High Stakes + Have Budget → Claude (96% quality, $9/1M)
Production Balance → GPT-4o (92% quality, $6.25/1M)
High Volume + Tight Budget → GPT-4o-mini (85% quality, $0.38/1M)
Very High Volume → GLM self-hosted (75% quality, $0 API)
```

### Decision Formula
```
If (decision_value / cost) > 1000: Use best quality
If (decision_value / cost) < 100: Use lowest cost
Else: Use balanced option
```

