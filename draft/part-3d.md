---
title: "Prompt Engineering From First Principles: The Mechanics They Don't Teach You - Part 3"
date: 2025-02-24T10:36:47.4747+05:30
draft: true
tags:
  - prompt-engineering
  - llm
  - ai
  - generative_ai
---

In [Part 1](./part-1.md), we learned that **LLMs are probability engines predicting the next token**. In [Part 2](./part-2.md), we discovered **how to steer those probabilities** through word choice, structure, and persona.

Now comes the crucial part: **How do you make LLMs actually reason and solve complex problems?**

The answer is: **You can't make them smarter. But you can structure your prompts so the computation they perform becomes more effective.**

This is where prompting techniques come in. Each one reshapes the problem in a different way, forcing the model to allocate its computation differently. In this post, we'll explore the most effective techniques for developers, with deep explanations of how and why they work.

---

## Understanding the Core Problem

Before diving into techniques, let's establish what we're solving:

**The Base Model Problem:**
```
Question → [Single forward pass] → Output
```

A single forward pass through the model is like one "thinking step." For complex problems, one step often isn't enough. The model needs multiple attempts, multiple paths, or a structured way to decompose the problem.

**What techniques do:** They restructure the input so the model performs multiple or structured computations instead of one.

---

## Technique 1: Chain of Thought (CoT) - Multiple Computation Passes

### The Problem

Ask most models to solve this:

> A cafeteria has 12 apples. They use 3 apples to make pies and then buy 0 more apples. How many apples are left?

Without guidance, the model might output **27** or **15** because it tries to compress the entire problem into a single probability calculation.

### Why It Works (The Mechanism)

When you ask for step-by-step thinking, you're not making the model smarter. You're **forcing it to generate intermediate tokens that become part of the context for the next step**.

**Without CoT (single pass):**
```
Input: "A cafeteria has 12 apples. They use 3 apples to make pies..."
Model calculates: P(answer | input)
Output: [potential error]
```

**With CoT (multiple passes):**
```
Step 1: Input → Model predicts reasoning tokens ("Start with 12 apples")
       These tokens are now part of the input
Step 2: Input + Step1 → Model predicts next step ("Use 3, leaves 9")
       These new tokens join the context
Step 3: Input + Step1 + Step2 → Model predicts next step ("Buy 0")
Step 4: Input + All previous → Model predicts final answer ("9")
```

Each step forces the model to recalculate attention over the entire context. Errors in reasoning tend to get caught because previous correct steps are now in the context influencing later predictions.

### How to Implement

**Basic CoT:**
```xml
<problem>
A cafeteria has 12 apples. They use 3 apples to make pies and then buy 0 more apples. How many apples are left?
</problem>

Let me work through this step by step:

Step 1: Start with the initial number of apples
Step 2: Calculate after using apples
Step 3: Calculate after buying apples
Step 4: Final answer
```

**Advanced CoT (with reflection):**
```xml
You are an AI assistant that uses a Chain of Thought approach with reflection to answer queries.

Follow these steps:

1. Think through the problem step by step within <thinking> tags.
2. Reflect on your thinking to check for errors within <reflection> tags.
3. Provide your final answer within <output> tags.

<problem>
A cafeteria has 12 apples. They use 3 apples to make pies and then buy 0 more apples. How many apples are left?
</problem>

<thinking>
Step 1: Initial state - the cafeteria starts with 12 apples
Step 2: They use 3 apples for pies - so 12 - 3 = 9
Step 3: They buy 0 more apples - so 9 + 0 = 9
<reflection>
Let me verify: Started with 12, used 3 (now 9), bought 0 (still 9). This is correct.
</reflection>
</thinking>

<output>
The cafeteria has 9 apples left.
</output>
```

### When CoT Excels

- Complex math problems
- Logic puzzles requiring multiple steps
- Code debugging (step through the execution)
- Multi-step reasoning (cause and effect)
- **When accuracy matters more than latency**

### The Cost

CoT increases token usage significantly. Each thinking step generates additional tokens that become part of the context, making the total prompt longer. For simple questions ("What is 2+2?"), this is wasteful.

---

## Technique 2: Chain of Draft (CoD) - CoT But Efficient

### The Problem with CoT

CoT is powerful but **token expensive**. If you're making thousands of API calls, the cost adds up quickly.

Research shows: **Verbose reasoning doesn't help. Concise reasoning does.**

### How CoD Works

Chain of Draft applies a key insight: **Humans don't write verbose notes when solving problems. We jot down essential information only.**

Instead of long explanations at each step, CoD asks for **5-word maximum drafts**:

```
Minimal draft per step: Keep each to 5 words
Only essential calculations/transformations
Final answer clearly marked
```

### Why This Works

By forcing minimalist expression, you:
1. **Reduce token overhead** - Each thinking step is smaller
2. **Force focus** - The model must identify what matters
3. **Maintain reasoning quality** - 5 words is enough for core logic

### Real Performance Impact

