# 🎓 Fine-Tuning & Alignment — Teaching AI Your Values 🧭

> **"Pre-training gives the model knowledge. Fine-tuning gives it a job. Alignment gives it ethics."** — The three phases of LLM development

```
┌─────────────────────────────────────────────────────────────────┐
│  🎮 LEVEL 6.7: FINE-TUNING & ALIGNMENT                          │
│                                                                 │
│  XP Reward: +350 🌟                                             │
│  Badge: 🎓 Alignment Engineer                                   │
│  Time: ~50 minutes                                              │
│  Industry Relevance: ⭐⭐⭐⭐⭐ (HOW ChatGPT became helpful!)     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📑 Table of Contents

1. [The Training Journey of an LLM](#-the-training-journey)
2. [Full Fine-Tuning vs Parameter-Efficient Methods](#-fine-tuning-approaches)
3. [LoRA & QLoRA — The Practical Revolution](#-lora--qlora)
4. [RLHF — Making AI Helpful, Harmless, Honest](#-rlhf)
5. [DPO — Simpler Alternative to RLHF](#-dpo)
6. [When to Fine-Tune vs When to Prompt/RAG](#-when-to-fine-tune)
7. [Java Implementation](#-java-implementation)
8. [Real-World Applications](#-real-world-applications)
9. [Interview Questions & Puzzles](#-interview-questions--puzzles)

---

## 🛤️ The Training Journey

### Three Phases of LLM Development

```
PHASE 1: PRE-TRAINING (Expensive! $1M-$100M!)
  Data: Internet-scale text (trillions of tokens)
  Goal: Learn language, facts, reasoning
  Result: A powerful but uncontrolled "base model"
  
  "The capital of France is..." → "Paris. The city is known for..."
  "How to hack a website?" → "Here are the steps: 1) Find..." 😱
  (Base model has no ethics — it just completes text!)

PHASE 2: FINE-TUNING (Moderate: $100-$100K)
  Data: Task-specific examples (thousands to millions)
  Goal: Specialize for a particular use case
  Result: Model that excels at your task
  
  Fine-tune for customer support:
  "How do I reset my password?" → helpful support response ✅

PHASE 3: ALIGNMENT (Critical! Makes it safe!)
  Data: Human preferences (which response is better?)
  Goal: Be helpful, harmless, and honest
  Result: ChatGPT, Claude, etc.
  
  "How to hack a website?" → "I can't help with that. 
   However, if you're interested in security, here are 
   ethical penetration testing resources..." ✅

              Pre-training → Fine-tuning → Alignment
              (knowledge)    (task skill)   (values)
              $$$$$$$        $$             $$$
```

---

## 🔧 Fine-Tuning Approaches

### The Spectrum

```
FULL FINE-TUNING:
  Update ALL parameters (billions!)
  ├── Pros: Maximum performance, full adaptation
  ├── Cons: Expensive (needs many GPUs!), catastrophic forgetting
  └── Cost: Training a 7B model ≈ $500-5000 on cloud GPUs

PARAMETER-EFFICIENT FINE-TUNING (PEFT):
  Update only a SMALL subset of parameters!
  ├── LoRA: Add tiny trainable matrices (0.1% of params!)
  ├── QLoRA: LoRA + quantization (runs on single GPU!)
  ├── Adapters: Insert small trainable layers
  └── Prefix Tuning: Only train prompt prefix embeddings

                Full Fine-Tune           LoRA
  Parameters    7,000,000,000       7,000,000 (0.1%!)
  GPU Memory    80GB+ (A100!)       16GB (consumer GPU!)
  Training Time Days                Hours!
  Performance   100% (baseline)     95-99% ← Almost as good!
  Cost          $5,000              $50 ← 100x cheaper!
```

---

## 🧩 LoRA & QLoRA

### LoRA: Low-Rank Adaptation

```
THE KEY INSIGHT:
  When you fine-tune a model, the weight CHANGES are LOW-RANK!
  Meaning: You don't need to update the full matrix!
  A small, thin matrix can capture most of the adaptation!

