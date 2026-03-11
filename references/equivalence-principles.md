# Equivalence Principles - In-Depth Guide

The Equivalence Principle is GenLayer's solution for achieving consensus on non-deterministic operations like LLM calls and web fetches.

## The Core Problem

Traditional blockchains require determinism - every node must compute the exact same result. But:
- LLMs produce varying outputs for the same prompt
- Web content can differ between fetches
- Timing differences cause data variations

GenLayer solves this with the **Leader/Validator Pattern**.

---

## How It Works

### Leader/Validator Pattern

1. **Leader Selection**: One validator randomly chosen as "leader"
2. **Leader Execution**: Leader runs the non-deterministic operation
3. **Result Broadcast**: Leader's result shared with validators
4. **Validation**: Validators verify/compare the result
5. **Consensus**: Majority agreement accepts the result

```
Transaction → Random Leader Selected → Leader Executes → Result to Validators
                                                              ↓
                                                    Validators Verify
                                                              ↓
                                                    Consensus Reached
```

---

## Decision Flowchart

```
Does your contract fetch web data or call LLMs?
├── Yes → gl.vm.run_nondet (leader + validator both execute independently)
│   ├── Comparing verdicts/categories? → match on verdict only, not reasoning
│   ├── Comparing numbers? → use tolerance (abs diff or %)
│   └── Comparing structured data? → extract stable fields only
└── No (pure deterministic logic only)
    ├── Result is bool/exact value? → strict_eq
    └── Need semantic comparison? → prompt_comparative
```

**Default: `gl.vm.run_nondet`** — this is the right choice for ~90% of contracts.

---

## Equivalence Principle Options

### 1. Custom Leader/Validator (`gl.vm.run_nondet`) — THE DEFAULT

**How it works:**
- You define both leader and validator functions
- Leader executes the non-deterministic operation (web fetch, LLM call)
- Validators independently execute the **same** operation and compare results
- Full control over validation logic and comparison tolerance

**This is the primary pattern used in production** (Rally, MergeProof, Molly.fun).

**When to use:**
- Any contract that fetches web data (essentially always)
- Any contract that calls LLMs
- When you need custom comparison logic (tolerance, field extraction)
- Production environments needing explicit error handling

**Basic example:**
```python
def leader():
    data = gl.nondet.web.render(url, mode="text")
    return {"price": extract_price(data), "timestamp": extract_time(data)}

def validator(leader_result):
    # Independently fetch and evaluate
    my_data = gl.nondet.web.render(url, mode="text")
    my_price = extract_price(my_data)
    
    # Allow 5% tolerance for timing differences
    price_diff = abs(leader_result["price"] - my_price) / my_price
    return price_diff < 0.05

result = gl.vm.run_nondet(leader=leader, validator=validator)
```

**LLM comparison example:**
```python
def leader():
    raw = gl.nondet.exec_prompt(classification_prompt)
    return _parse_llm_json(raw)

def validator(leader_result):
    raw = gl.nondet.exec_prompt(classification_prompt)
    my_result = _parse_llm_json(raw)
    # Compare only the verdict, not reasoning/explanation
    return leader_result.get("verdict") == my_result.get("verdict")

result = gl.vm.run_nondet(leader=leader, validator=validator)
```

**With custom eq_fn:**
```python
result = gl.vm.run_nondet(
    leader=lambda: risky_operation(),
    validator=lambda r: validate(r),
    eq_fn=lambda a, b: abs(a - b) < tolerance  # Custom comparison
)
```

**Advantages:**
- Full independent verification — most secure
- Handles all types of non-determinism
- Custom tolerance and comparison logic

**Disadvantages:**
- Slightly more verbose than shortcut options

---

### 2. Strict Equality (`strict_eq`)

**How it works:**
- All validators execute the same function
- Results must match **exactly**

**When to use:**
- Boolean operations that extract a deterministic flag
- Exact checksums or hashes
- Parsed data that must be byte-identical (e.g., a specific integer field)