According to research:
- **CoT**: 100% accuracy (baseline), 450 tokens average
- **CoD**: 98% accuracy, 35 tokens average (7.6% of CoT tokens)

You sacrifice 2% accuracy to save 92.4% tokens. For many applications, that trade-off is excellent.

### How to Implement

**Standard CoD:**
```xml
<problem>
A cafeteria has 12 apples. They use 3 apples to make pies and then buy 0 more apples. How many apples are left?
</problem>

Think step by step, but keep each step to 5 words maximum. Return the answer after ####.

Guidelines:
- Limit each step to 5 words
- Focus on essential calculations
- Maintain logical progression
- Mark final answer with ####

Step 1: Start with 12 apples
Step 2: Use 3, leaves 9
Step 3: Buy 0, still 9
####
Answer: 9 apples
```

## Technique 3: Reflection Fine-Tuning - Self-Correction in Reasoning

### The Problem

Even with CoT, models can make mistakes and not catch them. They generate reasoning that feels confident but is actually wrong.

**Example:**
```
Q: Is the following a valid Python function?

def add(a, b):
    return a + b + c

A: Yes, this looks correct.
```

The model made an error (undefined variable `c`) but didn't catch it.

### How Reflection Works

Reflection fine-tuning teaches the model to **explicitly check its own work**.

Instead of:
```
Input → Thinking → Output
```

You structure it as:
```
Input → Thinking → Reflection (did I make mistakes?) → Corrected Output
```

The key: The **reflection tags are visible to the model**, so it can see and correct errors before the final answer.

### Why This Works Mechanically

When the model sees `<reflection>` tags in its output, it:
1. Has already committed to reasoning in `<thinking>`
2. Re-reads that reasoning (multiple passes over context)
3. Calculates probabilities for "Is this right?" tokens
4. If it detects errors, it generates correction tokens

This is another form of **multiple computation passes**, but specifically focused on error detection.

### How to Implement

**Basic Reflection:**
```xml
You are a world-class AI system capable of complex reasoning and reflection.

Reason through the query inside <thinking> tags.
Then provide your final response inside <output> tags.
If you detect mistakes in your reasoning, correct yourself inside <reflection> tags.

<question>
Is the following Python valid?

def add(a, b):
    return a + b + c
</question>

<thinking>
Looking at this function:
- It takes two parameters: a and b
- The return statement uses a + b + c
- But 'c' is not defined as a parameter or variable
- This would cause a NameError at runtime
- So this is NOT valid Python
<reflection>
Wait, I need to reconsider. The function does have an undefined variable 'c'. This would indeed cause an error. My reasoning is correct.
</reflection>
</thinking>

<output>
No, this is not valid Python. The function references an undefined variable 'c' which would raise a NameError at runtime.
</output>
```

**Advanced Reflection (with explicit error categories):**
```xml
You are solving a logic problem.

Use <thinking> to work through it.
In <reflection>, specifically check for:
1. Did I misread any facts?
2. Did I apply the right logic?
3. Is my conclusion supported by evidence?

<problem>
There are three boxes. Box A has 5 apples. Box B has 3 apples. Someone moves 2 apples from Box A to Box C. How many apples are in Box A now?
</problem>

<thinking>
Box A starts with 5 apples.
2 apples are moved from Box A.
5 - 2 = 3 apples remain in Box A.

<reflection>
Check 1: Did I misread facts?
- Box A has 5 ✓
- Move 2 from A ✓
- That's all that matters for Box A

Check 2: Did I apply right logic?
- Subtracting is correct ✓

Check 3: Is my conclusion supported?
- 5 - 2 = 3 ✓
</reflection>
</thinking>

<output>
Box A has 3 apples.
</output>
```

### When Reflection Helps

- **Complex logic** where the model might make mistakes
- **Self-verification tasks** (code review, fact-checking)
- **When errors are expensive** (financial calculations, medical info)
- **Multi-part problems** where mistakes in one part affect the conclusion

### The Downside

Reflection **doubles the computation**. The model thinks, then re-thinks. This uses more tokens and latency. Only use when accuracy is critical.

---

## Technique 4: ReAct - Reasoning + Acting

### The Problem

Pure reasoning has a limit: **The model can't interact with the external world.**

If you ask: _"What's the capital of France?"_ the model can answer from training data. But if you ask: _"What's the current stock price of Tesla?"_ it can't—it needs to search the web.

ReAct (Reasoning + Acting) solves this by making the model **explicitly call actions**.

### How ReAct Works

The model outputs:
1. **Thought** - reasoning about what to do
2. **Action** - a command (search, calculate, lookup, etc.)
3. **Observation** - the result of that action (fed back as input)
4. Repeat until final answer

### Why This Works

ReAct breaks the "single forward pass" problem by creating a **loop**:

```
Thought → Action → Observation (new context) → Thought → Action → ...
```

Each action returns information that becomes part of the input for the next thought. The model has multiple "chances" to reason with fresh data.

### How to Implement