Original weight matrix W (4096 × 4096 = 16M params!):
  ┌──────────────────────────────────────┐
  │                                      │
  │         W (frozen! don't touch!)     │ 16M params
  │                                      │
  └──────────────────────────────────────┘

LoRA addition (rank r=8):
  ┌──────────┐     ┌──────────────────────────┐
  │    A     │  ×  │           B              │
  │(4096 × 8)│     │       (8 × 4096)         │
  └──────────┘     └──────────────────────────┘
     32K params  +     32K params = 64K params! (vs 16M!)
     
  Output = W×input + (A×B)×input
              ↑           ↑
           Frozen!    Trainable! (0.4% of original!)

It's like: Instead of rewriting an entire book,
           you just write sticky notes on the margins! 📝
```

### QLoRA: LoRA + Quantization

```
QLoRA = Quantize base model to 4-bit + apply LoRA

Base model (7B params):
  FP16: 14 GB GPU memory
  INT8: 7 GB
  INT4 (QLoRA): 3.5 GB! ← Fits on consumer GPU! 🎉

Plus LoRA adapters: +200MB (tiny!)

Total: Fine-tune a 7B model on a $500 gaming GPU! 🎮
(This democratized fine-tuning! Before: needed $50K+ hardware!)
```

### When to Use What

```
┌──────────────────────────────────────────────────────────────┐
│  Budget & Scale          │ Method                            │
├──────────────────────────┼───────────────────────────────────┤
│  Startup, small team     │ QLoRA (4-bit, single GPU)         │
│  Mid-size company        │ LoRA (16-bit, few GPUs)           │
│  Big tech / critical     │ Full fine-tune (many GPUs)        │
│  Just need good prompts  │ Don't fine-tune! Use RAG! 💰     │
└──────────────────────────┴───────────────────────────────────┘
```

---

## 🤝 RLHF — Reinforcement Learning from Human Feedback

### The Problem RLHF Solves

```
After pre-training + supervised fine-tuning:
  Model can write fluent text ✅
  Model follows instructions ✅
  BUT model might still:
    - Give harmful advice ❌
    - Hallucinate confidently ❌
    - Be unhelpful or verbose ❌
    - Reveal training data ❌

RLHF adds HUMAN JUDGMENT to fix these!
"Which response do you PREFER?" → train on that! 🎯
```

### The Three Steps of RLHF

```
STEP 1: SUPERVISED FINE-TUNING (SFT)
  Train on high-quality (prompt, response) pairs
  Written by humans or curated from model outputs
  Result: Model that can follow instructions
  
STEP 2: REWARD MODEL TRAINING
  Show humans TWO responses to the same prompt
  Humans say which is BETTER (preference data!)
  Train a "reward model" to predict human preference!
  
  "How do I make coffee?"
  Response A: "Step 1: Boil water. Step 2: Add grounds..." (clear! ✅)
  Response B: "Coffee is a complex beverage with origins in..." (verbose! ❌)
  Human: "A is better!"
  Reward model learns: concise + helpful = higher reward!

STEP 3: PPO (Proximal Policy Optimization)
  Use the reward model to further train the LLM!
  LLM generates response → Reward model scores it → Update LLM!
  
  Repeat millions of times → LLM learns to maximize reward! 📈

                ┌─────────────────────────────────────────┐
                │           RLHF PIPELINE                   │
                │                                          │
                │  ┌───────┐   Generate    ┌──────────┐  │
                │  │  LLM  │──────────────→│ Response │  │
                │  └───┬───┘               └────┬─────┘  │
                │      ↑                        │        │
                │      │ Update weights          │ Score  │
                │      │                        ↓        │
                │  ┌───┴────────────────────────────┐    │
                │  │       REWARD MODEL             │    │
                │  │  (trained on human preferences) │    │
                │  └────────────────────────────────┘    │
                └─────────────────────────────────────────┘
```

---

## ⚡ DPO — Direct Preference Optimization

### The Simpler Alternative (2023)

```
RLHF is COMPLEX:
  Need reward model + PPO training + careful hyperparameters
  Unstable! Hard to reproduce! Expensive!

DPO is SIMPLE:
  Skip the reward model entirely!
  Directly optimize the LLM on preference pairs!
  
  Given: (prompt, preferred_response, rejected_response)
  DPO loss: increase probability of preferred, decrease rejected!
  
  That's it! No reward model! No PPO! No instability! 🎉

RLHF:   SFT → Train Reward Model → PPO Loop (complex!)
DPO:    SFT → Direct optimization on preferences (simple!)

Results: DPO ≈ RLHF quality, much easier to implement!
  Llama-2: RLHF
  Llama-3: DPO ← Meta switched to DPO! (simpler, same quality!)
```

---

## 🤔 When to Fine-Tune

### The Decision Framework

```
DO YOU NEED TO FINE-TUNE? Use this flowchart:

"I want my AI to do X better"
           │
           ▼
  Can good prompting solve it?
  ├── YES → Don't fine-tune! Just improve prompts! ✅ (cheapest!)
  └── NO ↓
  
  Can RAG solve it? (model needs specific knowledge?)
  ├── YES → Build RAG pipeline! ✅ (no training needed!)
  └── NO ↓
  
  Do you need a specific OUTPUT STYLE or BEHAVIOR?
  ├── YES → Fine-tune! (model needs to learn your style!)
  └── NO ↓
  
  Do you need domain-specific REASONING?
  ├── YES → Fine-tune on domain data! (medical, legal, etc.)
  └── NO → You probably don't need to fine-tune! 🎯
```

### Comparison Table

| Method | Cost | Setup Time | When to Use |
|--------|------|-----------|-------------|
| Prompt Engineering | $0 | Minutes | First try! Always! |
| RAG | $-$$ | Days | Need specific knowledge |
| LoRA Fine-tune | $$-$$$ | Days-Weeks | Need specific behavior/style |
| Full Fine-tune | $$$$ | Weeks | Maximum control (rare!) |

### Real Examples

```
❌ DON'T fine-tune for:
  "Answer questions about our docs" → Use RAG! 
  "Respond in JSON format" → Use structured output prompt!
  "Be more concise" → Just say "Be concise" in system prompt!

✅ DO fine-tune for:
  "Write code in our company's specific style/patterns"
  "Classify support tickets into our custom 50 categories"
  "Generate medical reports following our hospital's format"
  "Translate technical docs with industry-specific terminology"
```

---

## ☕ Java Implementation

```java
/**
 * Fine-tuning workflow orchestration in Java
 * Note: Actual training runs in Python/PyTorch — 
 * Java orchestrates the pipeline and serves the model!
 */

// ─── FINE-TUNING JOB MANAGEMENT ──────────────────────────────

@Service
public class FineTuningOrchestrator {
    
    private final OpenAiFineTuningClient fineTuningClient;
    private final DataPreparationService dataPrep;
    
    /**
     * Prepare and submit a fine-tuning job to OpenAI
     * (OpenAI supports fine-tuning via API!)
     */
    public String submitFineTuningJob(List<TrainingExample> examples) {
        // 1. Validate and format training data
        List<ChatMessage> formattedData = examples.stream()
            .map(ex -> new ChatMessage(
                ex.getSystemPrompt(),
                ex.getUserMessage(),
                ex.getDesiredResponse()
            ))
            .toList();
        
        // 2. Validate data quality
        validateTrainingData(formattedData);
        
        // 3. Upload training file
        String fileId = fineTuningClient.uploadTrainingFile(formattedData);
        
        // 4. Create fine-tuning job
        FineTuningJobRequest request = FineTuningJobRequest.builder()
            .trainingFile(fileId)
            .model("gpt-4o-mini-2024-07-18")  // Base model
            .hyperparameters(Hyperparameters.builder()
                .nEpochs(3)
                .learningRateMultiplier(1.8)
                .batchSize(4)
                .build())
            .suffix("my-custom-model")  // Name suffix
            .build();
        
        return fineTuningClient.createJob(request).getId();
    }
    
    /**
     * Quality checks before expensive training!
     */
    private void validateTrainingData(List<ChatMessage> data) {
        if (data.size() < 50) {
            throw new IllegalArgumentException(
                "Need at least 50 examples! (100+ recommended)");
        }
        
        // Check for duplicates
        long uniqueCount = data.stream()
            .map(ChatMessage::getUserMessage)
            .distinct().count();
        
        if (uniqueCount < data.size() * 0.9) {
            throw new IllegalArgumentException(
                "Too many duplicates! %d/%d are unique".formatted(uniqueCount, data.size()));
        }
        
        // Check response quality (not empty, reasonable length)
        data.forEach(msg -> {
            if (msg.getAssistantResponse().length() < 10) {
                throw new IllegalArgumentException(
                    "Response too short: " + msg.getAssistantResponse());
            }
        });
    }
}

// ─── PREFERENCE DATA COLLECTION FOR DPO/RLHF ────────────────

@Service
public class PreferenceCollectionService {
    
    private final ChatModel model;
    private final PreferenceRepository preferenceRepo;
    
    /**
     * Generate response pairs for human evaluation
     */
    public ComparisonPair generatePairForEvaluation(String prompt) {
        // Generate two responses with different temperatures
        String responseA = model.call(new Prompt(prompt,
            OpenAiChatOptions.builder().withTemperature(0.3).build()
        )).getResult().getOutput().getContent();
        
        String responseB = model.call(new Prompt(prompt,
            OpenAiChatOptions.builder().withTemperature(0.9).build()
        )).getResult().getOutput().getContent();
        
        return new ComparisonPair(prompt, responseA, responseB);
    }
    
    /**
     * Record human preference (for future RLHF/DPO training)
     */
    public void recordPreference(String prompt, String preferred, 
                                  String rejected, String annotatorId) {
        PreferenceRecord record = PreferenceRecord.builder()
            .prompt(prompt)
            .chosenResponse(preferred)
            .rejectedResponse(rejected)
            .annotatorId(annotatorId)
            .timestamp(Instant.now())
            .build();
        
        preferenceRepo.save(record);
    }
    
    /**
     * Export preferences in DPO training format
     */
    public List<DPOExample> exportForTraining() {
        return preferenceRepo.findAll().stream()
            .map(r -> new DPOExample(
                r.getPrompt(),
                r.getChosenResponse(),
                r.getRejectedResponse()
            ))
            .toList();
    }
}

// ─── USING A FINE-TUNED MODEL ────────────────────────────────

@Configuration
public class FineTunedModelConfig {
    
    @Bean("fineTunedModel")
    public ChatModel fineTunedChatModel() {
        return new OpenAiChatModel(
            new OpenAiApi(System.getenv("OPENAI_API_KEY")),
            OpenAiChatOptions.builder()
                .withModel("ft:gpt-4o-mini-2024-07-18:my-org::abc123") // Fine-tuned!
                .withTemperature(0.2)
                .build()
        );
    }
}

@Service
public class SpecializedAssistant {
    
    @Qualifier("fineTunedModel")
    private final ChatModel fineTunedModel;
    
    // This model now responds in YOUR specific style! 🎯
    public String generateResponse(String input) {
        return fineTunedModel.call(input);
    }
}
```

---

## 🏢 Real-World Applications

### 🔷 ChatGPT — From GPT-4 to ChatGPT

```
GPT-4 (base) → ChatGPT:
  1. Pre-trained on internet text (knowledge!)
  2. SFT on high-quality conversations (instruction following!)
  3. RLHF with human raters (helpful, safe, honest!)

Without RLHF:
  "How to hotwire a car?" → Detailed instructions! 😱

With RLHF:
  "How to hotwire a car?" → "I can't help with that. If you're 
   locked out of your own car, here are legitimate options..." ✅

The RLHF is what makes ChatGPT FEEL different from a base model!
```

### 🔷 Bloomberg — Domain Fine-Tuning

```
BloombergGPT (50B params): Fine-tuned on financial data!

Base model: "What's a call option?"
  → Generic textbook answer

Bloomberg model: "What's a call option?"
  → Precise financial definition with current market context,
    relevant regulations, and practical trading implications!

Fine-tuning on domain data = EXPERT-level responses in that domain! 🎓
```

---

## 🎯 Interview Questions & Puzzles

### Question 1: "Explain RLHF and why it matters."

**Great Answer**: "RLHF is the process that turns a raw language model into a helpful assistant. It has three steps: (1) Supervised fine-tuning on curated (instruction, response) pairs. (2) Training a reward model on human preferences — humans compare two responses and pick which is better. (3) Using PPO (reinforcement learning) to optimize the LLM to maximize the reward model's score. Why it matters: pre-trained models have knowledge but no judgment — they'll happily help with harmful requests or give verbose, unhelpful answers. RLHF teaches the model human VALUES: be helpful, be safe, be honest. The alternative DPO skips the reward model and directly optimizes on preference pairs — simpler, similarly effective, and increasingly popular (used in Llama-3)."

### Question 2: "When would you fine-tune vs use RAG?"

**Great Answer**: "Fine-tune when you need to change HOW the model responds — its style, format, reasoning patterns, or personality. RAG when you need to change WHAT the model knows — giving it access to specific documents or current information. Examples: A law firm wanting AI to write briefs in their specific format → fine-tune (style). A law firm wanting AI to reference specific case law → RAG (knowledge). Often you use BOTH: fine-tune for style + RAG for knowledge. Key consideration: RAG is cheaper, faster to implement, and the knowledge can be updated without retraining. Fine-tuning is more expensive but produces more consistent style and can improve reasoning in specific domains. My default: start with RAG, only fine-tune if RAG quality isn't sufficient."

---

### 🧩 Puzzle: Choose Your Approach!

```
For each scenario, choose: Prompt | RAG | Fine-tune | RLHF

A) "Make our chatbot respond in our brand voice (casual, uses emojis)"
   → FINE-TUNE ✅ (specific style that prompting can't perfectly capture)

B) "Our model needs to know about products added this week"
   → RAG ✅ (dynamic knowledge, changes frequently!)

