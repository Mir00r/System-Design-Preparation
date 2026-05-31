# 🏗️ ML Pipeline Architecture — From Data to Production 🚀

> **"A model sitting in a notebook is worth nothing. A model in production serving millions of requests — that's engineering!"** — Every ML engineer at FAANG

```
┌─────────────────────────────────────────────────────────────────┐
│  🎮 LEVEL 8.1: ML PIPELINE ARCHITECTURE                         │
│                                                                 │
│  XP Reward: +350 🌟                                             │
│  Badge: 🏗️ Pipeline Architect                                   │
│  Time: ~55 minutes                                              │
│  Industry Relevance: ⭐⭐⭐⭐⭐ (System Design interview GOLD!)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📑 Table of Contents

1. [What is an ML Pipeline?](#-what-is-an-ml-pipeline)
2. [End-to-End Architecture](#-end-to-end-architecture)
3. [Data Pipeline (Ingestion → Features)](#-data-pipeline)
4. [Training Pipeline](#-training-pipeline)
5. [Serving Pipeline (Inference)](#-serving-pipeline)
6. [Monitoring & Feedback Loop](#-monitoring--feedback-loop)
7. [LLM-Specific Pipelines](#-llm-specific-pipelines)
8. [Java Implementation](#-java-implementation)
9. [Real-World Case Studies](#-real-world-case-studies)
10. [Interview Questions & Puzzles](#-interview-questions--puzzles)

---

## 🤔 What is an ML Pipeline?

### The Big Picture

```
ML Pipeline = The ENTIRE system that turns raw data into 
              predictions served to users at scale!

It's NOT just the model! The model is maybe 5% of the code! 😱

Google's famous paper (2015):

   ┌─────────────────────────────────────────────────────┐
   │                                                     │
   │  ┌─────────────────────────────────────────────┐   │
   │  │           Configuration                      │   │
   │  ├──────┬──────┬──────────┬──────┬─────────────┤   │
   │  │Data  │Data  │ Feature  │ ML   │ Monitoring  │   │
   │  │Collec│Verif │ Extract  │ Code │ & Testing   │   │
   │  │tion  │      │          │  🔥  │             │   │
   │  ├──────┴──────┴──────────┴──────┴─────────────┤   │
   │  │  Serving Infrastructure  │  Storage          │   │
   │  └─────────────────────────────────────────────┘   │
   │                                                     │
   └─────────────────────────────────────────────────────┘
   
   The tiny "ML Code" box in the middle? That's the actual model!
   Everything else is INFRASTRUCTURE! 🏗️
```

### Why Pipelines Matter (Not Just Notebooks!)

```
DATA SCIENTIST approach:
  Jupyter Notebook → train model → "it works!" → throws over wall 🧱
  
ML ENGINEER approach:
  Design pipeline → automate everything → deploy → monitor → retrain
  
The difference:
  ❌ Notebook: Works on my machine! (sample data, manual, no monitoring)
  ✅ Pipeline: Works in production! (real data, automated, observable)

"Everyone wants to do the model training part.
 Nobody wants to do the data plumbing part.
 But the plumbing is where the value is!" — Industry truth 💰