**Basic ReAct Pattern:**
```xml
You are an AI assistant with the ability to search for information.

Format your response as:
- Thought: [what you need to do]
- Action: Search [topic]
- Observation: [result of search]
- Repeat until you reach a final answer
- Final Answer: [your conclusion]

Question: What is the elevation range of the High Plains in the United States?

Thought: I need to find information about the High Plains and their elevation.

Action: Search [High Plains elevation United States]

Observation: The High Plains are a broad expanse of grassland that extends from Canada through the western Great Plains. Elevations range from approximately 1,800 to 7,000 feet (550 to 2,130 meters) as you move westward.

Thought: I have found the information I need about elevation range. The High Plains have a clear elevation range from 1,800 to 7,000 feet.

Final Answer: The elevation range of the High Plains in the United States is 1,800 to 7,000 feet (550 to 2,130 meters).
```

**Real-World Code Example (with tool calls):**
```python
import anthropic
import json

client = anthropic.Anthropic()

def process_tool_call(tool_name, tool_input):
    """Simulate tool results"""
    if tool_name == "search_web":
        return f"Search results for '{tool_input['query']}': [Mock search results]"
    elif tool_name == "calculate":
        return str(eval(tool_input['expression']))
    return "Unknown tool"

messages = [
    {
        "role": "user",
        "content": "What's the capital of France multiplied by the population of Germany divided by 2?"
    }
]

system_prompt = """You are an AI that can use tools to solve problems.

Use these tools:
- search_web: Search for information
- calculate: Perform calculations

Format your response with clear Thought/Action/Observation cycles."""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system=system_prompt,
    messages=messages,
    tools=[
        {
            "name": "search_web",
            "description": "Search the web for information",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                }
            }
        },
        {
            "name": "calculate",
            "description": "Perform calculations",
            "input_schema": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string"}
                }
            }
        }
    ]
)

print(response.content[0].text)
```

### When ReAct Excels

- **Information retrieval** (searching databases, APIs, web)
- **Multi-step problem solving** (need to gather info → process → gather more)
- **Real-time data** (stock prices, weather, current events)
- **Tool-heavy workflows** (code execution, calculations, database queries)

### The Trade-off

ReAct requires external tools and **multiple API calls** per question (one for each action). This increases latency and cost compared to pure reasoning techniques.

---

## Technique 5: Tree of Thoughts (ToT) - Multi-Path Exploration

### The Problem

Sometimes there are **multiple valid ways to solve a problem**. Chain of Thought explores one path. What if you explored many paths simultaneously?

Example: Solving a puzzle
- Path 1: Try approach A → gets stuck
- Path 2: Try approach B → reaches answer
- Path 3: Try approach C → gets stuck

With CoT, if you pick Path 1, you fail. With ToT, you explore all paths and pick the best one.

### How ToT Works

Tree of Thoughts is like **breadth-first search through a reasoning tree**:

```
        Root (Problem)
       /    |    \
    Path1  Path2  Path3
    /|\    /|\    /|\
   ... ... ... ... ... ...

Select highest-quality paths and branches.
```

For each "node" in the tree:
1. Generate multiple possible next steps
2. Evaluate quality of each step
3. Keep the best ones
4. Expand from the best
5. Repeat until solution found

### Why This Works

By exploring multiple paths simultaneously, you:
- **Recover from dead ends** (if one approach fails, others continue)
- **Find optimal solutions** (you pick the best path, not the first path)
- **Parallelize reasoning** (model generates multiple next-steps at each level)

### How to Implement

**Basic ToT Structure:**
```xml
You are an expert problem-solving agent designed to generate and evaluate multiple solution paths.

For this problem, I want you to:

1. Generate multiple potential approaches (at least 3)
2. Evaluate each approach on a scale of 0.1 to 1.0
3. Select the best approach
4. Execute it step by step

<problem>
Design a system to detect fraudulent transactions in real-time.

Requirements:
- Must catch 95% of fraud
- Latency under 100ms
- False positive rate under 1%
</problem>

Let me generate multiple approaches:

Approach 1: Rule-based detection
- Evaluate based on: Simple to implement, interpretable, limited coverage
- Score: 0.4 (Too many manual rules, scales poorly)

Approach 2: Machine Learning model
- Evaluate based on: Better coverage, harder to tune, good speed
- Score: 0.7 (Promising but needs good data)

Approach 3: Ensemble approach (Rules + ML + Heuristics)
- Evaluate based on: Comprehensive, maintainable, meets all requirements
- Score: 0.9 (Best: combines strengths of both)

Selected Approach: Ensemble approach (score 0.9)

Implementation:
[Detailed steps for selected approach]
```

