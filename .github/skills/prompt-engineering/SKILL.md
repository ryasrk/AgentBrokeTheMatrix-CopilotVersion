---
name: prompt-engineering
description: Prompt design, LLM integration, RAG pipelines, embeddings, tool calling, and cost optimization for building AI-powered features with OpenAI, Anthropic, and local models.
---

# Prompt Engineering & LLM Integration

Comprehensive patterns for building reliable, cost-effective LLM-powered features.

## When to Activate

- Designing system prompts or few-shot examples
- Building RAG (Retrieval-Augmented Generation) pipelines
- Implementing tool/function calling for agents
- Integrating LLM APIs (OpenAI, Anthropic, local models)
- Optimizing token usage and API costs
- Evaluating LLM output quality

## Core Principles

### 1. Structured Over Freeform

Always request structured output for programmatic consumption.

```python
# Good: Enforce JSON output with schema
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Extract license plate info. Return JSON only."},
        {"role": "user", "content": f"Image analysis result: {ocr_text}"},
    ],
    response_format={"type": "json_object"},
    temperature=0.0,  # Deterministic for extraction
)
result = json.loads(response.choices[0].message.content)
```

### 2. Defense in Depth for Prompt Injection

Never trust user input embedded in prompts.

```python
# Good: Separate user input from instructions
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},  # Fixed instructions
    {"role": "user", "content": sanitize(user_input)},  # User content isolated
]

# Bad: String interpolation with user input
prompt = f"Analyze this: {user_input}. Follow the rules above."  # INJECTION RISK
```

### 3. Cost-Aware Model Routing

Route requests to the cheapest model that meets quality requirements.

```python
def select_model(task_complexity: str) -> str:
    """Route to appropriate model by task complexity."""
    routing = {
        "simple": "gpt-4o-mini",      # Classification, extraction
        "moderate": "gpt-4o",           # Analysis, summarization
        "complex": "claude-opus-4-6",   # Reasoning, code generation
    }
    return routing.get(task_complexity, "gpt-4o")
```

## Prompt Design Patterns

### System Prompt Template

```python
SYSTEM_PROMPT = """You are a {role} specializing in {domain}.

## Task
{task_description}

## Output Format
Return a JSON object with these fields:
- field_1: (string) Description
- field_2: (number) Description
- confidence: (float) 0.0-1.0

## Rules
1. {rule_1}
2. {rule_2}
3. If unsure, set confidence below 0.5

## Examples
Input: {example_input}
Output: {example_output}
"""
```

### Few-Shot Pattern

```python
def build_few_shot_messages(examples: list[dict], query: str) -> list[dict]:
    """Build few-shot prompt from examples."""
    messages = [{"role": "system", "content": SYSTEM_PROMPT}]
    for ex in examples:
        messages.append({"role": "user", "content": ex["input"]})
        messages.append({"role": "assistant", "content": ex["output"]})
    messages.append({"role": "user", "content": query})
    return messages
```

### Chain of Thought

```python
COT_PROMPT = """Think step by step:
1. First, identify {aspect_1}
2. Then, analyze {aspect_2}
3. Finally, determine {conclusion}

Show your reasoning, then provide the final answer in JSON format.
"""
```

## RAG Pipeline Patterns

### Chunking Strategy

```python
def chunk_text(
    text: str,
    chunk_size: int = 512,
    chunk_overlap: int = 50,
    separators: list[str] | None = None,
) -> list[str]:
    """Split text into overlapping chunks at semantic boundaries."""
    if separators is None:
        separators = ["\n\n", "\n", ". ", " "]

    chunks = []
    # Try splitting at each separator level
    for sep in separators:
        if len(text) <= chunk_size:
            chunks.append(text)
            break
        parts = text.split(sep)
        current_chunk = ""
        for part in parts:
            if len(current_chunk) + len(part) > chunk_size:
                if current_chunk:
                    chunks.append(current_chunk.strip())
                current_chunk = current_chunk[-chunk_overlap:] + part
            else:
                current_chunk += sep + part
        if current_chunk:
            chunks.append(current_chunk.strip())
        break
    return chunks
```

### Retrieval + Reranking

```python
def retrieve_context(
    query: str,
    vector_store,
    top_k: int = 20,
    rerank_top_k: int = 5,
) -> list[str]:
    """Retrieve and rerank relevant chunks."""
    # Stage 1: Broad retrieval
    candidates = vector_store.similarity_search(query, k=top_k)

    # Stage 2: Rerank for precision
    reranked = reranker.rerank(
        query=query,
        documents=[c.page_content for c in candidates],
        top_k=rerank_top_k,
    )

    return [doc.text for doc in reranked]
```

### Context Window Management

```python
def build_context(chunks: list[str], max_tokens: int = 4000) -> str:
    """Pack chunks into context within token budget."""
    context_parts = []
    token_count = 0
    for chunk in chunks:
        chunk_tokens = count_tokens(chunk)
        if token_count + chunk_tokens > max_tokens:
            break
        context_parts.append(chunk)
        token_count += chunk_tokens
    return "\n\n---\n\n".join(context_parts)
```

## Tool Calling Patterns

### Tool Definition

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "detect_license_plate",
            "description": "Detect and read license plate from image file path",
            "parameters": {
                "type": "object",
                "properties": {
                    "image_path": {
                        "type": "string",
                        "description": "Path to the image file",
                    },
                    "confidence_threshold": {
                        "type": "number",
                        "description": "Minimum confidence (0.0-1.0)",
                        "default": 0.5,
                    },
                },
                "required": ["image_path"],
            },
        },
    }
]
```

### Tool Execution Loop

```python
async def agent_loop(messages: list[dict], max_iterations: int = 10) -> str:
    """Execute tool-calling agent loop with termination."""
    for _ in range(max_iterations):
        response = await client.chat.completions.create(
            model="gpt-4o", messages=messages, tools=tools
        )
        message = response.choices[0].message

        if message.tool_calls is None:
            return message.content  # Final answer

        messages.append(message)
        for tool_call in message.tool_calls:
            result = await execute_tool(tool_call.function.name, tool_call.function.arguments)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result),
            })

    return "Max iterations reached"
```

## Evaluation Patterns

```python
def evaluate_extraction(predictions: list[dict], ground_truth: list[dict]) -> dict:
    """Evaluate LLM extraction accuracy."""
    exact_matches = 0
    partial_matches = 0
    total = len(ground_truth)

    for pred, truth in zip(predictions, ground_truth):
        if pred == truth:
            exact_matches += 1
        elif any(pred.get(k) == truth.get(k) for k in truth):
            partial_matches += 1

    return {
        "exact_match": exact_matches / total,
        "partial_match": (exact_matches + partial_matches) / total,
        "total_samples": total,
    }
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| User input in system prompt | Isolate in user message role |
| No max_tokens | Always set to prevent runaway costs |
| Parsing LLM output with regex | Use JSON mode or structured output |
| Single retry on failure | Exponential backoff with jitter |
| Same model for all tasks | Route by complexity and cost |
| No evaluation | Build golden dataset and automated evals |
| Embedding entire documents | Chunk at semantic boundaries |
| No caching | Cache identical queries and embeddings |