```

---

## 🔄 End-to-End Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ML SYSTEM ARCHITECTURE                             │
│                                                                     │
│  ┌────────────┐    ┌──────────────┐    ┌────────────────────────┐  │
│  │ DATA       │    │  FEATURE     │    │  MODEL TRAINING         │  │
│  │ SOURCES    │───→│  STORE       │───→│  PIPELINE               │  │
│  │            │    │              │    │                          │  │
│  │ • Events   │    │ • Offline    │    │ • Data validation       │  │
│  │ • DB dumps │    │   features   │    │ • Train/val/test split  │  │
│  │ • APIs     │    │ • Online     │    │ • Model training        │  │
│  │ • Streams  │    │   features   │    │ • Hyperparameter tuning │  │
│  └────────────┘    └──────────────┘    │ • Model evaluation      │  │
│                                         │ • Model registry        │  │
│                                         └───────────┬────────────┘  │
│                                                     │               │
│  ┌────────────────────────────────────────────────┐ │               │
│  │ SERVING (INFERENCE)                             │ │               │
│  │                              ┌─────────────────┘ │               │
│  │  ┌──────────┐    ┌─────────▼──┐    ┌────────────┐              │
│  │  │ Request  │───→│   Model    │───→│  Response   │              │
│  │  │          │    │  Server    │    │  (+ cache)  │              │
│  │  └──────────┘    └────────────┘    └────────────┘              │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ MONITORING & FEEDBACK                                           │ │
│  │  • Prediction quality (drift detection!)                       │ │
│  │  • Latency & throughput                                        │ │
│  │  • Data quality                                                │ │
│  │  • Trigger retraining when quality drops!                      │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Data Pipeline

### Feature Store — The Missing Piece Most People Skip!

```
What: A centralized store for ML features (processed data!)
Why: 
  - Consistency between training and serving (critical!)
  - Reuse features across models (team productivity!)
  - Point-in-time correctness (avoid data leakage!)

OFFLINE Features (batch):
  Computed periodically (daily, hourly)
  Example: user_avg_order_value_30d, product_popularity_score
  Storage: Data warehouse (BigQuery, Redshift)
  
ONLINE Features (real-time):
  Computed per request at serving time
  Example: user_session_duration, items_in_cart_now
  Storage: Low-latency store (Redis, DynamoDB)

Feature Store architecture:
  ┌──────────────────────────────────────────────┐
  │              FEATURE STORE                     │
  │                                               │
  │  Batch Pipeline ──→ OFFLINE STORE (warehouse) │
  │                     (historical features)     │
  │                            ↕                  │
  │  Stream Pipeline ──→ ONLINE STORE (Redis)     │
  │                     (real-time features)      │
  │                                               │
  │  Training: reads from OFFLINE store           │
  │  Serving: reads from ONLINE store             │
  │  SAME features, SAME logic, different speed!  │
  └──────────────────────────────────────────────┘
```

### Data Validation — Catch Problems BEFORE Training!

```
Common data issues (that waste days of training!):

1. Schema changes: "Who added this new column?!" 
2. Missing values: "Why is 30% of age null today?!"
3. Distribution shift: "Average order value jumped 10x!"
4. Staleness: "This data is 3 days old, not real-time!"
5. Duplicates: "Same event logged 4 times!"

Validation checks:
  ✅ Schema validation (types, required fields)
  ✅ Statistical tests (distribution shift detection)
  ✅ Completeness checks (null %, row counts)
  ✅ Freshness checks (data timestamp vs now)
  ✅ Cross-feature consistency (age > 0, email format)
```

---

## 🏋️ Training Pipeline

### The Training Loop (Production Edition)

```
┌─────────────────────────────────────────────────────────────┐
│                 TRAINING PIPELINE                             │
│                                                             │
│  1. DATA PREP                                               │
│     ├── Load from feature store                             │
│     ├── Validate data quality                               │
│     ├── Split: train(70%) / val(15%) / test(15%)            │
│     └── Handle class imbalance (oversample/undersample)     │
│                                                             │
│  2. TRAINING                                                │
│     ├── Load previous best model (warm start)               │
│     ├── Train with early stopping                           │
│     ├── Log metrics (MLflow, W&B)                           │
│     └── Save checkpoints                                    │
│                                                             │
│  3. EVALUATION                                              │
│     ├── Evaluate on held-out test set                       │
│     ├── Compare vs current production model                 │
│     ├── Check for bias/fairness                             │
│     └── Generate model card (documentation)                 │
│                                                             │
│  4. REGISTRY                                                │
│     ├── If better → register new model version              │
│     ├── Add metadata (metrics, data version, params)        │
│     ├── Stage: dev → staging → production                   │
│     └── Trigger deployment pipeline                         │
└─────────────────────────────────────────────────────────────┘
```

### Model Registry — Version Control for Models!

```
Like Git, but for ML models:

my-fraud-detector/
├── v1.0 (2024-01-15) — Logistic Regression, AUC: 0.82
├── v1.1 (2024-02-20) — Random Forest, AUC: 0.89
├── v2.0 (2024-04-10) — XGBoost, AUC: 0.93  ← PRODUCTION
└── v2.1 (2024-05-01) — Neural Net, AUC: 0.94 ← STAGING (testing!)

Each version stores:
  - Model artifact (the actual weights/file)
  - Training data version (which data was used)
  - Hyperparameters (how it was trained)
  - Evaluation metrics (how good it is)
  - Code version (git SHA)
  - Environment (Python version, library versions)

Enables: Rollback, A/B testing, audit trail! 🎯
```

---

## ⚡ Serving Pipeline

### Serving Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│  BATCH SERVING (Offline)                                         │
│  ├── Run predictions on entire dataset periodically             │
│  ├── Store results in database/cache                            │
│  ├── Serve pre-computed results                                 │
│  ├── Latency: N/A (pre-computed!)                               │
│  └── Use case: Recommendation emails, risk scores              │
├─────────────────────────────────────────────────────────────────┤
│  REAL-TIME SERVING (Online)                                      │
│  ├── Compute predictions on-demand per request                  │
│  ├── Model loaded in memory, ready to serve                     │
│  ├── Latency: <100ms (or users leave!)                          │
│  └── Use case: Fraud detection, search ranking, chatbots        │
├─────────────────────────────────────────────────────────────────┤
│  STREAMING SERVING (Near real-time)                              │
│  ├── Consume event stream, predict per event                    │
│  ├── Results published to another stream                        │
│  ├── Latency: 100ms-5s                                          │
│  └── Use case: IoT anomaly detection, real-time bidding         │
└─────────────────────────────────────────────────────────────────┘
```

### Scaling Model Serving

```
Challenge: Model inference can be EXPENSIVE (especially LLMs!)

Solutions:
                                                
1. CACHING (Simple but effective!)
   Same input → same output. Cache it!
   Hit rate for recommendation: ~60%! That's 60% less compute!

2. BATCHING (GPU loves batches!)
   Collect N requests → process together → distribute results
   GPU utilization: 10% (single) → 80% (batched)!
   
3. MODEL OPTIMIZATION
   ├── Quantization (FP32 → INT8): 4x smaller, 2-3x faster!
   ├── Distillation (big model → small model): 10x faster!
   ├── Pruning (remove useless weights): 2-5x smaller!
   └── ONNX Runtime (cross-platform optimization)

4. AUTOSCALING
   ├── Scale replicas based on request queue depth
   ├── Scale down during low traffic (save $$$!)
   └── Use spot/preemptible instances for cost savings
```

---

## 📊 Monitoring & Feedback Loop

### What to Monitor (Production ML is DIFFERENT!)

```
Traditional software: "Is it up? Is it fast?"
ML systems: "Is it up? Is it fast? IS IT STILL CORRECT?!" 😱

MODEL PERFORMANCE MONITORING:
├── Prediction quality (accuracy, latency percentiles)
├── Data drift (input distribution changed!)
├── Concept drift (world changed, model is stale!)
├── Feature drift (individual feature distributions)
└── Feedback signals (user clicked? purchased? complained?)

WHEN TO RETRAIN:
├── Scheduled (weekly, monthly)
├── Triggered (quality drops below threshold!)
├── Continuous (online learning, always updating)
└── Manual (team decides based on monitoring dashboard)
```

### The Feedback Loop

```
                    ┌─────────────────┐
                    │  MONITORING     │
                    │  Dashboard      │
                    └────────┬────────┘
                             │ Alert: "Accuracy dropped 5%!"
                    ┌────────▼────────┐
                    │  DIAGNOSIS      │
                    │  "Why did it    │
                    │   degrade?"     │
                    └────────┬────────┘
                             │ Found: "New product category 
                             │         model never saw!"
                    ┌────────▼────────┐
                    │  RETRAIN        │
                    │  with new data  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  DEPLOY         │
                    │  new model      │──→ Back to monitoring! 🔄
                    └─────────────────┘
```

---

## 🤖 LLM-Specific Pipelines

### LLM Pipeline vs Traditional ML Pipeline