**Advanced ToT with Branching:**
```xml
Problem: Solve this logic puzzle

Given:
- There are 3 people: Alice, Bob, Carol
- One is a knight (always tells truth), one is a knave (always lies), one is a commoner (random)
- Alice says: "Bob is a knight"
- Bob says: "I am a knight"
- Carol says: "At least one of us is a knave"
- Who is who?

Generate multiple reasoning branches:

Branch A: Assume Alice is the knight
  - If Alice tells truth, Bob is a knight
  - But if Bob is a knight, he tells truth saying "I am a knight" ✓
  - Then Carol must be a commoner
  - But Carol says "At least one of us is a knave" - this is FALSE
  - If Carol is a commoner (unpredictable), this could work
  - Score: 0.6

Branch B: Assume Alice is the knave
  - If Alice lies, Bob is NOT a knight
  - Bob says "I am a knight" - if Bob is knave, this is a lie ✓
  - Then Carol must be the knight
  - Carol says "At least one of us is a knave" - this is TRUE ✓
  - All three roles assigned: Alice=knave, Bob=knave... wait, can't have two knaves
  - Score: 0.3

Branch C: Assume Alice is the commoner
  - Alice says "Bob is a knight" - Alice is unpredictable, so this could be true or false
  - If Bob is knight: "I am knight" is true ✓
  - Then Carol is knave: "At least one knave" - this is TRUE, but knaves lie...
  - Score: 0.4

Highest-scoring branch: Branch A
Final Answer: [Detailed solution from Branch A]
```

### When ToT Works Best

- **Complex problems** with multiple solution paths
- **Optimization problems** (find the best solution, not just any solution)
- **Ambiguous situations** where multiple answers could work
- **When you have enough context window** (ToT uses more tokens)

### The Cost

ToT generates multiple branches and evaluates them, using **significantly more tokens** than CoT. Only use for important, complex problems.

---

## Technique 6: Re-Reading (RE2) - Improving Input Processing

### The Problem

LLMs process the entire prompt at once, but there's a key limitation: **attention weights decay over distance**.

In a long document:
- Early tokens have moderate influence on later predictions
- Middle tokens have less influence
- Late tokens have the most influence

If critical information is early in your prompt, the model might "miss" it by the time it generates the answer.

### How Re-Reading Works

Instead of hoping the model reads everything carefully, **you explicitly ask it to re-read relevant sections**:

```
Original approach: Read document → Answer question

Re-Reading approach: Read document → Question → Re-read relevant parts → Answer question
```

The re-reading step forces the model to:
1. Generate tokens that represent reviewing the document
2. Recalculate attention over those sections
3. Let that influence the answer

### Why This Works

By asking the model to re-read, you create multiple attention passes:

**Pass 1 (initial reading):**
- Model reads: "The cafeteria had 12 apples on Monday..."
- Attention distributed across all tokens

**Pass 2 (re-reading):**
- Model specifically focuses on: "12 apples"
- Attention weights recalculated
- This information now dominates when generating the answer

It's like humans re-reading a key paragraph before answering a question.

### How to Implement

**Basic Re-Reading Pattern:**
```xml
<document>
The Global Trade Report 2024 shows that international commerce increased by 15% year-over-year. However, this growth was unevenly distributed. Developing nations saw 8% growth while developed nations saw 22% growth. The report specifically highlights that supply chain disruptions in Southeast Asia caused regional trade to decline by 3%.

Key statistics:
- Global growth: 15%
- Developed nations: 22%
- Developing nations: 8%
- SE Asia: -3%

The report concludes that recovery in manufacturing sectors is expected in Q2 2025.
</document>

Question: What was the trade growth rate in Southeast Asia?

Before answering, re-read the document and find the exact statistic:
```

**For Counting/Specific Retrieval:**
```xml
<document>
The strawberry patch has 47 strawberry plants. In early June, 12 were harvested, leaving 35. By mid-June, 8 more were picked, leaving 27. In late June, 15 were harvested, leaving 12 remaining.

The strawberry patch also has raspberry plants. There are 20 raspberry plants.
</document>

Question: How many times does the letter 'r' appear in the word "strawberry"?

Let me re-read the document and focus specifically on the word "strawberry":

The document says: "The strawberry patch has 47 strawberry plants..."

Now counting 'r's in "strawberry": s-t-r-a-w-b-e-r-r-y
Letters: 1-2-3-4-5-6-7-8-9-10
The letter 'r' appears at positions: 3, 8, 9

Answer: The letter 'r' appears 3 times in "strawberry"
```

**Code Example:**
```python
import anthropic

client = anthropic.Anthropic()

document = """
The company Q3 earnings report shows:
- Revenue: $1.2 billion (up 15% YoY)
- Operating expenses: $800 million
- Net profit: $400 million
- Cash reserves: $2.5 billion
- Employee count: 5,000 (up from 4,500)
"""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=500,
    messages=[{
        "role": "user",
        "content": f"""<document>
{document}
</document>

Question: What are the company's total cash reserves?

Before answering:
1. Re-read the document
2. Find the specific statistic for cash reserves
3. Provide the exact answer
"""
    }]
)

print(response.content[0].text)
```

### When RE2 Helps

- **Long documents** (where early information might be missed)
- **Specific data retrieval** (exact numbers, quotes)
- **Counting tasks** (counting letters, occurrences)
- **When precision matters**

### When RE2 Doesn't Help

- **Short documents** (1-2 paragraphs)
- **Tasks that don't require specific details**
- **Real-time applications** (extra re-reading adds latency)

