
---
title: "Prompt Engineering From First Principles: The Mechanics They Don't Teach You part-3"
date: 2025-12-21T10:36:47.4747+05:30
draft: true
tags:
---


How to write prompt for production practice and test in scallable way
- categorize the prompt use does it sinlge turn or multi turn (conversation) we will subcategorize it based on the use case but for sake of simplicity we will use single turn and multi turn and see how to write prompt for both
- HOw write and test and scale single prompt 
- How to write and test and scale multi turn prompt
- How to debug the prompt 
- *Evaluation:** How to define and test for "good" output beyond just looking correct. automatic 
- live monotoring


prompt optimization



---
title: "Prompt Engineering From First Principles: The Mechanics They Don't Teach You - Part 4"
date: 2025-02-24T10:36:47.4747+05:30
draft: true
tags:
  - prompt-engineering
  - llm
  - optimization
  - automation
  - dspy
---

In [Part 3](./part-3.md), we mastered **practical prompting techniques**: CoT, ReAct, Tree of Thoughts, and others. These techniques make prompts more effective—but they also make them **more expensive**.

A Chain of Thought might use 3x more tokens than a simple prompt. ReAct might make 5+ API calls. These improvements come with costs.

Part 4 addresses the operational reality: **How do you scale prompts to production without massive costs? How do you automate prompt engineering itself?**

We'll split this into two critical areas:

1. **Cost Reduction & Efficiency** - Making effective prompts cheaper
2. **Automation & Programmatic Approach** - Making prompt engineering scalable

---

## Section A: Cost Reduction & Efficiency

### Problem Statement

You've built an amazing prompt that works perfectly. But your app makes 1,000 requests per day. At $0.01 per 1,000 tokens (Claude pricing), each request with optimized prompting costs $0.05-0.10. That's $50-100 per day, or $1,500-3,000 per month.

Your business model breaks at scale.

The question: **How do you maintain quality while reducing cost?**

---

## Technique 1: Prompt Compression - Reducing Token Size

### The Problem

Your carefully crafted prompt might be:
- 500 words of context
- 10 examples (few-shot)
- Detailed instructions
- Total: 1,500+ tokens

But **not all tokens matter equally**.

Some tokens are "filler" (punctuation, connecting words, repeated concepts). Other tokens are critical (domain-specific terms, key examples, constraints).

### How Prompt Compression Works

Prompt compression uses a trained model to classify each token as "preserve" or "discard" based on importance:

**Three-step process:**

1. **Token Classification:**
   - A specialized model reads your prompt
   - Assigns each token a "preserve probability" (0-1)
   - Critical tokens get high probability, filler gets low

2. **Token Selection:**
   - You specify a target compression ratio (e.g., 50% = compress to half size)
   - The model selects the top tokens by preserve probability
   - Discards the rest

3. **Token Order Preservation:**
   - The selected tokens maintain their original order
   - Ensures grammatical coherence and meaning

### Why It Works

The preserve/discard classifier learns from examples. It figures out:
- Domain keywords are critical (preserve)
- Articles ("a", "the") are often redundant (discard)
- Examples are important but can be shortened
- Repeated instructions can be compressed

### Real-World Performance

**Example:**
```
Original prompt (1,500 tokens):
"You are a world-class customer service AI. Your job is to respond
to customer inquiries with empathy and accuracy. When customers write,
they often have problems or questions. You should address their problem
directly and accurately. Here are some examples of good responses..."

Compressed to 50% (750 tokens):
"Customer service AI. Respond with empathy, accuracy. Address problems
directly. Examples of good responses..."

Quality loss: ~5-10%
Token reduction: 50%
```

### Tools for Prompt Compression

1. **LLMLingua** (Microsoft Open Source)
   - Uses iterative compression with LLM feedback
   - Preserves semantic meaning
   - Works with any LLM

2. **GPTTrim**
   - Simpler approach: tokenization, stemming, removing spaces
   - Faster but less sophisticated

### When to Use Compression