```
Traditional ML Pipeline:          LLM Pipeline:
├── Collect training data          ├── Choose base model (GPT-4, Claude)
├── Feature engineering            ├── Design prompts (prompt engineering!)
├── Train model from scratch       ├── Build RAG pipeline (retrieval!)
├── Evaluate on test set           ├── Fine-tune (optional, expensive!)
├── Deploy model                   ├── Deploy with guardrails
└── Monitor predictions            └── Monitor quality + costs!

Key differences:
  - LLMs: Usually DON'T train from scratch (use API!)
  - LLMs: Prompt = your "feature engineering"
  - LLMs: RAG = your "training data at inference time"
  - LLMs: Cost is per-token (scales with usage!)
  - LLMs: Output is non-deterministic (harder to test!)
```

### LLM Serving Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  LLM APPLICATION PIPELINE                                        │
│                                                                  │
│  User Query                                                      │
│      │                                                           │
│      ▼                                                           │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │ Guardrail│→ │ RAG Retrieval│→ │ Prompt Assembly            │ │
│  │ (safety) │  │ (vector DB)  │  │ (system + context + query) │ │
│  └──────────┘  └──────────────┘  └─────────────┬─────────────┘ │
│                                                 │                │
│                                    ┌────────────▼────────────┐  │
│                                    │  LLM API Call            │  │
│                                    │  (with retry + fallback) │  │
│                                    └────────────┬────────────┘  │
│                                                 │                │
│  ┌───────────────┐  ┌──────────────┐  ┌───────▼─────────────┐ │
│  │ Response Cache │← │ Output Guard │← │ Response Parsing     │ │
│  │ (save $$$!)   │  │ (hallucination│  │ (structured output) │ │
│  └───────────────┘  │  detection)   │  └─────────────────────┘ │
│                      └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## ☕ Java Implementation