---

## Technique 7: CO-STAR Framework - Structured Prompt Design

### The Problem

Without structure, prompts become vague. The model doesn't know:
- What context matters
- What style you want
- Who the audience is
- What format to use

Result: Inconsistent, unpredictable outputs.

### How CO-STAR Works

CO-STAR is a **structural framework** that ensures you specify all critical dimensions:

- **C**ontext: Background information
- **O**bjective: What you want the model to do
- **S**tyle: Writing style to adopt
- **T**one: Emotional attitude of the response
- **A**udience: Who the response is for
- **R**esponse: Output format required

Each component steers the model's probability distribution toward your desired output.

### Why This Works Mechanically

By specifying all six dimensions, you:
1. **Reduce ambiguity** - Model doesn't have to guess
2. **Prime the latent space** - Specific tokens become more likely
3. **Set constraints** - Model avoids undesired token sequences
4. **Align attention** - All components point toward consistent output

### How to Implement

**Basic CO-STAR:**
```xml
# CONTEXT
I'm building an educational platform for teaching machine learning to beginners. The platform needs clear, accessible content.

# OBJECTIVE
Create a blog post introduction for a topic on "How Neural Networks Learn"

# STYLE
Clear and analogical - use real-world comparisons to explain technical concepts

# TONE
Friendly and encouraging, avoid intimidating jargon

# AUDIENCE
People with no machine learning background (age 18-35)

# RESPONSE
A blog post introduction (2-3 paragraphs), in markdown format, with engaging opening sentence

---

Now write the introduction:
```

**Advanced CO-STAR (for product recommendations):**
```xml
# CONTEXT
We're a startup selling productivity software (Notion-like product).
We want to recommend features to free-tier users to encourage upgrade.
Target user: Freelancer who uses our product for client management.

# OBJECTIVE
Write a personalized email recommending three premium features based on their usage patterns.

# STYLE
Professional but personable - like advice from a knowledgeable friend

# TONE
Helpful and non-pushy. Emphasize value, not pressure to buy.

# AUDIENCE
Freelancers (graphic designers, consultants, writers)
Already using free tier, so they understand basic features
Tech-savvy but value simplicity

# RESPONSE
Email format with:
- Subject line
- 2-3 sentence personalized opener
- 3 feature recommendations (one paragraph each)
- Clear CTA (Call To Action)
- Brief closing

---

Email:
```

**CO-STAR for Code Generation:**
```xml
# CONTEXT
Building a real-time chat application in Python.
Using async/await patterns for scalability.
Need database integration for message persistence.

# OBJECTIVE
Generate Python code for a function that saves chat messages to a database asynchronously.

# STYLE
Clean, production-ready code with best practices.

# TONE
Professional and straightforward - no comments, let code speak for itself.

# AUDIENCE
Senior backend engineers who will review and deploy this code.

# RESPONSE
Python function using async/await, with:
- Type hints on all parameters and return
- Error handling
- Database connection pooling
- No comments (self-explanatory)
```

### When CO-STAR Works Best

- **Any time you want consistency** across multiple outputs
- **When delegating prompts** to team members (they follow the framework)
- **Product content** (marketing copy, user-facing text)
- **Code generation** (engineers need specific patterns)
- **Team workflows** (everyone uses same framework)

### The Cost

CO-STAR adds **overhead to the prompt**. For simple queries ("What's 2+2?"), it's unnecessary. For complex tasks, it saves iteration and refinement.

---

## Comparison Matrix: When to Use Each Technique

| Technique | Problem Type | Token Overhead | Accuracy Gain | Best For |
|-----------|-------------|----------------|---------------|----------|
| **CoT** | Multi-step reasoning | 2-3x | 20-50% | Complex logic, math |
| **CoD** | Multi-step, cost-sensitive | 1.1x | 15-30% | High-volume, cost-critical |
| **Reflection** | Error-prone reasoning | 2.5x | 30-40% | Mission-critical accuracy |
| **ReAct** | Needs external info | 3-5x | 40-70%* | Search, tools, APIs |
| **ToT** | Multiple valid paths | 5-10x | 50-80% | Complex optimization |
| **RE2** | Long document retrieval | 1.3x | 25-35% | Information extraction |
| **CO-STAR** | Output consistency | 1.2x | 60%+ | Product quality, team alignment |

*ReAct gains depend heavily on tool availability and quality

---

## Combining Techniques: Real Example

Let's build a production-grade prompt using multiple techniques:

**Task:** Generate a code review for a pull request.

**Single technique (just CoT):**
```
Review this code:
[code snippet]
```