C) "Make the model refuse to discuss competitor products"
   → PROMPT + GUARDRAILS ✅ (system prompt + output filter)

D) "Our medical AI sometimes gives dangerous advice"
   → RLHF/DPO ✅ (align with safety values, learn from expert preferences)

E) "Model should output in our custom XML schema"
   → FINE-TUNE ✅ (or structured output with JSON schema if supported)

F) "Users want answers from our 10,000 page documentation"
   → RAG ✅ (classic RAG use case!)

Remember: Start simple (prompting) → Add RAG → Fine-tune last! 📊
```

---

## 🎯 Key Takeaways

| Concept | One-Liner | When |
|---------|-----------|------|
| Full Fine-Tune | Update all weights | Maximum quality, big budget |
| LoRA | Tiny trainable matrices (~0.1% params) | 95% quality, 1% cost! |
| QLoRA | LoRA + 4-bit quantization | Single GPU fine-tuning! |
| RLHF | Train with human preferences | Make model helpful & safe |
| DPO | Simpler RLHF alternative | Same quality, easier implementation |
| SFT | Supervised instruction tuning | First step after pre-training |

### The Golden Rule
```
Don't fine-tune until you've exhausted:
  1. Better prompting (free!)
  2. Few-shot examples (free!)
  3. RAG pipeline (moderate cost)
  4. Fine-tuning (expensive! last resort!)
  
90% of "we need to fine-tune" cases are actually 
"we need better prompts + RAG" cases! 💡
```

---

## ➡️ Next Up

👉 [Module 6.1: Text Representation →](./01_Text_Representation.md)

---

*"RLHF is like raising a child: you don't program them with rules, you show them good behavior and bad behavior, and they learn your values. That's why ChatGPT feels like it has a personality — it was raised well!"* 🧒🎓😄