- High-volume applications (1,000+ requests/day)
- Cost-sensitive scenarios
- When you've over-engineered your prompt
- Batch processing (where latency isn't critical)

### When NOT to Use

- **Critical accuracy tasks** (medical, legal, financial)
- **One-off/low-volume** queries (savings not worth it)
- **Real-time applications** (compression adds latency)

### Implementation Considerations

```
Trade-offs:
- Compression ratio 30%: ~1-2% quality loss, 70% cost savings
- Compression ratio 50%: ~5-10% quality loss, 50% cost savings
- Compression ratio 70%: ~15-25% quality loss, 30% cost savings

Test your specific use case before deploying.
```

---

## Technique 2: FrugalGPT - Intelligent Cost Optimization

### The Problem

Not all requests are equally complex. Some queries need:
- GPT-4 ($0.03 per 1,000 input tokens)
- Others work fine with GPT-4o-mini ($0.0005 per 1,000 tokens)

But you don't know which is which until you try.

**Standard approach:** Use GPT-4 for everything (expensive, consistent)
**Naive approach:** Use mini for everything (cheap, breaks on hard tasks)
**Smart approach:** Route intelligently based on task complexity

### How FrugalGPT Works

FrugalGPT is a framework with three key techniques:

#### 1. Prompt Adaptation - Optimize Examples & Context

**Concept:** You have 10 examples. Do you need all 10? Can you pick the best 3?

FrugalGPT suggests:
- **Identify top-k similar examples** - Use similarity metrics to find the best examples for each query
- **Combine similar requests** - Batch similar queries and use shared examples

**Example:**
```
Task: Email classification (spam vs. not spam)

Without optimization:
- Every query gets 10 examples
- 500 queries × 10 examples = 5,000 examples used

FrugalGPT approach:
- Calculate similarity between new email and existing examples
- Select top 3 most similar examples for each query
- 500 queries × 3 examples = 1,500 examples used

Result: 70% fewer examples, same accuracy
```

#### 2. Model Cascade - Route to Right-Sized Model

**Concept:** Use smaller models first, fall back to larger models only if needed.

**Process:**
1. Send query to **GPT-4o-mini** (cheapest)
2. Evaluate response quality/confidence
3. If confidence >= threshold, return answer
4. If confidence < threshold, send to **GPT-4** (more expensive)
5. Continue up the cascade until satisfied

**Cost Impact:**
```
Without cascade (use GPT-4 for all):
- 1,000 queries × $0.03 = $30

With cascade (mini → GPT-4):
- 800 queries to mini × $0.0005 = $0.40
- 200 queries to GPT-4 × $0.03 = $6.00
- Total: $6.40 (78% cost reduction)

Accuracy maintained if confidence thresholds are calibrated correctly.
```

**Confidence evaluation strategies:**
- **Output confidence**: Model's own uncertainty (available in some APIs)
- **Self-consistency**: Generate 3 responses, check if they agree
- **Semantic validation**: Check if output has expected structure/format
- **Human feedback loop**: In production, track which queries humans fix

#### 3. Better Prompt ≈ Worse Model

**Concept:** A well-optimized prompt with GPT-4o-mini might outperform a mediocre prompt with GPT-4.

Trade-off:
- **Invest in prompt engineering** → Use cheaper model
- **Use expensive model** → Accept mediocre prompts

Example:
```
Approach 1: Minimal prompt + GPT-4
Cost: $0.03 per query
Accuracy: 85%

Approach 2: Optimized prompt + GPT-4o-mini
Cost: $0.0005 per query
Accuracy: 87%

Conclusion: Better prompt + cheaper model > worse prompt + expensive model
```

### FrugalGPT Implementation Patterns

**Pattern 1: Simple Cascade**
```
1. Try with GPT-4o-mini
2. If response seems incomplete/wrong, retry with Claude 3.5 Sonnet
3. If still unsure, escalate to human
```

**Pattern 2: Adaptive Examples**
```
For each new query:
1. Calculate similarity to existing examples in database
2. Select top 3 most similar examples
3. Use only those examples in the prompt
4. Send to model
```

**Pattern 3: Combined Optimization**
```
1. Compress prompt to 50% size
2. Select top-3 examples via similarity
3. Route simple queries to mini, complex to GPT-4
4. Evaluate response confidence
5. If unsure, escalate
```

### When FrugalGPT Makes Sense

- **Production systems** with thousands of requests/day
- **Mixed complexity** - some tasks are hard, some are easy
- **Cost-sensitive** business models
- **You can afford latency** for cascade (extra API calls)

### When It Doesn't

- **Real-time requirements** (cascade adds latency)
- **Consistent quality needed** (cascade introduces variance)
- **Already using cheap models** (little room to optimize)

---

## Technique 3: LLM Cascade - Multi-Model Routing

### The Problem

Different models have different strengths:
- **GPT-4 Turbo**: Best overall reasoning ($0.03/1K tokens)
- **Claude 3.5 Sonnet**: Best instruction-following ($0.003/1K tokens)
- **GPT-4o-mini**: Fastest, cheapest ($0.00015/1K tokens)
- **Llama 3.1 70B**: Fast, reasonable cost ($0.0005/1K tokens)

Your app has **one task**. Which model should you use?

Answer: **All of them, intelligently.**

### How LLM Cascade Works

Instead of picking one model, LLM Cascade queries models **sequentially**:

1. **Send to model 1** (cheapest)
   - Get response
   - Evaluate quality/confidence
   - If good enough → return

2. **If not good enough, send to model 2** (next tier)
   - Get response
   - Evaluate
   - If good enough → return

3. **If still not good enough, send to model 3** (best)
   - Get response
   - Return regardless

### Why This Works

Most queries are "easy" and cheap models handle them fine. Only truly hard queries bubble up to expensive models.

**Cost structure:**
```
Tier 1 (GPT-4o-mini): $0.00015/1K tokens, handles 70% of queries
Tier 2 (Claude 3.5 Sonnet): $0.003/1K tokens, handles 25% of queries
Tier 3 (GPT-4 Turbo): $0.03/1K tokens, handles 5% of queries

Average cost per query:
= (0.70 × $0.00015) + (0.25 × $0.003) + (0.05 × $0.03)
= $0.00011 + $0.00075 + $0.0015
= $0.00236 per query

vs. Always using GPT-4: $0.03 per query (12x more expensive)
```

### Designing Confidence Metrics

The key to effective cascade is **knowing when to escalate**.

**Metric 1: Self-Consistency**
```
Generate response 3 times
If all 3 agree → confidence HIGH (return)
If 2/3 agree → confidence MEDIUM (escalate)
If all 3 different → confidence LOW (escalate)
```

**Metric 2: Semantic Validity**
```
Check if response matches expected structure
- Code generation: Does code compile?
- JSON output: Is it valid JSON?
- Math problems: Is answer numeric?

If VALID → confidence HIGH
If INVALID → confidence LOW (escalate)
```

**Metric 3: Perplexity Score**
```
Measure how "certain" the model was generating each token
Low perplexity = high confidence
High perplexity = low confidence (escalate)
```

**Metric 4: Length Heuristic**
```
Longer, detailed responses often indicate deeper reasoning
If response < 20 tokens → likely shallow (escalate)
If response > 100 tokens → likely thoughtful (return)
```

### Implementation Pattern

```
Function: query_with_cascade(prompt, thresholds):

    for model, threshold in cascade_models:
        response = query(model, prompt)
        confidence = evaluate_confidence(response)

        if confidence >= threshold:
            return response
        else:
            continue to next model

    # If all models fail confidence threshold
    return response_from_best_model
```

### Caveats

- **Different models have different output quality** - Compare carefully
- **Confidence metrics are heuristic** - They fail sometimes
- **Cascading adds latency** - Each tier is another API call
- **Track metrics in production** - Monitor where queries actually escalate

---

## Technique 4: LLM Approximation - Caching & Strategic Fine-Tuning

### The Problem

You're making the same query multiple times:
- Users asking similar questions
- Batch processing similar documents
- Testing the same prompt repeatedly

Each query costs money. Identical queries cost the same as different queries.

### How LLM Approximation Works

LLM Approximation has two complementary strategies:

#### Strategy 1: Cache LLM Requests

**Concept:** If you've already answered this question, don't ask again.

Implementation:
```
1. Hash the prompt (user input + system prompt)
2. Check if result is in cache
3. If YES → return cached result instantly
4. If NO → query LLM, cache result, return

Cache key: SHA256(system_prompt + user_message)
Cache TTL: Depends on task (15 minutes for news, 24 hours for evergreen)
```

**Cost Impact:**
```
Without caching:
- 1,000 queries/day × $0.01 average = $10/day

With caching (assuming 40% of queries are repeated):
- 600 new queries × $0.01 = $6
- 400 cached queries × $0 = $0
- Total: $6/day (40% savings)

On production scale: $300/month savings
```

**Example Cache Strategy for Q&A:**
```
Question: "What is photosynthesis?"

Query 1: Cache miss → Query LLM → Cache result
Query 2 (same question): Cache hit → Return cached result immediately
Query 3 (similar): "Explain photosynthesis" → Different hash → Cache miss

Cache hit rate typically 20-50% depending on your domain.
```

**Considerations:**
- **Cache invalidation**: When does cached data become stale?
- **User context**: Different users might need different answers
- **Prompt changes**: When you update prompts, cache becomes invalid

#### Strategy 2: Fine-Tune a Smaller Model in Parallel

**Concept:** While using GPT-4 in production, continuously fine-tune a smaller model on real user data. Eventually switch to the fine-tuned smaller model.

**Implementation:**
```
Phase 1: Establish baseline
- Run GPT-4 for all requests
- Log requests and responses
- Collect training data (user query → GPT-4 response)

Phase 2: Fine-tune smaller model
- Use collected data to fine-tune Claude 3.5 Haiku or GPT-4o-mini
- Run in parallel, don't affect production yet
- Compare quality of fine-tuned model vs. GPT-4

Phase 3: Gradual migration
- Route 10% of traffic to fine-tuned smaller model
- Monitor quality, user feedback
- If good, increase to 25%, then 50%, then 100%

Phase 4: Cost savings
- Smaller model is 10-50x cheaper
- After fine-tuning, quality is competitive
- Full migration to fine-tuned model
```

**Example Timeline:**
```
Month 1: Collect 1,000 Q&A pairs using GPT-4 ($30 cost)
Month 2: Fine-tune GPT-4o-mini on collected data ($5 cost)
Month 3: A/B test (90% GPT-4, 10% fine-tuned mini)
Month 4: Scale to 50% fine-tuned mini
Month 5: Full migration to fine-tuned mini

Result:
- Old cost: $30/month (using GPT-4 for everything)
- New cost: $0.50/month (using fine-tuned mini)
- Savings: 98% reduction
```

**When This Works:**
- **High volume, consistent domain** (customer support, Q&A)
- **You have months to prepare** (fine-tuning takes time)
- **Quality difference is acceptable** (95% of GPT-4 is good enough)

**When It Doesn't:**
- **Low volume** (fine-tuning cost > usage cost)
- **Rapidly changing requirements** (fine-tuned model becomes stale)
- **Quality must be perfect** (can't sacrifice accuracy)

---

## Section A Summary: Cost Reduction Strategies

| Technique | Cost Reduction | Effort | Time | Best For |
|-----------|----------------|--------|------|----------|
| **Compression** | 30-50% | Low | Immediate | Batch processing |
| **FrugalGPT** | 60-80% | Medium | 1-2 weeks | Mixed complexity tasks |
| **LLM Cascade** | 70-90% | Medium | 2-4 weeks | High-volume apps |
| **Caching** | 20-50% | Low | Immediate | Repetitive queries |
| **Fine-tuning** | 90%+ | High | 1-3 months | Long-term, stable domain |

**Strategy:**
1. Start with **Caching** (immediate, no risk)
2. Add **Compression** (quick win)
3. Implement **FrugalGPT** (good balance)
4. Plan **Fine-tuning** (long-term)
5. Use **Cascade** when you have multiple models available

---

---

## Section B: Automation & Programmatic Approach

### Problem Statement

You've written perfect prompts. But prompts are **fragile**:
- A tiny wording change breaks everything
- Examples that work for one task fail for another
- You can't test prompts systematically

Scaling prompts means **automating prompt engineering itself**.

Enter: **DSPy**

---

## Technique: DSPy - Declarative Self-Improving Language Programs

### What Is DSPy?

DSPy (Declarative Self-improving Language Programs) is a **framework for building and optimizing LM applications programmatically**.

Instead of hand-writing prompts, DSPy lets you:
1. **Define what you want** (signatures/specs)
2. **Chain operations** (modules)
3. **Automatically optimize** (teleprompters)

Think of it as "PyTorch for LLMs."

### Why Hand-Written Prompts Don't Scale

```
Manual approach:
1. Write prompt
2. Test it
3. It fails
4. Rewrite prompt
5. Test again
6. Repeat infinitely

Problems:
- Not systematic
- Hard to version control
- Can't A/B test reliably
- Breaks when tasks change
```

DSPy fixes this by making prompts **programmable and optimizable**.

### Core Concepts

#### 1. Signatures - Define Input/Output

A **signature** specifies:
- What inputs you need
- What output you want
- Optional descriptions

```python
# Simple signature
"question -> answer"

# With metadata
"""
question: The user's question
context: Optional background info
---
answer: A concise answer to the question
"""

# Python class (most explicit)
class GenerateAnswer(dspy.Signature):
    """Answer questions with short factoid answers."""
    context = dspy.InputField(desc="may contain relevant facts")
    question = dspy.InputField()
    answer = dspy.OutputField(desc="often between 1 and 5 words")
```

A signature is **declarative** - it says "what", not "how".

#### 2. Modules - Chain Operations

A **module** is a reusable unit that performs a task:

```python
# Simple prediction
predictor = dspy.Predict("question -> answer")
result = predictor(question="What is 2+2?")
print(result.answer)  # "4"

# Chain of Thought (built-in)
cot = dspy.ChainOfThought("question -> answer")
result = cot(question="Why is the sky blue?")
print(result.answer)

# Custom module
class RAG(dspy.Module):
    def __init__(self, num_passages=3):
        super().__init__()
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.generate = dspy.ChainOfThought("context, question -> answer")

    def forward(self, question):
        context = self.retrieve(question).passages
        return self.generate(context=context, question=question)
```

#### 3. Teleprompters - Automatic Optimization

A **teleprompter** (optimizer) automatically improves your program:

```python
# Before optimization
program = RAG()
accuracy = evaluate(program)  # 60% accurate

# Optimize with teleprompter
optimizer = dspy.BootstrapFewShot(metric=accuracy_metric)
program = optimizer.compile(program, trainset=training_data)
accuracy = evaluate(program)  # 85% accurate
```

The teleprompter:
- Generates demonstrations automatically
- Tests different prompts
- Selects best version
- Learns few-shot examples
- All without you writing anything

### How DSPy Works: A Complete Example

Let's build a Q&A system:

```python
import dspy

# Step 1: Define your data
class QuestionAnswerPair:
    def __init__(self, question, answer):
        self.question = question
        self.answer = answer

train_data = [
    QuestionAnswerPair("What is photosynthesis?",
                      "Process where plants use sunlight to create energy"),
    QuestionAnswerPair("Who wrote Romeo and Juliet?",
                      "William Shakespeare"),
    # ... more examples
]

# Step 2: Configure LLM (DSPy works with any LLM)
lm = dspy.OpenAI(model="gpt-4o-mini", api_key="your-key")
dspy.settings.configure(lm=lm)

# Step 3: Define signature (what we want)
class AnswerQuestion(dspy.Signature):
    """Answer a question concisely."""
    question = dspy.InputField()
    answer = dspy.OutputField(desc="short, factual answer")

# Step 4: Create module (how we do it)
class QASystem(dspy.Module):
    def __init__(self):
        super().__init__()
        # Use Chain of Thought for reasoning
        self.generate = dspy.ChainOfThought(AnswerQuestion)

    def forward(self, question):
        return self.generate(question=question)

# Step 5: Define metric (how good is good)
def accuracy_metric(example, pred, trace=None):
    return pred.answer.lower() == example.answer.lower()

# Step 6: Compile/optimize
program = QASystem()

optimizer = dspy.BootstrapFewShot(metric=accuracy_metric)
program = optimizer.compile(
    program,
    trainset=train_data,
    teacher_settings=dict(lm=lm)
)

# Step 7: Use your optimized program
test_question = "What is photosynthesis?"
result = program(question=test_question)
print(result.answer)
```

What just happened:
1. You defined WHAT (signature)
2. You defined HOW to organize it (module)
3. You defined GOOD (metric)
4. DSPy automatically found the BEST prompt and few-shot examples

**Result:** Your program improves from 60% to 85% accuracy **without you writing a single new prompt**.

### Advanced: Retrieval Augmented Generation (RAG) with DSPy

```python
import dspy

# Configure retrieval system
dspy.settings.configure(
    lm=dspy.OpenAI(model="gpt-4o-mini"),
    rm=dspy.ColBERTv2(url='http://localhost:8893/api/search')
)

# Step 1: Define signatures
class GenerateQuery(dspy.Signature):
    """Write a search query based on the question."""
    question = dspy.InputField()
    query = dspy.OutputField()

class GenerateAnswer(dspy.Signature):
    """Answer based on context."""
    context = dspy.InputField()
    question = dspy.InputField()
    answer = dspy.OutputField()

# Step 2: Create RAG module
class RAGSystem(dspy.Module):
    def __init__(self, num_passages=3):
        super().__init__()
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.generate_query = dspy.ChainOfThought(GenerateQuery)
        self.generate_answer = dspy.ChainOfThought(GenerateAnswer)

    def forward(self, question):
        # Step 1: Generate search query
        query = self.generate_query(question=question).query

        # Step 2: Retrieve relevant passages
        context = self.retrieve(query).passages

        # Step 3: Generate answer using context
        answer = self.generate_answer(
            context=context,
            question=question
        ).answer

        return dspy.Prediction(answer=answer)

# Step 3: Optimize
rag_system = RAGSystem()

def rag_metric(example, pred, trace=None):
    answer_match = pred.answer.lower() in example.answer.lower()
    return answer_match

optimizer = dspy.BootstrapFewShot(metric=rag_metric)
rag_system = optimizer.compile(
    rag_system,
    trainset=train_data,
    teacher_settings=dict(lm=dspy.OpenAI(model="gpt-4"))  # Use better model for optimization
)

# Use your optimized RAG
question = "What are the benefits of renewable energy?"
result = rag_system(question=question)
print(result.answer)
```

In this example:
- DSPy automatically figured out: "Which passages are relevant?"
- DSPy optimized the answer generation prompt
- DSPy selected the best few-shot examples
- All automatically from your training data

### DSPy vs. Manual Prompting

| Aspect | Manual Prompts | DSPy |
|--------|---|---|
| **Defining task** | Unstructured text | Signatures |
| **Chaining** | String concatenation | Modules + composition |
| **Optimization** | Trial and error | Automatic via teleprompter |
| **Testing** | Ad-hoc | Systematic metric-based |
| **Few-shot examples** | Hand-picked | Automatically selected |
| **Reproducibility** | Hard (prompt text varies) | Easy (code is version controlled) |
| **Scaling** | Hard (more manual work) | Natural (same code, larger data) |

### When to Use DSPy

**Use DSPy when:**
- You're building **production systems** (not one-offs)
- You have **training data** to optimize on
- You need **reproducible, version-controlled** prompts
- You want **systematic A/B testing**
- Your task requires **chaining multiple LLM calls**

**Use manual prompts when:**
- One-off tasks (analysis, writing)
- You don't have training data
- Rapid prototyping/experimentation
- Simple single-call tasks

### DSPy Limitations

1. **Requires training data** - You need examples to optimize
2. **Metric design is hard** - Defining "good" is non-trivial
3. **Learning curve** - Requires thinking programmatically
4. **Works best for structured tasks** - Open-ended tasks are harder

### DSPy Optimizers Explained

Different teleprompters for different scenarios:

**BootstrapFewShot:**
- Selects best examples from training data
- Generates few-shot demonstrations
- Best for: Most tasks, simple optimization

**BootstrapFewShotWithRandomSearch:**
- Tries random combinations of examples
- More thorough search
- Best for: When initial attempts fail

**BootstrapFewShotWithOptuna:**
- Uses Bayesian optimization
- Finds optimal number of examples
- Best for: Cost-sensitive applications

**LabeledFewShot:**
- Simple: just add training examples
- No optimization
- Best for: Quick baseline

### Example: Real-World Email Classification

```python
import dspy

# Data
class Email:
    def __init__(self, subject, body, label):
        self.subject = subject
        self.body = body
        self.label = label  # "spam" or "not_spam"

train_emails = [
    Email("Limited time offer!", "Buy now for 50% off", "spam"),
    Email("Meeting tomorrow at 2pm", "Please confirm attendance", "not_spam"),
    # ... more examples
]

# Configure
dspy.settings.configure(
    lm=dspy.OpenAI(model="gpt-4o-mini"),
)

# Define task
class ClassifyEmail(dspy.Signature):
    """Classify email as spam or not spam."""
    subject = dspy.InputField()
    body = dspy.InputField()
    label = dspy.OutputField(desc="spam or not_spam")

# Build module
class EmailClassifier(dspy.Module):
    def __init__(self):
        super().__init__()
        self.classifier = dspy.ChainOfThought(ClassifyEmail)

    def forward(self, subject, body):
        return self.classifier(subject=subject, body=body)

# Metric
def classification_metric(example, pred, trace=None):
    return pred.label.lower() == example.label.lower()

# Optimize
classifier = EmailClassifier()
optimizer = dspy.BootstrapFewShot(metric=classification_metric)
classifier = optimizer.compile(
    classifier,
    trainset=train_emails[:50],  # Use 50 emails to optimize
    teacher_settings=dict(lm=dspy.OpenAI(model="gpt-4"))
)

# Use
result = classifier(
    subject="Congratulations! You've won a prize",
    body="Click here to claim your prize"
)
print(result.label)  # "spam"
```

Result: **Automatic email classifier, no manual prompts written**.

### Production Deployment with DSPy

```python
# Save optimized program
classifier.save("email_classifier.json")

# In production
classifier = EmailClassifier()
classifier.load("email_classifier.json")

# Use for inference
result = classifier(subject=user_subject, body=user_body)
```

The saved program includes:
- Optimized prompts (learned by DSPy)
- Few-shot examples (selected by DSPy)
- Metadata (model used, date optimized, etc.)

---

## Section B Summary: Why DSPy Matters

DSPy solves the **scalability problem** of manual prompt engineering:

| Stage | Manual | DSPy |
|-------|--------|------|
| **Development** | Write prompts → test → fail → rewrite | Define signature → compile |
| **Optimization** | Try different phrasings | Teleprompter auto-optimizes |
| **Testing** | Ad-hoc spot checks | Metric-based systematic testing |
| **Deployment** | Hope it still works | Versioned, reproducible code |
| **Iteration** | Add new prompts manually | Recompile with new data |

**Key insight:** Treating prompts as **code**, not as **text**, makes them scalable.

---

## Part 4 Summary

### Cost Reduction (Section A)
1. **Prompt Compression** - Reduce token count 30-50%
2. **FrugalGPT** - Select best examples, cascade models
3. **LLM Cascade** - Route to right-sized model based on confidence
4. **Caching** - Return cached results for repeated queries
5. **Fine-tuning** - Invest in smaller model, switch long-term

### Automation (Section B)
1. **DSPy** - Programmatic prompt engineering
   - Signatures: Define what you want
   - Modules: Chain operations
   - Teleprompters: Auto-optimize
   - Metric-driven: Test systematically

---

## The Decision Tree: What to Use When

```
Starting a new LLM project?
├─ One-off analysis/writing → Manual prompts
├─ Small prototype (< 100 requests/day)
│  ├─ Caching alone
│  └─ Move to FrugalGPT if cost explodes
└─ Production system (> 1,000 requests/day)
   ├─ Add Caching immediately
   ├─ Add Compression if queries are large
   ├─ Implement FrugalGPT for cost control
   ├─ Plan fine-tuning for 6-month horizon
   └─ Use DSPy for structure + automation
```

---

## What's Next? Part 5

In Part 5, we'll move beyond optimization to **advanced strategies**:
- Building reliable LLM systems (evaluation, monitoring, failure recovery)
- Real-time LLM applications (streaming, partial responses)
- Prompt chaining and orchestration at scale

See you there!

---

Have experiences with DSPy, caching strategies, or model cascading? Share in the comments!

Check the [GitHub](https://github.com/programmerraja/PromptEngineering) for complete DSPy examples and production-ready templates.