```java
/**
 * Production ML Pipeline components in Java/Spring Boot
 * Feature Store + Model Serving + Monitoring
 */

// ─── FEATURE STORE SERVICE ─────────────────────────────────────

@Service
public class FeatureStoreService {
    
    private final RedisTemplate<String, Map<String, Object>> onlineStore;
    private final BigQueryClient offlineStore;
    
    /**
     * Get features for real-time serving (low latency!)
     */
    public Map<String, Object> getOnlineFeatures(String entityId, List<String> featureNames) {
        String key = "features:" + entityId;
        Map<String, Object> allFeatures = onlineStore.opsForValue().get(key);
        
        if (allFeatures == null) {
            // Cache miss — compute on the fly or fallback
            return computeFeaturesRealtime(entityId, featureNames);
        }
        
        // Return only requested features
        return featureNames.stream()
            .filter(allFeatures::containsKey)
            .collect(Collectors.toMap(f -> f, allFeatures::get));
    }
    
    /**
     * Batch compute and store features (runs nightly)
     */
    @Scheduled(cron = "0 0 2 * * *") // 2 AM daily
    public void refreshOfflineFeatures() {
        String query = """
            SELECT user_id,
                   AVG(order_total) as avg_order_30d,
                   COUNT(*) as order_count_30d,
                   MAX(order_date) as last_order_date
            FROM orders
            WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
            GROUP BY user_id
            """;
        
        List<UserFeatures> features = offlineStore.query(query);
        
        // Push to online store for real-time access
        features.forEach(f -> onlineStore.opsForValue().set(
            "features:" + f.getUserId(), f.toMap(),
            Duration.ofHours(25) // TTL slightly > refresh interval
        ));
    }
}

// ─── MODEL SERVING SERVICE ─────────────────────────────────────

@Service
public class ModelServingService {
    
    private final FeatureStoreService featureStore;
    private final ChatModel llmModel;
    private final CircuitBreaker circuitBreaker;
    private final MeterRegistry metrics;
    private final Cache<String, PredictionResult> cache;
    
    public PredictionResult predict(PredictionRequest request) {
        Timer.Sample timer = Timer.start(metrics);
        
        try {
            // 1. Check cache
            String cacheKey = request.computeCacheKey();
            PredictionResult cached = cache.getIfPresent(cacheKey);
            if (cached != null) {
                metrics.counter("ml.predictions.cache_hit").increment();
                return cached;
            }
            
            // 2. Fetch features
            Map<String, Object> features = featureStore.getOnlineFeatures(
                request.getEntityId(), 
                request.getRequiredFeatures()
            );
            
            // 3. Call model with circuit breaker
            PredictionResult result = circuitBreaker.run(() -> {
                return callModel(features, request);
            }, throwable -> {
                metrics.counter("ml.predictions.fallback").increment();
                return getFallbackPrediction(request);
            });
            
            // 4. Cache and return
            cache.put(cacheKey, result);
            
            // 5. Log for monitoring
            logPrediction(request, result);
            
            return result;
            
        } finally {
            timer.stop(metrics.timer("ml.predictions.latency"));
        }
    }
    
    private void logPrediction(PredictionRequest request, PredictionResult result) {
        // Async log for monitoring pipeline
        metrics.counter("ml.predictions.total").increment();
        metrics.gauge("ml.predictions.confidence", result.getConfidence());
        
        // Log to monitoring system for drift detection
        monitoringService.logAsync(new PredictionLog(
            request.getEntityId(),
            request.getFeatures(),
            result.getPrediction(),
            result.getConfidence(),
            Instant.now()
        ));
    }
}

// ─── DRIFT DETECTION SERVICE ───────────────────────────────────

@Service
public class DriftDetectionService {
    
    /**
     * Check if input data distribution has shifted
     * Uses Population Stability Index (PSI)
     */
    @Scheduled(fixedRate = 3600000) // Every hour
    public void checkDataDrift() {
        // Get current hour's feature distributions
        Map<String, Distribution> current = getRecentDistributions(Duration.ofHours(1));
        
        // Compare to baseline (training data distribution)
        Map<String, Distribution> baseline = getBaselineDistributions();
        
        for (String feature : current.keySet()) {
            double psi = calculatePSI(baseline.get(feature), current.get(feature));
            
            metrics.gauge("ml.drift.psi." + feature, psi);
            
            if (psi > 0.2) { // Significant drift!
                alertService.sendAlert(
                    "DATA DRIFT DETECTED: Feature '%s' PSI=%.3f (threshold: 0.2). "
                    + "Consider retraining!".formatted(feature, psi)
                );
            }
        }
    }
    
    private double calculatePSI(Distribution baseline, Distribution current) {
        // Population Stability Index
        double psi = 0;
        for (int i = 0; i < baseline.getBins().length; i++) {
            double expected = baseline.getBins()[i] + 0.0001; // avoid log(0)
            double actual = current.getBins()[i] + 0.0001;
            psi += (actual - expected) * Math.log(actual / expected);
        }
        return psi;
    }
}
```

---

## 🏢 Real-World Case Studies

### 🔷 Uber — ML Platform (Michelangelo)

```
Scale: 
  - 1000s of models in production
  - Millions of predictions per second
  - 100s of ML engineers

Architecture:
  1. Feature Store: Offline (Hive) + Online (Cassandra)
  2. Training: Spark + Horovod (distributed training)
  3. Serving: Custom serving layer, <10ms latency
  4. Monitoring: Automatic drift detection + alerting

Key lesson: "The platform matters more than any individual model!"
```

### 🔷 Netflix — Recommendation Pipeline

```
What you see on Netflix homepage = ML pipeline output!

Pipeline:
  1. Data: 200M members × viewing history × time × device
  2. Features: 1000+ features (recently watched, time of day, etc.)
  3. Models: Ensemble of 10+ models (candidate generation + ranking)
  4. Serving: Pre-compute top-N for each user (batch) + 
             real-time personalization (online)
  5. A/B testing: Every change is A/B tested!

Scale: Generates $1B+/year in value (via retention!)
```

---

## 🎯 Interview Questions & Puzzles

### Question 1: "Design an ML pipeline for fraud detection at a bank."