**⚠️ NOT for LLM outputs** — LLMs rarely produce identical text across validators.
Use `run_nondet` with a validator that compares the relevant field instead.

**Example:**
```python
def check_website():
    content = gl.nondet.web.render("https://example.com", mode="text")
    return "maintenance" in content.lower()  # bool → deterministic

is_maintenance = gl.eq_principle.strict_eq(check_website)
```

**Advantages:**
- Simplest and most predictable
- No LLM overhead for validation

**Disadvantages:**
- Any variation causes MAJORITY_DISAGREE
- Completely unsuitable for LLM text outputs

---

### 3. Prompt Comparative (`prompt_comparative`)

**How it works:**
- Leader AND validators execute the same task
- Results compared using LLM with given criteria

**When to use:**
- Simple LLM classification shortcut (when you don't need custom validator logic)
- Semantic equivalence matters but you don't want to write a custom validator
- Note: costs more (extra LLM call for comparison)

**Parameters:**
- `func`: Function to execute
- `criteria`: String describing what must match

**Example:**
```python
def get_sentiment():
    prompt = f"""
    Classify the sentiment of this text as positive, negative, or neutral.
    Text: {user_text}
    Respond with only the classification word.
    """
    return gl.nondet.exec_prompt(prompt)

result = gl.eq_principle.prompt_comparative(
    get_sentiment,
    "The sentiment classification must be the same"
)
```

**Advantages:**
- Handles LLM output variations
- Still verifies independently

**Disadvantages:**
- Higher computational cost (all validators execute + extra LLM comparison)
- Less control than `run_nondet`

---

## Special Cases

### 4. Prompt Non-Comparative (`prompt_non_comparative`)

> ⚠️ **WARNING**: This option should ONLY be used for pure NLP tasks where there is
> **no web fetching**. Validators do NOT re-execute — they only check whether the
> leader's output looks reasonable according to criteria. If your contract calls
> `web.get` or `web.render` inside the function, validators will blindly trust
> whatever the leader fetched. This defeats the entire purpose of multi-validator
> consensus. **When in doubt, use `run_nondet`.**

**How it works:**
- Only LEADER executes the full task
- Validators check if result meets criteria (without re-executing)

**Parameters:**
- `input`: Callable returning the input data
- `task`: String describing what to do
- `criteria`: Validation rules for the output

**Legitimate use case:**
```python
# Pure NLP: summarize a string that was passed as a method argument
# No web fetch, no external data — just text processing
result = gl.eq_principle.prompt_non_comparative(
    lambda: article_text,  # Input data from method argument
    task="Summarize the main points in 3 bullet points",
    criteria="""
    - Summary must have exactly 3 bullet points
    - Each point must be factually present in the original
    - Total length under 100 words
    - No hallucinated information
    """
)
```

**Advantages:**
- Most efficient (leader-only execution)
- Flexible criteria-based validation

**Disadvantages:**
- Validators don't independently verify the underlying data
- Blind trust on any web-fetched content
- Relies entirely on criteria quality

---

### 5. Unsafe Custom Pattern (`run_nondet_unsafe`)

Same as `run_nondet` but without sandbox protection.

**When to use:**
- Performance-critical applications
- Simple validators that won't error
- When you need maximum speed

**⚠️ Warning:** Validator errors become Disagree status directly.

```python
result = gl.vm.run_nondet_unsafe(
    leader=lambda: compute_value(),
    validator=lambda v: isinstance(v, int) and 0 < v < 1000
)
```

---

## Writing Secure Validators

### ❌ Bad Example
```python
def bad_validator(leader_result):
    return True  # Always accepts - DANGEROUS!
```

This allows a malicious leader to return arbitrary data.

### ✅ Good Example - Independent Verification
```python
def good_validator(leader_result):
    # Fetch data independently
    my_data = gl.nondet.web.render(url, mode="text")
    my_result = parse_score(my_data)
    
    # Compare with tolerance
    return abs(leader_result["score"] - my_result) <= 1
```

### ✅ Good Example - LLM Validation
```python
def llm_validator(leader_result):
    prompt = f"""
    Verify this analysis is reasonable for the given data:
    Data: {original_data}
    Analysis: {leader_result}
    
    Respond only with: valid or invalid
    """
    verdict = gl.nondet.exec_prompt(prompt)
    return "valid" in verdict.lower()
```

---

## Key Principles for Custom Validators

### 1. Independent Verification
Don't blindly trust the leader. Verify independently when possible.

### 2. Tolerance for Non-Determinism
For AI/web data, allow reasonable variations:
```python
# Use similarity thresholds
def validator(leader_text):
    my_text = get_content()
    similarity = compute_similarity(leader_text, my_text)
    return similarity > 0.9

# Account for timing
def validator(leader_price):
    my_price = fetch_price()
    return abs(leader_price - my_price) / my_price < 0.02  # 2% tolerance
```

### 3. Compare Stable Fields Only
When comparing LLM outputs, extract the decision field, not the full response:
```python
def validator(leader_result):
    raw = gl.nondet.exec_prompt(prompt)
    my_result = _parse_llm_json(raw)
    # Compare verdict only — reasoning text will vary
    return leader_result.get("verdict") == my_result.get("verdict")
```

### 4. Error Handling
```python
def safe_validator(leader_result):
    if isinstance(leader_result, Exception):
        # Leader had error
        return False
    
    try:
        return verify(leader_result)
    except Exception:
        return False
```

### 5. Security First
When in doubt, reject. It's better to fail a transaction than accept bad data.

---

## Best Practices Summary

| Scenario | Recommended Approach | Notes |
|----------|---------------------|-------|
| **Web fetch + LLM** | `run_nondet` ← **DEFAULT** | ~90% of contracts |
| **Web fetch + boolean** | `run_nondet` with bool validator | Independent fetch per validator |
| **Price / numeric data** | `run_nondet` with tolerance | e.g., < 2% diff |
| **Pure boolean flag** | `strict_eq` | Only if 100% deterministic |
| **Exact data match** | `strict_eq` | Only if no LLM involved |
| **LLM classification (shortcut)** | `prompt_comparative` | Extra cost |
| **Pure NLP, no web** | `prompt_non_comparative` | No re-execution by validators |
| **Performance critical** | `run_nondet_unsafe` | No sandbox protection |

---

## Common Pitfalls

### 1. Using `strict_eq` for LLM outputs
LLMs rarely produce identical text. Always use `run_nondet` with a validator that
compares the relevant field, or use `prompt_comparative`.

### 2. Using `prompt_non_comparative` with web fetches
Validators won't re-fetch. They'll trust whatever the leader fetched. This is a
security issue. Use `run_nondet` instead.

### 3. Comparing full LLM text instead of verdict field
LLM responses vary in phrasing. Extract and compare only the decision field:
```python
# Bad
return leader_result == my_result  # Full text comparison

# Good
return leader_result.get("verdict") == my_result.get("verdict")
```

### 4. Weak validation criteria
Be specific:
```python
# Bad
criteria = "Response should be reasonable"

# Good
criteria = """
- Must be valid JSON with keys: sentiment, confidence
- Sentiment must be one of: positive, negative, neutral
- Confidence must be a number between 0 and 1
"""
```

### 5. No tolerance for timing
Web data changes. Allow reasonable differences:
```python
# Bad
return leader_price == my_price

# Good
return abs(leader_price - my_price) < 0.01 * my_price
```

### 6. Accessing storage in non-det blocks
Storage is inaccessible in non-deterministic blocks:
```python
# Bad
def process():
    return gl.nondet.exec_prompt(f"Data: {self.data}")  # ERROR!

# Good
data_copy = gl.storage.copy_to_memory(self.data)
def process():
    return gl.nondet.exec_prompt(f"Data: {data_copy}")
```
