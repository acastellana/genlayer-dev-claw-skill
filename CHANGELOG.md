# Changelog

All notable changes to this skill will be documented in this file.

## [1.2.0] - 2026-03-11

### Changed
- **Equivalence principles reordered** — `gl.vm.run_nondet` is now documented as the primary default (~90% of use cases), not option 4. Order is now: `run_nondet` → `strict_eq` → `prompt_comparative` → `prompt_non_comparative`.
- Quick Start "Contract with LLM" example updated to use `gl.vm.run_nondet` with leader/validator pattern
- Quick Start "Contract with Web Access" example updated to use `gl.vm.run_nondet`
- Decision flowchart in `references/equivalence-principles.md` rewritten to lead with `run_nondet`
- Best Practices Summary table updated: `run_nondet` is now the default row for web+LLM scenarios
- Error handling updated to `gl.vm.UserError` with `[EXPECTED]` / `[EXTERNAL]` / `[TRANSIENT]` / `[LLM_ERROR]` prefix convention

### Added
- `_parse_llm_json` helper pattern in SKILL.md Best Practices — handles `exec_prompt` returning `dict` or `str` (was root cause of ERC-8183 bounty failures)
- GenVM Linter section: `genvm-linter` install/usage, common catches (mandatory pre-deploy)
- Direct Mode Testing section: `genlayer-testing-suite` for fast local test runs
- `prompt_non_comparative` prominent warning: blind trust on web-fetched data if used with `web.get`/`web.render`
- `gl.vm.run_nondet` and `genvm-linter` added as trigger keywords in frontmatter
- New production learnings:
  - `exec_prompt` return type (dict vs str) — the ~7 UNDETERMINED failure bug
  - Bridge call isolation pattern for local testing
  - Consensus timing guidance (~90-180s on Studionet)
- Local vs Studionet gap table updated: `exec_prompt` return type and `gl.evm.*` availability rows

## [1.0.0] - 2026-01-30

### Added
- Initial public release
- Complete SDK API reference (`references/sdk-api.md`)
- Contract examples with annotations (`references/examples.md`):
  - Storage Contract (basic)
  - User Storage (per-address data)
  - Wizard of Coin (LLM decision making)
  - Football Prediction Market (web + LLM)
  - LLM ERC20 (token with AI validation)
  - Log Indexer (vector embeddings)
  - Intelligent Oracle (simple)
  - Production Oracle (full-featured)
- Equivalence Principles guide (`references/equivalence-principles.md`)
- Deployment guide with CLI commands (`references/deployment.md`)
- GenVM internals documentation (`references/genvm-internals.md`)
- Cross-link to companion `genlayer-claw-skill` for concepts/pitching

### Fixed
- Production Oracle example updated to use current SDK API:
  - `gl.nondet.exec_prompt()` instead of `gl.exec_prompt()`
  - `gl.nondet.web.render()` instead of `gl.get_webpage()`
  - `gl.eq_principle.prompt_comparative()` instead of `gl.eq_principle_prompt_comparative()`
  - `gl.message.sender_address` instead of `gl.message.sender_account`
  - Proper class definition with `gl.Contract` instead of `@gl.contract` decorator

### Changed
- Expanded trigger keywords for better discoverability
