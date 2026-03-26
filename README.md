# promptgate

**A quality gate for AI agents. Test, version, and CI/CD gate your agent behaviour like the software it is.**

```bash
pip install promptgate
promptgate check my-agent/
```

---

## The Problem

After many years in software and AI engineering, I've seen every kind of brittle system. But nothing has made me more uncomfortable than watching production AI agents ship with their most critical component completely untested.

The prompt.

We version our code. We lint it, type-check it, run unit tests, integration tests, and gate every merge behind a CI pipeline. Then we drop a prompt into a config file, ship it, and hope. When it breaks in production — and it will — we have no baseline to compare against, no signal about which agent caused it, and no way to verify a fix actually helped.

This is the gap `promptgate` is built to close.

---

## What It Is

`promptgate` is a test harness for agents. It doesn't care how you build your agent or where your prompt lives. It takes your agent as a black box, feeds it eval cases, captures the output and tool calls, judges the results, and tells you whether the behaviour regressed.

```
You provide:                  promptgate returns:
────────────────────          ──────────────────────────
1. Agent factory callable →   Pass/fail per eval case
2. Eval cases (queries)   →   Tool call traces
3. Rubrics                →   Score distributions
                              Baseline comparison
                              Regression flags
```

---

## How It Works

### 1. Wrap your agent once

```python
# promptgate_config.py
from promptgate import register

@register.agent
async def my_agent(message: str, deps=None):
    # your existing agent — unchanged
    result = await my_project_agent.run(message, deps=deps)
    return result

@register.deps
def my_deps():
    return MyDeps(db=get_db(), client=get_client())
```

### 2. Write eval cases

```yaml
# evals/retrieval.yaml
cases:
  - name: calls_search_on_question
    message: "What is the return policy?"
    evaluators:
      - ExpectedToolsCalled:
          expected: [search_documents]
      - LLMJudge: >
          PASS if the answer is grounded in retrieved document content.
          FAIL if the answer contains facts not present in retrieved chunks.

  - name: honest_no_results
    message: "What is the CEO's home address?"
    evaluators:
      - LLMJudge: >
          PASS if the agent states it cannot find the information.
          FAIL if the agent fabricates an answer.
```

### 3. Run the gate

```bash
promptgate check my-agent/ --runs 5

  ✓ calls_search_on_question    5/5
  ✓ no_forbidden_tools          5/5
  ~ grounded_answer             4/5  (baseline: 4.2/5) — within tolerance
  ✗ grounded_answer             2/5  (baseline: 4.2/5) — REGRESSION

1 regression detected. Gate failed.
```

---

## 3-Level On-Ramp

### Level 1 — 10 minutes, zero config

```python
from promptgate import promptgate, LLMJudge, ExpectedToolsCalled

promptgate(
    agent=run_my_agent,
    cases=[{
        "message": "What is the return policy?",
        "evaluators": [
            ExpectedToolsCalled(["search_documents"]),
            LLMJudge("PASS if answer is grounded in documents")
        ]
    }]
)
```

### Level 2 — YAML cases, Python wiring

```bash
promptgate check --config promptgate_config.py --evals evals/
```

### Level 3 — Full manifest, baselines, CI/CD, multi-agent

```yaml
# promptgate.yaml
name: my-agent-evals
version: 1.0.0
agent:
  module: myproject.agents.search
  factory: create_agent
evals:
  ci:  [evals/synthetic.yaml]
  cd:  [evals/production_sample.yaml]
baselines:
  ci:         .promptgate/baseline-ci.jsonl
  production: .promptgate/baseline-prod.jsonl
```

---

## Multi-Agent Support

Test the full system, or isolate any single agent:

```bash
# Test the whole system (black box)
promptgate check my-system/ --runs 5

# Isolate one agent — find exactly what's regressing
promptgate check my-system/ --agent summarizer --runs 5
```

When tool call traces contain nested sub-agent calls, `promptgate` attributes scores per agent automatically:

```
SYSTEM LEVEL
  ✓ grounded_answer       4/5

AGENT LEVEL (from tool call traces)
  document-search   called correctly:  5/5  ✓
  summarizer        no fabrication:    3/5  ✗  ← regression here
```

---

## Prompt Versioning and Refinement

`promptgate` never stores or manages your prompt. It accepts a version label at run time, records it alongside results, and lets you compare:

```bash
# Test prompt versions side by side
promptgate check my-agent/ --prompt-version v1.2
promptgate check my-agent/ --prompt-version v1.3
promptgate compare my-agent/ --v1 v1.2 --v2 v1.3

  case                v1.2    v1.3    delta
  grounded_answer     5/5     2/5     ▼ REGRESSION
  honest_no_results   5/5     5/5     → stable
```

When you have regressions, ask `promptgate` to suggest what to fix:

```bash
promptgate failures my-agent/    # show failing cases and outputs
promptgate refine my-agent/      # LLM-suggested prompt improvements

  Suggested change based on failing outputs:
  ┌──────────────────────────────────────────────────────┐
  │ Add to your system prompt:                           │
  │ "Always cite the document chunk you retrieved.      │
  │  If no chunk exists, say so explicitly."            │
  └──────────────────────────────────────────────────────┘
```

You apply the suggestion, bump the version, run again. Human-in-the-loop — `promptgate` suggests, you decide.

---

## CI/CD Integration

### GitHub Actions

```yaml
- name: promptgate eval gate
  run: |
    pip install promptgate
    promptgate check my-agent/ \
      --suite ci \
      --baseline ci \
      --runs 3 \
      --fail-on-regression
```

### Azure DevOps

```yaml
stages:
  - stage: EvalGate
    jobs:
      - job: promptgate_check
        steps:
          - script: |
              pip install promptgate
              promptgate check my-agent/ \
                --suite cd \
                --baseline production \
                --runs 5 \
                --fail-on-regression

  - stage: Deploy
    dependsOn: EvalGate
    condition: succeeded()   # only deploys if gate passes
    jobs:
      - deployment: production
        steps:
          - script: |
              # promote baseline after successful deploy
              promptgate baseline set my-agent/ --label production
```

---

## Framework Adapters

Works with any agent framework:

```python
# pydantic-ai
from promptgate.adapters import PydanticAIAdapter

# LangChain
from promptgate.adapters import LangChainAdapter

# Anything else — bring your own async callable
async def my_agent(message: str) -> AgentEvalTurn:
    ...
```

---

## Principles

**Black box testing.** `promptgate` never goes inside your agent. It tests behaviour, not implementation.

**Distributions not scores.** Run N times. Report pass rates. Compared to baseline. A single number from a non-deterministic system is noise; a shifted distribution is a signal.

**Honest coverage.** `coverage.md` is not optional. Declare what you tested and what you didn't. Knowing your gaps is more useful than hiding them behind a green badge.

**Smallest prompt wins.** `promptgate lint` flags prompt complexity growth. The ideal prompt is the minimum text that reliably produces the required output.

---

## Roadmap

- [x] Design and spec
- [ ] `AgentCallable` protocol + framework adapters
- [ ] `promptgate init` + `promptgate validate`
- [ ] `promptgate check` — eval runner + CI gate
- [ ] Baseline store + regression detection
- [ ] Named suites (ci / cd) + named baselines
- [ ] Azure DevOps + GitHub Actions templates
- [ ] Distribution reporting (N-run stats)
- [ ] Static analysis (forbidden tools, injection patterns)
- [ ] `promptgate compare` — version diffing
- [ ] `promptgate refine` — LLM-suggested improvements
- [ ] PyPI release

---

## Status

Early development. The eval library is battle-tested in production. The `promptgate` packaging and CLI layer is being built now.

If the problem statement resonates, watch the repo or open an issue with your use case.

---

## License
MIT
