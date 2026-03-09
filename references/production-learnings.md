# Production Learnings

Hard-won lessons from building and deploying GenLayer contracts to Studionet with real AI jury evaluation and cross-chain bridge integration.

## GenVM Storage Type Restrictions

**`int` is NOT a valid storage type.** The SDK raises `TypeError('use bigint or one of sized integers')`, but GenVM WASI runtime may surface this as an opaque `exit_code 1` with no error message in deployment logs.

```python
# ❌ FAILS
class Bad(gl.Contract):
    count: int       # TypeError or exit_code 1
    items: list      # AssertionError: use DynArray

# ✅ WORKS
class Good(gl.Contract):
    count: u256          # sized unsigned integer
    ratio: float         # 8-byte double — YES, float works
    when: datetime.datetime  # with timezone support
    big_num: bigint      # variable-length signed (slower)
    label: str
```

**Full valid storage types:** `str`, `bool`, `bytes`, `float`, `datetime.datetime`, `Address`, `u8`–`u256`, `i8`–`i256`, `bigint`, `TreeMap[K,V]`, `DynArray[T]`, `Array[T, Literal[N]]`, custom `@allow_storage` classes.

**Key trap:** `int` works in local direct-mode tests (pure Python) but crashes on GenVM. Always use sized types.

## Multimodal Prompting (Images)

`gl.nondet.exec_prompt()` accepts an `images` parameter for vision tasks:

```python
def nondet():
    resp = gl.nondet.web.get(f"https://ipfs.io/ipfs/{cid}")
    images = []
    if resp and resp.status == 200 and resp.body:
        images.append(resp.body)
    return gl.nondet.exec_prompt(prompt_text, images=images)
```

- `images` accepts `list[bytes | Image]` — raw image data, not URLs
- Always check `resp.status == 200` and `resp.body` before using
- Include fetch status in prompt so jury knows which evidence was available
- Use `response_format='json'` for structured output: returns `dict` instead of `str`

## Evidence Design for AI Jury Consensus

5 validators with different LLMs evaluate the same evidence. Evidence quality directly determines consensus success.

**Reliable consensus:**
- Composite images with all relevant context visible at once
- Matching identifiers across documents for cross-validation
- Clear timestamps with timezone offsets: `2026-04-07 14:30:00 -04:00`
- Explicit evidence hierarchy in guidelines (which document is authoritative)

**Breaks consensus:**
- Irrelevant document types → jury returns UNDETERMINED for wrong reasons
- Typos in evidence images → AI reads exactly what's there, no "intended" content
- Ambiguous date formats without timezone offsets

## Frozen Guidelines Pattern

Store evaluation guidelines as immutable, versioned constants:

```python
GUIDELINES = {
    "my-guideline-v1": "Evaluate the statement using only the submitted evidence..."
}

class Court(gl.Contract):
    guideline_version: str

    def __init__(self, guideline_version: str, ...):
        if guideline_version not in GUIDELINES:
            raise Exception(f"Unknown guideline '{guideline_version}'")
```

Different guideline text → different LLM output → consensus failure. Version string on-chain for audit.

## Constructor-Time Evaluation

For oracle/court contracts, run AI evaluation in the constructor:

```python
def __init__(self, evidence_cid: str, statement: str, bridge_addr: str, ...):
    stmt = statement  # copy to local for closure
    def nondet():
        resp = gl.nondet.web.get(f"https://ipfs.io/ipfs/{evidence_cid}")
        return gl.nondet.exec_prompt(f"Evaluate: {stmt}", images=[resp.body])

    result_str = gl.eq_principle.prompt_non_comparative(nondet, task="...", criteria="...")
    self.verdict = json.loads(result_str)["verdict"]

    # Bridge to EVM in same tx
    bridge = gl.get_contract_at(Address(bridge_addr))
    bridge.emit().send_message(...)
```

Deploy = evaluate = bridge in one atomic transaction.

## Cross-Chain Verdict Code Mapping

When bridging GenLayer → EVM, verdict codes must match exactly:

```python
# GenLayer
VERDICT_APPROVE = 1
VERDICT_REJECT = 2
```

```solidity
// Solidity
uint8 constant VERDICT_APPROVE = 1;
uint8 constant VERDICT_REJECT = 2;
```

Mismatch = silent wrong execution on EVM side.

## Local Tests vs Studionet — The Gap

| Behavior | Local (direct mode) | Studionet |
|----------|-------------------|-----------|
| Speed | ~0.4s per test | ~100s per write tx |
| LLMs | Mocked (identical) | 5 different real LLMs |
| `strict_eq` on LLM text | ✅ works (mocked) | ❌ MAJORITY_DISAGREE |
| Storage type `int` | ✅ works (Python) | ❌ TypeError/crash |
| `prompt_non_comparative` | ❌ needs conftest patch | ✅ works natively |

Local tests catch logic bugs. Studionet catches GenVM + consensus bugs. Always test on Studionet before declaring production-ready.

## Studionet Tips

- Fund via `sim_fundAccount` with 10M+ tokens (free)
- Wait for FINALIZED, not just ACCEPTED, before bridging verdicts
- `readContract` may fail 5-30s post-deploy — use retry with backoff
- One `gl.Contract` subclass per file — enforced at runtime
- Verify constants via `cast call` immediately after deploy

## Bridge Relay Wallet Funding

Bridge path: Base → GenLayer → zkSync → LayerZero → Base. Relay wallet needs ETH on every chain it touches. Dry wallet = silently stuck verdicts with no error. Maintain >0.01 ETH per chain.

## IPFS Evidence

- Pin before deploying contracts (validators must fetch during evaluation)
- `ipfs.io` gateway is free but unreliable; `gateway.pinata.cloud` is better for pinned content
- Court sheet images typically 80-85KB PNGs — within GenVM fetch limits

## Additional Learnings (2026-03-09)

### GenVM `gen_getContractSchemaForCode` Failures
- Returns `invalid_contract not_utf8_text` for some deployed contracts
- Contracts still function correctly — `sim_getTransactionsForAddress` confirms activity
- Use `sim_getTransactionsForAddress` to verify contract existence when schema reads fail
- Likely caused by GenVM version mismatch between deploy time and query time

### `int` Type Still Broken in Contract Storage
- GenVM does NOT support Python `int` for contract storage fields
- Use `str` or `u256` instead
- Contracts deployed with `int` fields fail at schema read time

### Forge 1.5 Dry-Run Default
- `forge create` in Forge 1.5+ defaults to dry-run and deploys nothing
- Use Node.js + viem/ethers deploy scripts, or `forge script --broadcast`

### GenLayer Studio Explorer URL Structure
- `/transactions/<hash>` works (200)
- `/contracts/<address>` returns 404 — no contract detail pages exist
- Link to deploy transaction instead of contract address