**Great Answer**: "I'd design it with these components: (1) **Data Pipeline**: Stream transaction events via Kafka, compute real-time features (transaction velocity, amount vs average, geo anomalies) and batch features (historical patterns, account age). Store in a feature store with both online (Redis) and offline (BigQuery) components. (2) **Training**: Daily batch training on labeled fraud data with significant class imbalance handling (SMOTE + class weights). Use an ensemble (XGBoost + neural net). Register in model registry with A/B comparison. (3) **Serving**: Real-time scoring at <50ms (hard requirement for payment authorization!). Use model ensemble with pre-computed batch features + real-time stream features. Cache for same-card repeated checks. (4) **Monitoring**: Track precision/recall daily, alert on drift. Special attention to false positive rate (blocking legitimate customers is expensive!). Feedback loop from fraud investigation team labels."

### Question 2: "What's the difference between data drift and concept drift?"

**Great Answer**: "Data drift (covariate shift): The input distribution changes, but the relationship between inputs and outputs stays the same. Example: Your fraud model was trained on US transactions, now you're seeing European transactions — different patterns, same underlying fraud indicators. Fix: retrain on new data distribution. Concept drift: The relationship between inputs and outputs changes. Example: During COVID, legitimate customers started buying differently (more online, unusual merchants) — what LOOKS like fraud isn't. The 'concept' of fraud changed! Fix: relabel data and retrain. Data drift is easier to detect (just monitor feature distributions). Concept drift requires monitoring actual outcomes (did flagged transactions turn out to be fraud?)."

---

### 🧩 Puzzle: Pipeline Design Challenge

```
Scenario: You're building an AI-powered customer support system 
that auto-responds to 60% of tickets.

Design the pipeline! Consider:
- What's the training data?
- How do you serve?
- What do you monitor?
- When do you retrain?

Answer:
  TRAINING:
  - Data: Historical tickets + human agent responses (labeled)
  - Features: Ticket text embedding, customer tier, product, sentiment
  - Model: Fine-tuned LLM or RAG over knowledge base
  - Eval: Human eval + automated metrics (BLEU, user satisfaction)
  
  SERVING:
  - Incoming ticket → classify (can auto-respond?)
  - If confidence > 0.85 → auto-respond + mark for review
  - If confidence < 0.85 → route to human agent
  - Response time target: <5 seconds
  
  MONITORING:
  - Customer satisfaction after auto-response (thumbs up/down)
  - Escalation rate (auto → human = failure!)
  - Response quality (human review sample)
  - Topic drift (new issues the model hasn't seen)
  
  RETRAINING TRIGGERS:
  - Satisfaction drops below 4.0/5.0
  - Escalation rate exceeds 15%
  - New product launch (new ticket types!)
  - Weekly with latest human-approved responses

Key insight: The 0.85 confidence threshold is your safety valve!
Too high → too few auto-responses (defeats the purpose)
Too low → bad responses → angry customers → $$$ lost! 💸
```

---

## 🎯 Key Takeaways

| Concept | One-Liner | Remember |
|---------|-----------|----------|
| ML Pipeline | End-to-end system: data → model → serving → monitoring | "Model is 5% of the code!" |
| Feature Store | Centralized, reusable feature computation | "Same features for train & serve!" |
| Model Registry | Version control for models | "Like Git, but for ML" |
| Batch vs Real-time | Pre-compute vs on-demand predictions | "Latency vs freshness trade-off" |
| Data Drift | Input distribution changes | Monitor feature distributions |
| Concept Drift | World changes, model is stale | Monitor actual outcomes |
| Feedback Loop | Monitor → detect → retrain → deploy | "ML never stops! It's a loop!" |

---

## ➡️ Next Up

👉 [Module 8.2: Model Serving & Inference →](./02_Model_Serving_Inference.md)

---

*"The hardest part of ML isn't building the model. It's keeping it working 6 months later when the data has changed, the team has changed, and nobody remembers why that one feature is log-transformed."* 🏗️😅

---

*Previous: [← Chain of Thought](../AI_Design_Patterns/04_Chain_of_Thought.md) | Next: [Java AI Ecosystem →](../AI_Java_Developers/01_Java_AI_Ecosystem.md)*