**Production approach (CoT + Reflection + CO-STAR):**
```xml
# CONTEXT
We're reviewing a pull request for our payment processing system.
Security is critical. Performance matters but is secondary.
This code will be deployed to production.

# OBJECTIVE
Generate a thorough code review identifying security issues, performance problems, and improvement suggestions.

# STYLE
Technical and direct - highlight problems clearly.

# TONE
Professional and constructive - focus on code, not person.

# AUDIENCE
Backend engineers who wrote this code and will implement feedback.

# RESPONSE
Code review as markdown with:
- Security Issues section
- Performance Issues section
- Suggestions section
- Each issue rated as Critical/Major/Minor

---

Now review this pull request:

<code_to_review>
def process_payment(user_id, amount, card_token):
    """Process a payment transaction"""

    # Log the transaction
    logger.info(f"Processing payment: {user_id} - {amount} - {card_token}")

    # Call payment gateway
    response = requests.post(
        "https://payment-api.example.com/charge",
        json={"user": user_id, "amount": amount, "token": card_token},
        timeout=5
    )

    return response.json()
</code_to_review>

Let me think through this step by step:

<thinking>
Step 1: Security check
- Card token is being logged - PII leak! ✗
- No SSL verification - security issue ✗
- No rate limiting - abuse possible ✗
- Direct requests.post - should use client library ✗

Step 2: Performance check
- 5 second timeout might be too long
- No retries defined
- Synchronous request blocks thread

Step 3: Code quality check
- No error handling
- No input validation

<reflection>
This code has multiple critical security issues. The card token logging is a severe data leak. The SSL verification missing makes it vulnerable to MITM attacks. I should mark these as Critical.
</reflection>
</thinking>

## Code Review

### Security Issues (Critical)
1. **Card Token Logging** - CRITICAL
   - Line 7: Card token is logged in plaintext
   - This is a PII violation and security risk
   - Remove sensitive data from logs

2. **Missing SSL Verification** - CRITICAL
   - requests.post() should have verify=True (default) but should be explicit
   - Consider using our payment client library instead of direct requests

### Performance Issues (Major)
1. **Blocking Operation** - MAJOR
   - Should use async/await for payment processing
   - Current approach blocks the entire thread

2. **Timeout Configuration** - MINOR
   - 5 seconds might be reasonable, but should match SLA

### Suggestions (Minor)
- Add input validation for user_id, amount
- Implement retry logic with exponential backoff
- Add proper error handling and logging (without sensitive data)
- Use company's payment client library instead of raw requests

---
```

Notice how this combines:
- **CO-STAR**: For consistent structure
- **CoT**: For step-by-step analysis
- **Reflection**: For catching own mistakes
- **RE2**: For detailed code review

---

## The Deeper Insight

All these techniques work through the **same fundamental principle: they reshape the computation from a single pass to multiple passes or more structured passes.**

A model's intelligence emerges from how much computation it's allowed to do. A single forward pass has strict limits. Multiple structured passes can solve exponentially harder problems.

Your job as a prompt engineer:
1. **Diagnose** what the model struggles with
2. **Select** the technique that adds the right kind of structure
3. **Implement** that technique properly
4. **Measure** whether it actually improves results

---

## Summary: Your Technical Toolkit

- **CoT**: For reasoning problems where you need the model to show its work
- **CoD**: For CoT-like reasoning but on a tight token budget
- **Reflection**: When accuracy is paramount and the model might make errors
- **ReAct**: When the model needs to interact with external tools/APIs
- **ToT**: For complex optimization where multiple paths exist
- **RE2**: For improving accuracy on long documents or retrieval tasks
- **CO-STAR**: For ensuring consistency and structure in all your prompts

Master these seven techniques, and you've mastered the practical mechanics of prompt engineering.

In Part 4, we'll shift focus to **optimization techniques**: How to reduce costs, automate prompt improvement, and scale these techniques to production.

---

Feel free to share your experiences in the comments. Which technique have you found most useful?

I've set up a [GitHub](https://github.com/programmerraja/PromptEngineering) repository for this series with complete code examples and test cases for each technique. Check it out and star if helpful!



- notice period
- food/ accoumudation
- work timing and days 5?
- remote / hybrid
- leave policy
- dress code
- Hike cycle
- bounus rentions bounus..



---
title: "Prompt Engineering From First Principles: The Mechanics They Don't Teach You - Part 3"
date: 2025-02-24T10:36:47.4747+05:30
draft: true
tags:
  - prompt-engineering
  - llm
  - ai
  - generative_ai
---

In [Part 1](./part-1.md), we understood **LLMs are probability engines**. In [Part 2](./part-2.md), we learned **how to steer those probabilities** through word choice, structure, and persona.

Now comes Part 3: **Practical Prompting Techniques**.

We're going to skip the generic templates floating around the internet ("Try this magic prompt!" or "Top 10 Techniques!"). Instead, we'll explore **why certain techniques work at the mechanical level**, and when to use them.

---

## The Core Principle: Understanding Your Problem First

Before throwing a technique at your problem, ask yourself: **What's actually failing?**

Is it:
- **Reasoning?** (The model can't solve multi-step problems)
- **Precision?** (The model gives correct-ish answers but not exactly what you want)
- **Format?** (The output structure is wrong)
- **Knowledge?** (The model lacks domain context)

Each failure requires a different technique. Most people skip this diagnosis step and just try everything.

---

## Technique 1: Chain of Thought (CoT) – When Your Model Can't Reason

### The Problem

Remember the apple problem from Part 2?

> A cafeteria has 12 apples. They use 3 apples to make pies and then buy 0 more apples. How many apples are left?

If you ask most models this in one shot, they might say **27** or **15** because they're compressing multiple logical steps into a single forward pass.

### Why It Works (The Mechanism)

When you ask the model to **"Think step by step,"** you're not making it smarter. You're reshaping the computation:

**Without CoT:**
```
Question → [Single probability calculation] → Answer
```

**With CoT:**
```
Question → [Step 1 calc] → Step 1 (added to context) → [Step 2 calc] → Step 2 → ... → Answer
```

Each step becomes part of the context for the next step. The model has **multiple chances** to correct itself rather than one high-risk calculation.

### When to Use It

- Complex math problems
- Logic puzzles
- Multi-step processes (e.g., writing code, debugging)
- **When accuracy matters more than speed**

### The Prompt Pattern

```xml
<task>Solve this problem step by step.</task>

<problem>
A cafeteria has 12 apples. They use 3 apples to make pies and then buy 0 more apples. How many apples are left?
</problem>

Let's think through this step by step:

1. Initial state:
2. After using apples:
3. After buying apples:
4. Final answer:
```

### Common Mistake

Many people write: _"Let's think step by step"_ at the end of their prompt.

Instead, structure it so the model completes each step **before seeing the next question**. This forces iterative computation.

---

## Technique 2: Few-Shot Examples – Bootstrapping the Model's Understanding

### The Problem

Vague tasks fail. The model doesn't know what "good output" looks like.

**Bad:** "Summarize this text"
- Result: Too long, wrong tone, missing key points

**Good:** "Summarize this text in 1-2 sentences, focusing on actionable insights"
- Result: Better, but still inconsistent

### Why Few-Shot Works

Few-shot examples don't teach the model new information. Instead, they **constrain the probability space**.

When the model sees your examples, it:
1. Infers the format, style, and depth you want
2. Aligns its attention heads with your use case
3. Reduces ambiguity about what "good" means

### When to Use It

- When the output format is non-standard
- When you need consistency across many calls
- When the task is specific to your domain

### The Prompt Pattern

```xml
<examples>
<example>
<input>The global economy entered recession in Q3 2024</input>
<summary>Economic downturn started Q3 2024</summary>
</example>
<example>
<input>New AI regulations passed in the EU requiring transparency in model training data</input>
<summary>EU mandates AI transparency in training</summary>
</example>
</examples>

<task>Now summarize this in the same style:</task>
<input>Tesla's new manufacturing plant increased capacity by 40% while cutting costs</input>
```

### The Math Behind It

Research shows:
- **1-2 examples:** Marginal improvement
- **3-5 examples:** Significant improvement (often 20-50% accuracy boost)
- **10+ examples:** Diminishing returns (or worse, the model overfits to your examples)

**Rule of thumb:** Use 3-5 high-quality examples. More isn't always better.

---

## Technique 3: Negative Examples – Preventing Hallucination

### The Problem

Models don't know what NOT to do. They're optimized for predicting the most likely token, which sometimes includes hallucinations.

**Prompt:** "List 5 benefits of using LLMs in healthcare"
- Model might invent statistics that sound plausible but are false

### Why Negative Examples Work

Negative framing activates unwanted token sequences in the model's mind, making them **less likely**.

But here's the key: **Be explicit about what not to do.**

**Weak:** "Don't make up facts"
- The model still thinks about "made up facts"

**Strong:** "Do not cite statistics unless they come from the provided sources. If you don't know, say 'I don't know.'"
- The model learns to predict a refusal token rather than a hallucinated statistic

### When to Use It

- When hallucination is a risk
- For factual claims that require sources
- When you need safe outputs (healthcare, legal, finance)

### The Prompt Pattern

```xml
<instructions>
You will answer questions about our product.

DO:
- Reference features we explicitly provide
- Say "I don't have that information" if uncertain
- Ask clarifying questions if needed

DO NOT:
- Invent features we don't have
- Cite statistics without sources
- Assume user intent without asking
</instructions>

<context>
Our product supports: real-time collaboration, version history, and export to PDF
</context>

<question>Can your product sync with Notion?</question>
```

---

## Technique 4: Stop Sequences – Controlling Output Length

### The Problem

The model keeps generating after it's done. You ask for 5 points and get 20 rambling paragraphs.

### Why It Works

Stop sequences tell the model: _"When you see this token/phrase, stop generating."_

The model doesn't refuse; it just stops sampling at that point.

### When to Use It

- When you need exact output length
- To prevent repetitive output
- To cut off unwanted follow-up text

### The Prompt Pattern

```
Generate exactly 3 code review tips. Format as bullet points.

- Tip 1
- Tip 2
- Tip 3

[STOP]
```

And set the API parameter: `stop=["\n\n"]` or `stop=["---"]`

---

## Technique 5: Attention Visualization Through Prompting

### The Problem

You're not sure which parts of your prompt the model is "focusing on."

### The Insight

When you write a long prompt, not all tokens get equal attention. The model's attention mechanism (from Part 1) allocates more "weight" to relevant tokens.

### How to Leverage It

Structure your prompt so important information gets **syntactic emphasis**:

**Weak:**
```
Please analyze this customer feedback about our new dashboard and identify common themes.

Customer 1 said...
Customer 2 said...
```

**Strong:**
```xml
<task>Identify common themes</task>

<important>
FOCUS ON: Pain points and feature requests
IGNORE: General praise
</important>

<feedback>
Customer 1: ...
Customer 2: ...
</feedback>
```

The XML tags act like attention hooks. The model's attention mechanism "sees" `<important>` tags and weights those tokens higher, which cascades into the rest of the computation.

---

## Technique 6: Context Window Management – Single Read Problem

### The Problem (From Your Draft)

"LLM can only read one time it cannot go back and refer the text again"

This is partially true but needs clarification.

### The Reality

LLMs process the entire prompt **simultaneously**, not sequentially. But here's what's true:

- The model's **attention weights decay over distance**
- Early tokens have less influence on later predictions
- The model can't "re-read" previous context unless you explicitly make it part of the input again

### The Solution: Re-Reading Technique

Ask the model to **re-read** relevant sections before answering:

```xml
<document>
[Long document here]
</document>

<question>
How many times is "stakeholder" mentioned in the document?
</question>

Re-read the document above and count:

```

Why this works:
- You force the model to predict tokens that represent re-reading
- This causes the model to recalculate attention over the document
- It's like hitting "refresh" on which tokens matter

---

## Technique 7: Using Thinking Models for Complex Problems

### The New Frontier

Recent models like OpenAI's o1 and upcoming Claude Thinking models introduce **extended reasoning**.

Instead of one forward pass → answer, they generate:

1. **Thinking section:** The model reasons (but this is hidden)
2. **Output section:** The model provides the answer

### Why It Matters for Your Framework

This breaks the "single pass" limitation. The model **actually reasons iteratively**, not just appears to.

**Impact on prompting:**
- You can ask harder questions
- The thinking happens "for free" (you only pay for output tokens)
- But you need different prompt structures

### The New Prompt Pattern

```xml
<task>
Solve this step by step. You have space to think through this deeply.
</task>

<problem>
Can GPT-4 solve differential equations? Provide reasoning and cite evidence.
</problem>
```

The model will use its thinking tokens (invisible to you) to:
1. Search its knowledge
2. Reason about limitations
3. Generate a well-reasoned answer

---

## Putting It All Together: A Real Example

Let's apply multiple techniques to a complex prompt:

### Task: Code Review Assistant

**Bad Version:**
```
Review this code and tell me what's wrong.

[code snippet]
```

**Good Version (Multi-Technique):**
```xml
<task>Review this code for security and performance issues</task>

<context>
We're building a web API that processes user payments. Security is critical.
</context>

<do_not>
- Don't suggest minor style changes
- Don't mention things that already follow industry standards
</do_not>

<examples>
<example>
<code>
password = request.args['password']
store_in_db(password)
</code>
<review>
1. Password is in plaintext URL parameter (security issue)
2. Should hash password before storage (use bcrypt)
3. Use POST not GET for sensitive data
</review>
</example>
</examples>

<code_to_review>
[code snippet]
</code_to_review>

Think through this step by step:
1. Identify the security vulnerabilities
2. Check for performance issues
3. Rate overall quality
```

This combines:
- ✅ **Negative examples** (what to ignore)
- ✅ **Few-shot examples** (format and style)
- ✅ **Chain of thought** (step-by-step structure)
- ✅ **Context** (domain understanding)
- ✅ **XML structure** (attention allocation)

---

## Summary: The Technique Selection Matrix

| Problem | Technique | Why It Works |
|---------|-----------|------------|
| Model can't solve multi-step logic | Chain of Thought | Multiple forward passes allow correction |
| Output format is inconsistent | Few-Shot Examples | Constrains probability space with patterns |
| Model hallucinates | Negative Examples | Reduces probability of false tokens |
| Output too long/rambling | Stop Sequences | Hard cutoff on generation |
| Model misses key info | Re-Reading | Forces attention recalculation |
| Complex reasoning needed | Thinking Models | Extended computation at reasoning time |

---

## The Deeper Truth

All these techniques work for the same reason: **They reshape the probability distribution.**

Your job as a prompt engineer isn't to be creative with words. It's to:

1. **Diagnose** what the model is struggling with
2. **Understand** the mechanism (probability distribution, attention, computation)
3. **Apply** the right constraint to push probabilities in your favor

In the next part, we'll cover advanced techniques like **retrieval-augmented generation (RAG)**, **prompt optimization**, and **when to fine-tune instead of prompt engineering**.

But for now, master these techniques. They're the foundation of everything else.

---

Feel free to share your thoughts in the comments. What techniques have you found most effective?

I've also set up a [GitHub](https://github.com/programmerraja/PromptEngineering) repository for this series with examples and code. Check it out!
